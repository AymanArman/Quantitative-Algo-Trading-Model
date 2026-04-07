---
name: Project Context
description: QT Model course assignment details, constraints, deadlines, and grading criteria
type: project
---

Clustering-based S&P 500 sector trading model in R, submitted as .qmd + self-contained .html + final_context.md.

**Due:** 2026-04-07 at 6pm EDT (hard deadline — presentation recorded, deleted after grading)
**Weight:** 20%, individual, peer graded before Prof grading

**Assets:**
- Benchmark: SPY
- Sector ETFs: XLB, XLC (from 2018), XLE, XLF, XLG, XLI, XLK, XLP, XLRE (from 2019), XLV, XLU, XLY
- Training: June 2000 – Dec 2024 | Testing: Jan 2025 – March 2026
- Daily data frequency, monthly rebalancing

**Hard constraints:**
- All data via API only — no local CSV/Excel files
- All analysis in R inside Quarto — Python penalized
- Must use tidymodels workflow
- HTML must be self-contained (penalty: -5 pts)
- Public GitHub repo, link in email submission
- Mental model/decision rules visualized with mermaid.ai
- Kalman Filter used alongside clustering to trigger trades

**Strategy:** Fully defined in `strat_brainstorm.md`. Core architecture:
1. Kalman Filter smooths rolling PC2 loadings (cyclical/defensive character per ETF)
2. Clustering (k-means, elbow method, K≥3) groups ETFs using 4 features: Kalman-filtered PC2 loading, relative strength vs SPY, beta to SPY, momentum consistency (hit rate)
3. Cross-sectional momentum ranks clusters by rolling average return; capital allocated to top cluster only
4. Within top cluster: long only, inverse rank weighting among positive-return ETFs, normalised to sum to 1
5. Monthly rebalancing; daily data and signal computation

**Status:** Sections 1–5 complete. Next session: Section 6 (Portfolio Infrastructure) and Section 7 (Signal Rules + Simulation).

**Section 1 decisions locked:**
- `adj_open = open * (adjusted / close)` — derived adjusted open for consistency with Yahoo's adjusted close
- Four log return columns computed: `cc_return` (close-close, primary for analysis/PCA), `oo_return`, `co_return`, `oc_return`
- SPY and XLG are benchmark/comparison only — not clustered, not traded
- Data confirmed clean: no zero/negative prices; 4540 dates with <13 symbols (expected, driven by XLC/XLG/XLRE shorter histories)
- `df` is the master dataframe

**Section 2 decisions locked:**
- `algo_df` is the canonical algo dataset — all subsequent sections add columns to it
- Columns: `date, symbol, open (= adj_open), close (= adjusted), cc_return, oo_return, co_return, oc_return`
- Both price columns are adjusted — raw open/close dropped
- `train_df`: `date <= 2024-12-31` | `test_df`: `2025-01-01 <= date <= 2026-03-31`
- No parameter fitting on test set — lookback selection (Section 3) and all model fitting use training data only

**Section 3 decisions locked:**
- `lookback_days <- 63L` — selected on literature grounds (Jegadeesh & Titman 1993); empirical correlation tests were inconclusive (weak negative r, <38% ETFs significant)
- Two diagnostic analyses run: absolute return correlation and relative return vs SPY — both weak, documented in QMD
- `H <- 21L` (holding period); Chan 2013 non-overlapping rule applied (stride = H)

**Section 4 decisions locked:**
- `algo_df` (long) used for all feature engineering; `model_df` (wide) created at end of Section 4
- Rolling PCA daily, `compute_rolling_pc2()` cached via knitr; `apply_kalman_pc2()` separate function
- Kalman Filter: Q = 0.001, R = 0.1 (locked)
- XLC/XLRE excluded from PCA until they have full 63-day history
- Features added to `algo_df`: `pc2_raw`, `pc2_filtered`, `cluster_character`, `rel_strength`, `beta`, `hit_rate`
- `model_df`: wide format, one row per date, ETF features as columns; SPY/XLG excluded

**Section 5 decisions locked:**
- Dynamic monthly clustering: k-means re-fit at each rebalancing date using current feature values
- K selected via elbow method on training data (WCSS vs K=2–8); K hardcoded as `K <- 4L` (user selected 4 after elbow inspection)
- Features scaled via `step_normalize()` in tidymodels recipe before each fit
- Cluster label consistency intentionally not addressed — strategy ranks clusters by rolling return at each date, never tracks a named cluster over time
- `cluster_assignments` long df (date, symbol, cluster_id) joined onto `model_df`
- Visualisations: elbow chart (plotly), 3D animated scatter (rel_strength × beta × hit_rate, colored by cluster_id)

**Section 6 decisions locked:**
- Starting capital: $1,000,000
- Execution convention: Option 1 (split rebalance day) — old_weights × co_return (gap), new_weights × oc_return (intraday); non-rebalance days use cc_return
- `pv_open` is a transient intermediate, not a stored column; `portfolio_value` is always end-of-day close
- First rebalance: Jan 2025 day 1; lookback window uses last 63 days of training data (Oct–Dec 2024); no test-period warm-up
- Cash earns 0% — no risk-free rate applied; zero transaction costs; full execution at open

**Section 7 decisions locked:**
- Top cluster: highest equal-weighted rolling avg cc_return over 63 days (can be negative)
- Rotation buffer: 0 (removed) — empirically tested; buffer caused strategy to stall in stale clusters, removing it beat SPY over test period
- ETF weighting: inverse rank, positive-return ETFs only; renormalised to sum to 1
- All-negative edge case: implicit cash via zero weights (Option A) — documented explicitly
- Ties broken alphabetically by symbol

**Status:** Sections 1–7 written. Next: render and verify output, then Section 8.

**Why:** Course assignment with strict reproducibility and workflow requirements.
**How to apply:** All code decisions must respect tidymodels, API-only data, and R-only constraints. Strategy decisions are locked in strat_brainstorm.md — read that file for full rationale before planning.
