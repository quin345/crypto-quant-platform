# Top 20 Alpha Hypotheses

| Field | Value |
|---|---|
| Author | Research Director |
| Date | 2026-11-19 |
| Status | Initial registry seed — all hypotheses at **Hypothesis** stage (ADR-003 §2.1) |
| Related | MARKET_STRUCTURE.md, RESEARCH_ROADMAP.md, RESEARCH_GOVERNANCE.md, ADR-003 |

---

## 0. How to Read This Document

These are **hypotheses, not strategies**. None has been tested on our data. Most will be rejected; that is the expected outcome. A healthy pipeline rejects the majority of what it tests.

Each hypothesis states:

- **Claim** — falsifiable statement.
- **Edge source** — risk premium / structural flow / segmentation / speed (MARKET_STRUCTURE.md §1).
- **Who pays** — the counterparty and why they keep paying.
- **Data** — minimum data required (ADR-002 coverage phase).
- **First test** — the cheapest test that could kill it.
- **Kill criteria** — pre-registered rejection conditions.
- **Fidelity** — minimum simulation tier for an experiment of record (ADR-003 §2.6).

Thresholds in kill criteria are **initial defaults** and will be finalized in each hypothesis's registry record before any experiment of record.

### Scoring

| Dimension | Scale | Meaning |
|---|---|---|
| **P** — Prior | 1–5 | Probability the edge exists *net of our costs* |
| **D** — Durability | 1–5 | Likely persistence over 2+ years |
| **F** — Feasibility | 1–5 | Our data/infrastructure can test and trade it in Phase 2–3 |
| **C** — Capacity | 1–5 | Capital it can absorb before decaying |
| **Score** | P × D × F × C (max 625) | Used to rank, not to decide |

---

## 1. Summary Ranking

| # | ID | Name | Family | Edge source | P | D | F | C | Score | Tier |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | H-2026-001 | Mid-cap perp funding carry with liquidity filter | Funding | Risk premium | 4 | 4 | 5 | 3 | 240 | A |
| 2 | H-2026-002 | Funding forecastability for entry/exit timing | Funding | Risk premium | 4 | 4 | 5 | 3 | 240 | A |
| 3 | H-2026-003 | Cross-venue funding differential | Funding | Segmentation | 4 | 3 | 4 | 3 | 144 | A |
| 4 | H-2026-004 | Dated-futures basis vs expected perp funding | Funding | Risk premium / segmentation | 3 | 4 | 3 | 4 | 144 | B |
| 5 | H-2026-005 | Funding-crowding as a risk-off conditioner | Funding (risk overlay) | Structural flow | 4 | 4 | 5 | 2 | 160 | A |
| 6 | H-2026-006 | Liquidation-cascade overshoot reversal | Stat arb | Structural flow | 3 | 4 | 4 | 2 | 96 | B |
| 7 | H-2026-007 | Cross-sectional carry factor across perps | Stat arb | Risk premium | 3 | 3 | 5 | 4 | 180 | A |
| 8 | H-2026-008 | Market-residual reversal in alt perps | Stat arb | Structural flow / liquidity provision | 3 | 3 | 4 | 3 | 108 | B |
| 9 | H-2026-009 | Token-unlock supply pressure | Stat arb (event) | Structural flow | 3 | 4 | 3 | 2 | 72 | B |
| 10 | H-2026-010 | New-listing funding and premium decay | Funding / stat arb | Structural flow | 3 | 3 | 4 | 2 | 72 | B |
| 11 | H-2026-011 | Funding-settlement positioning flow | Stat arb | Structural flow | 2 | 3 | 4 | 2 | 48 | C |
| 12 | H-2026-012 | Cross-sectional time-series momentum in perps | Stat arb | Behavioural / risk premium | 2 | 2 | 5 | 4 | 80 | C |
| 13 | H-2026-013 | Perp leads spot / major leads alts (minutes) | Stat arb | Speed / information | 3 | 2 | 4 | 2 | 48 | C |
| 14 | H-2026-014 | Stablecoin-quote basis | Cross-exchange | Risk premium / segmentation | 3 | 3 | 3 | 3 | 81 | B |
| 15 | H-2026-015 | Secondary-venue price convergence (taker) | Cross-exchange | Speed | 2 | 2 | 2 | 2 | 16 | C |
| 16 | H-2026-016 | Passive quoting on secondary venue, hedged on primary | Cross-exchange / MM | Liquidity provision + segmentation | 3 | 3 | 2 | 2 | 36 | C |
| 17 | H-2026-017 | Order-flow imbalance short-horizon predictability | MM (signal) | Information | 3 | 2 | 2 | 1 | 12 | C |
| 18 | H-2026-018 | Large-tick instrument queue value | MM | Structural (microstructure) | 3 | 3 | 2 | 2 | 36 | C |
| 19 | H-2026-019 | Toxicity-filtered quoting using cross-venue signals | MM | Liquidity provision | 2 | 3 | 1 | 2 | 12 | C |
| 20 | H-2026-020 | Funding-aware inventory skew for MM | MM | Risk premium | 3 | 3 | 1 | 2 | 18 | C |

