# Planning — Quantitative Algo Trading Model

> **Workflow note:** Phases are defined structurally here. Each phase is refined in detail immediately before execution. The Quarto document IS the deliverable — each section mixes narrative, implementation, and analysis sequentially.

---

## Deliverables

- `model.qmd` — Quarto source (R only, all implementation inline)
- `model.html` — self-contained rendered HTML (`embed-resources: true`)
- `final_context.md` — locked decisions, assumptions, deferred items
- Public GitHub repo, link submitted via email

**Hard constraints:**
- All data via API only
- R + tidymodels only (Python penalized)
- HTML self-contained
- Mermaid diagram of decision flow
- Kalman Filter + clustering both present

---

## Document Structure & Phases

### Section 1 — Data
Pull daily OHLCV for all 12 sector ETFs (XLB, XLC, XLE, XLF, XLG, XLI, XLK, XLP, XLRE, XLV, XLU, XLY) and SPY via API.
- Training: June 2000 – Dec 2024
- Test: Jan 2025 – Mar 2026
- Handle shorter histories: XLC (from 2018), XLRE (from 2019)
- Explore and visualise the raw data; clean and confirm quality
- Compute all three return conventions per ETF per day: open-open, close-open, open-close — convention selected later without reworking data
- Output: tidy long-format dataframe `{date, ticker, open, close, oo_return, co_return, oc_return}`

*Refine before execution:* API source, handling of missing dates, adjusted vs unadjusted prices.

---

### Section 2 — Lookback Period Selection
Empirically select a single universal lookback period for rolling returns.
- Test: 21, 63, 126, 252 trading days (≈ 1, 3, 6, 12 months)
- Method: correlate rolling average return at each lookback to next-period return; assess p-value
- Run on SPY, then equal-weighted sector ETF average
- Handle XLC/XLRE shorter histories separately; verify result holds on full universe from 2019
- Selection rule: shortest lookback with largest significant correlation; document any SPY vs sector conflict
- Output: single locked integer `lookback_days` used for all downstream computation

*Refine before execution:* exact correlation/predictiveness methodology, conflict resolution rule.

---

### Section 3 — Algo Dataset Construction
Build the base dataset that all features and signals attach to.
- Compute rolling returns per ETF over `lookback_days`
- This is the canonical table — every subsequent section adds columns to it
- Output: `{date, ticker, rolling_return}` indexed by trading day over training period

*Refine before execution:* confirm rolling return definition (arithmetic vs log), alignment with rebalance dates.

---

### Section 4 — Feature Engineering
Add the four clustering features to the algo dataset.

**4a — PCA → PC2 → Cyclical/Defensive Label**
- Rolling PCA on sector return covariance matrix over `lookback_days`
- Extract PC2 loadings per ETF per day
- Sign-align: XLK loading on PC2 forced positive; flip all loadings if needed
- Apply Kalman Filter to raw PC2 loadings (per ETF independently) to smooth estimation noise
- Derive qualitative label: positive filtered loading = cyclical, negative = defensive
- Output column: `pc2_filtered`, `cluster_character` (cyclical/defensive)

**4b — Relative Strength vs SPY**
- Rolling average return of ETF minus rolling average return of SPY over `lookback_days`
- Output column: `rel_strength`

**4c — Beta to SPY**
- Rolling OLS slope of ETF daily returns on SPY daily returns over `lookback_days`
- Output column: `beta`

**4d — Momentum Consistency (Hit Rate)**
- Proportion of positive return days over `lookback_days`
- Output column: `hit_rate`

*Refine before execution:* Kalman Filter state model definition (Q, R initialisation), rolling PCA implementation, feature scaling approach.

---

### Section 5 — Clustering
Assign each ETF to a cluster for each day over the training dataset.
- tidymodels workflow: `recipe` (scale features) + `k_means` spec
- Elbow method: fit K = 2 through 8, select K at elbow of within-cluster SS; K ≥ 3 required
- Assign cluster ID per ETF per day
- Validate: XLK should consistently fall in the cyclical cluster
- Output column: `cluster_id`

*Refine before execution:* cluster label stability across time.

---

### Section 6 — Portfolio Infrastructure
Build the execution and accounting framework before applying any signal.
- Define execution convention: open-open, close-open, or open-close (resolve here with rationale)
- Per-ETF columns: `weight`, `value`, `trade_value`, `quantity`
- Portfolio-level columns: `portfolio_value`, `daily_return`
- Rebalance sequence (from strat_brainstorm.md):
  1. Mark holdings to closing price → current portfolio value
  2. Compute new target weights
  3. New weights × portfolio value → target values
  4. Subtract current values → trade_value per ETF
  5. Divide by price → quantity to buy/sell
- Starting capital: TBD during phase refinement
- Assume full execution at rebalance price, zero market impact, zero transaction costs (explicit assumption)
- Output: portfolio accounting table ready to receive signals

*Refine before execution:* starting capital, handling of first rebalance, return convention criteria.

---

### Section 7 — Signal Rules + Simulation
Apply momentum signal logic and simulate the strategy over the test set.
- Cluster ranking: equal-weighted rolling average return per cluster; top cluster selected
- Rotation buffer: only rotate if new top cluster outperforms current by > X% (X resolved empirically)
- ETF ranking within top cluster: by rolling average return
- Weight assignment: inverse rank normalised to 1; zero weight for negative-return ETFs
- Signal: `signal ∈ {0, 1}` per ETF per rebalance
- Simulate monthly rebalancing from Jan 2025 – Mar 2026 using infrastructure from Section 6
- Output: full trade log + daily portfolio value series over test period

*Refine before execution:* rotation buffer threshold, handling of ties in ranking, edge case where all ETFs in top cluster have negative returns.

---

### Section 8 — Evaluation
Analyse and evaluate strategy results against SPY benchmark.
- Metrics: annualised return, annualised Sharpe (risk-free rate TBD), max drawdown, Calmar ratio
- Turnover: average monthly portfolio turnover; flag if materially high
- Benchmark: SPY buy-and-hold over same test period
- Evaluate test period only for strategy performance; training period used for methodology validation
- Equity curve plot, drawdown plot, turnover chart
- Discuss results honestly — what worked, what didn't, what assumptions matter most

*Refine before execution:* risk-free rate assumption, annualisation convention.

---

## Phase Dependencies

```
Section 1 → Section 2 → Section 3 → Section 4 → Section 5 → Section 6 → Section 7 → Section 8
```

Strictly sequential. Each section must produce clean, validated output before the next begins.

---

## Locked Assumptions

- Execution at rebalance-day price, no market impact
- Zero transaction costs (stated explicitly in document)
- Long only
- Capital fully deployed within top cluster at all times
- XLK is the PC2 sign-alignment anchor
- Single universal lookback period for all rolling computations
- All three return conventions (open-open, close-open, open-close) computed upfront; convention selection is tentative — determined by specific criteria during Section 6
- K is fixed globally, selected once via elbow method on training data

---

## Deferred Design Decisions (resolve during execution)

- API source for data (Section 1)
- Exact lookback period — output of Section 2
- Kalman Filter Q and R values (Section 4)
- K is fixed globally — selected once via elbow method on training data (Section 5)
- Starting capital (Section 6)
- Rotation buffer threshold (Section 7)
- Risk-free rate for Sharpe (Section 8)
