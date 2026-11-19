# Crypto Market Structure — Research Primer

| Field | Value |
|---|---|
| Author | Research Director |
| Date | 2026-11-19 |
| Status | Living document — revise quarterly |
| Related | ADR-001, ADR-002, ADR-003, TOP_20_ALPHA_HYPOTHESES.md |

---

## 0. Purpose

This document describes how crypto markets are built, who trades in them, and **where structural edges come from**. Every hypothesis on this platform must be able to point to a section of this document and say *who is on the other side of the trade and why they will keep being there*.

> **Verification rule.** Venue mechanics (funding formulas, intervals, caps, fee tiers, liquidation rules) change often and differ between venues. Statements here describe typical patterns. Before any experiment of record, the exact rules must be confirmed from venue documentation and stored as bitemporal Reference data (ADR-002 §2.1). No research result may depend on a mechanic that is only described here.

---

## 1. The Edge Taxonomy

A durable edge has an identifiable, persistent source. We classify every hypothesis into one of four sources, ordered roughly by durability:

| Source | Mechanism | Who pays | Durability | Main threat |
|---|---|---|---|---|
| **Risk premium** | We are paid to carry a risk others want to shed (basis, liquidation, venue, inventory) | Leveraged traders, hedgers | **High** — persists while the risk persists | Tail events; crowding compresses the premium |
| **Structural / forced flow** | Someone must trade regardless of price (liquidations, funding avoidance, index rebalances, token unlocks, settlement) | Constrained participants | **Medium–High** — rules change slowly | Rule changes; front-running by others |
| **Market segmentation / friction** | Capital cannot move freely between venues, collateral types, or jurisdictions | Participants who can't access both sides | **Medium** — erodes as plumbing improves | Integration (better bridges, unified margin, ETFs) |
| **Speed / information** | We see or act on information before others | Slower traders | **Low** for us — arms race against better-resourced firms | Competitors with lower latency and better fee tiers |

**Research Director stance:** Phase 2 prioritizes risk-premium and structural-flow edges. Speed edges are researched only where we can show a non-speed advantage (e.g. a niche venue, a capacity range large firms ignore).

---

## 2. Participants

| Participant | Behaviour | Relevance to us |
|---|---|---|
| **Leveraged retail / directional speculators** | Trend-following, momentum-chasing, high leverage, price-insensitive at extremes | Main source of persistent positive funding; liquidation fuel; adverse-selection-light flow for MMs |
| **Professional market makers** (prop firms, venue-designated MMs) | Low latency, VIP or negotiated fee tiers, often rebates | Our main competitors in MM and cross-exchange arb; they set the cost floor |
| **Basis / carry funds** | Long spot (or ETF), short perps/futures to harvest funding/basis | Competitors in funding arb; their capital inflow compresses carry |
| **ETF authorized participants and TradFi desks** | Arbitrage ETF creation/redemption; trade CME futures | Linked crypto to TradFi basis; compressed some spot-futures dislocations, especially in BTC/ETH |
| **Arbitrageurs** (cross-venue, triangular, stablecoin) | Keep prices aligned across venues | Competitors; their capacity limits define how fast dislocations close |
| **Liquidation engines and insurance funds** | Forced closure of under-margined positions | Generate price-insensitive flow; create overshoots |
| **Projects, foundations, early investors** | Scheduled token unlocks, treasury sales, market-maker loan deals | Predictable supply events, often in illiquid tokens |
| **Venue itself** | Sets fees, rules, listing decisions, sometimes runs affiliated trading | Rule changes are the largest regime risk; venue failure is the largest tail risk |

---

## 3. Venues and Instruments

### 3.1 Venue types

| Type | Characteristics | Phase |
|---|---|---|
| **CEX spot** | Central limit order book; multiple quote currencies (USDT, USDC, USD, others) | 2 |
| **CEX perpetual futures** | No expiry; anchored to index via funding; most of crypto's volume; high leverage | 2 (core) |
| **CEX dated futures** | Fixed expiry; basis converges at settlement | 2b / 3 |
| **CEX options** | Concentrated in few venues; volatility surface | Non-goal (ADR-001 §2.5) |
| **Regulated futures (e.g. CME)** | Institutional; different hours and margin regime | Reference / signal only |
| **DEX / on-chain** | AMMs, on-chain perps; MEV; settlement on-chain | Non-goal (ADR-001 §2.5) |

### 3.2 Why venue concentration matters

Price discovery concentrates on a small number of venues and instruments (typically the most liquid perps). Secondary venues tend to follow. This creates **lead-lag** at short horizons, but the leading venue is also where the fastest competitors sit. Edges on secondary venues are often more accessible but have small capacity.

---

## 4. Core Mechanics

### 4.1 Perpetual funding

- Perps track their index through periodic **funding payments** between longs and shorts.
- Typical structure: funding ≈ premium component (perp price vs index, averaged over the interval) + an interest-rate component, subject to a clamp/cap. **Exact formula, clamp, and averaging window are venue-specific — verify.**
- Intervals differ by venue and sometimes by instrument (commonly 8h, also 4h or 1h; some venues shorten the interval during extreme moves). **Verify per instrument.**
- Venues often publish a **predicted** funding rate during the interval; realized funding is fixed at settlement.
- **Structural fact:** in most regimes, aggregate leveraged demand is net long, so funding is positive more often than negative. Shorts (us, when hedged) collect. This is a **risk premium**: the hedged short is exposed to basis spikes, liquidation of the short leg in sharp rallies, and venue risk.

