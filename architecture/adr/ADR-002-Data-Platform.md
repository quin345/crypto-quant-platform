# ADR-002: Data Platform

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-19 |
| Decider | Systems Architect |
| Consulted | Data Engineering (primary owner), Research Director, Backtesting, Execution, Risk |
| Supersedes | — |
| Related | ADR-001, ADR-003, ADR-004 |

---

## 1. Context

ADR-001 establishes that:

- Iceberg on object storage is the system of record; QuestDB and Redpanda are serving and transport layers.
- A single canonical event model is shared by capture, research, simulation, and production.
- Any production session must be deterministically replayable.
- Funding arbitrage and statistical arbitrage come first; cross-exchange arbitrage and market making come later and need much finer data.

The data platform is therefore not a "market data database". It is the **evidence base** for every research claim and every incident review. If the data is wrong, every downstream conclusion is wrong, and wrong in ways that are usually invisible in a backtest.

### Crypto-specific data hazards

These hazards drive most of the decisions below:

| Hazard | Example | Consequence if ignored |
|---|---|---|
| Unreliable exchange timestamps | Venue `ts` reflects matching engine, gateway, or publish time depending on the venue and channel | Phantom lead-lag and phantom cross-exchange spreads |
| Sequence gaps on websocket feeds | Dropped messages during reconnects or bursts | Corrupt order book reconstruction; fictitious arbitrage |
| Silent schema changes | Venue adds/renames fields, changes precision | Parsing errors or, worse, silent misinterpretation |
| Instrument lifecycle | Listings, delistings, contract-multiplier changes, symbol renames, tick-size changes | Survivorship bias; wrong notional and PnL |
| Funding mechanics differ per venue | Interval (1h/4h/8h), caps, predicted vs realized, settlement timing | Funding arb PnL overstated or mistimed |
| Fee schedules change and are tiered | Maker/taker by tier, rebates, promotional periods | MM and arb backtests that ignore fees are meaningless |
| Historical vendor data gaps | Third-party data with undocumented cleaning or interpolation | Unknown provenance; non-reproducible results |

### Assumptions challenged

- **"QuestDB can be the system of record."** It is excellent for time-series ingest and recent-window queries, but it is not where we want long-term immutable history, snapshot-based reproducibility, or schema evolution governance. Using it as the source of truth couples research reproducibility to a database's retention and upgrade cycle.
- **"Record normalized data only."** Normalizers have bugs. If only normalized data is kept, a normalizer bug permanently destroys history. Raw payloads must be retained.
- **"Exchange timestamp is the event time."** It is *one* of the times. We need at least three (Section 2.3).
- **"Buy historical data and skip capture."** Vendor history is useful for long-horizon research, but it cannot validate our own latency, our own feed quality, or our own fills. Self-capture starts in Phase 2 regardless of vendor purchases.
- **"Capture everything from every venue."** Contradicts *Data Integrity over Quantity*. Coverage expands only when a registered hypothesis requires it (ADR-003).

---

## 2. Decision

### 2.1 Layered storage model

```
 Venue WS/REST
      │
      ▼
 ┌──────────────┐   raw payload + receive metadata
 │ Capture      │──────────────────────────────┐
 │ (per venue)  │                              ▼
 └──────────────┘                     ┌─────────────────┐
      │ Redpanda: raw.<venue>.<chan>  │ L0 RAW (Iceberg) │  immutable, append-only
      ▼                               └─────────────────┘
 ┌──────────────┐                              │ re-normalize on bug fix
 │ Normalizer   │◀─────────────────────────────┘
 └──────────────┘
      │ Redpanda: md.<event_type>  (canonical schema)
      ├────────────────────────▶ QuestDB (hot window, monitoring, ad-hoc)
      ▼
 ┌────────────────────┐
 │ L1 NORMALIZED       │  canonical events, versioned schema
 │ (Iceberg)           │
 └────────────────────┘
      │ batch jobs (reproducible, versioned)
      ▼
 ┌────────────────────┐
 │ L2 DERIVED          │  bars, book snapshots, funding panels, features
 │ (Iceberg)           │
 └────────────────────┘
```

