# 📊 Global Portfolio Engine (GPE) v48.0

### Strict Real-Data Quantitative Portfolio Research Engine (ETF & Crypto)

Global Portfolio Engine (GPE) is a **fully client-side**, browser-based quantitative portfolio research platform designed for systematic asset allocation analysis across **Global ETFs** and **Cryptocurrencies**.

GPE operates exclusively on **real historical market data**:

* **ETFs & equities** via Yahoo Finance Chart API (through public CORS proxies)
* **Cryptocurrencies** via Binance public REST API (spot USDT pairs)

The engine simulates portfolio NAV evolution under realistic assumptions — including **transaction costs, management fees, rebalance frequency, and capital injection (DCA)** — and evaluates results using **risk-adjusted performance metrics** and advanced interactive visualizations.

🌐 Live Application (GitHub Pages)
[https://waranyutrkm.github.io/portfolio-engine-etf-crypto/](https://waranyutrkm.github.io/portfolio-engine-etf-crypto/)

---

## 1. Project Objective

> **How do portfolio allocation strategies behave over time when subjected to real market dynamics, parameter variation, and realistic cost structures?**

GPE focuses on **portfolio-level allocation behavior**, not execution microstructure.
It is intended for **education, quantitative research, and strategy exploration** — not for live trading or investment advice.

---

## 2. Core Design Principles

* **Strict real data only** (no mock, no synthetic prices)
* **Deterministic & reproducible** given identical data availability
* **Risk-adjusted evaluation first**, raw return second
* **Parameter robustness over point optimization**
* **No backend, no database, no server**

---

## 3. System Architecture Overview

**Market Data (Yahoo / Binance)**
→ **Strict Validation**
→ **Master Timeline Union + Forward-Fill**
→ **Daily Returns & Rolling Volatility**
→ **Momentum Signals**
→ **Strategy-Based Target Weights**
→ **Execution Engine (Rebalance / Smart DCA)**
→ **Daily NAV Curve**
→ **Metrics, Surfaces, Robustness & Monte Carlo**

---

## 4. Market Data Ingestion

### 4.1 Cryptocurrency Mode

* Source: Binance REST API
* Market: Spot USDT pairs
* Timeframe: Daily (`1d`)
* Price: Close
* Max history: ~1000 observations

### 4.2 ETF / Equity Mode

* Source: Yahoo Finance Chart API
* Access: Public proxy
* Timeframe: Daily (`1d`)
* Range: Up to 5 years
* Price: Close (not adjusted close)

---

## 5. Strict Fetch Policy

An asset is included **only if**:

* API request succeeds
* At least 50 daily observations are returned

Failed assets are dropped silently.
If fewer than two assets remain, execution halts.

---

## 6. Master Timeline Alignment (Union + Forward-Fill)

Let ( D ) be the union of all dates across valid assets.

For each asset ( k ), aligned price series ( \tilde{P}_{k,t} ) is constructed as:

[
\tilde{P}*{k,t} =
\begin{cases}
P*{k,t} & \text{if available} \
\tilde{P}_{k,t-1} & \text{otherwise}
\end{cases}
]

This ensures consistent portfolio valuation.

---

## 7. Backtest Window Selection

User-selected Start and End dates are mapped to the master timeline:

* ( startIdx = \min { t \mid D_t \ge Start } )
* ( endIdx = \min { t \mid D_t > End } )

Guardrail:
[
(endIdx - startIdx) \ge 30
]

---

## 8. Daily Returns

For asset ( k ) on day ( t ):

[
r_{k,t} = \frac{P_{k,t}}{P_{k,t-1}} - 1
]

Used for volatility, Sharpe/Sortino, and Monte Carlo estimation.

---

## 9. Momentum Signal

Momentum signal is defined as:

[
signal_k(t) = \frac{P_k(t)}{P_k(t-LB)} - 1
]

Where ( LB ) is the lookback period (days).

---

## 10. Portfolio Allocation Strategies

*(Formulas & Worked Examples)*

Let:

* ( N ) = number of active assets
* ( \sigma_k ) = annualized volatility
* ( w_k ) = target weight

---

### 10.1 Equal Weight (1/N)

[
w_k = \frac{1}{N}
]

**Example**
4 assets → each receives 25%.

---

### 10.2 Rank-Based Momentum (Linear Score)

[
Score_i = N - rank_i + 1
\quad,\quad
\sum Score = \frac{N(N+1)}{2}
]

[
w_i = \frac{Score_i}{\sum Score}
]

**Example**

| Asset | Momentum | Rank | Weight |
| ----- | -------- | ---- | ------ |
| A     | +12%     | 1    | 40%    |
| B     | +8%      | 2    | 30%    |
| C     | +3%      | 3    | 20%    |
| D     | −2%      | 4    | 10%    |

---

### 10.3 Top 3 Momentum (Equal)

[
w_k =
\begin{cases}
\frac{1}{3} & rank_k \le 3 \
0 & \text{otherwise}
\end{cases}
]

---

### 10.4 Top 3 Momentum (Ranked)

[
w_i = \frac{K - rank_i + 1}{\frac{K(K+1)}{2}},\quad K=3
]

**Example** → weights: 50%, 33.3%, 16.7%

---

### 10.5 Top 50% Selection

[
K = \lceil N/2 \rceil
\quad,\quad
w_k = \frac{1}{K}\ \text{for top }K
]

---

### 10.6 Absolute Momentum (Trend Filter)

[
Eligible = {k \mid signal_k(t) > 0}
]

[
w_k =
\begin{cases}
\frac{1}{|Eligible|} & k \in Eligible \
0 & \text{otherwise}
\end{cases}
]

---

### 10.7 Dual Momentum

Filter ( signal_k(t) > 0 ), select top 3, equal weight.

---

### 10.8 Inverse Volatility

[
raw_k = \frac{1}{\sigma_k}
\quad,\quad
w_k = \frac{raw_k}{\sum raw}
]

---

### 10.9 Adaptive Aggressive Allocation (AAA)

1. Use assets with ( signal_k(t) > 0 ); else fallback to top 3
2. Apply inverse volatility weighting

[
w_k \propto \frac{1}{\sigma_k}
]

---

## 11. Execution Engine

### 11.1 Initial Allocation

Front-end fee:
[
Capital_0 = Initial \times (1 - FrontFee)
]

Trading fee:
[
NetBuy_k = Buy_k \times (1 - TradeFee)
]

Shares:
[
Shares_k = \frac{NetBuy_k}{Price_k}
]

---

### 11.2 Lump-Sum Rebalance

Turnover:
[
Turnover = \sum |Target_k - Current_k|
]

Trading cost:
[
Cost = Turnover \times TradeFee
]

---

### 11.3 Smart DCA (No-Sell)

Deficit:
[
Deficit_k = \max(0,\ Ideal_k - Current_k)
]

New capital is allocated proportionally to deficits.

---

## 12. Management Fee (Daily Drag)

[
DailyFee = \frac{MgmtFee}{365}
]

Holdings scaled daily:
[
Shares_k \leftarrow Shares_k \times (1 - DailyFee)
]

---

## 13. Performance Metrics

### CAGR

[
CAGR = \left(\frac{NAV_{end}}{NAV_{start}}\right)^{1/Years} - 1
]

### Volatility

[
\sigma_{annual} = std(r_t)\times\sqrt{365}
]

### Sharpe

[
Sharpe = \frac{mean(r_t - rf_{daily})\times365}{\sigma_{annual}}
]

### Sortino

[
Sortino = \frac{mean(excess)\times365}{DownsideDev_{annual}}
]

### Max Drawdown

[
MDD = \min_t \left(\frac{NAV_t - Peak_t}{Peak_t}\right)
]

### Calmar

[
Calmar = \frac{CAGR}{|MDD|}
]

---

## 14. Grid Search & Robustness

All combinations of:

* Strategy
* Lookback
* Rebalance

Robustness score:
[
Robust = mean(Self + Neighbors)
]

---

## 15. Monte Carlo Simulation

Geometric Brownian Motion:

[
P_{t+1} = P_t \cdot e^{(\mu - 0.5\sigma^2) + \sigma Z}
]

Where ( Z \sim \mathcal{N}(0,1) )

---

## 16. Research Caveats

* Survivorship bias possible
* Close prices only
* Proxy reliability affects ETF availability
* Monte Carlo is exploratory, not predictive

---

## 17. Disclaimer

This project is for **educational and research purposes only**.
It does **not** constitute financial advice.
Use at your own risk.
