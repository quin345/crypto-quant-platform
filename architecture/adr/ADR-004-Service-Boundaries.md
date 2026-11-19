# ADR-004: Service Boundaries

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-19 |
| Decider | Systems Architect |
| Consulted | Data Engineering, Backtesting, Execution, Risk, DevOps, Research Director |
| Supersedes | — |
| Related | ADR-001, ADR-002, ADR-003 |

---

## 1. Context

ADR-001 defines five planes, two speed tiers, intent-based strategies, independent fail-closed risk, and one-way credential isolation. ADR-002 defines the data platform; ADR-003 defines the research workflow. This ADR assigns **services**, **owners**, **communication paths**, and **deployment boundaries**.

AGENTS.md requires that each component have exactly one primary owner, that responsibilities not overlap, and that boundaries be clear. Service boundaries are therefore drawn primarily along **ownership and failure-domain lines**, and only secondarily along technical lines.

### Assumptions challenged

- **"Microservices from day one."** A small team operating twenty services spends its time on operations, not research. Boundaries are defined logically now; physical separation happens only where a failure-domain, security, or latency reason exists (Section 2.5).
- **"Risk is a service the order passes through."** On the standard tier, yes. On the latency tier, a network hop for every order defeats the purpose. Risk must be *both* an in-process check and an independent out-of-process authority (Section 2.4).
- **"Kubernetes for everything."** Kubernetes suits research, data, and standard-tier workloads. Latency-tier processes need pinned cores, predictable networking, and minimal scheduler interference; they run on dedicated hosts (containerized, but not necessarily orchestrated by Kubernetes).
- **"The exchange is the source of truth for positions."** It is the source of truth for *its own* balances. Our internal position view can and does diverge (missed fills, websocket drops). Reconciliation must be continuous and owned independently of execution.
- **"Strategy code can call the exchange directly for simplicity."** This collapses alpha, risk, and execution ownership and puts credentials in research-adjacent code. Prohibited.

---

## 2. Decision

### 2.1 Service catalogue

| # | Service | Plane | Tier | Owner | Language (default) | Responsibility |
|---|---|---|---|---|---|---|
| 1 | **Market Data Capture** (one instance per venue) | Data | Standard → Latency* | Data Engineering | Rust | Connect to venue feeds; stamp receive times; publish raw payloads; detect gaps; resync |
| 2 | **Normalizer** | Data | Standard | Data Engineering | Rust | Raw → canonical events; schema validation; dead-letter routing |
| 3 | **Storage Sinks** | Data | Standard | Data Engineering | Rust or JVM connectors | Redpanda → Iceberg (L0/L1) and → QuestDB |
| 4 | **Reference Data Service** | Data | Standard | Data Engineering | Python or Rust | Instruments, fees, funding schedules, symbol mapping; bitemporal; read API |
| 5 | **Data Quality Monitor** | Data | Standard | Data Engineering | Python | Checks from ADR-002 §2.6; validity masks; alerts |
| 6 | **Research Environment** | Research | — | Systems Architect (platform); researchers (content) | Python (Polars/PyArrow) | Notebooks, experiment pipelines, data access library; read-only data access |
| 7 | **Simulation Engine** | Simulation | — | Backtesting | Rust core, Python API | Event-driven replay, fill-model tiers F1–F3, live-vs-sim parity |
| 8 | **Strategy Runtime** | Production | Standard and Latency | Execution (host); strategy teams (strategy code) | Python (standard) / Rust (latency) | Hosts strategy logic behind the Strategy API; emits intents |
| 9 | **Pre-Trade Risk** | Production | Both | Risk | Rust | Validates intents/orders against limits; fail-closed |
| 10 | **Order Management System (OMS)** | Production | Both | Execution | Rust | Intent → order translation; order state machine; routing; idempotency |
| 11 | **Exchange Adapters** (one per venue) | Production | Both | Execution | Rust | Venue-specific REST/WS order APIs; **only holder of trading credentials** |
| 12 | **Position & Reconciliation** | Production | Standard | Risk | Rust | Authoritative internal positions and PnL; continuous reconciliation against venue balances |
| 13 | **Risk Monitor & Kill Switch** | Production | Independent | Risk | Rust | Post-trade limits, exposure aggregation, kill authority (Section 2.4) |
| 14 | **Execution Recorder** | Production → Data | Standard | Execution | Rust | Publishes own orders/fills/latencies to the data plane (ADR-001 principle 7) |
| 15 | **Observability Stack** | Cross-cutting | — | DevOps | Off-the-shelf | Metrics, logs, traces, alerting, dashboards |
| 16 | **Governance Registry** | Governance | — | Research Director (content); Systems Architect (structure) | Repository artifacts | ADRs, hypothesis registry, approvals, audit trail |

