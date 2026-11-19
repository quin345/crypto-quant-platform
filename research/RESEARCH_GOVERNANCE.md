# Research Governance

| Field | Value |
|---|---|
| Author | Research Director |
| Date | 2026-11-19 |
| Status | Proposed |
| Related | AGENTS.md (Governance), ADR-003 (Research Workflow), RESEARCH_ROADMAP.md |

---

## 0. Purpose and Scope

ADR-003 defines the research **workflow architecture**: lifecycle stages, reproducibility contract, statistical standards, and fidelity tiers. This document defines **how the Research Director exercises authority** within that workflow: decision rights, gate thresholds, review practice, and the rules that keep research honest.

Where this document and ADR-003 conflict, ADR-003 governs until amended by a superseding ADR.

**The governing question for every decision: "Is this edge real, and will it still be real when our capital is in it?"**

---

## 1. Decision Rights

| Decision | Decider | Must consult | Recorded in |
|---|---|---|---|
| Register a hypothesis | Research Director | Data Engineering (data availability) | Registry |
| Approve experiment of record design (pre-registration) | Research Director | Backtesting (fidelity tier) | Registry |
| Pass Experiment gate | Research Director | Peer reviewer | Experiment Report |
| Authorize holdout evaluation | Research Director | — | Holdout log |
| Pass Validation gate | Research Director | Risk | Experiment Report |
| Start Paper trading | Research Director | Execution, Backtesting | Registry |
| Start Pilot, set Pilot capital | Research Director **and** Risk (joint) | Execution | Pilot approval record |
| Scale to Production | Research Director **and** Risk (joint) | Systems Architect (capacity of infra) | Production approval record |
| Reduce / retire a strategy | Research Director; **Risk may reduce or halt unilaterally** | Strategy owner | Retirement note |
| Change gate thresholds | Research Director | Systems Architect if ADR-003 is affected | This document (versioned) |
| Request new data or infrastructure | Research Director | Systems Architect | Registry + ADR if architectural |

Risk's authority to halt is never subject to research approval. Research's authority to approve is never sufficient alone to commit capital.

---

## 2. Research Ownership

Resolving ADR-003 Open Question 1:

| Family | Owning team | Rationale |
|---|---|---|
| Funding arbitrage | **Statistical Arbitrage** (interim) | Shared carry, basis, and portfolio modeling skills |
| Statistical arbitrage | Statistical Arbitrage | AGENTS.md |
| Cross-exchange arbitrage | **Market Making** (interim) | Shared microstructure, latency, and book-modeling skills |
| Market making | Market Making | AGENTS.md |
| Execution-cost research (e.g. H-017) | Execution, with Research Director review | Owns TCA and order placement |

Interim assignments are reviewed at the Phase 2b gate. If funding arbitrage becomes the main capital user, a dedicated funding team should be created.

Each hypothesis has **exactly one** named owner.

---

## 3. Gate Thresholds (Defaults)

These are defaults. A hypothesis may pre-register stricter thresholds; it may not pre-register looser ones without Research Director approval recorded *before* the experiment.

### 3.1 Experiment gate (development data)

| Criterion | Default threshold |
|---|---|
| Economic rationale | Written; names who pays and why they persist |
| Pre-registration | Success/kill criteria recorded before the experiment ran |
| Net performance | Positive after full cost model at our achievable fee tier |
| Cost robustness | Positive after ×1.5 costs |
| Statistical significance | Deflated Sharpe ratio (or equivalent) significant at 95% given recorded trial count |
| Sub-period stability | Positive in ≥ 75% of non-overlapping sub-periods (period length pre-registered) |
| Concentration | No single instrument > 25% of PnL; no single 30-day window > 30% of PnL |
| Benchmark | Beats the naive baseline for the family |
| Reproducibility | Manifest complete; re-run reproduces headline metrics |

### 3.2 Validation gate (holdout)

| Criterion | Default threshold |
|---|---|
| Holdout performance | Sign matches development; Sharpe ≥ 50% of development Sharpe |
| Robustness battery | Parameter sensitivity (neighbouring parameters also profitable), venue split, liquidity split |
| Tail scenarios | Loss in stated stress scenarios (venue outage, depeg, squeeze/ADL, liquidation cascade) quantified and accepted by Risk |
| Capacity | Capacity estimate with method stated; Pilot size well inside it |

A sharp drop from development to holdout is **evidence of overfitting**, not bad luck, unless proven otherwise.

### 3.3 Paper and Pilot gates

| Gate | Criterion |
|---|---|
| Paper → Pilot | Live-vs-sim decision divergence within tolerance; no unexplained behaviour; minimum duration pre-registered (default: 4 weeks or 100 trades, whichever later) |
| Pilot → Production | Realized costs within ±25% of modeled; realized PnL within the pre-registered confidence band of expectation; minimum duration (default: 3 months) |

### 3.4 Family-specific additions

| Family | Additional requirement |
|---|---|
| Funding arb | Funding modeled with exact venue settlement times and caps; ADL and liquidation of each leg simulated; collateral and transfer costs included |
| Stat arb | Point-in-time universe including delisted instruments; beta and sector exposure reported |
| Cross-exchange arb | Clock-sync quality reported; opportunities measured on `ts_recv`; leg-risk simulated; transfer friction modeled |
| Market making | F3 fidelity; markout analysis at multiple horizons; fee tier sensitivity; queue-position assumptions validated against own fills |

---

## 4. Rules for Honest Research

### 4.1 Trial accounting

- Every variant run on development data counts as a trial: parameter changes, universe changes, feature changes, date-range changes.
- Trials are logged in the registry automatically where tooling permits, otherwise manually. **Under-reporting trials is a governance violation**, treated more seriously than a failed hypothesis.
- Exploration on shared datasets still consumes trials for the hypothesis it informs.

