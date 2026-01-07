# 📊 Portfolio Engine — ETF & Crypto Edition

Portfolio Engine — ETF & Crypto Edition is a fully client-side, web-based quantitative portfolio backtesting engine designed to analyze and compare portfolio allocation strategies across Global ETFs and Cryptocurrencies. The engine fetches real market data from Yahoo Finance (via public proxy) for ETFs and from the Binance API for crypto assets, simulates portfolio behavior under different strategies, rebalancing rules, and fee assumptions, and evaluates performance using risk-adjusted metrics and interactive visualizations.

🌐 Live Application (GitHub Pages)  
https://waranyutrkm.github.io/portfolio-engine-etf-crypto/

---

## 1. Project Purpose

This project is built to answer a single core research question:

How does a portfolio allocation strategy behave over time when applied to ETFs and cryptocurrencies under realistic market conditions, transaction costs, management fees, and rebalancing schedules?

The engine focuses on portfolio-level behavior rather than individual trade execution. It is intended for education, quantitative research, and strategy exploration, not for live trading or investment advice.

Key design goals:
- Emphasize risk-adjusted performance instead of raw returns
- Compare multiple allocation strategies in a consistent framework
- Explore parameter sensitivity (lookback period and rebalance frequency)
- Provide a fully browser-based research tool with no backend dependency

---

## 2. High-Level System Flow

Market Data Sources (Yahoo Finance / Binance)  
→ Price Normalization and Timeline Alignment  
→ Daily Return and Volatility Calculation  
→ Momentum Signal Generation  
→ Strategy-Based Weight Allocation  
→ Execution Engine (Rebalance or Smart DCA with Fees)  
→ Daily Portfolio NAV Curve  
→ Performance Metrics and Visualizations  

---

## 3. Data Ingestion

### 3.1 Crypto Assets
- Source: Binance REST API (public endpoints)
- Market: Spot (USDT pairs)
- Timeframe: Daily
- Price used: Close price
- Maximum history: Approximately 1000 days (Binance limit)

### 3.2 Global ETFs
- Source: Yahoo Finance (via public CORS proxy)
- Timeframe: Daily
- Price used: Adjusted Close or Close

If a data request fails, the engine falls back to internally generated mock data to prevent UI crashes. Mock data is strictly for demonstration and should not be used for analysis.

---

## 4. Master Timeline Alignment

Assets may have different start dates or missing observations. To ensure consistency across the portfolio:
- All unique dates across assets are merged into a master timeline
- Missing prices are forward-filled using the most recent available close
- All asset price series are aligned to the same date index

This allows accurate day-by-day portfolio valuation and comparison.

---

## 5. Backtest Window Selection

The user defines a start date and end date. The engine locates the corresponding indices in the master timeline, slices all aligned price series accordingly, and enforces a minimum data length to avoid unstable or misleading results.

---

## 6. Daily Return Calculation

For each asset k on day t, the daily return is calculated as:

r_t = (P_t / P_{t-1}) − 1

Daily returns are used throughout the system for volatility estimation, correlation analysis, Sharpe and Sortino ratios, and Monte Carlo simulations.

---

## 7. Momentum Signal Generation

Momentum is calculated using a simple lookback-based return:

signal_k(t) = (P_k(t) / P_k(t − LB)) − 1

Where LB is the lookback period in days (for example, 30, 60, or 90). This signal represents relative strength and is used to rank assets.

---

## 8. Volatility Estimation

Volatility is estimated from daily returns:

σ_daily = standard deviation of daily returns  
σ_annual = σ_daily × √365

If volatility is zero or undefined, a small fallback value is applied to avoid division-by-zero errors.

---

## 9. Portfolio Allocation Strategies

Each strategy outputs target weights that sum to 100 percent unless no asset qualifies.

### 9.1 Equal Weight
w_k = 1 / N

### 9.2 Rank-Based Allocation
Assets are ranked by momentum:

Score_i = N − rank_i + 1  
SumScore = N(N + 1) / 2  
w_i = Score_i / SumScore

### 9.3 Top 3 Equal
w_top = 1 / 3

### 9.4 Top 3 Ranked
SumScore = 3(3 + 1) / 2 = 6  
w1 = 3/6, w2 = 2/6, w3 = 1/6

### 9.5 Top 50 Percent
K = ceil(N / 2)  
w = 1 / K

### 9.6 Absolute Momentum (Trend Filter)
If signal_k > 0, the asset is eligible.  
Eligible assets are equally weighted; assets with negative momentum receive zero weight.