### 4.2 Mark price, index price, and liquidation

- Liquidations are triggered on **mark price** (usually index-derived), not last trade — this protects against manipulation but means the perp can trade away from mark.
- **Index** is a weighted composite of spot venues. Index composition and outlier rules matter when one constituent venue dislocates.
- When a position is liquidated, the venue's engine closes it in the market; if the loss exceeds margin, the **insurance fund** absorbs it; if the fund is insufficient, **auto-deleveraging (ADL)** closes profitable opposing positions. **ADL is a direct risk to hedged carry positions**: our profitable short can be force-closed during a squeeze, leaving the spot leg unhedged.

### 4.3 Margin and collateral

- Cross vs isolated margin; unified accounts on some venues allow spot holdings as perp collateral, on others they do not.
- **Collateral fragmentation** is a core friction: capital on venue A cannot instantly support a position on venue B. Transfers depend on chain confirmation times, withdrawal processing, and limits. This friction is a *source* of cross-venue funding and price differentials and a *risk* when trying to capture them.

### 4.4 Fees

- Maker/taker schedule tiered by 30-day volume and sometimes by holdings of the venue token. Rebates at the top tiers.
- **For a new entrant, fees are the single largest determinant of viability for MM and cross-exchange arb.** A strategy that only works at VIP tiers is not a strategy for us until we are at that tier.
- All research must state the fee tier assumed (ADR-003 §2.4) and show results at our *actual achievable* tier.

### 4.5 Order book and microstructure

- Tick size and lot size relative to price vary widely. **Large-tick instruments** (tick ≫ typical spread) create queue-priority value; small-tick instruments make the spread itself competitive.
- Websocket feeds deliver incremental updates with sequence numbers; gaps happen (ADR-002 §2.6).
- Rate limits on order placement/cancellation constrain quoting strategies; they differ by venue and by account tier.
- Self-trade prevention, post-only, reduce-only, and cancel-on-disconnect order features vary by venue and affect strategy design and risk (ADR-004 §2.4).

### 4.6 Stablecoins and quote currencies

- The same asset trades against several quote currencies (USDT, USDC, fiat USD). These are not identical: stablecoins carry issuer, redemption, and regulatory risk and can trade at small premia/discounts.
- Cross-quote price differences are partly **risk premium** (stablecoin risk) and partly **segmentation** (who can access fiat rails).

### 4.7 Events

| Event | Why it matters |
|---|---|
| Funding settlements | Position adjustments around settlement; funding-avoidance flows |
| Listings / delistings | New perps often have extreme funding and thin books; delistings create survivorship bias |
| Token unlocks | Scheduled supply; often in thin markets |
| Contract changes | Multiplier, tick, leverage, or funding-rule changes; regime breaks |
| Venue incidents | Outages, withdrawal halts, insolvency — the largest tail risk in crypto |
| Macro / TradFi | ETF flows, CME session times, rate decisions affect basis and funding |
| Weekends | TradFi closed; liquidity thinner; different participant mix |

---

## 5. What Has Changed (and Why It Matters for Durability)

| Change | Effect on edges |
|---|---|
| **Spot ETFs and institutional basis trading** | BTC/ETH carry more competed; funding/basis premia in majors compressed relative to early cycles. Long-tail perps less affected. |
| **More professional MMs and better venue tooling** | Simple cross-venue arbitrage in majors largely competed away; residual dislocations last milliseconds. |
| **Venue failures (historical)** | Counterparty risk is real and must be priced; a carry return that ignores venue default risk is overstated. |
| **Shorter / adaptive funding intervals on some venues** | Changes funding predictability and cross-venue comparability. |
| **Proliferation of long-tail perps** | More, smaller, less-competed opportunities; higher manipulation and liquidity risk. |

**Implication:** durable edges for us are most likely in (a) carry/risk premia with better *risk management* than competitors, (b) structural flows in the mid-liquidity tier that large firms find too small, and (c) careful cost engineering. They are least likely in raw speed on major pairs.

---

## 6. Where We Can Plausibly Compete

| Edge type | Can we compete? | Condition |
|---|---|---|
| Funding carry in majors | Yes, but low margin | Excellent cost and risk control; realistic about compression |
| Funding carry in mid-caps | **Likely best opportunity** | Liquidity-aware sizing; liquidation/ADL risk modeling |
| Cross-venue funding differentials | Plausible | Collateral pre-positioning; transfer risk modeled |
| Structural flow (liquidations, unlocks, settlement) | Plausible | Clean event data; honest sample sizes |
| Cross-sectional stat arb on perps | Plausible | Strict overfitting control; capacity-aware |
| Cross-exchange taker arbitrage in majors | **Unlikely** | Would need fee tiers and latency we don't have |
| Market making in majors | **Unlikely** at start | Fee tier, latency, and queue disadvantage |
| Market making in mid-liquidity / large-tick instruments | Possible later | Requires F3 simulation and demonstrated adverse-selection control |

---

## 7. Open Research Questions on Structure

1. Which two venues for Phase 2? Criteria: perp liquidity breadth (not just majors), data API quality, sub-account and scoped-key support, cancel-on-disconnect, fee schedule, jurisdictional access.
2. How competed is mid-cap funding carry after costs today? (First experiment — see H-001.)
3. How often does ADL actually hit hedged short positions during squeezes on candidate venues?
4. What is our realistically achievable fee tier within 12 months of trading?
