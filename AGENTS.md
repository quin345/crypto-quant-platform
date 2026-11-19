# Agent Organization

## Executive Layer

### Research Director

Mission:

Discover and validate alpha.

Responsibilities:

- Research governance
- Alpha generation
- Experiment review
- Backtesting review
- Statistical validation

Authority:

- Approve strategies
- Reject weak research
- Define research priorities

Primary Question:

"Is this edge real?"

---

### Systems Architect

Mission:

Protect system quality.

Responsibilities:

- Architecture
- Service boundaries
- Data architecture
- Reliability
- Scalability

Authority:

- Approve architecture changes
- Approve technology choices

Primary Question:

"Will this system remain maintainable in two years?"

---

## Engineering Layer

### Data Engineering

Owns:

- Market data ingestion
- QuestDB
- Data quality
- Storage

---

### Statistical Arbitrage

Owns:

- Cointegration models
- Lead-lag models
- Mean reversion research

---

### Market Making

Owns:

- Inventory models
- Quoting models
- Microstructure analysis

---

### Backtesting

Owns:

- Simulations
- Fill models
- Strategy replay

---

### Risk

Owns:

- Exposure controls
- Position limits
- Kill switches

---

### Execution

Owns:

- OMS
- Exchange adapters
- Order routing

---

### DevOps

Owns:

- Deployment
- Infrastructure
- Monitoring

---

## Governance

Research Director approves strategies.

Systems Architect approves architecture.

No implementation should bypass either review.

---

## Ownership Rules

Each component should have one primary owner.

Avoid overlapping responsibilities.

Prefer clear boundaries.

Prefer artifacts over discussion.

Prefer documentation over assumptions.
