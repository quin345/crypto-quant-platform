# Research Roadmap

| Field | Value |
|---|---|
| Author | Research Director |
| Date | 2026-11-19 |
| Status | Proposed — reviewed at every phase gate |
| Related | ADR-001 §2.3 (strategy sequencing), ADR-002 §2.9 (coverage), ADR-003, TOP_20_ALPHA_HYPOTHESES.md |

---

## 1. Research Objective

**Find at least one durable, capacity-meaningful, market-neutral edge and take it to Pilot with real capital, while building the evidence base and discipline that let us find the next ones.**

"Durable" means: an identifiable edge source (MARKET_STRUCTURE.md §1), positive in every major sub-period tested, robust to ×1.5 costs, and with a named counterparty whose behaviour we expect to persist.

We optimize for **quality of evidence per unit of infrastructure**, not number of strategies.

---

## 2. Guiding Bets

1. **Risk premia before speed.** Funding/carry first; latency-dependent families only after measurement says the opportunity exists for *us*.
2. **Mid-liquidity before majors.** Majors are most competed; long-tail is most manipulable. The middle is our hunting ground.
3. **Measure before building.** Cross-exchange arb and MM start as measurement studies on recorded data, not engineering projects.
4. **Negative results are deliverables.** A clean kill of a hypothesis saves capital and is reported with equal weight.
5. **Execution quality is alpha.** For carry strategies, cost control is a large share of the edge; execution research (H-017) is funded early.

---

## 3. Phases

Timing is indicative and depends on infrastructure delivery (ADR-001 to ADR-004). **Gates are evidence-based, not date-based.**

### Phase 1 — Foundations (now → indicative Q1 2027)

| Workstream | Deliverable | Owner |
|---|---|---|
| Market structure | MARKET_STRUCTURE.md verified against venue documentation; venue selection recommendation | Research Director |
| Hypothesis registry | Top 20 hypotheses converted into registry records with pre-registered criteria | Research Director + strategy teams |
| Cost model | Fee, slippage, funding, borrow, and transfer cost model spec per candidate venue, at our achievable tier | Research Director + Execution |
| Data requirements | Coverage spec for Phase 2a handed to Data Engineering (ADR-002 §2.9) | Research Director → Data Engineering |
| Simulation requirements | F0/F1 requirements spec handed to Backtesting | Research Director → Backtesting |
| Governance | RESEARCH_GOVERNANCE.md adopted; report templates written | Research Director |

**Exit gate:** venues selected; Tier A hypotheses fully registered; cost model and data coverage specs accepted by owners.

### Phase 2a — Funding Carry (indicative Q1–Q2 2027)

| Hypothesis | Goal |
|---|---|
| H-001 Mid-cap carry | Experiment of record → Validation |
| H-002 Funding forecastability | Experiment of record |
| H-005 Crowding risk overlay | Experiment of record |
| H-003 Cross-venue funding | Exploration → Experiment |
| H-010 Listing decay | Exploration |

Parallel: begin **self-capture** of top-of-book and trades on the selected venues (needed later for H-013/H-015 and for cost-model calibration). Capture starts early because recorded history cannot be bought retroactively.

**Exit gate (Phase 2a review):**
- **Go:** at least one of H-001/H-002/H-003 passes Validation → proceed to Paper, then Pilot.
- **Pivot:** all fail on costs but gross carry exists → focus on execution cost reduction (H-017, fee tier) before more alpha research.
- **Stop:** gross carry net of risk does not compensate venue/tail risk → reassess the platform's strategy mix with Systems Architect; do not proceed to Phase 3 infrastructure.

### Phase 2b — Statistical Arbitrage and Pilot (indicative Q2–Q3 2027)

| Hypothesis | Goal |
|---|---|
| Phase 2a winner(s) | Paper → Pilot with small capital |
| H-007 Carry factor | Experiment of record |
| H-008 Residual reversal | Experiment of record |
| H-006 Liquidation overshoot | Exploration → Experiment |
| H-009 Token unlocks | Exploration (data provenance first) |
| H-004 Dated basis | Exploration (if dated-futures data available) |
| H-012 Momentum | Benchmark only |