### 4.2 Holdout custody

- Holdout period boundaries are set by the Research Director per family and recorded in the registry.
- Researchers do not read holdout data before the Validation gate. Data Engineering provides holdout access only via a tagged snapshot released on Research Director authorization (ADR-002 §2.7).
- One holdout evaluation per hypothesis version. Results of every holdout evaluation are recorded, pass or fail.
- Holdouts age: once a holdout period has been used by many hypotheses, it is retired and a newer period becomes the holdout.

### 4.3 Peer review

- Every Experiment Report is reviewed by one researcher not on the owning team before the Experiment gate.
- Reviewer uses the red-flag checklist (§5) and signs off in the report.

### 4.4 Separation of discovery and approval

- The Research Director does not approve their own hypotheses. A Research Director-owned hypothesis is reviewed by a senior researcher and Risk.

### 4.5 Recording failure

- Every hypothesis reaches a recorded conclusion: Passed / Rejected / Abandoned (with reason).
- Rejected hypotheses may be reopened only as a new version with a stated reason (new data, new mechanism) and inherit their prior trial count.

---

## 5. Red-Flag Checklist (for Reviewers)

Any "yes" requires an explanation in the report before the gate can pass.

| # | Red flag |
|---|---|
| 1 | Uses `ts_exchange` (or bar close) for event ordering where `ts_recv` should be used |
| 2 | Universe defined using information from after the start of the test (survivorship / look-ahead) |
| 3 | Fills at mid, at last trade, or at the bar close without slippage |
| 4 | Fee tier better than what we currently have or contractually expect |
| 5 | Ignores funding on perp legs, or borrow costs on spot shorts |
| 6 | Trades through intervals marked invalid in the data validity mask |
| 7 | Performance driven by a small number of days, instruments, or events |
| 8 | Sharpe > 3 for a non-HFT strategy (usually a bug) |
| 9 | Parameters at the edge of the search range, or sharp performance cliff nearby |
| 10 | Results not reproducible from the manifest |
| 11 | No economic explanation of who pays |
| 12 | Trial count not stated, or implausibly low for the complexity of the final model |
| 13 | Capacity not estimated, or estimated without reference to depth |
| 14 | Tail scenarios (venue failure, depeg, ADL) not modeled for strategies holding positions across venues |
| 15 | Fidelity tier below the minimum for the family (ADR-003 §2.6) |

---

## 6. Durability Assessment

Every hypothesis passing the Experiment gate receives a durability assessment, reviewed annually while live:

| Question | Evidence expected |
|---|---|
| Who pays, and why? | Named participant type and motive (MARKET_STRUCTURE.md §2) |
| Why haven't others removed it? | Capacity, operational difficulty, risk others won't take, segmentation |
| What would kill it? | Specific market changes (rule change, new product, competitor entry, fee changes) |
| Is it decaying? | Trend in edge over time in the historical data |
| Is it crowded? | Correlation with known public factors or strategies |
| What is our advantage? | Why *we* can capture it (cost, risk management, niche) |

An edge with no answer to "why haven't others removed it?" is presumed to be a data or modeling error until proven otherwise.

---

## 7. Live Strategy Oversight

| Mechanism | Frequency | Owner |
|---|---|---|
| Expected-vs-realized review | Weekly in Pilot, monthly in Production | Strategy owner → Research Director |
| Decay-criteria check | Continuous (automated, ADR-003 §2.7) | Strategy owner; Risk alerted |
| Durability re-assessment | Annually, or on market-structure change | Research Director |
| Post-incident research review | After any loss event exceeding Risk thresholds | Research Director + Risk |

**Decay actions** (pre-registered at Pilot approval):

1. Breach of first decay threshold → review within one week; may continue at reduced size.
2. Breach of second threshold → reduce to Pilot size; root-cause analysis required.
3. Breach of kill threshold → retire; retirement note required.

---

## 8. Review Cadence

| Forum | Frequency | Purpose |
|---|---|---|
| Research review | Weekly | Progress on active hypotheses; new ideas into the log |
| Gate reviews | As needed | Formal gate decisions with written record |
| Portfolio review | Monthly | Strategy mix, concentration, correlation, capacity |
| Roadmap review | At each phase gate (RESEARCH_ROADMAP.md) | Reprioritize hypotheses; go/pivot/stop decisions |
| Governance audit | Quarterly | Reproducibility sample, trial-accounting audit, holdout log check |

---

## 9. Required Artifacts (Templates to Produce)

| Artifact | Stage | Template status |
|---|---|---|
| Idea log entry | Idea | To write |
| Hypothesis record | Hypothesis | To write (fields per ADR-003 §2.2) |
| Exploration notes | Exploration | Free form; trial log mandatory |
| Experiment Report | Experiment, Validation | To write (content per ADR-003 §2.4) |
| Holdout log | Validation | To write |
| Paper / Pilot report | Paper, Pilot | To write |
| Durability assessment | Experiment onward | To write (§6) |
| Retirement note | Retired | To write |

---

## 10. Governance Failure Modes and Monitoring

| Failure mode | Detection |
|---|---|
| Process bypassed for "quick tests" in production | Any order from a strategy ID without a Pilot/Production approval record — monitored by Risk |
| Trial under-reporting | Quarterly audit comparing compute/experiment logs with registry trial counts |
| Holdout leakage | Access logs on holdout snapshots |
| Governance too heavy, research goes off-platform | Time-per-stage metrics; researcher feedback each quarter |
| Research Director as bottleneck | Gate review latency tracked; delegate Experiment-gate approval to senior researchers once ≥ 3 strategies have reached Pilot |
| Rubber-stamp reviews | Rejection rate tracked; a near-zero rejection rate is itself a red flag |