Rows are ordered by family and dependency, not by score. Tier is a Research Director judgment informed by the score plus dependencies and roadmap fit (e.g. H-005 is Tier A because it is a cheap risk overlay on H-001).

**Tiers:** **A** = start in Phase 2a/2b. **B** = start when data and F1 simulation are in place, or after A results are in. **C** = deferred until its infrastructure prerequisite is justified by an A/B result, or kept as a cheap exploratory question.

**Portfolio note:** Tier A deliberately concentrates on the funding family. This is a conscious bet on the most durable edge source (risk premium) with the cheapest infrastructure. The diversification risk is accepted for Phase 2 and must be revisited at the Phase 2b review (RESEARCH_ROADMAP.md).

---

## 2. Funding Arbitrage Hypotheses

### H-2026-001 — Mid-cap perp funding carry with liquidity filter

- **Claim:** A delta-hedged portfolio (long spot or low-funding hedge, short perp) across liquid mid-cap perps earns funding, net of fees, slippage, borrow, and hedge rebalancing, with a Sharpe ratio above the same strategy in BTC/ETH alone and positive returns in each half-year sub-period.
- **Edge source:** Risk premium.
- **Who pays:** Leveraged long speculators, who pay funding for leverage. Large carry funds concentrate in majors because of capacity; mid-caps remain less competed.
- **Data:** Funding (realized + predicted), mark/index, top-of-book and trades for perp and spot, borrow rates; ≥ 2 years incl. one drawdown regime; delisted instruments included.
- **First test:** Gross funding minus a conservative cost model across the universe, by liquidity bucket. If gross carry in mid-caps is not meaningfully above majors after costs, stop.
- **Kill criteria:** Net annualized return after ×1.5 costs below the risk-free stablecoin yield available on the same venues; or more than 50% of PnL from a single 30-day window; or tail loss (squeeze/ADL scenario) exceeding 12 months of expected carry.
- **Key risks:** Short squeezes, ADL of the short leg, spot illiquidity on exit, venue default, depegging of collateral.
- **Fidelity:** F1.

### H-2026-002 — Funding forecastability for entry/exit timing

- **Claim:** Next-interval and next-day realized funding is predictable from the current premium index, predicted funding, open-interest change, and recent funding history, with enough accuracy that a timing rule (hold carry only when forecast cumulative funding over the holding period exceeds round-trip cost) beats always-on carry (H-001) after costs.
- **Edge source:** Risk premium (improves capture efficiency rather than a new premium).
- **Who pays:** Same as H-001; the edge is avoiding periods when we are not paid enough for the risk.
- **Data:** As H-001 plus open interest.
- **First test:** Out-of-sample forecast skill (rank IC and hit rate) of a simple persistence model vs a no-change baseline. Funding is known to be autocorrelated; the question is whether the *excess* over persistence pays for turnover.
- **Kill criteria:** Timing rule improves net Sharpe by less than 0.2 over always-on carry, or improvement not present in the holdout.
- **Fidelity:** F1.

