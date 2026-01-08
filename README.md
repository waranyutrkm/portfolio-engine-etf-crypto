# 📊 Global Portfolio Engine (GPE) v45.1 — Strict Real Data (ETF & Crypto)

Global Portfolio Engine (GPE) v45.1 is a **fully client-side**, browser-based quantitative portfolio backtesting engine for **Global ETFs** and **Cryptocurrencies**.  
It fetches **real market data only** — **ETFs from Yahoo Finance** (via a public CORS proxy) and **crypto from Binance REST API** — then simulates portfolio behavior across multiple allocation strategies, lookback/rebalance grids, and fee assumptions. Results are evaluated using risk-adjusted metrics and interactive visualizations (Chart.js + Plotly).

🌐 Live Application (GitHub Pages)  
https://waranyutrkm.github.io/portfolio-engine-etf-crypto/

---

## 1) Project Purpose

This project answers one core research question:

**How does a portfolio allocation strategy behave over time when applied to ETFs and cryptocurrencies under realistic market conditions, transaction costs, management fees, and rebalancing schedules?**

The engine focuses on **portfolio-level allocation and NAV evolution**, not exchange-grade execution microstructure. It is built for **education, quantitative research, and strategy exploration** — **not** for live trading and **not** investment advice.

**Key design goals**
- Emphasize **risk-adjusted performance** (Sharpe/Sortino/Calmar) over raw return
- Compare strategies under a **consistent, repeatable framework**
- Explore **parameter sensitivity** (Lookback × Rebalance) via full grid search
- Run fully in-browser with **no backend**, no database, no server

---

## 2) High-Level System Flow (Matches v45.1)

**Market Data Sources (Yahoo / Binance)**  
→ **Strict Fetch & Validation** (drop failed assets; no mock)  
→ **Master Timeline Union** + Forward-Fill (align series)  
→ **Daily Returns** + Rolling **Volatility**  
→ **Momentum Signals** (Lookback return)  
→ **Strategy-Based Target Weights**  
→ **Execution Engine** (Lump Sum Rebalance OR Smart DCA No-Sell) + Fees  
→ **Daily NAV Curve**  
→ **Metrics + Visualizations** (Equity, Drawdown, Heatmaps, Surface, Monte Carlo)

---

## 3) Data Ingestion (Strict Real Data Only)

### 3.1 Crypto Assets (CRYPTO Mode)
- Source: **Binance REST API** (public endpoints)
- Market: Spot **USDT pairs** (e.g., `BTCUSDT`)
- Timeframe: **1d**
- Price used: **Close** (`kline[4]`)
- Limit: **1000 candles** (`limit=1000`) → ~1000 daily observations

### 3.2 ETF / Stock Assets (ETF Mode)
- Source: **Yahoo Finance Chart API** (`query1.finance.yahoo.com/v8/finance/chart/{symbol}`)
- Access method: fetched via **public proxy** to bypass browser CORS (AllOrigins)
- Range: `5y`, interval `1d`
- Price used: **Close** (Yahoo `quote[0].close`)  
  *(Note: this implementation uses `close`, not `adjusted close`.)*

### 3.3 Strict Fetch Policy (No Mock, No Fallback)
**v45.1 does not generate mock data.**  
If a request fails or returns insufficient history, that asset is **dropped**.

Validation rule:
- An asset is considered usable only if:
  - fetch succeeded **AND**
  - `length > 50` daily points (minimum threshold)

If all assets fail:
- the engine stops with a critical error message
- **no analysis is produced**

---

## 4) Master Timeline Alignment (Union + Forward-Fill)

Different assets have different listings, missing days, or partial histories.  
GPE aligns them using a master timeline:

1) Collect **all unique dates** across the successfully loaded assets  
2) Sort dates to create a **master timeline**
3) For each asset, build an aligned price array and **forward-fill** missing observations using the last available close:

- If a date has no quote, the engine uses the previous known close  
- This avoids discontinuities and enables consistent portfolio valuation

> This forward-fill is a standard portfolio backtesting technique for handling missing observations in daily close data.

---

## 5) Backtest Window Selection

The user chooses **Start** and **End** dates.

The engine:
- finds `startIdx` = first master date ≥ Start
- finds `endIdx` = first master date > End
- slices the aligned series accordingly

Guardrail:
- requires at least ~30 days of data inside the chosen range  
- otherwise stops with “Insufficient data in selected range.”

---

## 6) Daily Return Calculation

For each asset \(k\) on day \(t\):

\[
r_{k,t} = \frac{P_{k,t}}{P_{k,t-1}} - 1
\]

Daily returns are used for:
- rolling volatility estimation
- correlation heatmap (recent window)
- Sharpe / Sortino calculations
- Monte Carlo drift/vol extraction

---

## 7) Momentum Signal Generation (Lookback Return)

Momentum signal (relative strength proxy) is computed as:

\[
signal_k(t) = \frac{P_k(t)}{P_k(t-LB)} - 1
\]