\* Capture for latency-tier strategies (cross-exchange arb, MM) runs co-located with the strategy runtime on the latency-tier host and feeds the strategy directly; it *also* publishes to Redpanda for recording.

**Ownership gaps resolved here** (AGENTS.md did not specify):

- **Position & Reconciliation → Risk**, not Execution. The component that places orders should not be the sole judge of what positions exist.
- **Execution Recorder → Execution**, as producer; Data Engineering owns the schema and storage of what it records.
- **Research Environment platform → Systems Architect** until a platform/tooling owner exists (see Open Questions).

**Ownership gaps NOT resolved here** (see Open Questions): treasury/collateral movement across venues; secrets and key custody; legal/compliance.

### 2.2 Core contracts

Boundaries are defined by contracts, not by deployment. Four contracts are architecturally significant:

| Contract | Between | Content | Owner |
|---|---|---|---|
| **Canonical Event Schema** | Data plane ↔ everyone | ADR-002 §2.2 | Data Engineering (approved by Systems Architect) |
| **Strategy API** | Strategy code ↔ Simulation Engine & Strategy Runtime | Callbacks for market events, timers, and own-order events; emits `Intent` objects; access to a read-only state view (positions, reference data). No I/O, no clock access except through the API, no direct network | Systems Architect (approved jointly with Backtesting and Execution) |
| **Intent** | Strategy ↔ Risk/OMS | Target position or quote set, per instrument, with urgency/constraints and strategy ID. Not an order | Execution (approved by Risk) |
| **Order/Fill Events** | OMS/Adapters ↔ Risk, Position, Recorder | Order lifecycle state machine and fills in canonical form | Execution |

The **Strategy API is the most important boundary in the system.** It is what makes simulation and production run the same strategy code (ADR-001 principle 1; ADR-003 §2.6). Strategy code that cannot run unchanged in the Simulation Engine is not deployable.

### 2.3 Communication topology

```
                         STANDARD TIER (Kubernetes)
 ┌─────────┐  raw   ┌──────────┐ canonical ┌──────────────┐ intents ┌──────────┐ orders ┌──────────┐
 │ Capture │──────▶│Normalizer│──────────▶│   Strategy   │────────▶│ Pre-Trade│───────▶│   OMS    │
 └─────────┘       └──────────┘  Redpanda │   Runtime    │ Redpanda│   Risk   │        └────┬─────┘
                        │                 └──────────────┘         └──────────┘             │
                        ▼                                                                  ▼
              Sinks → Iceberg / QuestDB                                            ┌──────────────┐
                                                                                   │   Exchange   │──▶ Venue
                                                                                   │   Adapters   │
                                                                                   └──────┬───────┘
                                       order/fill events (Redpanda)                       │
              ┌───────────────────────────────────────────────────────────────────────────┘
              ▼                       ▼                         ▼
     ┌─────────────────┐    ┌──────────────────┐     ┌────────────────────┐
     │ Position & Recon│    │ Risk Monitor &   │     │ Execution Recorder │──▶ Data plane
     │  (vs. venue API)│───▶│ Kill Switch      │     └────────────────────┘
     └─────────────────┘    └────────┬─────────┘
                                     │ kill: independent path (own credentials, cancel-all)
                                     ▼
                                   Venues

                         LATENCY TIER (dedicated host, single process)
 ┌───────────────────────────────────────────────────────────────────────────────┐
 │ Capture ─▶ Strategy ─▶ Pre-Trade Risk (in-process lib) ─▶ OMS ─▶ Adapter ─▶ Venue │
 │      (shared memory / in-process queues; no network hops on the hot path)     │
 │      async tap ─▶ Redpanda (recording, risk monitor, position)                │
 └───────────────────────────────────────────────────────────────────────────────┘
```

Rules:

- **Standard tier:** services communicate via Redpanda topics using the canonical schema. Request/response (gRPC) only for reference-data lookups and control-plane operations.
- **Latency tier:** the hot path is a single process (or a small set of processes sharing memory) on a dedicated host. The Risk, OMS, and Adapter components are **libraries owned by their respective teams**, linked into the latency-tier binary. Ownership of code is preserved even when deployment is merged.
- **All tiers** publish order/fill/position events to Redpanda asynchronously so that the Position, Risk Monitor, and Recorder services see a unified view.
- **No service in the Research or Simulation planes may publish to production topics.** Topic ACLs enforce this.

### 2.4 Risk architecture: defense in depth

