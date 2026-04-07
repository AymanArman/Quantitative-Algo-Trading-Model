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

---

Use `nrow(.)` inside gt() tab_style() chains causes errors — gt does not resolve the pipe placeholder in that context.

**Why:** Caused a render failure in the XLK validation table.
**How to apply:** Always compute row counts as a named variable before the gt() chain and reference that variable inside cells_body().

---

plotly `scatter3d` animations require `redraw = TRUE` in `animation_opts()` — `redraw = FALSE` causes frames not to update.

**Why:** 3D cluster chart was static despite animation slider being present.
**How to apply:** Always use `redraw = TRUE` for scatter3d; `redraw = FALSE` is fine for 2D charts.

---

plotly animation frame labels must sort chronologically — use `format(date, "%Y-%m")` not `format(date, "%b %Y")`.

**Why:** "%b %Y" sorts alphabetically (Apr before Jan), scrambling the date slider order.
**How to apply:** Always use "%Y-%m" as the frame variable for plotly animations. Display-friendly labels can go in hover text instead.
