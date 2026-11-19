# ADR-003: Research Workflow

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-19 |
| Decider | Systems Architect (workflow architecture); Research Director (gate criteria and approvals) |
| Consulted | Statistical Arbitrage, Market Making, Backtesting, Risk, Data Engineering |
| Supersedes | — |
| Related | ADR-001, ADR-002, ADR-004 |

---

## 1. Context

ADR-001 defines the platform as a research factory whose primary question, per AGENTS.md, is: **"Is this edge real?"**

In systematic trading, the dominant failure mode is not bad infrastructure. It is **false discovery**: a strategy that looks profitable in research because of overfitting, look-ahead bias, unrealistic fills, ignored costs, or selective reporting, and that loses money live. Crypto amplifies this:

- Short histories and frequent regime changes (bull/bear cycles, venue collapses, regulatory events).
- Survivorship bias from delisted tokens and failed venues.
- Costs (fees, funding, borrow, slippage) that are large relative to typical edge.
- Many researchers testing many variants on the same small dataset.

The research workflow is therefore an **error-control system**, not a productivity tool. Its job is to make false discoveries expensive and real discoveries traceable.

### Assumptions challenged

- **"A good Sharpe ratio is evidence."** Not without knowing how many trials produced it. Ten thousand parameter combinations will produce an impressive Sharpe on noise.
- **"Notebooks are fine for research."** For exploration, yes. As the record of an approved result, no: hidden state, out-of-order execution, and unpinned data make them non-reproducible.
- **"Walk-forward validation solves overfitting."** It reduces it. Reusing the same walk-forward windows across many iterations re-introduces overfitting at the meta level. A true holdout must be consumed sparingly.
- **"Paper trading proves the fill model."** Paper trading cannot observe market impact, queue position, or adverse selection of our own orders. Only small live trading can.
- **"Process slows research."** Lightweight process slows *bad* research. The workflow must be cheap at early stages and strict only at gates where capital is committed.

---

## 2. Decision

### 2.1 Research lifecycle

Every piece of alpha research moves through explicit stages. Each stage has a required artifact and a gate.

```
 Idea ─▶ Hypothesis ─▶ Exploration ─▶ Experiment ─▶ Validation ─▶ Paper ─▶ Pilot ─▶ Production ─▶ Retired
          (registered)                 (of record)   (holdout)               (small live)  (scaled)
   │           │              │             │             │           │         │           │
   └───────────┴──────────────┴─────────────┴─────────────┴───────────┴─────────┴───────────┴──▶ Rejected
                                         (any stage may reject; rejections are recorded)
```

| Stage | Purpose | Required artifact | Gate / approver |
|---|---|---|---|
| **Idea** | Capture a thought cheaply | One paragraph in the idea log | None |
| **Hypothesis** | State a falsifiable claim with an economic rationale | Hypothesis record (Section 2.2) | Research Director: is it worth data/compute? Is the data available (ADR-002)? |
| **Exploration** | Descriptive analysis; check the effect exists at all | Exploration notes (notebooks allowed) | Researcher self-assessment; logs trial count |
| **Experiment** | Pre-registered, reproducible test of the hypothesis on development data | Experiment Report (Section 2.4) | Research Director review |
| **Validation** | Single test on untouched holdout data + robustness battery | Validation section of Experiment Report | Research Director approval; Risk consulted on capacity/tail risk |
| **Paper** | Run in production runtime against live data with simulated fills | Paper-trading report: live-vs-sim divergence | Backtesting + Execution sign off on parity |
| **Pilot** | Small live capital to measure real fills, slippage, and impact | Pilot report: realized vs expected costs and PnL | Research Director + Risk (limits) |
| **Production** | Scaled capital within risk limits | Ongoing monitoring dashboard | Risk sets limits; Research Director owns continuation |
| **Retired** | Edge decayed or invalidated | Retirement note with cause | Research Director |

**Rejections are first-class results.** A rejected hypothesis is recorded with its evidence so that it is not rediscovered and re-tested without accounting for the prior trials.

### 2.2 Hypothesis registry

A single registry (initially version-controlled Markdown/YAML in this repository; tooling later if needed) holds one record per hypothesis:

| Field | Description |
|---|---|
| ID | `H-YYYY-NNN` |
| Family | Funding arb / stat arb / cross-exchange arb / MM |
| Claim | Falsifiable statement (e.g. "Predicted funding on venue A leads realized funding on venue B by ≥1 interval with IC > 0.05") |
| Economic rationale | *Why* the edge should exist and who is on the other side |
| Required data | Datasets, venues, granularity, history length (ties to ADR-002 coverage) |
| Success criteria | Metrics and thresholds defined **before** the experiment |
| Kill criteria | What result rejects the hypothesis |
| Cost model | Fee tier, slippage model, funding/borrow assumptions |
| Owner | One researcher / team |
| Status | Lifecycle stage |
| Trial log | Count and description of variants tested |