### 9.7 Dual Momentum
Assets must satisfy both positive momentum and top relative performance. The top three qualifying assets are equally weighted.

### 9.8 Inverse Volatility (Risk Parity Style)
raw_k = 1 / volatility_k  
w_k = raw_k / sum(raw_k)

### 9.9 Adaptive Aggressive Allocation (AAA)
If positive-momentum assets exist, they are selected; otherwise, the top three ranked assets are used. Weights are assigned using inverse volatility.

---

## 10. Execution Engine

The engine supports two execution modes.

### 10.1 Lump Sum Rebalance Mode

Initial capital after front-end fee:
NetInitial = InitialCapital × (1 − FrontFee%)

Trading fee on each transaction:
NetTrade = TradeAmount × (1 − TradingFee%)

At each rebalance date:
Value_k = Shares_k × Price_k  
NAV = Cash + Σ(Value_k)  

Target_k = NAV × w_k  
Turnover = Σ |Target_k − Value_k|  

Cost = Turnover × TradingFee%  
NetNAV = NAV − Cost  

Shares_k = (NetNAV × w_k) / Price_k  

Transaction logs record the percentage of total portfolio traded.

### 10.2 Smart DCA Mode (No-Sell)

Capital injection:
Inflow = DCA_Amount  

Target after injection:
Ideal_k = (CurrentPortfolioValue + Inflow) × w_k  
Deficit_k = max(0, Ideal_k − CurrentValue_k)  

Allocation of new capital:
Allocation_k = Deficit_k / Σ(Deficit)  
Buy_k = Inflow × Allocation_k  

Trading fees apply only to new purchases. Logs are expressed as percentages of injected capital.

---

## 11. Fee Model

Front-End Fee (one-time):
InitialNet = Initial × (1 − FrontFee%)

Trading Fee:
NetTrade = Amount × (1 − TradingFee%)

Management Fee (annual, deducted daily):
DailyFee = (ManagementFee% / 100) / 365  
NAV_t = NAV_{t−1} × (1 − DailyFee)

Shares are adjusted proportionally to reflect daily NAV decay.

---

## 12. Performance Metrics

Let NAV_t be the portfolio value series.

CAGR:
TotalReturn = NAV_end / NAV_start  
Years = number_of_days / 365  
CAGR = (TotalReturn)^(1 / Years) − 1  

Volatility:
σ_annual = standard deviation(daily returns) × √365  

Sharpe Ratio:
rf_daily = (RiskFreeRate% / 100) / 365  
Sharpe = (mean(excess returns) × 365) / σ_annual  

Sortino Ratio:
Sortino = (mean(excess returns) × 365) / downside deviation  

Max Drawdown:
Peak_t = max(NAV_0 … NAV_t)  
Drawdown_t = (NAV_t − Peak_t) / Peak_t  
MaxDrawdown = min(Drawdown_t)  

Calmar Ratio:
Calmar = CAGR / |MaxDrawdown|  

Win Rate:
WinRate = number of positive return days / total days  

---

## 13. Grid Search

The engine evaluates all combinations of strategies, lookback periods, and rebalance frequencies. Each combination is fully simulated and ranked by Sharpe Ratio, with results sortable and filterable in the UI.

---

## 14. Visualization Modules

- Equity Curve (linear and logarithmic)
- Drawdown Curve
- Monthly Return Heatmap (compounded)
- Monthly Drawdown Heatmap (worst monthly drawdown)
- Correlation Heatmap
- 3D Hyper-Parameter Surface (Lookback × Rebalance × Metric)
- Time-Warp Surface Animation
- Monte Carlo Simulation (GBM-style, multiple paths)

Monte Carlo simulations are used for scenario exploration only, not prediction.

---

## 15. Limitations

- Uses daily close prices only
- No intraday data or slippage modeling
- Data availability depends on public APIs
- Results are regime-dependent
- Smart DCA does not rebalance existing holdings
- Not suitable for live trading execution

---

## 16. Best Practices

- Do not optimize solely for Sharpe Ratio
- Always inspect Max Drawdown and Calmar Ratio
- Prefer smooth parameter surfaces over sharp peaks
- Compare against benchmarks such as BTC or global ETF indices
- Use heatmaps to understand year-by-year behavior

---

## 17. Disclaimer

This project is for educational and research purposes only.  
It does not constitute financial advice.  
ETF and cryptocurrency markets involve significant risk.  
Use at your own risk.