### H-2026-003 — Cross-venue funding differential

- **Claim:** For the same underlying, funding rates on two venues diverge persistently enough that long perp on venue A / short perp on venue B earns the differential net of costs and collateral carry, without spot legs.
- **Edge source:** Segmentation — different user bases and collateral that cannot freely move.
- **Who pays:** Venue-segmented leveraged traders.
- **Data:** Funding for both venues with exact intervals and settlement times; mark prices; transfer times and costs between venues.
- **First test:** Distribution of funding differentials normalized to a common interval; fraction of time the differential exceeds round-trip cost; persistence of the sign.
- **Kill criteria:** Net return after collateral rebalancing costs below H-001's; or differentials are dominated by interval-mismatch artefacts (e.g. 1h vs 8h settlement timing) that vanish when properly aligned.
- **Key risks:** Mark-price divergence between venues triggering liquidation on one leg; collateral transfer delays during stress; venue-specific funding caps.
- **Fidelity:** F1.

### H-2026-004 — Dated-futures basis vs expected perp funding

- **Claim:** The annualized basis of dated futures systematically deviates from expected cumulative perp funding over the same horizon, and a spread position (dated future vs perp) captures the mispricing net of costs.
- **Edge source:** Risk premium (term premium) and segmentation (different user bases for dated futures).
- **Who pays:** Participants who prefer fixed-cost hedging (dated) vs floating (perp); TradFi-linked basis trades.
- **Data:** Dated futures prices and expiries, perp funding history, index.
- **First test:** Compare implied basis to realized cumulative funding over matched horizons historically.
- **Kill criteria:** Mean mispricing after costs not distinguishable from zero; or entirely explained by a single regime.
- **Fidelity:** F1. **Dependency:** dated-futures data (ADR-002 Phase 2b/3).

### H-2026-005 — Funding crowding as a risk-off conditioner

- **Claim:** Extreme positive funding combined with rapid open-interest growth predicts elevated probability of a sharp downside move and liquidation cascade over the following 1–7 days. Using this as a *risk conditioner* (reduce carry size, widen hedges) improves H-001's drawdown profile more than it costs in foregone carry.
- **Edge source:** Structural flow — crowded leverage must unwind.
- **Who pays:** Not a direct PnL source; it reduces our losses to the same forced flows.
- **Data:** Funding, OI, liquidation events, prices.
- **First test:** Event study of forward returns and realized volatility conditional on funding/OI percentiles, with honest count of independent events.
- **Kill criteria:** No statistically robust conditional difference after multiple-testing correction; or drawdown reduction smaller than carry foregone.
- **Note:** Directional trading on this signal is **out of scope** (platform is market-neutral first). It is evaluated only as a risk overlay.
- **Fidelity:** F1.

### H-2026-010 — New-listing funding and premium decay

- **Claim:** Newly listed perps exhibit elevated funding and perp-over-index premium in their first weeks that decays predictably; a hedged short-perp position over that window earns more than steady-state carry net of the higher costs and risks.
- **Edge source:** Structural flow — speculative demand around listings with limited initial short supply.
- **Who pays:** Early speculative longs.
- **Data:** Listing dates (Reference data), funding and prices from listing day, spot availability for hedging.
- **First test:** Event study over all listings (including later-delisted ones) of funding and premium path.
- **Kill criteria:** Edge concentrated in a handful of listings; or hedging the spot leg is infeasible (no spot market / no borrow) for most cases.
- **Key risks:** Manipulation, extreme squeezes, thin books.
- **Fidelity:** F1.

---

## 3. Statistical Arbitrage Hypotheses

### H-2026-006 — Liquidation-cascade overshoot reversal

