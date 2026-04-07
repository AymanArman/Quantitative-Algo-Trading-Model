---
name: Execution Feedback
description: Feedback on how to approach execution — what to keep, what to avoid
type: feedback
---

Do not remove cc_return from the return computation block — user wants it kept as the primary close-to-close return for analysis and PCA.

**Why:** I attempted to remove it when cleaning up; user corrected this immediately.
**How to apply:** cc_return is a planned output of Section 1 and feeds downstream feature engineering. Never drop it.

---

Workflow is: plan phases at high level in planning.md, then refine each phase in detail immediately before executing it. This is a deliberate deviation from the standard full-upfront planning rule, accepted due to deadline pressure.

**Why:** Hard deadline (2026-04-07 6pm EDT) made full upfront planning impractical.
**How to apply:** Before starting each section, read only that section of planning.md, refine it, get approval, then execute.

---

No parameter fitting on the test set — all model fitting, lookback selection, PCA, and clustering use training data only (June 2000 – Dec 2024).

**Why:** User explicitly corrected this when the plan said lookback would be computed on the full date range.
**How to apply:** Any rolling window calibration, elbow method, or signal threshold selection must use train_df only.