Hypotheses without an economic rationale are allowed at the Exploration stage but cannot pass the Experiment gate. "The data shows it" is not a rationale.

### 2.3 Reproducibility contract

An **experiment of record** is reproducible if and only if it pins:

| Element | Mechanism |
|---|---|
| Data | Iceberg snapshot IDs or tags for every table read (ADR-002 §2.7); tagged to prevent expiry |
| Code | Git commit SHA of a clean working tree |
| Environment | Locked dependency file (e.g. `uv.lock`) and container image digest |
| Configuration | Declarative experiment config (parameters, universe, date ranges, cost model, fill model, validity-mask policy) committed alongside results |
| Randomness | Explicit seeds for all stochastic components |
| Outputs | Results written to an experiment store keyed by experiment ID, with the manifest of all the above |

Rules:

- Experiments of record are executed as **versioned pipelines** (scripts/modules invoked with a config), not notebooks. Notebooks may *read* experiment outputs for presentation.
- Re-running an experiment of record from its manifest must reproduce headline metrics exactly (deterministic components) or within a stated tolerance (stochastic components).
- Experiment tracking tooling is deliberately left open (candidates: MLflow, a lightweight manifest table in Iceberg). The manifest content above is decided; the tool is not.

### 2.4 Experiment Report — required content

1. Hypothesis ID and pre-registered success/kill criteria.
2. Data: snapshot IDs, universe, date ranges, validity-mask policy, known data issues.
3. Method: signal construction, portfolio/position logic, execution assumptions.
4. Cost model: fees (tier stated), slippage, funding, borrow, transfer/withdrawal costs where relevant.
5. Fill model and its tier (Section 2.6).
6. Results on development data, with **trial count** and multiple-testing-adjusted statistics.
7. Robustness: parameter sensitivity, sub-period stability, venue and instrument breakdown, cost sensitivity (e.g. ×1.5 and ×2 fees).
8. Capacity estimate and its assumptions.
9. Failure modes and tail scenarios (venue outage, depeg, liquidation cascade, funding spike).
10. Conclusion against the pre-registered criteria — including if the conclusion is "rejected".

### 2.5 Statistical standards

| Standard | Rule |
|---|---|
| Data split | Development / validation (walk-forward within development) / **holdout**. Holdout is the most recent contiguous period (proposal: ≥ 6 months for daily-horizon strategies) plus at least one distinct regime where available. |
| Holdout discipline | Holdout is evaluated **once** per hypothesis version. A second look requires a new hypothesis version and is recorded as such. |
| Leakage control | Purged and embargoed cross-validation for overlapping labels; point-in-time universe and reference data (bitemporal, ADR-002); event ordering by `ts_recv`. |
| Multiple testing | Report trial count; use deflated Sharpe ratio or an equivalent correction; report the distribution of results across trials, not only the best. |
| Minimum evidence | Thresholds per family are set by the Research Director in the registry; defaults: statistically significant after correction, positive after ×1.5 costs, and stable sign across sub-periods. |
| Survivorship | Universe includes delisted instruments and failed venues for the periods they existed. |
| Benchmarks | Every result is reported against a naive baseline (e.g. unconditional carry for funding arb). |

### 2.6 Simulation fidelity tiers

Fill realism must match the strategy's latency sensitivity (ADR-001 §1). Using a low-fidelity model for a high-sensitivity strategy is grounds for rejection.

| Tier | Model | Acceptable for |
|---|---|---|
| **F0 — Vectorized** | Bar-based, fill at next bar with cost haircut | Exploration only, any family |
| **F1 — Event-driven, taker** | Event-driven replay on top-of-book/trades; taker fills with spread + slippage model; latency injected | Funding arb, stat arb experiments of record |
| **F2 — Event-driven, book-aware** | Replay on L2 deltas; depth-consuming fills; per-venue latency distributions; leg-risk modeling | Cross-exchange arb |
| **F3 — Queue-aware** | L2/L3 replay with queue-position estimation, own-order impact, adverse-selection measurement; calibrated against our own fills | Market making |

- F1+ simulation runs **the same strategy code and event model** as production (ADR-001 principle 1; ADR-004 Strategy API). Divergence between simulated and live behavior during Paper and Pilot is a tracked metric.
- Fill models are calibrated with production fill data as it accumulates (ADR-001 principle 7). Until then, fill-model assumptions are stated as assumptions.

### 2.7 Live monitoring and decay

From Paper onward, each strategy has:

- **Expected vs realized** dashboards: PnL, turnover, costs, slippage, fill rates against the Experiment Report's projections.
- **Divergence alerts**: live-vs-replay divergence (replay the live session through simulation and compare decisions, ADR-001 principle 3).
- **Decay criteria** set at Pilot approval: e.g. rolling realized Sharpe below a threshold for N periods, or slippage exceeding the modeled value by X%, triggers review and possible reduction to Pilot or Retirement.