- **Claim:** After clusters of forced liquidations, prices overshoot and partially revert over minutes to hours. A hedged (vs market factor) contrarian position entered after liquidation clusters earns positive returns net of costs.
- **Edge source:** Structural flow — liquidation engines sell regardless of price.
- **Who pays:** Liquidated traders (via the engine).
- **Data:** Liquidation events (where published), trades, top-of-book, OI change as a liquidation proxy where events aren't published.
- **First test:** Event study on cleaned liquidation clusters; reversal magnitude vs spread and fees.
- **Kill criteria:** Reversal smaller than round-trip cost at our fee tier; or reversal only appears when entry is at the cascade low (look-ahead).
- **Key risks:** Catching cascades that continue; publication of liquidation data is incomplete on some venues.
- **Fidelity:** F1 (F2 preferred for entry modelling during thin books).

### H-2026-007 — Cross-sectional carry factor across perps

- **Claim:** Ranking perps by funding (carry) and going short high-carry / long low-carry, beta-neutralized, earns a positive premium net of costs — i.e. high-funding perps underperform the funding they pay.
- **Edge source:** Risk premium (carry) plus behavioural over-leveraging.
- **Who pays:** Crowded longs in high-funding names.
- **Data:** Funding, prices, liquidity metrics across the full perp universe incl. delisted.
- **First test:** Quintile portfolio returns, beta-neutral, with realistic rebalancing frequency.
- **Kill criteria:** Net Sharpe < 0.5 in development; sign instability across sub-periods; returns explained by exposure to a single sector or to illiquidity.
- **Fidelity:** F1.

### H-2026-008 — Market-residual reversal in alt perps

- **Claim:** Idiosyncratic returns of alt perps (residual after BTC/ETH factor exposure) mean-revert at horizons of hours to a day, sufficiently to be traded net of costs in a diversified, beta-neutral book.
- **Edge source:** Liquidity provision against uninformed idiosyncratic flow.
- **Who pays:** Liquidity-demanding traders in alts.
- **Data:** Bars at 1m–1h, liquidity metrics.
- **First test:** Residual autocorrelation by horizon and liquidity bucket; turnover-adjusted returns.
- **Kill criteria:** Edge disappears in the top-liquidity half of the universe (i.e. only exists where costs are highest); or net Sharpe < 0.7 after ×1.5 costs.
- **Fidelity:** F1.

### H-2026-009 — Token-unlock supply pressure

- **Claim:** Tokens with large scheduled unlocks (relative to circulating supply and daily volume) exhibit negative abnormal returns around the unlock window, capturable via a hedged short perp.
- **Edge source:** Structural flow — predictable supply from recipients who sell.
- **Who pays:** Holders who don't hedge; recipients selling into the market.
- **Data:** Unlock schedules (third-party; provenance must be documented — ADR-002 vendor policy), prices, funding (short perps may pay funding).
- **First test:** Event study by unlock size bucket.
- **Kill criteria:** Effect not robust to excluding the largest 10% of events; or funding/borrow costs exceed the abnormal return; or effect fully priced in before the earliest point we can enter.
- **Key risks:** Small sample; data quality of unlock schedules; short squeezes.
- **Fidelity:** F1.

### H-2026-011 — Funding-settlement positioning flow

- **Claim:** Traders reduce positions just before funding settlement to avoid paying, and re-establish after, creating a predictable intraday price pattern around settlement timestamps on high-funding instruments.
- **Edge source:** Structural flow.
- **Who pays:** Funding-avoiding traders.
- **Data:** Trades and top-of-book around settlement; funding.
- **First test:** Average returns and order-flow imbalance in windows around settlement, conditional on funding level.
- **Kill criteria:** Pattern magnitude below spread + fees; pattern absent on venues with continuous or short-interval funding.
- **Fidelity:** F1. **Prior is low** — likely competed; kept because the test is cheap.

### H-2026-012 — Time-series and cross-sectional momentum in perps

