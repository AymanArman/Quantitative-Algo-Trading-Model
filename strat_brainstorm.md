# Strategy Brainstorm

## Core Philosophy

**Primary strategy type:** Cross-sectional momentum across S&P 500 sector ETFs.

**Why momentum:**
Momentum strategies allow active risk management without contradicting entry or exit signals — unlike mean-reversion strategies where cutting a losing position conflicts with the signal itself. This property makes momentum particularly well-suited to disciplined, rules-based trading.
> Chan, E. (2013). *Algorithmic Trading: Winning Strategies and Their Rationale.* Wiley. (Chapter on momentum strategies)

**Why momentum + clustering:**
Cross-sectional momentum strategies pair naturally with clustering analysis. Clustering can identify which sectors are behaving similarly at a given time — momentum signals can then be applied within or across those clusters to rank and select positions, rather than treating all sectors as independent.

## Clustering Method

Use PCA on a sector-return covariance matrix to separate defensive vs cyclical ETFs, using XLK's loading on PC2 as a sign-alignment anchor.

**Rationale:** PC2 naturally captures the cyclical/defensive split in sector returns after removing the market factor (PC1). Since PCA eigenvectors are sign-ambiguous, we impose a convention: XLK (tech) is pinned to a positive loading on PC2. ETFs that co-move with tech are labelled cyclical (positive loading); those that move against it are labelled defensive (negative loading). This is a heuristic — it is defensible but can break down in regimes where a sector's co-movement with tech decouples from its economic character (e.g. XLE during oil shocks).

## Momentum Strategy

Momentum operates at two layers:

**Layer 1 — Cluster momentum (universe selection):**
At each rebalance, rank clusters by their rolling average return over the selected lookback period. Allocate capital only to the top performing cluster. This is the regime filter — it ensures we are invested in the strongest behavioural group, not just any group.

**Layer 2 — ETF momentum within cluster (security selection):**
Within the top performing cluster, rank individual ETFs by their rolling average return over the same lookback period. Take long positions in the top performers. This ensures we hold the strongest names within the strongest regime.

**Rebalancing and exits:**
Cluster membership and rankings are recomputed at every rebalance. Cluster labels have no continuity — what matters is whether an ETF belongs to the top performing group at each rebalance period. If an ETF shifts out of the top cluster, it is exited. If a new ETF enters the top cluster with strong momentum, it is added. Positions are tracked at the ETF level; clusters are a dynamic filter applied fresh at each rebalance.

**Why clustering adds value over pure momentum:**
A standalone momentum strategy might chase a single hot ETF that is an outlier within a weak group. Requiring both the cluster and the ETF within it to show strong momentum adds a confirmation layer — the ETF must be strong and belong to a strong regime.

**On systematic exits:**
When the top cluster shifts, all ETFs not in the new top cluster are exited immediately — even if they remain individually profitable. This is intentional. A rules-based momentum strategy derives its risk management advantage precisely from this discipline: exit and entry signals never contradict each other. Holding a position because it "feels" like it should recover introduces discretionary override and undermines the strategy.
> Chan, E. (2013). *Algorithmic Trading: Winning Strategies and Their Rationale.* Wiley.

**Concerns:**

- *Turnover risk:* When two clusters have similar return profiles, small shifts in cluster membership could trigger frequent rotations with marginal signal differences. This generates transaction costs that may outweigh the signal. Turnover rate will be tracked and evaluated during backtesting.
- *Buffer threshold:* A rotation buffer may be introduced — only rotate to a new top cluster if it outperforms the current cluster by more than X%. This filters noise-driven marginal shifts from genuine regime changes. Left as a design parameter to be determined empirically during backtesting.

## Portfolio Construction

**Number of clusters:**
Minimum of 3. Final K selected empirically using the elbow method — the point where adding another cluster produces meaningfully diminishing reduction in within-cluster variance. This keeps clusters distinct enough that return profiles don't collapse toward each other.

**Position sizing within the top cluster:**
All ETFs in the top cluster are eligible. Weights are return-based — stronger performing ETFs within the cluster receive larger allocations. Exact weighting scheme to be determined in planning.

**Long only:**
The strategy takes no short positions. All sector ETFs are diversified and carry inherent positive market exposure — shorting them risks introducing large market-driven swings that would dilute and obscure the sector rotation signal. The strategy instead capitalises on winners rather than penalising laggards.

**Weight assignment:**
Weights are assigned only within the highest performing cluster. ETFs with negative rolling returns over the lookback period receive zero weight and no position. Among positive-return ETFs, weights are assigned by inverse rank normalised to sum to 1:

```r
# N = number of positive-return ETFs in top cluster
# rank 1 = best performer
weight = (N - rank + 1) / (N * (N + 1) / 2)
```