| Layer | Content | Mutability | Primary consumers |
|---|---|---|---|
| **L0 Raw** | Exact venue payload bytes + capture metadata (receive timestamps, connection ID, capture host, capture software version) | Append-only, never rewritten | Normalizer re-runs, forensic investigation |
| **L1 Normalized** | Canonical events (Section 2.2) | Rewritten only via a versioned re-normalization that produces a new snapshot; old snapshots retained per policy | Simulation, research, replay |
| **L2 Derived** | Bars, periodic book snapshots, funding/basis panels, research features | Rebuilt from L1 by versioned jobs | Research, monitoring |
| **Reference** | Instruments, fee schedules, funding schedules, venue metadata, calendar events (maintenance, delistings) | Bitemporal (valid-time + record-time) | Every layer |

**System of record:** L0 and Reference are the irreducible sources. L1 and L2 are reproducible from them; they are stored for performance, not because they are authoritative.

**Physical format:** Parquet files managed as Apache Iceberg tables on S3-compatible object storage.

### 2.2 Canonical event model

One versioned schema family, owned by Data Engineering and approved by the Systems Architect. Minimum event types:

| Event type | Purpose | Needed first by |
|---|---|---|
| `Trade` | Public prints | Stat arb |
| `BookTop` | Best bid/ask + sizes | Funding arb, stat arb |
| `BookDelta` | L2 incremental update with venue sequence number | Cross-exchange arb, MM |
| `BookSnapshot` | Full L2 snapshot (venue-provided or reconstructed) | Cross-exchange arb, MM |
| `FundingRate` | Predicted and realized funding, interval, settlement time | Funding arb |
| `MarkIndexPrice` | Mark, index, and their components where available | Funding arb, risk |
| `OpenInterest` | Open interest snapshots | Research |
| `Liquidation` | Public liquidation events where published | Research |
| `BorrowRate` | Spot margin borrow rates | Funding arb (spot leg) |
| `OrderEvent` / `Fill` | Our own order lifecycle and fills (from production) | Fill-model calibration, TCA |
| `FeedStatus` | Connect / disconnect / gap / resync markers | Everything (data validity masks) |

Schema rules:

- Prices and quantities are fixed-point decimals with instrument-specific scale, never binary floats, in L1. Research may cast to float in L2 and analysis code.
- Every event carries `instrument_id` (internal, stable) — not the venue symbol. Venue symbol mapping lives in Reference data.
- Schemas are versioned; breaking changes require a new version and a migration note. A schema registry (Redpanda's built-in registry is the default candidate) enforces compatibility on the streaming path.
- Serialization format on the bus is an open sub-decision (candidates: Protobuf, FlatBuffers/SBE for the latency tier). It is deferred to a dedicated ADR (see Consequences), but the logical schema above is decided here.

### 2.3 Time model

Every event carries:

| Field | Meaning | Source |
|---|---|---|
| `ts_exchange` | Time asserted by the venue (as documented per channel) | Venue payload |
| `ts_recv` | Wall-clock time the capture host received the message | Capture host, disciplined clock |
| `ts_recv_mono` | Monotonic receive counter | Capture host |
| `seq_venue` | Venue sequence / update ID, if provided | Venue payload |
| `seq_capture` | Gapless per-connection sequence assigned at capture | Capture process |

Rules:

- **Simulation and research order events by `ts_recv`** (what we could actually have known), not `ts_exchange`. `ts_exchange` is used for latency analysis and venue-side ordering checks.
- Capture hosts must run disciplined clock sync (chrony/PTP where available) and **record their clock offset estimate** as a metric. Data captured with clock error above a threshold is flagged, not silently used.
- The exact clock-sync standard and the latency budget for the latency tier are deferred to a clock-synchronization ADR (ADR-001 §5).

### 2.4 Roles of each technology

| Technology | Role | Not used for |
|---|---|---|
| **Iceberg + Parquet on object storage** | System of record; snapshot-based reproducibility (every experiment pins a snapshot ID); schema evolution; partition evolution | Low-latency queries; streaming |
| **Redpanda** | Transport between capture, normalizer, sinks, and standard-tier services; short-term buffer (days) enabling replay of recent data | Long-term storage; source of truth |
| **QuestDB** | Hot window (proposal: 7–30 days) of normalized data for live dashboards, data-quality monitoring, and ad-hoc SQL | Research datasets of record; backtests that must be reproducible |
| **Polars / PyArrow** | Research and batch compute over Iceberg/Parquet | Production hot path |
| **Iceberg catalog** | Table metadata and snapshot management | — (catalog choice is an open question) |

Why both QuestDB and Iceberg: they answer different questions. QuestDB answers "what is happening now and was the feed healthy in the last hour?" Iceberg answers "what exactly did the data look like when this experiment was run?" Collapsing them forces one to do a job it is weak at.

### 2.5 Partitioning and layout (initial)

- L0: partition by `venue`, `channel`, `date(ts_recv)`.
- L1: partition by `event_type`, `venue`, `date(ts_recv)`; sort within files by `instrument_id`, `ts_recv`.
- L2: partitioned by the access pattern of the derived dataset (usually `date` and frequency).
- Iceberg partition evolution allows revision without rewrite; revisit after query patterns are observed for one quarter.
- Small-file compaction is a required scheduled job, not an optimization.

### 2.6 Data quality

Data quality is a first-class product with owners, checks, and SLOs.

| Check | Layer | Action on failure |
|---|---|---|
| Sequence gap detection (`seq_venue`) | Capture / Normalizer | Emit `FeedStatus{gap}`; trigger resync; mark interval invalid |
| Book integrity (crossed book, negative size, checksum mismatch where venue provides checksums) | Normalizer | Mark book invalid until next snapshot |
| Schema conformance | Normalizer | Route to dead-letter topic; alert; never drop silently |
| Clock offset over threshold | Capture | Flag interval; alert |
| Volume/coverage anomaly vs trailing baseline | L1 daily job | Alert; block downstream L2 build for that partition |
| Cross-venue sanity (price deviation beyond plausible band) | L1 daily job | Flag for review |
| Reference data consistency (every `instrument_id` resolves for its valid-time) | L1 daily job | Block downstream |

**Validity masks:** every dataset consumed by research and simulation must include, or be joinable to, a validity mask derived from `FeedStatus` and quality checks. Backtests must not trade through invalid intervals; they either skip or treat the interval as a forced flat, as declared in the experiment config (ADR-003).

### 2.7 Data access contract for research

- Research reads **Iceberg tables via a pinned snapshot ID** (or tag). "Latest" is permitted only for exploration, never for an experiment of record.
- Research never reads from production Redpanda topics or the production QuestDB directly; a read replica or the Iceberg layer is used. (Credential and blast-radius isolation per ADR-001 principle 7.)
- A small, versioned **data access library** wraps catalog access, snapshot pinning, instrument resolution, and validity masks. It is the only supported entry point for research code.

### 2.8 Retention (initial proposal)

| Layer | Retention | Rationale |
|---|---|---|
| L0 Raw | Indefinite (cold storage tier after 90 days) | Irreplaceable |
| L1 Normalized | Indefinite for current schema version; prior snapshots expired after 180 days unless tagged by an experiment of record | Reproducibility of approved research |
| L2 Derived | Rebuildable; retain tagged snapshots only | Cost |
| Redpanda | 3–7 days | Buffer and recent replay |
| QuestDB | 7–30 days | Hot window only |

Experiments of record (ADR-003) **tag** the Iceberg snapshots they depend on; tagged snapshots are exempt from expiry.

### 2.9 Coverage roadmap

Coverage is driven by hypotheses, not ambition:

| Phase | Coverage |
|---|---|
| 2a | Funding, mark/index, top-of-book, trades for perps on 2 venues; ~20–50 liquid instruments; vendor history for funding and bars where provenance is documented |
| 2b | Add spot top-of-book, borrow rates (funding arb spot legs); expand instruments as stat-arb hypotheses require |
| 3 | Full L2 deltas for a small instrument set on the arb venues, once a cross-exchange hypothesis is registered |
| 4 | Own order lifecycle and queue-position-relevant data for MM research |

---

## 3. Alternatives Considered

| Alternative | Why rejected |
|---|---|
| **QuestDB as system of record** | Ties reproducibility to a single database's retention and upgrade path; no snapshot isolation for experiments; harder schema governance. Kept as the hot serving layer. |
| **Plain Parquet on object storage (no table format)** | Simpler, but no atomic commits, no snapshot pinning, no schema/partition evolution, and race conditions between writers and readers. Reproducibility would depend on file-naming conventions. |
| **Delta Lake instead of Iceberg** | Technically viable. Iceberg chosen for broader engine neutrality (Polars/PyArrow/DuckDB/Spark/Trino readers, REST catalog standard) and alignment with CLAUDE.md preferences. |
| **Kafka instead of Redpanda** | Equivalent semantics. Redpanda chosen for single-binary operations, lower tail latency, and built-in schema registry, which matter for a small team. |
| **ClickHouse / TimescaleDB / kdb+ as the analytical store** | ClickHouse is a strong alternative for interactive analytics and should be revisited if Polars-on-Iceberg proves too slow. kdb+ rejected on licensing cost and talent pool. Timescale offers less benefit than QuestDB for our ingest pattern. |
| **Normalized-only capture** | A normalizer bug would permanently corrupt history. |
| **Vendor-only data** | Unknown provenance; cannot validate our own latencies, gaps, or fills. |

---

## 4. Tradeoffs

| We gain | We pay |
|---|---|
| Reproducibility via Iceberg snapshot pinning | Catalog operations; snapshot expiry and compaction jobs |
| Forensic recoverability via raw retention | Storage cost (mitigated by cold-storage tiering and compression) |
| Correct "what could we have known" ordering via `ts_recv` | Need disciplined clocks and capture hosts close to venues |
| Validity masks prevent trading through bad data in backtests | More complex research data access; every dataset carries an extra mask |
| Separation of hot (QuestDB) and historical (Iceberg) | Two query surfaces; risk of users querying the wrong one — mitigated by the access library and policy |
| Fixed-point canonical types | Conversion cost and care in research code |

---

## 5. Consequences

- Data Engineering owns capture, normalization, the canonical schema, the data access library, quality checks, and Iceberg/QuestDB/Redpanda data operations. DevOps owns the underlying infrastructure (ADR-004).
- Every research experiment of record must reference Iceberg snapshot IDs (ADR-003).
- Production fills and order events must flow back into L0/L1 (ADR-001 principle 7; ADR-004).
- Follow-up ADRs required:
  - **Serialization & schema registry** (wire format per tier, compatibility rules).
  - **Iceberg catalog selection** (REST catalog implementation, e.g. Nessie, Polaris, Lakekeeper, or a managed catalog).
  - **Clock synchronization standard** and capture host placement.
  - **Vendor data policy** (which vendors, provenance requirements, how vendor and self-captured data are reconciled).

---

## 6. Development Rule Compliance

| Rule | Answer |
|---|---|
| Objective | Provide a trustworthy, reproducible evidence base for research, simulation, and incident review. |
| Owner | Data Engineering (implementation & operations); Systems Architect (schema and architecture approval). |
| Acceptance criteria | (1) Any L1 partition can be rebuilt bit-identically from L0 + Reference with a given normalizer version. (2) An experiment pinned to a snapshot ID returns identical data six months later. (3) Every gap in a captured feed is represented by a `FeedStatus` event and appears in the validity mask. (4) Data-quality checks run daily with results queryable and alerting wired. (5) Research code accesses data only through the access library. |
| Failure modes | Silent sequence gaps; clock drift producing fake lead-lag; normalizer bug corrupting L1 (recoverable via L0); catalog outage blocking research; snapshot expiry deleting data an approved experiment depends on; small-file explosion degrading query performance; QuestDB used as a de facto source of truth; capture host failure creating unmarked holes. |
| Monitoring | Per-feed message rate, gap count, reconnect count, end-to-end capture lag (`ts_recv − ts_exchange` distribution), clock offset, dead-letter volume, Redpanda consumer lag, Iceberg commit latency and file counts, daily quality-check pass rate, storage growth. |

---

## 7. Open Questions

1. Which Iceberg catalog implementation? (Requires hosting decision; managed vs self-hosted.)
2. Hosting region(s) for capture hosts relative to venue matching engines — affects both data quality and later latency-tier viability.
3. Which vendor(s), if any, for historical backfill, and what provenance standard do they meet?
4. Exact QuestDB hot-window length — driven by monitoring needs, to be measured.
5. Do we need a reconstructed L3/queue model for MM research, or is L2 sufficient? Defer until MM is a registered hypothesis.
6. Do we capture public data from venues we do not trade on (for price discovery / lead-lag research)? Default: no, unless a registered hypothesis requires it.