**Exit gate (Phase 2b review):**
- Pilot realized costs within tolerance of modeled costs (calibrates every future backtest).
- Live-vs-sim divergence measured and acceptable (ADR-003 §2.7).
- Review portfolio concentration: if all approved strategies are carry, decide whether to diversify deliberately.

### Phase 3 — Microstructure Measurement (indicative Q3–Q4 2027)

**Measurement, not trading.** Uses self-captured data from Phase 2a onward.

| Hypothesis | Goal |
|---|---|
| H-013 Lead-lag | Experiment on `ts_recv`-ordered data |
| H-015 Secondary-venue convergence | Opportunity count at our latency and fee tier |
| H-017 OFI | Predictive power by horizon |
| H-014 Stablecoin basis | Experiment |
| H-011 Settlement flow | Cheap exploration |

**Exit gate — latency-tier decision:**
- Build latency tier (ADR-004 §2.5) **only if** H-015 or H-016 shows net opportunity value that exceeds the estimated cost of building and operating the latency tier for 2 years, at our *current* fee tier.
- Otherwise: record result, keep capture running, and re-test annually or when fee tier improves.

### Phase 4 — Market Making Research (2028+, conditional)

Only entered if Phase 3 gate passes **or** a fee-tier/rebate arrangement materially changes the economics.

| Hypothesis | Goal |
|---|---|
| H-018 Large-tick queue value | Markout study (F3) |
| H-016 Hedged passive quoting | F3 experiment |
| H-019 Toxicity filter | Conditional on H-013/H-017 |
| H-020 Funding-aware skew | Overlay |

---

## 4. Research Capacity Allocation

| Category | Share of research time | Purpose |
|---|---|---|
| **Core** (current phase Tier A/B) | ~60% | Move the highest-scoring hypotheses through gates |
| **Infrastructure-of-evidence** (cost models, data validation, fill-model calibration) | ~25% | Raises the quality of every result |
| **Exploration** (new ideas, Tier C cheap tests) | ~15% | Keeps the pipeline fed; idea log maintenance |

---

## 5. Dependencies on Engineering

| Research need | Engineering deliverable | ADR | Needed by |
|---|---|---|---|
| Funding, mark/index, top-of-book, trades, 2 venues, ≥2y history | Data platform L0/L1, vendor backfill with provenance | ADR-002 | Phase 2a start |
| Reproducible experiments | Snapshot pinning, access library, experiment manifest | ADR-002, ADR-003 | Phase 2a experiments of record |
| F1 simulation | Event-driven engine with taker fill model | ADR-003 §2.6 | Phase 2a experiments of record |
| Paper / Pilot | Strategy Runtime, Risk, OMS, Adapters, Recorder (standard tier) | ADR-004 | Phase 2a → 2b |
| Self-captured L2 + synchronized clocks | Capture hosts, clock sync ADR | ADR-002 | Phase 3 |
| F2/F3 simulation | Book-aware and queue-aware engines | ADR-003 §2.6 | Phase 3 / 4 |

**Research Director commitment:** research will not request infrastructure beyond this table without a registered hypothesis that requires it.

---

## 6. Key Risks to the Roadmap

| Risk | Mitigation |
|---|---|
| Carry is fully competed after costs | Phase 2a "pivot" and "stop" outcomes defined in advance |
| Venue failure during Pilot | Small Pilot capital, sub-accounts, venue exposure limits (Risk) |
| History too short / regime-specific | Require multiple sub-periods; prefer mechanisms with clear economic rationale over pure statistics |
| Over-concentration in carry | Explicit portfolio review at Phase 2b gate |
| Data quality issues discovered late | Data validation is 25% of capacity, not an afterthought |
| Team bandwidth | Tier C deferred without guilt; fewer hypotheses tested properly beats many tested badly |

---

## 7. Success Metrics (12 months)

| Metric | Target |
|---|---|
| Hypotheses tested to a recorded conclusion | ≥ 8 |
| Experiments of record reproducible on audit | 100% |
| Strategies in Pilot | ≥ 1 |
| Pilot realized cost vs modeled cost | Within ±25% |
| Latency-tier go/no-go decision made on evidence | Yes |
| Strategies trading without a full artifact trail | 0 |