This ensures full capital deployment (weights always sum to 1) regardless of how many ETFs qualify, while still rewarding stronger performers with larger allocations and avoiding concentration risk from return-proportional weighting.

**Rebalancing mechanics:**
Portfolio rebalancing occurs monthly. We assume full execution at month-end closing price with no market impact — defensible given the liquidity of the sector ETFs traded. This is noted as an explicit assumption in the document.

The binary trade signal (-1, 0, 1) handles entry and exit only. Sizing and partial rebalancing are handled separately via the following columns per ETF:

- `weight` — target allocation at rebalance, return-based, sums to 1 across held ETFs
- `value` — `weight × portfolio_value` — dollar amount allocated
- `trade_value` — `value_t - value_{t-1}` — positive = buy, negative = sell
- `quantity` — `value / price` — number of shares

**Rebalance sequence on rebalance day:**
1. Mark all current holdings to today's closing price → compute current portfolio value
2. Compute new target weights from momentum ranking within top cluster
3. Multiply new weights × current portfolio value → new target values per ETF
4. Subtract current marked-to-market values → trade_value per ETF
5. Divide trade_value by price → quantity to buy or sell

This ensures weights always sum correctly against actual current portfolio value, not stale prior values. Capital flows automatically from weaker to stronger holdings through the weight recomputation, without requiring a separate signal.

## Clustering Features

Four features, all computed over the universal lookback period (except PC2 which is Kalman filtered):

1. **PC2 loading (Kalman filtered)** — structural cyclical/defensive character derived from rolling PCA, sign-aligned to XLK
2. **Relative strength vs SPY** — rolling average return of ETF minus SPY; captures genuine outperformance, replaces raw return as it contains it and adds market-adjustment
3. **Beta to SPY** — market sensitivity; orthogonal to performance direction
4. **Momentum consistency (hit rate)** — proportion of positive return days over the lookback; proxies volatility while adding directional consistency as a distinct layer

**Kalman Filter applied to PC2 only:**
- *Relative strength & hit rate:* already smooth by construction; filtering adds lag that blunts the momentum signal
- *Beta:* sufficiently stable over a long enough window; added complexity not justified
- *Hit rate:* discrete, bounded proportion — Kalman Filter assumes continuous Gaussian noise, statistically inappropriate here

## Lookback Period Selection

Prior to building the model, we empirically select a single universal lookback period for rolling average returns.

**Methodology:**
- Test a small set of economically motivated lookback periods (1, 3, 6, 12 months) on SPY and on the equal-weighted average return of all available sector ETFs
- For each lookback, measure correlation to today's return and assess statistical significance (p-value)
- Prefer the shortest lookback with the largest correlation and a significant p-value — following Chan's methodology
- If SPY and the sector ETF average agree → use that period with high confidence
- If they disagree → take the average of the best-performing lookback across individual ETFs weighted by correlation strength, and note the discrepancy in the document

**Why one universal lookback:**
ETF-specific lookback periods would make cross-sectional momentum rankings incomparable — ranking XLK on a 3-month signal against XLE on a 6-month signal uses different measuring sticks. Since the strategy depends on ranking ETFs against each other, a single lookback is a hard requirement for internal consistency.

**Note on ETF availability:**
XLC (from 2018) and XLRE (from 2019) have shorter histories. The lookback analysis will first be run on ETFs with full history from 2000, then verified on the full universe from 2019 onward to confirm the result holds.

## Core Model Architecture

The strategy separates three distinct responsibilities:

1. **Clustering** — group ETFs by behavioural similarity using multiple features (returns, volatility, PC2 loading, etc.). Answers: *who is in which regime right now?*
2. **Momentum signal** — rank ETF performance within or across clusters to determine position direction and sizing. Answers: *what do I buy and sell?*
3. **Kalman Filter** — smooth noisy input features (particularly PC2 loadings) before they enter the clustering model or signal generation. Answers: *what is the true underlying value of this feature?*

These three components are sequential and non-overlapping in responsibility. The Kalman Filter feeds into clustering; clustering feeds into the momentum signal; the momentum signal drives trades.

## Kalman Filter

A Kalman Filter is applied to the raw rolling PC2 loadings for each sector ETF independently.

**Purpose:** Raw PC2 loadings are noisy — sliding the window forward daily causes loadings to bounce due to estimation error, not genuine regime shifts. The Kalman Filter smooths this noise to produce a cleaner estimate of each sector's true underlying cyclical/defensive position over time.

**What it enables:** By tracking the filtered loading series, we can identify genuine crossovers — moments when a sector meaningfully drifts from defensive to cyclical character (or vice versa) — without being misled by day-to-day estimation noise. These crossovers are the primary signal driving the momentum strategy.
