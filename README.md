# Portfolio Analytics & Risk Dashboard

A single-file web app (`portfolio-dashboard.html`) that turns a portfolio of holdings into performance, risk, diversification and allocation insights, then suggests what to do about them.

- No build step, no dependencies, no network calls. HTML, CSS and vanilla JavaScript only.
- Works offline in any modern browser (Chrome, Edge, Firefox, Safari).
- Supports light and dark mode automatically.

> **Important:** the price history is a **seeded simulation** (see [Data](#data)), not real market data. All numbers are illustrative until you connect real prices. This is educational software, not investment advice.

---

## Quick start

1. Download `portfolio-dashboard.html`.
2. Open it in a browser (double-click it). Nothing to install.

## Using the dashboard

| Control | Where | What it does |
| --- | --- | --- |
| Risk-free % | Header | Rate used in Sharpe, Sortino, alpha and the optimizer. Default 6.5%. |
| Target % (Equity, Bonds, Gold, Cash) | Header | Your target allocation. If the four do not add to 100 they are normalised. |
| Market move slider | Header | Sets a custom market shock (−50% to +20%) in the stress test. |
| Quantity boxes | Holdings table | Change shares held. Press Enter or click away and everything recalculates. |

## What you see

1. **Health score (0–100)** with a verdict and the three most important actions.
2. **Key metrics:** value, invested amount, total gain, today's change, XIRR, return versus benchmark, volatility, Sharpe, Sortino, maximum drawdown, beta and alpha, 1-day VaR/CVaR, effective number of holdings.
3. **Growth vs benchmark** and an **underwater (drawdown) chart**.
4. **Allocation vs target** and **equity sector exposure**.
5. **Risk contribution:** each asset's share of total risk against its share of capital.
6. **Correlation heatmap.**
7. **Efficient frontier:** 1,200 random long-only portfolios coloured by Sharpe, with your portfolio, the max-Sharpe portfolio and the minimum-variance portfolio marked.
8. **1-year Monte Carlo fan chart** (5–95% and 25–75% bands, median line).
9. **Stress tests** (market −10/−20/−30%, custom slider, rates +1%, gold +10%).
10. **Strategy:** ranked recommendations plus a rebalance table with ₹ Buy/Sell amounts.
11. **Optimizer view:** current vs max-Sharpe vs minimum-variance weights.
12. **Holdings table** with editable quantities.

---

## Data

Each asset is one row in the `A` array near the top of the script:

```js
// [symbol, name, class, sector, qty, startPrice, simBeta, idioVol, annualDrift, buyDay]
['HDFCBANK','HDFC Bank','Equity','Financials',80,1600,1,.12,.13,60]
```

- `class` must be one of `Equity`, `Bonds`, `Gold`, `Cash`.
- Sector alerts apply only to `Equity` rows whose sector is not `Diversified`.
- `simBeta`, `idioVol`, `annualDrift` only drive the simulation. They are not used once you supply real prices.
- Average cost is the simulated price on `buyDay`, so cost, P&L and XIRR are consistent.

The simulation uses a seeded random generator, so the result is identical on every load. It is a market factor plus per-asset noise, with one crash episode (days 250–285) so drawdown and stress metrics have something to show.

### Using real data

Replace the simulation block with your own data:

| Variable | Needed shape |
| --- | --- |
| `P[i]` | Array of `T + 1` daily closing prices for asset `i` (same order as `A`). |
| `mr` | Array of `T` daily benchmark returns (for example Nifty 50). |
| `T` | Number of daily returns (`prices − 1`). |
| `A[i][9]` | Index of the purchase day. For real cost basis, replace the `cost` line in `compute()` with your average cost. |

Everything else (returns, covariance, betas, charts, strategy) derives from these.

---

## Method

Risk analytics use the **current weights held constant** (daily-rebalanced) over the history. They answer "how has a portfolio like this behaved?", not "what did my past trades return?". XIRR is the money-weighted measure that uses actual purchase dates.

| Metric | Formula |
| --- | --- |
| Daily portfolio return | rₚ = Σ wᵢ rᵢ |
| Volatility | σ = stdev(rₚ) × √252 (sample standard deviation) |
| Sharpe | (mean(rₚ) × 252 − r_f) / σ |
| Sortino | (mean(rₚ) × 252 − r_f) / downside deviation, measured against r_f/252 |
| Beta | Cov(rₚ, r_m) / Var(r_m) |
| Alpha (Jensen) | (Rₚ − r_f) − β (R_m − r_f), annualised |
| Maximum drawdown | min over t of (NAV_t / running peak − 1) |
| VaR 95% (1 day) | Negative 5th percentile of historical daily returns, times portfolio value |
| CVaR 95% | Negative average of returns at or below that percentile |
| XIRR | r solving Σ CFₖ / (1 + r)^(tₖ) = 0, found by bisection |
| Risk contribution | RCᵢ = wᵢ (Σw)ᵢ / (wᵀΣw); contributions sum to 100% |
| Effective holdings | 1 / Σ wᵢ² (inverse Herfindahl index) |
| Diversification ratio | Σ wᵢσᵢ / σₚ |

Σ is the annualised sample covariance matrix of asset returns.

### Optimizer

- **Objective:** maximise Sharpe (or minimise variance), long-only, each asset capped at 25%.
- **Algorithm:** projected gradient ascent (2,500 iterations). Each step is projected onto the capped simplex {0 ≤ wᵢ ≤ cap, Σw = 1} using bisection on a shift τ.
- **Safeguard:** the result is compared with the best of 1,200 random capped portfolios and the better one is kept.
- **Expected returns:** shrunk to reduce estimation error: μᵢ = 0.5 × historical mean + 0.5 × (r_f + βᵢ × 5.5%), where 5.5% is an assumed equity risk premium.

### Monte Carlo

400 lognormal paths over 252 trading days. The drift is (μ − σ²/2), with μ the shrunk expected return and σ the historical volatility of your current mix. It uses a fixed seed, so results are repeatable. It shows a plausible range, not a forecast, and ignores fat tails and volatility clustering.

### Stress tests

Equity holdings move by their estimated beta × the market shock. For a market shock *s*, bonds move by −0.1 × *s* and gold by −0.25 × *s* (so −20% equities gives about +2% bonds, +5% gold). Cash does not move. Rates +1% applies −3% to equity (× beta) and −6% to bonds. These are rough scenario estimates.

---

## Strategy engine

| Signal | Trigger | Suggested action |
| --- | --- | --- |
| Allocation drift | Class more than 5 pp from target (over 10 pp is high severity) | Buy or sell the ₹ amount needed; use new contributions first |
| Single-stock concentration | One stock above 15% (over 20% is high severity) | Trim to 15% |
| Sector concentration | One equity sector above 30% | Add to other sectors |
| Low diversification | Effective holdings below 6 | Spread across more positions |
| Risk concentration | Asset has over 12% of risk and over 1.6× its capital weight | Reduce or offset |
| Drawdown | Portfolio more than 7% below its peak | Follow a pre-set rebalance/hold rule |
| Beta | Above 1.1 or below 0.6 | Adjust equity exposure |
| Better mix available | Optimizer Sharpe exceeds current by more than 0.15 | See optimizer weights |

The rebalance table shows **Hold** when a class is within ±2 pp of target.

**Health score** starts at 100 and subtracts:

| Penalty | Amount | Cap |
| --- | --- | --- |
| Allocation drift | 1.2 per pp of total drift | 25 |
| Effective holdings below 8 | 3 per holding below 8 | 20 |
| Stock above 15% | 2 per pp above | 15 |
| Sector above 30% | 1.5 per pp above | 15 |
| Sharpe below 1 | 10 per unit below 1 | 15 |
| Drawdown worse than −15% | 0.8 per pp beyond | 10 |

Labels: 80 and above **Well balanced**, 60–79 **Needs attention**, below 60 **Action needed**. The thresholds are opinionated defaults; edit them in `render()`.

---

## Built-in tests

Nine self-checks run on every load and print to the browser console. The footer shows `Self-checks: 9/9 passed`:

1. Maximum drawdown of 100 → 120 → 90 → 110 is −25%.
2. XIRR of −100 then +110 a year later is 10%.
3. Capped-simplex projection sums to 1 and respects bounds.
4. Weights sum to 1.
5. wᵀΣw equals the annualised variance of the portfolio return series.
6. Risk contributions sum to 100%.
7. Portfolio beta equals Σ wᵢβᵢ.
8. The max-Sharpe portfolio is no worse than the current one.
9. Optimizer weights obey the 25% cap.

## Limitations

- Simulated prices. Real use needs a market-data source.
- No CSV import, no saved data, no login. Edits reset on reload.
- Taxes, fees, dividends and corporate actions are not modelled.
- History-based statistics assume the past resembles the future, which it may not.
- Optimizer output is sensitive to return estimates; treat it as directional.

## Roadmap ideas

CSV transaction import and average-cost calculation, live price feed, multiple portfolios, saved state, dividend tracking, PDF reports, factor and fat-tail risk models.

## Disclaimer

For education and analysis only. Nothing here is investment, tax or legal advice. Consult a SEBI-registered adviser before making investment decisions.
