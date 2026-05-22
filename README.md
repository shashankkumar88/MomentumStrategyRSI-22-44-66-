# Alok Jain RSI (22, 44, 66) Momentum Engine with SQLite Caching

A production-grade algorithmic trading script built in Python utilizing the **Angel One SmartAPI**. The strategy is based on Alok Jain's (Weekend Investing) momentum model, which calculates an equidistant triple-horizon Relative Strength Index (RSI) across the **CNX 200 (Nifty 200)** universe, ranks them, and isolates the top 20 highest-momentum assets.

## 🚀 Core Features
*   **Triple-Horizon Momentum Math:** Computes RSI on 22, 44, and 66-day boundaries and averages them to score overall trend strength.
*   **Bulletproof CNX 200 Scrip Mapping:** Includes a resilient static component filter combined with Angel One’s open scrip master download to avoid brittle web-scraping bugs.
*   **Persistent SQLite Caching:** Implements an internal data layer (`momentum_cache.db`) that caches downloaded daily candles. Consecutive intraday script runs execute in milliseconds instead of rebuilding connections.
*   **Smart Rate-Limiting:** Incorporates strict 400ms thread throttling to gracefully navigate Angel One's server query limits (Max 3 calls/sec).

---

## 🛠️ Step-by-Step Angel One SmartAPI Configuration

To run this algorithm live, you must register a free developer account with Angel One to acquire API endpoints and app credentials.

### Step 1: Create a SmartAPI Developer Account
1. Go to the [Angel One SmartAPI Portal](https://angelbroking.com).
2. Click **Sign Up** and complete your profile registration using your Angel One client ID details.
3. Log in to your newly activated developer dashboard.

### Step 2: Create a Trading App Link
1. In your SmartAPI Dashboard, click on **Create An App**.
2. Fill out the application details:
   *   **App Name:** `RSI Momentum Engine` (or any name you prefer)
   *   **Redirect URL:** `http://localhost:8080` (or any dummy URL; this script runs via backend scripts and doesn't require an active landing redirect page).
3. Click **Save/Create**.
4. Your dashboard will now display two crucial values:
   *   **API Key (Client ID)**
   *   **App Secret**

### Step 3: Enable TOTP (Time-Based One-Time Password)
Angel One requires multi-factor authentication (MFA) via a TOTP seed token for programmatic automated logins.
1. Log into your standard Angel One mobile trading app or web platform.
2. Navigate to **Profile/Account Settings** -> **Security** -> **Enable External TOTP** (or Manage TOTP).
3. The platform will display a **QR Code** alongside a string of alphanumeric characters known as the **TOTP Secret/Seed Key**.
4. **Copy this Secret Key!** You will need this key to programmatically generate your live login OTP tokens inside Python. *Do not share this key.*

---

## 💻 Local Project Setup

### 1. Prerequisite Packages
Ensure you have Python 3.8+ installed. Clone this repository and install the technical analysis and broker connection modules:

```bash
pip install smartapi-python pandas ta requests
```

### 2. Configure Credentials
Open the execution script file and locate lines 13–16. Replace the placeholder strings with your actual Angel One credentials:

```python
# =====================================================================
# 1. INITIALIZATION & CACHE DATABASE SETUP
# =====================================================================
API_KEY = "YOUR_SMARTAPI_API_KEY"           # Obtained from Developer Dashboard
CLIENT_CODE = "YOUR_ANGEL_CLIENT_ID"       # Your retail trading login ID
PASSWORD = "YOUR_RETAIL_LOGIN_PASSWORD"     # Your retail account login password
TOTP_TOKEN = "YOUR_TOTP_ALPHANUMERIC_SEED"  # The secret seed string saved in Step 3
```

### 3. Execution Run
Execute the main engine file from your terminal terminal:

```bash
python momentum_engine.py
```

---

## 📊 Strategy Mechanics & Architecture

1. **Watchlist Compiling:** The script fetches the live 30MB token tracking sheet from Angel One's open servers, cross-references it against our curated list of 200 CNX liquid symbols, and assigns the required `token` IDs.
2. **Local DB Verification:** Before calling the remote servers, the script verifies if data for the current timestamp exists in `momentum_cache.db`. If found (Cache Hit), it processes calculations offline instantly.
3. **Indicator Evaluation:** 
   $$\text{Momentum Score} = \frac{\text{RSI}(22) + \text{RSI}(44) + \text{RSI}(66)}{3}$$
4. **Output Profile:** Outputs a clean, sorted Pandas DataFrame printing the top 20 stocks boasting the strongest technical momentum profiles ready for portfolio allocation.

---

## ⚠️ Disclaimer
*This repository is for educational and technical demonstration purposes only. Algorithmic trading involves substantial financial risk. Past performance does not guarantee future market returns. Always perform intensive dry-run paper trading operations before exposing live capital to execution layers.*
