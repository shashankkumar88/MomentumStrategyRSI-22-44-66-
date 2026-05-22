import time
import sqlite3
import requests
import pandas as pd
from datetime import datetime
from ta.momentum import RSIIndicator
from SmartApi import SmartConnect

# =====================================================================
# 1. INITIALIZATION & CACHE DATABASE SETUP
# =====================================================================
# Configure your personal Angel One developer credentials here
API_KEY = "your_api_key"
CLIENT_CODE = "your_client_code"
PASSWORD = "your_password"
TOTP_TOKEN = "your_totp_token" 

DB_FILE = "momentum_cache.db"

def init_cache_db():
    """Initializes the local SQLite database schema for persistent candle storage."""
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS daily_candles (
            symbol TEXT,
            date TEXT,
            open REAL,
            high REAL,
            low REAL,
            close REAL,
            volume INTEGER,
            PRIMARY KEY (symbol, date)
        )
    """)
    conn.commit()
    conn.close()

# Start database framework and establish Angel One session tracking
init_cache_db()
smart_conn = SmartConnect(api_key=API_KEY)
session_data = smart_conn.generateSession(CLIENT_CODE, PASSWORD, TOTP_TOKEN)

# =====================================================================
# 2. COMPLETE NIFTY 200 COMPONENT REGISTRY MATRIX
# =====================================================================
# Hardcoded symbol baseline to avoid fragile third-party web scrapers
NIFTY200_RAW_SET = {
    "ABB", "ACC", "ADANIENSOL", "ADANIENT", "ADANIGREEN", "ADANIPORTS", "ADANIPOWER", "ATGL", "AWL",
    "ABCAPITAL", "ABFRL", "ALKEM", "AMBUJACEM", "APOLLOHOSP", "APOLLOTYRE", "ASHOKLEY", "ASIANPAINT", "ASTRAL", "AUROPHARMA",
    "AUBLANK", "DMART", "AXISBANK", "BAJAJ-AUTO", "BAJFINANCE", "BAJAJFINSV", "BAJAJHLDNG", "BALKRISIND", "BANDHANBNK",
    "BANKBARODA", "BANKINDIA", "BATAINDIA", "BEL", "BERGEPAINT", "BHARATFORG", "BHEL", "BPCL", "BHARTIARTL", "BIOCON",
    "BOSCHLTD", "BRITANNIA", "BSOFT", "CGPOWER", "CANBK", "CHAMBLFERT", "CHOLAFIN", "CIPLA", "COALINDIA", "COFORGE",
    "COLPAL", "CONCOR", "COROMANDEL", "CROMPTON", "CUMMINSIND", "DLF", "DABUR", "DALBHARAT", "DEEPAKNITR", "DELHIVERY",
    "DIVISLAB", "DIXON", "LALPATHLAB", "DRREDDY", "EICHERMOT", "ESCORTS", "EXIDEIND", "FEDERALBNK", "FACT", "FORTIS",
    "GAIL", "GLAND", "GLENMARK", "GODREJCP", "GODREJPROP", "GRANULES", "GRASIM", "GUJGASLTD", "GSPL", "HCLTECH",
    "HDFCBANK", "HDFCLIFE", "HDFCAMC", "HFCL", "HAVELLS", "HEROMOTOCO", "HINDALCO", "HINDPETRO", "HINDUNILVR", "HINDZINC",
    "HUDCO", "ICICIBANK", "ICICIGI", "ICICIPRULI", "IEX", "IOC", "IRCTC", "IRFC", "IREDA", "IGL",
    "INDUSTOWER", "INDUSINDBK", "NAUKRI", "INFY", "INDIGO", "IPCALAB", "ITC", "JINDALSTEL", "JSL", "JSWENERGY",
    "JSWSTEEL", "JIOFIN", "JUBLFOOD", "KPITTECH", "KALYANKJIL", "KOTAKBANK", "LTTS", "LTIM", "LT", "LICI",
    "LUPIN", "MRF", "M&MFIN", "M&M", "MANAPPURAM", "MARICO", "MARUTI", "MFSL", "MAXHEALTH", "MAZDOCK",
    "METROPOLIS", "MPHASIS", "MCX", "MUTHOOTFIN", "NHPC", "NMDC", "NTPC", "NATIONALUM", "NAVINFLUOR", "NESTLEIND",
    "OBEROIRLTY", "ONGC", "OIL", "PAYTM", "OFSS", "POLICYBZR", "PIIND", "PFC", "PVRINOX",
    "PAGEIND", "PERSISTENT", "PETRONET", "PIDILITIND", "PEL", "POLYCAB", "POONAWALLA", "POWERGRID", "PRESTIGE",
    "RELIANCE", "RBLBANK", "RECLTD", "RHIM", "RVNL", "SBICARD", "SBILIFE", "SBIN", "SAIL",
    "SRF", "SANOFIA", "SHREECEM", "SHRIRAMFIN", "SIEMENS", "SJVN", "SONACOMS", "STARHEALTH", "SUNPHARMA", "SUNTV",
    "SUPREMEIND", "SUZLON", "SYNGENE", "TVSMOTOR", "TATACOMM", "TATACONSUM", "TATAELXSI", "TATAMOTORS", "TATAPOWER", "TATASTEEL",
    "TATAINVEST", "TATATECH", "TCS", "TECHM", "NIACL", "RAMCOCEM", "TITAN", "TORNTPOWER", "TRENT", "TRIDENT",
    "TIINDIA", "UPL", "ULTRACEMCO", "UNIONBANK", "UBL", "VGUARD", "VBL", "VEDL", "IDEA",
    "VOLTAS", "WHIRLPOOL", "WIPRO", "YESBANK", "ZOMATO", "ZYDUSLIFE"
}

def generate_clean_watchlist():
    """Maps Nifty 200 ticker elements directly against Angel One token pairs."""
    master_url = "https://angelbroking.com"
    try:
        scrip_data = requests.get(master_url, timeout=15).json()
        raw_df = pd.DataFrame(scrip_data)
        nse_df = raw_df[(raw_df['exch_seg'] == 'NSE') & (raw_df['symbol'].str.endswith('-EQ'))].copy()
        nse_df['clean_base'] = nse_df['symbol'].apply(lambda x: str(x).replace('-EQ', '').strip())
        filtered_df = nse_df[nse_df['clean_base'].isin(NIFTY200_RAW_SET)]
        deduped_df = filtered_df.drop_duplicates(subset=['token'])
        return [{"symbol": row['symbol'], "token": str(row['token'])} for _, row in deduped_df.iterrows()]
    except Exception as e:
        print(f"Watchlist dynamic generation sequence failed: {str(e)}")
        return []

WATCHLIST = generate_clean_watchlist()

# =====================================================================
# 3. SQLite DATA ACCESS & API FALLBACK LAYER
# =====================================================================
def get_data_with_cache(stock):
    """Pulls local DB rows if available for today. Otherwise, downloads from Angel API."""
    today_str = datetime.now().strftime("%Y-%m-%d")
    conn = sqlite3.connect(DB_FILE)
    
    # Verify if records exist locally matching today's system stamp
    check_query = "SELECT COUNT(*) FROM daily_candles WHERE symbol = ? AND date LIKE ?"
    cursor = conn.cursor()
    cursor.execute(check_query, (stock['symbol'], f"{today_str}%"))
    has_today_data = cursor.fetchone()[0] > 0
    
    if has_today_data:
        # Cache Hit: Extract directly via local storage file
        df = pd.read_sql_query("SELECT * FROM daily_candles WHERE symbol = ? ORDER BY date ASC", conn, params=(stock['symbol'],))
        conn.close()
        return df, True
        
    # Cache Miss: Fetch historical window from broker API node
    conn.close()
    start_date = (pd.Timestamp.now() - pd.Timedelta(days=140)).strftime("%Y-%m-%d %H:%M")
    end_date = pd.Timestamp.now().strftime("%Y-%m-%d %H:%M")
    
    query_params = {
        "exchange": "NSE",
        "symboltoken": stock['token'],
        "interval": "ONE_DAY",
        "fromdate": start_date,
        "todate": end_date
    }
    
    try:
        response = smart_conn.getCandleData(query_params)
        if response and response.get('status') and response.get('data'):
            df = pd.DataFrame(response['data'], columns=['date', 'open', 'high', 'low', 'close', 'volume'])
            df['symbol'] = stock['symbol']
            
            # Pipe fresh market data vectors into local SQLite engine
            conn = sqlite3.connect(DB_FILE)
            df.to_sql('daily_candles', conn, if_exists='append', index=False, method='multi')
            
            # Run cleanup to prune duplicate rows across overlapping calendar boundaries
            cursor = conn.cursor()
            cursor.execute("""
                DELETE FROM daily_candles 
                WHERE rowid NOT IN (SELECT MIN(rowid) FROM daily_candles GROUP BY symbol, date)
            """)
            conn.commit()
            conn.close()
            
            df['close'] = df['close'].astype(float)
            return df.sort_values(by="date").reset_index(drop=True), False
    except Exception as e:
        print(f"Error gathering live candles for {stock['symbol']}: {str(e)}")
        
    return None, False

# =====================================================================
# 4. TRIPLE-HORI_ZON RSI STRATEGY SCANNERS
# =====================================================================
def run_momentum_screening_cycle():
    if not WATCHLIST:
        print("Scrip universe empty. Aborting run.")
        return pd.DataFrame()
        
    momentum_records = []
    print(f"Scanning {len(WATCHLIST)} assets via Cached Momentum Framework...")
    
    for i, stock in enumerate(WATCHLIST):
        df, is_cached = get_data_with_cache(stock)
        
        # Guard verification ensuring 66 lookback criteria can compute
        if df is not None and len(df) >= 68:
            r22 = RSIIndicator(close=df['close'], window=22).rsi().iloc[-1]
            r44 = RSIIndicator(close=df['close'], window=44).rsi().iloc[-1]
            r66 = RSIIndicator(close=df['close'], window=66).rsi().iloc[-1]
            
            # Alok Jain's structural momentum formula matrix
            momentum_score = (r22 + r44 + r66) / 3.0
            
            momentum_records.append({
                "Trading_Symbol": stock['symbol'],
                "Momentum_Score": round(momentum_score, 2),
                "RSI_22": round(r22, 1),
                "RSI_44": round(r44, 1),
                "RSI_66": round(r66, 1)
            })
            
        # Enforce 400ms sleep buffer ONLY during active API connection drops
        if not is_cached:
            time.sleep(0.4)

    # Sort array to deliver rank profiles
    master_rankings = pd.DataFrame(momentum_records)
    if not master_rankings.empty:
        master_rankings = master_rankings.sort_values(by="Momentum_Score", ascending=False).reset_index(drop=True)
        print("\n" + "="*15 + " TOP 20 MOMENTUM PICKS (CACHED) " + "="*15)
        print(master_rankings.head(20))
    return master_rankings

# =====================================================================
# 5. PIPELINE EXECUTION ENTRY POINT
# =====================================================================
ranked_universe = run_momentum_screening_cycle()
