# Learning Roadmap (Newcomer Track)

| Field | Value |
|---|---|
| Status | Draft |
| Date | 2026-11-19 |
| Audience | New contributors without a quant or data-engineering background |
| Related | ADR-001..004, RESEARCH_ROADMAP, RESEARCH_GOVERNANCE, TOP_20_ALPHA_HYPOTHESES, MARKET_STRUCTURE |

> Titles and availability were compiled from memory. Verify editions, links and prices before purchasing.

## 0. How to Use This Document

The repo's Phase 1 documents assume background knowledge. This roadmap maps that background to resources, in the order the project needs it, so you can review the ADRs and hypotheses critically instead of just accepting them.

Priority tags: **[Core]** read before reviewing the docs, **[Next]** read during Phase 1, **[Later]** needed only in Phase 2+ (skim now).

Phase 1 produces designs, not code. Do not skip ahead to Rust or exchange APIs.

---

## 1. Suggested Sequence

| Stage | Duration (part-time) | Focus | Unlocks reading of |
|---|---|---|---|
| A | Weeks 1-3 | Market basics, perps and funding | MARKET_STRUCTURE, hypotheses H-001..005 |
| B | Weeks 3-6 | Statistics and research discipline | RESEARCH_GOVERNANCE, ADR-003 |
| C | Weeks 5-9 | Data engineering and architecture | ADR-001, ADR-002, ADR-004 |
| D | Weeks 8-12 | Strategy-specific depth | Hypotheses H-006..020, RESEARCH_ROADMAP Phases 2-4 |

Stages overlap by design: alternate theory (A, B) with architecture reading (C).

---

## 2. Stage A: Markets, Perpetuals, Funding

| Resource | Tag | Why |
|---|---|---|
| *Trading and Exchanges*, Larry Harris | Core | Order books, makers vs takers, spreads, adverse selection. Vocabulary for MARKET_STRUCTURE and every market-making hypothesis |
| "Fundamentals of Perpetual Futures", Ackerer, Hugonnier, Jermann (SSRN/arXiv) | Core | Theory of perp pricing and funding. Foundation of H-001..005 |
| Exchange docs for funding, mark/index price and liquidation (Binance, Bybit, OKX, Deribit) | Core | Mechanics differ per venue. ADR-002's timestamp and schema hazards come from here |
| *A Random Walk Down Wall Street*, Burton Malkiel | Optional | Healthy skepticism about edges |
| *Mastering Bitcoin*, Antonopoulos | Optional | Only if transfer and settlement mechanics are unclear |
| *Empirical Market Microstructure*, Joel Hasbrouck | Later | Needed for Phase 3 microstructure measurement |

**Check yourself:** explain why a funding payment is not free money (basis risk, liquidation, collateral, venue risk).

---

## 3. Stage B: Statistics and Research Discipline

This stage matters most. RESEARCH_GOVERNANCE exists because naive backtesting produces false positives.

| Resource | Tag | Why |
|---|---|---|
| *An Introduction to Statistical Learning* (free PDF) | Core | Overfitting, cross-validation, regression |
| Bailey & López de Prado, "The Deflated Sharpe Ratio" | Core | Basis for trial accounting (Governance 4.1) |
| Bailey et al., "The Probability of Backtest Overfitting" | Core | Basis for holdout custody (Governance 4.2) |
| *Advances in Financial Machine Learning*, López de Prado | Next | Read the backtesting, labeling and leakage chapters first. Skip the rest at first |
| *Quantitative Trading* and *Algorithmic Trading*, Ernest Chan | Next | Mean reversion, cointegration, practical pitfalls (H-006..008) |
| Georgia Tech "Machine Learning for Trading" (free, Udacity) | Next | Hands-on Python intro |
| *Systematic Trading*, Robert Carver | Next | Position sizing and realistic expectations |
| *Analysis of Financial Time Series*, Ruey Tsay | Later | Time-series modelling reference |
| *Active Portfolio Management*, Grinold & Kahn | Later | Alpha, information ratio, the fundamental law |

**Check yourself:** explain why testing 200 ideas and reporting the best one invalidates a Sharpe ratio, and how the gate thresholds in Governance 3 respond to that.

---

## 4. Stage C: Data Platform and Architecture

| Resource | Tag | Why |
|---|---|---|
| *Designing Data-Intensive Applications*, Martin Kleppmann | Core | Logs, streams, replay, consistency. Underpins ADR-002 and ADR-004 |
| *Fundamentals of Data Engineering*, Reis & Housley | Core | Ingestion, storage layers, data quality |
| Michael Nygard, "Documenting Architecture Decisions" (free) | Core | Origin of the ADR format used here |
| *Fundamentals of Software Architecture*, Richards & Ford | Next | Tradeoff analysis and service boundaries |
| *Release It!*, Michael Nygard | Next | Failure modes, circuit breakers. Relevant to ADR-004 risk defense in depth |
| Apache Iceberg docs, *Apache Iceberg: The Definitive Guide* | Next | System of record in ADR-002 |
| QuestDB and Redpanda docs, Redpanda University (free) | Next | Serving and transport layers |
| Polars user guide, PyArrow docs | Next | Research data access contract |
| *Python for Data Analysis*, Wes McKinney | Optional | If pandas/Python fundamentals are weak |
| *The Rust Programming Language* ("the Book", free) | Later | Execution layer. Phase 1 forbids premature implementation, so skim only |

**Check yourself:** explain why exchange timestamps cannot be trusted (ADR-002 hazards), and what the canonical event model and time model do about it.

---

## 5. Stage D: Strategy-Specific Depth

Read only what the strategy being reviewed needs.

| Family | Hypotheses | Resources |
|---|---|---|
| Funding arbitrage | H-001..005, H-010 | Perpetuals paper (Stage A), exchange docs, Paradigm and Kaiko research blogs |
| Statistical arbitrage | H-006..009, H-011..013 | Chan, Vidyamurthy *Pairs Trading*, Tsay |
| Cross-exchange arbitrage | H-014..016 | Harris, Hasbrouck, exchange withdrawal and transfer docs |
| Market making | H-017..020 | Avellaneda & Stoikov (2008), Cartea, Jaimungal & Penalva *Algorithmic and High-Frequency Trading*, *Algorithmic Trading and DMA* (Barry Johnson) |

Market making is Phase 4 and conditional (RESEARCH_ROADMAP), so skip it for now.

---

## 6. Ongoing Reading

- SSRN and arXiv q-fin for papers
- Quantocracy and QuantStart for blog aggregation
- Paradigm, Kaiko, Amberdata for crypto market structure
- MIT OCW 18.S096, "Topics in Mathematics with Applications in Finance" (free)

---

## 7. Applying This to the Repo

Active reading is better than passive:

1. After Stage B, review RESEARCH_GOVERNANCE 3 and 5 and list any threshold or red flag you think is too loose or too strict. Challenging assumptions is a project rule.
2. After Stage C, pick one "Open Questions" section (ADR-002 or ADR-004 section 7) and draft a short answer with tradeoffs.
3. Pick one hypothesis, write what data it needs, and check that ADR-002's coverage roadmap provides it.

Record findings as comments or proposed ADR amendments, not as silent edits.

---

## 8. Failure Modes of Learning

| Failure | Mitigation |
|---|---|
| Reading advanced ML before basic statistics | Follow the stage order |
| Treating books and blog strategies as proven alpha | Run them through the Governance red-flag checklist |
| Learning tools (Rust, Kubernetes) before the objective is clear | Respect "Research First"; defer to Later items |
| Reading without output | Use the exercises in section 7 |