- **Claim:** Trend and cross-sectional momentum signals on perps earn positive risk-adjusted returns net of costs and funding.
- **Edge source:** Behavioural (under-reaction / herding) and crash-risk premium.
- **Who pays:** Late or slow-reacting participants.
- **Data:** Daily/hourly bars, funding.
- **First test:** Standard specifications only (no parameter search) on development data.
- **Kill criteria:** Results depend on parameter choice; or performance concentrated in one bull phase; or net of funding the edge disappears (momentum longs often pay funding).
- **Note:** Not market-neutral by construction; included as a **benchmark** and possible portfolio component, with a strong prior that it is crowded and regime-dependent. Strict trial accounting applies.
- **Fidelity:** F1.

### H-2026-013 — Perp leads spot / majors lead alts (seconds to minutes)

- **Claim:** Price changes in the most liquid perps lead spot and alt perps at horizons of seconds to minutes; a trading rule on the lagging instrument earns positive returns net of taker costs at our fee tier.
- **Edge source:** Speed / information.
- **Who pays:** Slower participants on the lagging instrument.
- **Data:** Trades and top-of-book with reliable `ts_recv` (ADR-002 §2.3) on both venues.
- **First test:** Cross-correlation of returns by lag using receive time; profitability of the simplest taker rule at our fee tier.
- **Kill criteria:** Lead-lag exists but is not profitable at our latency and fee tier (most likely outcome); or effect disappears when ordering by `ts_recv` instead of `ts_exchange`.
- **Value even if killed:** The lag structure is an input to H-016, H-017, H-019 and to execution quality.
- **Fidelity:** F2.

---

## 4. Cross-Exchange Arbitrage Hypotheses

### H-2026-014 — Stablecoin-quote basis

- **Claim:** Price differences for the same asset quoted in different stablecoins/fiat (e.g. X/USDT vs X/USDC vs X/USD) are partly predictable from stablecoin premia and flows, and a hedged position earns above the risk-free rate net of costs and conversion frictions.
- **Edge source:** Risk premium (stablecoin risk) and segmentation (fiat-rail access).
- **Who pays:** Participants restricted to one quote currency.
- **Data:** Multi-quote spot books, stablecoin pair prices, conversion costs.
- **First test:** Decompose cross-quote spread into the stablecoin rate and residual; test residual mean-reversion.
- **Kill criteria:** Residual smaller than conversion costs; returns are just compensation for stablecoin depeg risk at an unattractive ratio.
- **Fidelity:** F1–F2.

### H-2026-015 — Secondary-venue price convergence (taker-taker)

- **Claim:** Prices on secondary venues converge to the primary venue with a measurable lag, and taker-taker arbitrage is profitable after fees at our achievable tier.
- **Edge source:** Speed.
- **Who pays:** Stale quotes on secondary venues.
- **Data:** L2 deltas on both venues with synchronized clocks.
- **First test:** **Measurement only:** count opportunities exceeding (fees + half-spread × 2 + latency buffer) using our capture latency. No strategy development unless counts are material.
- **Kill criteria:** Opportunities net of costs insufficient to cover infrastructure costs of the latency tier.
- **Note:** Strong prior that this is competed away. It exists in the list as the **gate for whether the latency tier is ever built** (ADR-001 §2.3).
- **Fidelity:** F2.

### H-2026-016 — Passive quoting on secondary venue, hedged on primary

- **Claim:** Posting passive quotes on a less liquid venue and hedging fills on the liquid venue earns the secondary venue's wider spread minus hedge cost and adverse selection.
- **Edge source:** Liquidity provision + segmentation.
- **Who pays:** Takers on the secondary venue.
- **Data:** L2 on both venues; our own fill data (Pilot) for calibration.
- **First test:** Simulated spread capture minus hedge cost using F2, with conservative fill assumptions; sensitivity to latency.
- **Kill criteria:** Adverse selection on passive fills exceeds spread captured; hedge slippage dominant.
- **Fidelity:** F2 minimum, F3 for experiment of record.

