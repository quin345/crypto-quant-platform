# ADR-001: System Vision

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-19 |
| Decider | Systems Architect |
| Consulted | Research Director, all engineering owners |
| Supersedes | — |
| Related | ADR-002, ADR-003, ADR-004 |

---

## 1. Context

The platform's stated goal is a **research factory**: discover, validate, deploy, and monitor alpha in crypto markets across four strategy families:

1. Statistical Arbitrage
2. Market Making
3. Cross-Exchange Arbitrage
4. Funding Rate Arbitrage

These four families look similar on a slide but have very different system requirements. Treating them as one problem is the most likely way to build the wrong platform.

| Family | Decision horizon | Latency sensitivity | Data granularity needed | Dominant risk | Capital model |
|---|---|---|---|---|---|
| Funding arbitrage | Hours to days | Low (seconds acceptable) | Funding rates, mark/index, borrow rates, 1s–1m bars | Basis blowout, liquidation, venue/counterparty failure | Hedged, capital-heavy, multi-venue collateral |
| Statistical arbitrage | Minutes to days | Low to moderate | Trades, top-of-book, bars | Regime change / cointegration breakdown | Market-neutral portfolio |
| Cross-exchange arbitrage | Milliseconds to seconds | High | L2 books from multiple venues, synchronized clocks | Leg risk, stale quotes, transfer delays | Pre-positioned inventory on each venue |
| Market making | Sub-millisecond to milliseconds | Very high | Full L2/L3 deltas, own order lifecycle, queue position | Adverse selection, inventory | Inventory risk on single venue |

### Assumptions challenged

- **"One platform serves all four equally."** False. Market making and cross-exchange arbitrage need a latency-optimized hot path that the slower strategies do not need and should not pay for in complexity.
- **"We can compete in market making from day one."** Unlikely. Crypto MM on major venues is dominated by firms with colocation-equivalent placement, VIP fee tiers, and rebate programs. Without favorable fee tiers, most MM edge is consumed by fees and adverse selection. MM should be a *researched* hypothesis before it is an *engineered* capability.
- **"More data is better."** CLAUDE.md already rejects this. Data integrity and timestamp correctness matter more than venue count.
- **"Kubernetes everywhere."** Kubernetes is appropriate for research, data, and batch workloads. It is not automatically appropriate for latency-critical trading processes (see ADR-004).
- **"Backtests are evidence."** Backtests are evidence only when the data, fill model, and trial count are known. Otherwise they are marketing (see ADR-003).

---

## 2. Decision

### 2.1 What the platform is

A **research-first trading system** composed of five planes with strict one-way dependencies:

```
                ┌───────────────────────────────────────────────┐
                │              Governance Plane                  │
                │ ADRs · hypothesis registry · approvals · audit │
                └───────────────────────────────────────────────┘
   ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
   │   Data     │───▶│  Research  │───▶│ Simulation │───▶│ Production │
   │   Plane    │    │   Plane    │    │   Plane    │    │   Plane    │
   └────────────┘    └────────────┘    └────────────┘    └────────────┘
         ▲                                                     │
         └──────────── recorded live events, fills ────────────┘
                     (production feeds data, never research)
```

- **Data Plane** – capture, normalize, store, and serve market and reference data. (ADR-002)
- **Research Plane** – hypothesis-driven, reproducible analysis. (ADR-003)
- **Simulation Plane** – event-driven backtesting and replay using the same strategy logic and event model as production.
- **Production Plane** – strategy runtime, risk, order management, exchange connectivity. (ADR-004)
- **Governance Plane** – the artifacts and approvals that gate movement between planes.

### 2.2 Guiding architectural principles

1. **Single canonical event model.** Market data, orders, fills, and positions are expressed in one versioned schema used by capture, research, simulation, and production. Research/production skew is the most expensive class of bug in this domain.
2. **Iceberg on object storage is the system of record.** QuestDB and Redpanda are serving and transport layers, not sources of truth (ADR-002).
3. **Deterministic replay.** Any production session must be replayable from recorded inputs to reproduce strategy decisions. This is the foundation for debugging, backtest validation, and incident review.
4. **Strategies emit intents, not orders.** Strategy logic produces target positions or quote intents. Risk and execution turn intents into orders. This separates alpha ownership from execution ownership (ADR-004).
5. **Risk is independent and fail-closed.** Risk controls cannot be disabled by strategy code, and the kill switch must work when the rest of the system is broken.
6. **Two speed tiers, not one.** A *standard tier* (bus-based, Python-permissible for signal generation, Kubernetes-hosted) and a *latency tier* (Rust, direct market-data feed, dedicated hosts). A strategy is assigned to a tier by its measured latency sensitivity, not by preference.
7. **Research cannot reach production; production feeds research.** Research environments never hold trading credentials. Production outputs (fills, latencies, slippage) flow back into the data plane as research inputs.

### 2.3 Strategy sequencing

Build order follows the ratio of *research value* to *infrastructure cost*:

