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

**Status:** Brainstorming complete. Ready for planning.

**Why:** Course assignment with strict reproducibility and workflow requirements.
**How to apply:** All code decisions must respect tidymodels, API-only data, and R-only constraints. Strategy decisions are locked in strat_brainstorm.md — read that file for full rationale before planning.
