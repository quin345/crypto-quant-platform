# Quant Platform Constitution

You are working on a professional quantitative cryptocurrency trading platform.

## Platform Goals

Build a research-first system capable of:

- Statistical Arbitrage
- Market Making
- Cross-Exchange Arbitrage
- Funding Arbitrage

The platform must support:

- Hypothesis generation
- Data collection
- Backtesting
- Risk management
- Production execution

---

## Core Principles

### Research First

Research quality is more important than implementation speed.

Never build infrastructure without a clear research objective.

### Architecture First

Produce design documents before major implementation.

### Small Contexts

Avoid loading the entire repository when possible.

Use relevant files and artifacts only.

### Reproducibility

Research results must be reproducible.

### Data Integrity

Data quality is more important than data quantity.

---

## Technology Preferences

Research:
- Python
- Polars
- PyArrow

Data:
- QuestDB
- Apache Iceberg
- Parquet

Streaming:
- Redpanda

Execution:
- Rust

Infrastructure:
- Kubernetes
- Docker

---

## Required Artifacts

Major work should be documented through:

- ADRs
- Research Reports
- Experiment Reports
- Design Specifications

---

## Development Rules

Before implementing a feature:

1. Define objective
2. Define owner
3. Define acceptance criteria
4. Define failure modes
5. Define monitoring requirements

Never skip architecture reasoning.

Always explain tradeoffs.

Always challenge assumptions.

Prefer maintainability over quick solutions.

---

## Current Project Phase

Phase 1

Goals:

- System architecture
- Research roadmap
- Alpha hypothesis generation
- Data platform design

Avoid premature implementation.

---

## Task routing

Subagents live in `.claude/agents/`: `scout` (haiku), `implementer` (sonnet), `architect` (opus).

The main session is the judgment layer and runs on Opus (set in `.claude/settings.json`). It plans, routes, reviews child output and makes the final call; it does not do bulk work.

- For any task touching 3+ files or with separable parts, first write a short plan that splits it into independent subtasks.
- Spawn independent subtasks in parallel, in a single turn. Run dependent ones sequentially.
- Difficulty rubric:
  - Easy (lookup, grep, summarize, single-file mechanical change) -> `scout`
  - Medium (clear spec, known pattern, tests exist or are easy to write) -> `implementer`
  - Hard (ambiguous requirements, architecture, concurrency, security, data migrations, bug with unknown cause) -> `architect`
- When unsure, pick the cheaper tier first and escalate one tier if the result is incomplete or tests fail.
- Give every child a self-contained brief: goal, relevant paths, constraints, expected output format.
- The main session only plans, delegates, and integrates results. It doesn't do bulk work itself.
- After children finish, review their outputs and run tests before reporting done.