Where:
- \(LB\) = lookback period (days) from user input (supports comma-separated grid)

This signal is used to:
- rank assets (Rank / Top strategies)
- qualify assets (Absolute Momentum / Dual Momentum / AAA filters)

---

## 8) Volatility Estimation (Rolling)

Volatility is computed from daily returns:

- take the last ~30 days of returns ending at \(t\)
- compute standard deviation \( \sigma_{daily} \)
- annualize using \( \sqrt{365} \)

\[
\sigma_{annual} = \sigma_{daily} \times \sqrt{365}
\]

Safety fallback:
- if the computed volatility is zero/undefined, the engine uses `0.01` to avoid division-by-zero in inverse-vol logic.

> Note: annualization uses **365** (crypto-style calendar), even in ETF mode.

---

## 9) Portfolio Allocation Strategies (Implemented in v45.1)

Each strategy produces target weights \( w_k \) over the **active universe** (only assets that successfully loaded).

### 9.1 Equal Weight (1/N)
\[
w_k = \frac{1}{N}
\]

### 9.2 Rank-Based (Linear Score by Momentum Rank)
Sort by momentum descending, rank 1 is best.

\[
Score_i = N - rank_i + 1,\quad
SumScore = \frac{N(N+1)}{2},\quad
w_i = \frac{Score_i}{SumScore}
\]

### 9.3 Top 3 Leader (Equal)
Pick top 3 by momentum:

\[
w_{top} = \frac{1}{K},\quad K=\min(3,N)
\]

### 9.4 Top 3 Leader (Ranked)
For top \(K\le 3\):

\[
w_1=\frac{K}{S},\ w_2=\frac{K-1}{S},\ w_3=\frac{K-2}{S},\quad S=\frac{K(K+1)}{2}
\]

### 9.5 Top 50% Selection
\[
K = \lceil N/2 \rceil,\quad w = 1/K\ \text{for top K}
\]

### 9.6 Absolute Momentum (Trend Filter)
Eligible if \( signal_k(t) > 0 \).  
Eligible assets are equally weighted; others get 0.

### 9.7 Dual Momentum
Filter by \( signal_k(t) > 0 \), then take **top 3** among eligible and equal weight.

### 9.8 Inverse Volatility (Risk Parity Style)
\[
raw_k = \frac{1}{\sigma_k},\quad
w_k = \frac{raw_k}{\sum raw}
\]

### 9.9 Adaptive Aggressive Allocation (AAA)
- If there are positive-momentum assets → use them  
- Else → fall back to top 3 momentum assets  
- Allocate using inverse volatility on the chosen subset

---

## 10) Execution Engine (Two Modes in v45.1)

The engine simulates portfolio NAV day-by-day and applies:
- management fee daily
- rebalance logic on schedule
- transaction fee on trades (or on new buys in DCA mode)

### 10.1 Initial Buy (Day 0)
The engine invests the (fee-adjusted) initial capital into target weights.

- Initial capital after front-end fee:
\[
Cash_0 = Initial \times (1 - FrontFee\%)
\]

- Trading fee on buys:
\[
NetBuy = Amount \times (1 - TradeFee\%)
\]

Then shares are set:
\[
Shares_k = \frac{NetBuy_k}{Price_k}
\]

### 10.2 Lump Sum Rebalance Mode (Sell/Buy)
Triggered every `RB` days.

- Compute current holdings value:
\[
Value_k = Shares_k \times Price_k
\]
\[
NAV = Cash + \sum Value_k
\]

- Compute target values:
\[
Target_k = NAV \times w_k
\]
- Turnover (absolute traded value estimate):
\[
Turnover = \sum |Target_k - Value_k|
\]
- Estimated trading cost:
\[
Cost = Turnover \times TradeFee\%
\]
- Net NAV after cost:
\[
NetNAV = NAV - Cost
\]
- New shares:
\[
Shares_k = \frac{NetNAV \times w_k}{Price_k}
\]

Logs:
- record **% of total portfolio** bought/sold per asset on rebalance

### 10.3 Smart DCA Mode (No-Sell)
Triggered every `RB` days **only if DCA > 0**.

- Inflow:
\[
Inflow = DCA
\]
- Current asset values:
\[
CurrentValue_k = Shares_k \times Price_k
\]
\[
CurrentTotal = \sum CurrentValue_k
\]
- Ideal post-injection allocation:
\[
Ideal_k = (CurrentTotal + Inflow) \times w_k
\]
- Deficit:
\[
Deficit_k = \max(0, Ideal_k - CurrentValue_k)
\]
- Allocate new capital:
  - If total deficits > 0, distribute inflow proportional to deficits  
  - Else fallback to target weights

- Apply trading fees **only on the new buys**:
\[
NetBuy_k = Buy_k \times (1 - TradeFee\%)
\]
\[
Shares_k += \frac{NetBuy_k}{Price_k}
\]

Logs:
- record allocations as **% of injected capital (100% total)**