| Order | Family | Rationale |
|---|---|---|
| 1 | Funding arbitrage | Low latency demands; edge is observable in public data; exercises multi-venue data, position, and risk infrastructure without a hot path. |
| 2 | Statistical arbitrage | Reuses the same standard-tier infrastructure; adds portfolio construction and regime monitoring. |
| 3 | Cross-exchange arbitrage | Requires the latency tier, clock synchronization, and multi-venue inventory management. Only start after tier-1 capture data proves latency-adjusted spreads exist net of fees. |
| 4 | Market making | Requires queue-position-aware simulation, L2/L3 data, and favorable fee tiers. Treat as a research program first; engineer only after a fee-adjusted, adverse-selection-adjusted edge is demonstrated on recorded data. |

This sequencing is a recommendation for the Research Director; it is overridable by research evidence, but any override must state which infrastructure prerequisites are being pulled forward.

### 2.4 Quality attribute priorities

When attributes conflict, resolve in this order:

1. **Correctness** (data, positions, PnL)
2. **Safety** (risk limits, kill switch, credential isolation)
3. **Reproducibility**
4. **Maintainability**
5. **Observability**
6. **Latency** (only for latency-tier components)
7. **Cost**
8. **Throughput / scale**

### 2.5 Explicit non-goals (Phase 1–2)

- Sub-10µs tick-to-trade (FPGA, kernel bypass). Not justified without a demonstrated edge that requires it.
- On-chain / DEX strategies (MEV, AMM arbitrage). Different data model, custody model, and risk profile. Revisit with a separate ADR.
- Options and structured products.
- Multi-tenant or external-capital infrastructure.
- A general-purpose ML platform. Feature and model tooling is added only when a hypothesis requires it.

---

## 3. Alternatives Considered

| Alternative | Why rejected |
|---|---|
| **Adopt an existing framework end-to-end** (e.g. Nautilus Trader, Hummingbot, Freqtrade) | Strong candidates for components or reference designs, especially Nautilus (Rust core, Python API, event-driven parity). Rejected as the *whole* platform because it would dictate the data model and research workflow. **Re-evaluate** using Nautilus as the simulation/strategy runtime in a dedicated ADR before building our own — the build-vs-adopt question is open, not closed. |
| **Single-language system (all Python)** | Fine for standard tier; inadequate for latency tier and for the safety properties wanted in OMS/risk. |
| **Single-language system (all Rust)** | Slows research iteration dramatically. Research velocity is the primary constraint in Phase 1–2. |
| **Build for market making first** | Highest infrastructure cost, lowest prior probability of edge for a new entrant. |
| **Notebook-driven research without a registry** | Fast initially; produces unreproducible and over-fitted results. Directly contradicts the Reproducibility principle. |

---

## 4. Tradeoffs

| We gain | We pay |
|---|---|
| Research/production parity via one event model | Upfront schema design effort; schema changes become governed events |
| Safety from intent-based strategies and independent risk | An extra hop and an extra service to operate |
| Reduced complexity by deferring the latency tier | Cross-exchange and MM edges cannot be traded early, even if they exist |
| Reproducibility via Iceberg snapshots and replay | Storage cost of retaining raw data; operational cost of a catalog |
| Polyglot (Python research, Rust execution) | Two toolchains, two hiring profiles, an FFI/serialization boundary |

---

## 5. Consequences

- ADR-002 must define the canonical event model and the system of record.
- ADR-003 must define how a hypothesis moves from idea to production and what evidence is required at each gate.
- ADR-004 must define services, owners, communication paths, and the two-tier split.
- Further ADRs are expected for: canonical schema & serialization, build-vs-adopt for simulation/strategy runtime, clock synchronization, secrets and key custody, hosting/region strategy, and capital/treasury management across venues.

---

## 6. Development Rule Compliance

| Rule | Answer |
|---|---|
| Objective | Establish a shared system vision that constrains all subsequent architecture decisions. |
| Owner | Systems Architect (architecture); Research Director (strategy sequencing). |
| Acceptance criteria | ADR-002…004 are consistent with this vision; every planned component maps to exactly one plane and one owner; strategy sequencing is accepted or explicitly overridden by the Research Director. |
| Failure modes | Vision too abstract to constrain decisions; premature commitment to the latency tier; vision ignored by implementation; strategy sequencing contradicted by research evidence but not updated. |
| Monitoring | Quarterly ADR review; every new component design must cite which plane, tier, and principle it serves. Deviations require a superseding ADR. |

---

## 7. Open Questions

1. Which venues are in scope for Phase 2? (Proposal: two CEX venues with deep perp markets, chosen for API quality and data availability, not volume rank alone.)
2. What fee tiers are realistically attainable? This determines whether MM and cross-exchange arbitrage are viable at all.
3. What is the team size? Several decisions (Kubernetes, Iceberg catalog, schema registry) carry operational cost that a very small team may not absorb.
4. Build vs adopt for the simulation and strategy runtime (Nautilus Trader or similar).
5. Jurisdiction and venue-access constraints (legal/compliance ownership is not defined in AGENTS.md).
