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
Dynamic monthly clustering: re-fit k-means at each rebalancing date using current feature values. ETFs shift clusters over time as their character changes — this is the core of the regime-aware strategy.

**Feature scaling:**
- Features (pc2_filtered, rel_strength, beta, hit_rate) are on very different scales — beta ~1, hit_rate ~0–1, pc2_filtered and rel_strength in small decimals
- K-means is distance-based so unscaled features let whichever feature has the largest variance dominate
- tidymodels recipe with `step_normalize()` applied before each monthly fit

**K selection — elbow method (training data only):**
- Fit K = 2 through 8 on training data; plot WCSS vs K as a line chart with points
- Select K at the elbow; K ≥ 3 required per strategy design
- K locked once and applied to all monthly fits

**Dynamic clustering:**
- At each rebalancing date (first trading day of each month), fit k-means on that date's feature values across all ETFs with complete features
- Output: `cluster_id` per ETF per rebalancing date
- XLK validation: table showing XLK's cluster assignment over time — confirms it consistently falls in the cluster with the highest pc2_filtered (cyclical group)

**Note — cluster label consistency is intentionally not addressed:**
- K-means randomly renumbers clusters each run (cluster 1 this month ≠ cluster 1 next month)
- This is irrelevant to the strategy: at each rebalancing date we identify the top cluster by its rolling average return, not by its label
- We never track a named cluster over time — we only ask "which cluster is performing best right now?" and allocate to it
- Attempting to align labels across runs would add complexity with no strategic benefit

**Visualisation:**
- Elbow chart: WCSS vs K, line + points, project color scheme
- 3D animated scatter (monthly frames): x = rel_strength, y = beta, z = hit_rate; color = cluster_id; one point per ETF per frame — shows ETFs moving through feature space and cluster assignments shifting over time

**Output:** `cluster_id` column added to `model_df` for each rebalancing date; feeds directly into Section 7 signal logic

---

### Section 6 — Portfolio Infrastructure
Build the execution and accounting framework before applying any signal.

**Starting capital:** $1,000,000

**Execution convention — Option 1 (split rebalance day):**
- Rebalance occurs on the first trading day of each month (day T); signal computed from data through T-1 close
- On rebalance day T, two P&L events are accounted for separately:
  - Gap: `old_weights × co_return_T` (T-1 close → T open, on old positions)
  - Intraday: `new_weights × oc_return_T` (T open → T close, on new positions)
  - Intermediate `pv_open` computed as transient variable; not stored as a column
- Non-rebalance days: `weights × cc_return` (close-to-close)
- `portfolio_value` column always represents end-of-day (close) value — single column, no split needed

**First rebalance:**
- Deploy $1,000,000 on first trading day of Jan 2025
- Signal computed using last 63 days of training data (Oct–Dec 2024) as the lookback window
- No warm-up period consumed from the test set — training data provides it
- Model fitting boundary unchanged: June 2000 – Dec 2024

**Rebalance sequence:**
  1. Compute `pv_open = pv_yesterday_close × (1 + sum(old_weights × co_return_T))`
  2. Compute new target weights from signal (Section 7)
  3. `new_weights × pv_open` → target values per ETF
  4. Subtract current values → `trade_value` per ETF
  5. Divide by open price → `quantity` to buy/sell
  6. `pv_close = pv_open × (1 + sum(new_weights × oc_return_T))`

**Assumptions (state explicitly in document):**
- Full execution at open price on rebalance day; zero market impact; zero transaction costs
- Cash positions earn 0% — no risk-free rate applied

**Per-ETF columns:** `weight`, `value`, `trade_value`, `quantity`
**Portfolio-level columns:** `portfolio_value` (end-of-day close), `daily_return`

Output: portfolio accounting table ready to receive signals.

---

### Section 7 — Signal Rules + Simulation
Apply momentum signal logic and simulate the strategy over the test set.

**Cluster ranking:**
- Equal-weighted rolling average `cc_return` over `lookback_days = 63` per cluster
- Top cluster = highest average return (can be negative — see edge case below)

**Rotation buffer:**
- Set at 2 SD of the training-period rank-1 vs rank-2 spread distribution (~0.148%); derived quantitatively
- On first rebalance (no prior cluster): no buffer applied
- **Design decision (surface in reflection):** When buffer blocks a rotation, the strategy stays in `prev_cluster` rather than going to cash. Alternative considered: treat sub-buffer spread as full uncertainty and exit to cash. Rejected because buffer uncertainty (competitor not convincing enough) is distinct from signal failure (current cluster gone cold). Cash is reserved for the all-negative edge case only.

**ETF ranking within top cluster:**
- Rank constituent ETFs by rolling average `cc_return` over `lookback_days`
- Ties broken by symbol alphabetically (arbitrary, consistent)

**Weight assignment:**
- Inverse rank weighting: `weight_i = (1 / rank_i) / sum(1 / rank_j)` for positive-return ETFs only
- ETFs with non-positive rolling average return receive `weight = 0`
- Weights renormalised to sum to 1 across positive-return ETFs

**All-negative edge case (Option A — implicit cash):**
- If all ETFs in the top cluster have non-positive rolling average return, all weights = 0
- Portfolio holds 100% cash for that period; earns 0%
- This is a natural consequence of the weighting rule — no separate mechanism needed
- Document explicitly: *"Strategy moves to cash when no ETF in the top cluster shows positive momentum"*

**Signal:** `signal ∈ {0, 1}` per ETF per rebalance date

**Simulation:** Monthly rebalancing Jan 2025 – Mar 2026 using infrastructure from Section 6

Output: full trade log + daily `portfolio_value` series over test period.

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