> Smart DCA does not sell existing holdings; it only uses new money to move closer to target weights.

---

## 11) Fee Model (Exactly as Implemented)

### 11.1 Front-End Fee (One-time)
\[
InitialNet = Initial \times (1 - FrontFee\%)
\]

### 11.2 Trading Fee (Per execution)
- Lump Sum rebalance: applied to turnover estimate  
- Smart DCA: applied only to new buys

### 11.3 Management Fee (Annual, Deducted Daily)
Daily management fee factor:
\[
DailyMgmtFee = \frac{MgmtFee\%}{100 \times 365}
\]

Instead of tracking cash fee explicitly, v45.1 applies the fee by scaling down holdings:

- NAV reduced by fee
- shares scaled by:
\[
Shares_k \leftarrow Shares_k \times (1 - DailyMgmtFee)
\]

This ensures the NAV curve reflects ongoing management fee drag.

---

## 12) Performance Metrics (As Used in v45.1)

Let \(NAV_t\) be the simulated portfolio series.

### 12.1 CAGR (%)
\[
TotalReturn = \frac{NAV_{end}}{NAV_{start}}
\]
\[
Years = \max(\frac{days}{365}, 0.1)
\]
\[
CAGR = (TotalReturn^{1/Years} - 1)\times 100
\]

### 12.2 Annualized Volatility
\[
\sigma_{annual} = std(daily\ returns) \times \sqrt{365}
\]

### 12.3 Sharpe Ratio
Risk-free rate input is annual percent.

\[
rf_{daily}=\frac{rf\%}{100\times 365}
\]
\[
Sharpe = \frac{mean(r_t - rf_{daily}) \times 365}{\sigma_{annual}}
\]

### 12.4 Sortino Ratio
Downside deviation uses **excess returns** (after rf_daily):

\[
DownsideDev_{daily} = \sqrt{\frac{\sum (excess_t^2 \cdot \mathbb{1}_{excess_t<0})}{N}}
\]
\[
DownsideDev_{annual} = DownsideDev_{daily}\times \sqrt{365}
\]
\[
Sortino = \frac{mean(excess)\times 365}{DownsideDev_{annual}}
\]

### 12.5 Max Drawdown (%)
\[
Peak_t = \max(NAV_0,\dots,NAV_t)
\]
\[
DD_t = \frac{NAV_t - Peak_t}{Peak_t}
\]
\[
MDD = \min(DD_t)\times 100
\]

### 12.6 Calmar Ratio
In v45.1, MDD is stored as negative percent; Calmar uses absolute drawdown:

\[
Calmar = \frac{CAGR}{|MDD|}
\]

### 12.7 Win Rate (%)
\[
WinRate = \frac{\#\{r_t>0\}}{N}\times 100
\]

---

## 13) Grid Search (Tournament Mode)

The engine evaluates:
- Strategy (either one or **ALL**)
- All Lookback values in the comma list
- All Rebalance values in the comma list

Each combination is simulated independently and stored as a row in the Strategy Matrix.  
Default ranking is by **Sharpe Ratio (descending)** with sort/filter/pagination.

---

## 14) Visualization Modules (v45.1 UI)

- **Equity Curve** (Chart.js)  
  - compare selected strategy vs benchmark
  - linear/log toggle
- **Drawdown Curve** (Chart.js)
- **Monthly Return Heatmap** (compounded monthly return)
- **Monthly Drawdown Heatmap** (worst drawdown in the month)
- **Correlation Heatmap** (Plotly heatmap; recent window)
- **3D Hyper-Parameter Surface** (Plotly surface)  
  - Lookback × Rebalance × Metric  
  - optional Mean ± SD planes
  - time-warp slider/animation that recomputes metrics on partial horizon
- **Monte Carlo Forecast** (Plotly paths; GBM-style)  
  - drift/vol estimated from realized daily returns of the selected strategy  
  - used for scenario exploration only

---

## 15) Strict Real Data Constraints (Important)

- **No mock data.** If fetch fails, the asset is dropped.
- ETF data depends on a **public proxy**; outages/rate limits can reduce available symbols.
- Uses **daily close** prices only (no intraday).
- Annualization uses **365** for volatility and Sharpe/Sortino scaling.
- Universe is **user-specified/current tickers** (survivorship bias can exist if you only test today’s survivors).

---

## 16) Best Practices for Research Use

- Don’t optimize solely for Sharpe — always inspect **MDD** and **Calmar**
- Prefer parameter regions with smooth surfaces, not single-point peaks
- Compare against an appropriate benchmark (ETF mode: SPY; crypto mode: BTC by default)
- Use heatmaps to understand regime sensitivity and year-by-year behavior
- Treat Monte Carlo as **scenario exploration**, not prediction

---

## 17) Disclaimer

This project is for **educational and research purposes only**.  
It does **not** constitute financial advice.  
ETF and cryptocurrency markets involve significant risk.  
Use at your own risk.