### 2.8 Roles

| Role | Responsibility in workflow |
|---|---|
| Research Director | Owns registry, gate criteria, approvals at Hypothesis/Experiment/Validation/Pilot; rejects weak research |
| Strategy teams (Stat Arb, MM) | Own hypotheses, experiments, and reports in their family. Funding and cross-exchange arb ownership: see Open Questions |
| Backtesting | Owns the simulation engine, fill-model tiers, and live-vs-sim parity measurement |
| Data Engineering | Owns data availability, snapshot tagging, validity masks |
| Risk | Consulted at Validation; sets limits at Pilot and Production |
| Execution | Signs off parity at Paper; provides TCA at Pilot/Production |
| Systems Architect | Owns the workflow architecture (this ADR), reproducibility tooling standards |

---

## 3. Alternatives Considered

| Alternative | Why rejected |
|---|---|
| **Notebook-centric workflow** | Fast but irreproducible; hidden state; no trial accounting. Retained for Exploration only. |
| **Full MLOps platform up front** (feature store, model registry, orchestration) | Premature for Phase 1–2 and violates "never build infrastructure without a clear research objective". Add components when a registered hypothesis needs them. |
| **No holdout; rely on walk-forward only** | Meta-overfitting through repeated iteration on the same windows. |
| **Paper trading as the final gate** | Cannot measure impact, queue position, or real adverse selection. Pilot with small capital is required. |
| **Single fidelity simulator for all strategies** | Either too slow for research iteration (if high fidelity) or misleading for latency-sensitive strategies (if low fidelity). |
| **Heavyweight approval at every stage** | Discourages exploration and drives research off-platform. Gates are lightweight early and strict only where capital is committed. |

---

## 4. Tradeoffs

| We gain | We pay |
|---|---|
| Lower false-discovery rate | Slower path from idea to live capital |
| Traceability of every approved strategy | Discipline overhead: registry entries, manifests, reports |
| Honest statistics via trial accounting and holdout discipline | Fewer strategies pass; some real but marginal edges will be rejected |
| Fidelity matched to strategy | Must build and maintain multiple fill-model tiers |
| Recorded rejections prevent re-testing the same idea | Researchers must record failures, which is culturally uncomfortable |

---

## 5. Consequences

- Repository gains `research/registry/` (hypotheses), `research/experiments/` (configs and reports), and report templates — structure to be specified in a design spec, not code, in Phase 1.
- Backtesting must deliver F1 before any funding/stat-arb experiment of record can pass the Experiment gate.
- ADR-004 must expose a single Strategy API shared by simulation and production runtime.
- ADR-002 snapshot tagging becomes a dependency of every experiment of record.
- Follow-up artifacts: Research Report template, Experiment Report template, Hypothesis record template, and an ADR for experiment-tracking tooling.

---

## 6. Development Rule Compliance

| Rule | Answer |
|---|---|
| Objective | Make false discoveries expensive and real discoveries traceable from idea to production. |
| Owner | Research Director (gates, criteria, registry); Systems Architect (workflow architecture and reproducibility standard). |
| Acceptance criteria | (1) Every strategy in Paper or later has a registry entry, an Experiment Report, and a reproducible manifest. (2) A randomly chosen experiment of record reproduces from its manifest. (3) Every Experiment Report states trial count and adjusted statistics. (4) Holdout evaluations are logged and limited to one per hypothesis version. (5) Live strategies have expected-vs-realized monitoring and decay criteria. |
| Failure modes | Process bypassed ("just a quick test in prod"); trial counts under-reported; holdout reused informally; fill model too optimistic for the strategy's tier; snapshot expired before reproduction; registry abandoned because it is too heavy; live-vs-sim divergence ignored. |
| Monitoring | Registry metrics (hypotheses by stage, rejection rate, time per stage); quarterly reproducibility audit of a sample of experiments; live-vs-sim divergence per strategy; decay-criteria breaches; count of strategies in production without a complete artifact trail (target: zero). |

---

## 7. Open Questions

1. AGENTS.md has no owner for **funding arbitrage** or **cross-exchange arbitrage** research. Proposal: assign funding arb to Statistical Arbitrage (shared carry/basis modeling skills) and cross-exchange arb to Market Making (shared microstructure skills), or create dedicated teams. Research Director to decide.
2. Holdout length and regime requirements per strategy family.
3. Experiment-tracking tool choice (MLflow vs Iceberg manifest table vs other).
4. Pilot capital sizing policy — joint Risk / Research Director decision.
5. Should research be peer reviewed by a second researcher before the Experiment gate, in addition to the Research Director?
