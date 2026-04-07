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

### Section 2 — Algo Dataset Construction
Build the base dataset that all features and signals attach to.
- This is the canonical table — every subsequent section adds columns to it
- Initial columns: `{date, ticker, open, close, oo_return, co_return, oc_return}`
- Rolling returns over `lookback_days` added after Section 3 locks the parameter
- Covers full date range; SPY and XLG included for benchmark columns but not for feature/cluster columns

*Refine before execution:* confirm return definitions (arithmetic vs log), alignment with rebalance dates.

---

### Section 3 — Lookback Period Selection
Empirically select a single universal lookback period for rolling returns.
- Computed in a **separate diagnostic table**, draws from Section 2 `cc_return`
- Run on **training data only** (June 2000 – Dec 2024) — no parameter fitting on test set; all ETFs including SPY and XLG
- Candidates: L = 21, 63, 126, 252 trading days (≈ 1, 3, 6, 12 months); holding period H = 21 days fixed

**Correlation methodology (Chan 2013 non-overlapping rule):**
- All candidates have L ≥ H, so stride = H = 21 days (non-overlapping holding periods)
- Index sequence per ETF × L: `seq(from = L + 1, to = nrow - H, by = H)`
- At each step point `i`: signal = `mean(cc_return[(i-L):(i-1)])`; forward = `sum(cc_return[i:(i+H-1)])`
- This produces a two-column intermediate table (signal, forward_return) — one row per step point
- Run `cor.test()` on those two columns → extract `r` and `p_value`
- Approximate sample counts: L=21→~291, L=63→~281, L=126→~278, L=252→~271

**Analysis 1 — Absolute return correlation (retained):**
- Signal: `mean(cc_return[(i-L):(i-1)])`; forward: `sum(cc_return[i:(i+H-1)])`
- Output table (one row per ETF × L): `symbol, L, H, n_obs, r, r_sq, p_value`
- Aggregation table (one row per L): median r, median r², % ETFs significant
- Note: absolute lookback returns are still used downstream as clustering features (Section 4) — this analysis is diagnostic, not the primary selection basis

**Analysis 2 — Relative return vs SPY (primary selection basis):**
- Same methodology and stride logic as Analysis 1
- Signal: `mean((ETF_cc_return - SPY_cc_return)[(i-L):(i-1)])`; forward: `sum((ETF_cc_return - SPY_cc_return)[i:(i+H-1)])`
- SPY excluded from its own relative return test; XLG excluded from clustering but included here as diagnostic
- Rationale: strategy ranks clusters by relative outperformance vs market — relative return correlation directly validates the signal used for selection
- Same output tables as Analysis 1

**Selection rule (applied to Analysis 2):** shortest L where median r is maximised and majority of ETFs significant; document any conflicts with Analysis 1

**Presentation:** both analyses rendered as gt tables (summary + detail) using project standard color scheme

**Output:** single locked integer `lookback_days` used for all downstream algo dataset computation

---

### Section 4 — Feature Engineering
Add the four clustering features to `algo_df` (long format), then pivot to wide `model_df` at the end.

**Dataframe strategy:**
- `algo_df` (long) — master dataset; all feature engineering done here via `group_by(symbol)`
- `model_df` (wide) — created once at the end of Section 4 by pivoting; one row per date, each ETF's features as columns; input for clustering (Section 5) and portfolio simulation (Sections 6–7)

**4a — PCA → PC2 → Cyclical/Defensive Label**

*Rolling PCA (daily, not static):*
- ETF universe: 11 tradeable sector ETFs only — SPY and XLG excluded; XLC and XLRE excluded until they have a full `lookback_days` window of history
- Function `compute_rolling_pc2(algo_df, lookback_days)`:
  - Internally pivots long → wide (date × ETF columns of cc_return) for matrix operations
  - At each date t: extract trailing `lookback_days` rows → compute covariance matrix → eigen decomposition → extract PC2 loadings per ETF
  - Sign-align: XLK loading on PC2 forced positive; flip all loadings on that date if needed
  - Returns long df: `date, symbol, pc2_raw`
- Snapshot verification displayed after this step: eigenvector matrix for one representative date (ETFs as rows, PCs as columns) to confirm cyclical/defensive split

*Kalman Filter (separate function, run after PCA):*
- Function `apply_kalman_pc2(pc2_raw_df)`:
  - 1D local level model per ETF independently: true state evolves slowly, raw PC2 loading is noisy observation
  - Locked parameters: Q = 0.001 (process noise), R = 0.1 (observation noise)
  - Returns long df: `date, symbol, pc2_raw, pc2_filtered`
- Output columns added to `algo_df`: `pc2_raw`, `pc2_filtered`, `cluster_character` (cyclical if pc2_filtered > 0, defensive otherwise)

**4b — Relative Strength vs SPY**
- Rolling average return of ETF minus rolling average return of SPY over `lookback_days`
- Computed in long format via `group_by(symbol)`
- Output column added to `algo_df`: `rel_strength`

**4c — Beta to SPY**
- Rolling OLS slope of ETF daily returns on SPY daily returns over `lookback_days`
- Computed in long format via `group_by(symbol)`
- Output column added to `algo_df`: `beta`

**4d — Momentum Consistency (Hit Rate)**
- Proportion of positive return days over `lookback_days`
- Computed in long format via `group_by(symbol)`
- Output column added to `algo_df`: `hit_rate`

**End of Section 4 — pivot to wide:**
- `model_df <- algo_df %>% pivot_wider(...)` — one row per date, columns: `date, {ETF}_pc2_filtered, {ETF}_cluster_character, {ETF}_rel_strength, {ETF}_beta, {ETF}_hit_rate`
- SPY and XLG excluded from `model_df`
- `algo_df` retained as-is for reference

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
- Exact lookback period — output of Section 3
- Kalman Filter Q and R values (Section 4)
- K is fixed globally — selected once via elbow method on training data (Section 5)
- Starting capital (Section 6)
- Rotation buffer threshold (Section 7)
- Risk-free rate for Sharpe (Section 8)