Risk is enforced at four independent layers. Any one layer can stop trading.

| Layer | Where | What it checks | Fails |
|---|---|---|---|
| **L1 Pre-trade (in-path)** | Pre-Trade Risk service (standard) or linked library (latency) | Order size, price bands vs reference, per-instrument/per-strategy position limits, order rate limits, notional limits | Closed: reject order |
| **L2 Post-trade (out-of-path)** | Risk Monitor | Aggregated exposure across strategies and venues, PnL drawdown, margin utilization, liquidation distance, reconciliation breaks | Closed: issue strategy halt or global kill |
| **L3 Kill Switch** | Risk Monitor, with its own venue credentials (cancel/close permissions only where venues support scoped keys) | Manual or automatic trigger | Cancels all open orders and optionally flattens, bypassing OMS |
| **L4 Venue-side** | Exchange features | Cancel-on-disconnect / dead-man's-switch, venue-level position limits, sub-account isolation | Venue cancels our orders if we go silent |

Requirements:

- The kill switch must work when Redpanda, Kubernetes, the OMS, and the Strategy Runtime are all down. It therefore runs outside the main Kubernetes cluster's failure domain (separate host or separate cluster) and talks to venues directly.
- Each strategy trades from a **dedicated venue sub-account** where the venue supports it, bounding blast radius.
- Loss of risk heartbeat (L2 → OMS) for longer than a configured interval causes OMS to stop sending new orders (fail-closed).

### 2.5 Physical deployment boundaries

Logical services are grouped into deployment units only where a reason exists:

| Deployment unit | Contains | Reason for boundary |
|---|---|---|
| **Data cluster** (Kubernetes) | Capture (standard), Normalizer, Sinks, Reference Data, Data Quality, QuestDB, Redpanda | Shared infrastructure; independent of trading failure domain |
| **Research cluster** (Kubernetes) | Research Environment, Simulation Engine workers, experiment pipelines | Bursty compute; **no trading credentials**; read-only access to data |
| **Trading cluster — standard tier** (Kubernetes, separate namespace or cluster) | Strategy Runtime (standard), Pre-Trade Risk, OMS, Exchange Adapters, Position & Recon, Execution Recorder | Credential boundary; change control stricter than research |
| **Latency hosts** (dedicated, near venue) | Latency-tier binary (Capture + Strategy + Risk lib + OMS + Adapter) | Latency; CPU pinning; no orchestrator interference. Only provisioned when cross-exchange arb or MM passes the Experiment gate (ADR-003) |
| **Risk/Kill host** (separate failure domain) | Risk Monitor & Kill Switch | Must survive failure of everything else |
| **Observability** | Metrics, logs, alerts | Must survive trading-cluster failure to diagnose it |

Phase 2 may consolidate the Data and Trading-standard clusters onto shared infrastructure with namespace isolation **if** team size requires it, provided the credential boundary (only Exchange Adapters and Kill Switch hold trading keys) and the Risk/Kill failure-domain separation remain intact. Those two boundaries are non-negotiable.

### 2.6 Security boundaries

- **Trading credentials** exist only in Exchange Adapters and the Kill Switch. Keys are scoped (no withdrawal permission on any trading key), IP-allowlisted where venues support it, and stored in a secrets manager (custody design deferred to a dedicated ADR).
- **Withdrawal / transfer permissions** are not held by any automated service in Phase 1–2.
- **Research plane** has no network path to venue order endpoints and no read access to production secrets.
- Every order carries a strategy ID and intent ID for audit.

### 2.7 Repository boundaries (logical)

Monorepo with top-level areas aligned to owners, so code ownership (e.g. CODEOWNERS) matches service ownership:

| Area | Owner |
|---|---|
| `architecture/` | Systems Architect |
| `schemas/` (canonical events, Intent, Strategy API definitions) | Data Engineering / Systems Architect per contract |
| `data/` (capture, normalizer, sinks, quality, reference) | Data Engineering |
| `research/` (registry, experiments, access library) | Research Director / strategy teams |
| `simulation/` | Backtesting |
| `strategies/<family>/` | Owning strategy team |
| `execution/` (runtime host, OMS, adapters, recorder) | Execution |
| `risk/` (pre-trade, monitor, kill switch, position & recon) | Risk |
| `deploy/` (infrastructure, manifests, observability) | DevOps |

This is a target layout for design purposes; it is not created until Phase 2 implementation begins.

---

## 3. Alternatives Considered