---

## 5. Market Making Hypotheses

### H-2026-017 — Order-flow imbalance short-horizon predictability

- **Claim:** Order-flow imbalance at the top of book predicts the next mid-price move at sub-second to second horizons on our target instruments.
- **Edge source:** Information.
- **Who pays:** Not standalone PnL; it is a quoting and execution input.
- **Data:** L2 deltas, trades.
- **First test:** Predictive R²/IC of OFI on next mid change by horizon, out-of-sample.
- **Kill criteria:** Predictive power decays below a useful horizon given our latency.
- **Value:** Improves execution for every family (smarter order timing), even if MM is never built.
- **Fidelity:** F2.

### H-2026-018 — Large-tick instrument queue value

- **Claim:** On instruments where tick size is large relative to typical spread, early queue position earns positive expected value net of adverse selection, accessible at non-VIP fee tiers.
- **Edge source:** Microstructure structure.
- **Who pays:** Takers crossing a one-tick spread.
- **Data:** L2/L3-equivalent data, queue estimation, our own order lifecycle.
- **First test:** Identify large-tick instruments; estimate fill probability and post-fill markout by queue position from recorded data.
- **Kill criteria:** Expected markout after maker fees negative at our tier; or queue access dominated by incumbents (we never get early queue position).
- **Fidelity:** F3.

### H-2026-019 — Toxicity-filtered quoting using cross-venue signals

- **Claim:** Pulling or skewing quotes when cross-venue signals (H-013) or OFI (H-017) indicate informed flow reduces adverse selection enough to make quoting profitable on mid-liquidity instruments.
- **Edge source:** Liquidity provision with information advantage.
- **Who pays:** Uninformed takers.
- **Data:** Multi-venue L2, own fills.
- **First test:** Markout analysis of simulated fills, with and without the filter.
- **Kill criteria:** Filter benefit smaller than missed-fill cost; our latency too slow for filter to act in time.
- **Dependency:** Positive results from H-013 or H-017.
- **Fidelity:** F3.

### H-2026-020 — Funding-aware inventory skew for MM

- **Claim:** Skewing MM inventory toward the side that receives funding adds a carry component that improves net MM PnL without materially increasing inventory risk.
- **Edge source:** Risk premium.
- **Who pays:** Leveraged traders paying funding.
- **Data:** As H-018 plus funding.
- **First test:** Simulation overlay on any MM strategy that has passed F3.
- **Kill criteria:** Inventory risk increase exceeds funding earned.
- **Dependency:** A working MM strategy.
- **Fidelity:** F3.

---

## 6. Dependencies Between Hypotheses

```
H-001 ──▶ H-002 (timing)        H-005 (risk overlay on H-001/H-003)
  │
  ├──▶ H-003 (cross-venue)  ──▶ H-020 (carry in MM)
  └──▶ H-010 (listings)

H-013 (lead-lag) ──▶ H-015 (latency-tier gate)
       │
       └──▶ H-016, H-019        H-017 (OFI) ──▶ H-019
H-018 (queue value) ──▶ H-019 ──▶ H-020
```

---

## 7. Hypotheses Considered and Excluded

| Idea | Reason excluded |
|---|---|
| Triangular arbitrage on a single venue | Competed to near-zero; pure latency |
| On-chain/DEX arbitrage, MEV | Non-goal (ADR-001 §2.5) |
| Options volatility risk premium | Non-goal (ADR-001 §2.5); revisit with separate ADR |
| Sentiment/social-media signals | Data provenance poor, high overfitting risk; revisit only with a clear economic mechanism |
| ML "find any pattern" over features | Not a hypothesis; violates pre-registration. ML is a method, admissible only to test a stated claim |
| Weekend/time-of-day seasonality | Low prior, high multiple-testing risk; may be tested as a *conditioner* inside other hypotheses |
