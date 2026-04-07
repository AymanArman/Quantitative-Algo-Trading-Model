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

**Strategy:** Clustering drives signals from indicator mental model; Kalman Filter triggers trades.

**Why:** Course assignment with strict reproducibility and workflow requirements.
**How to apply:** All code decisions must respect tidymodels, API-only data, and R-only constraints.