| Alternative | Why rejected |
|---|---|
| **Monolith (single trading process for everything)** | Simplest to start, but merges risk and execution failure domains, mixes credential scope, and makes independent kill impossible. The latency tier deliberately uses a monolithic *process* but keeps independent *ownership* and an external risk authority. |
| **Fine-grained microservices for all components** | Operational overhead disproportionate to team size; network hops on paths that do not need them. |
| **Risk only as an in-path service** | Adds a network hop to every latency-tier order; does not catch cross-strategy aggregate exposure; cannot act if the path itself fails. |
| **Risk only as a post-trade monitor** | Too late to stop a fat-finger or runaway loop; pre-trade checks are mandatory. |
| **Strategies place orders directly** | Collapses alpha/risk/execution ownership; credentials in strategy code; no consistent audit trail. |
| **Synchronous gRPC between all standard-tier services** | Tighter coupling; lost events on failure; harder replay. Redpanda gives a durable, replayable event log that doubles as the recording path. |
| **Run latency tier on Kubernetes** | Possible with CPU manager and host networking, but adds scheduler and CNI variability for little benefit on a handful of dedicated hosts. Revisit if the latency fleet grows. |

---

## 4. Tradeoffs

| We gain | We pay |
|---|---|
| Clear single ownership per component | More coordination at contract boundaries (Strategy API, Intent) |
| Same strategy code in simulation and production | Strategy API constraints (no direct I/O) feel restrictive to researchers |
| Independent kill switch that survives system failure | A separate failure domain and a second set of scoped credentials to manage |
| Durable, replayable standard-tier communication via Redpanda | Milliseconds of added latency on the standard tier — acceptable for funding/stat arb |
| Latency tier without network hops | A different deployment model to operate; libraries must be designed for in-process linking from day one |
| Reconciliation owned by Risk | Risk team must understand venue balance APIs in detail |

---

## 5. Consequences

- The **Strategy API** and **Intent** contracts require dedicated Design Specifications before any strategy or simulation code is written. They are the critical path for Phase 2.
- Risk and OMS components must be built as libraries with thin service wrappers so they can run in both tiers.
- Venue selection (ADR-001 Open Question 1) must consider sub-account support, scoped API keys, and cancel-on-disconnect support, since the risk architecture depends on them.
- DevOps must provision at least three failure domains (data/research, trading, risk/kill) plus observability, even in a minimal Phase 2 setup.
- Follow-up ADRs/specs required: Strategy API spec, Intent spec, secrets and key custody, treasury/collateral management, latency-tier host and network design, incident-response runbook design.

---

## 6. Development Rule Compliance

| Rule | Answer |
|---|---|
| Objective | Define services, owners, contracts, and failure domains so the system stays safe and maintainable as strategy families are added. |
| Owner | Systems Architect (boundaries and contracts); each service's primary owner per Section 2.1. |
| Acceptance criteria | (1) Every service has exactly one primary owner. (2) Only Exchange Adapters and the Kill Switch hold trading credentials. (3) A strategy runs unchanged in Simulation Engine and Strategy Runtime. (4) A kill-switch drill succeeds with the trading cluster and Redpanda deliberately unavailable. (5) Research and Simulation planes cannot publish to production topics (verified by ACL test). (6) Position reconciliation detects an injected discrepancy within the configured interval. |
| Failure modes | Strategy API leaks I/O, breaking sim/prod parity; risk library diverges from risk service in behavior; kill switch shares a hidden dependency with the trading cluster (DNS, secrets manager, network); OMS and Position disagree with no alert; ownership erodes as teams "just fix" other teams' services; over-segmentation creating operational load the team cannot sustain. |
| Monitoring | Per-service health and owner on-call routing; intent → order → ack latency per tier; pre-trade rejection rate by reason; reconciliation break count and age; risk heartbeat freshness; scheduled kill-switch drills (proposal: monthly, plus after any change to risk or adapters); live-vs-sim divergence per strategy (ADR-003). |

---

## 7. Open Questions

1. **Treasury and collateral management** (moving capital between venues for funding and cross-exchange arb) has no owner. It is operationally critical and high-risk. Proposal: Risk owns policy, a dedicated Treasury function owns execution, with no automated withdrawals in Phase 1–2.
2. **Secrets and key custody**: owner (DevOps vs Risk) and tooling.
3. **Research Environment platform owner**: Systems Architect is a placeholder; a research-platform engineering role may be needed.
4. Does the standard-tier Strategy Runtime host Python strategies directly, or must all production strategies be Rust with Python only for research? (Trade-off: research-to-production speed vs runtime safety and parity.) Depends on the build-vs-adopt decision for the simulation/strategy runtime (ADR-001 Open Question 4).
5. Legal/compliance ownership and its interaction with venue access and sub-account structure.
