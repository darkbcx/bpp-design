# ADR-0012: Cross-context consistency

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/14-cross-context-consistency.md](../gaps/resolved/14-cross-context-consistency.md)
- **Builds on**: [ADR-0011](0011-domain-events.md) (transactional outbox, at-least-once delivery)

## Context

[ADR-0011](0011-domain-events.md) established the event substrate — outbox, dispatcher, envelope, at-least-once delivery. What remained: the **consistency rules** that govern multi-context operations.

Specifically:
- When state changes touch multiple contexts, what consistency do we provide?
- Can a single use case write to multiple contexts atomically?
- How do multi-step flows (like order placement, when Gap 11 lands) coordinate?
- How fresh must reads be?
- What happens when a subscriber fails?

These choices set how every cross-context feature is built. Get them wrong and the system either becomes brittle (over-tight coupling) or silently inconsistent.

## Options considered

### Consistency model

**A. Strong everywhere (distributed transactions / 2PC across context DBs).** Rejected — doesn't scale, doesn't survive service extraction, brittle to partial failures.
**B. Strong within context, eventual across contexts** — chosen.
**C. Eventual within contexts too.** Rejected — pushes significant complexity into every aggregate.

### Cross-context atomicity

**A. Multi-context transactions in a monolith.** Rejected — even when physically possible, code that depends on cross-context atomicity becomes impossible to extract later.
**B. One context per transaction; cross-context via events** — chosen.

### Multi-step flow orchestration

**A. Formal Saga framework** (persistent coordinators, declarative compensation). Rejected for v1 — overkill for flows that all fit in a single request.
**B. Application-Layer orchestration with hand-coded compensation** — chosen. Revisit when Gap 11 introduces longer-running flows.

### Read consistency

**A. Strict consistency** (all reads via owning context, no caches or projections). Rejected — over-constrains future read-side optimization.
**B. Per-query tolerance** — chosen. Transactional reads sync against the owning context; browse-time reads MAY be eventual.

### Failure handling

**A. Silent drop.** Rejected — loses events.
**B. Block dispatcher.** Rejected — cascading failure.
**C. Per-subscriber retry with backoff; stuck-row review by ops** — chosen.

## Decision

### 1. Consistency model

- **Within a context:** strong consistency. A transaction commits all writes for that context's aggregates atomically.
- **Across contexts:** eventual consistency. State propagates via domain events from the outbox. Subscribers see the change after a small (typically millisecond-scale) delay.

### 2. Cross-context atomicity

- A single Application-Layer use case writes to **one context only** in its transaction.
- Cross-context state propagation is **always via events** (outbox in the emitting context; subscribers in others).
- Even in a modular monolith — where multiple contexts may share a physical database — code does not span contexts in a single transaction. The architectural boundary is enforced in code, not the DB.

### 3. Multi-step flow orchestration

- **No formal Saga framework in v1.**
- Multi-context flows are coordinated by **Application-Layer orchestration**: a use case in the orchestrating context (typically the one that owns the outcome) calls into other contexts via synchronous Application-Layer ports.
- On failure mid-flow, **the orchestrating use case explicitly compensates** by calling reverse operations in already-affected contexts. Compensation is hand-coded per use case.
- Will be revisited when [Gap 11](../gaps/11-order-and-fulfillment-phasing.md) lands — order/payment/fulfillment may justify a Saga concept.

**Illustrative pattern** (will materialize in Gap 11):

```
Order.Place(items, voucher_code):
  reservation = Inventory.Reserve(items)             # may throw
  try:
    discount = Promotion.ValidateVoucher(voucher_code)  # may throw
  except:
    Inventory.ReleaseReservation(reservation)        # compensate
    raise
  return Order.Create(items, discount, reservation)
```

### 4. Read consistency

- **Transactional reads** — queries that immediately gate a state change (`Inventory.Reserve` reading `StockLevel`, `Order.Confirm` reading the active reservation, etc.) — MUST be synchronous against the owning context. No caches, no projections.
- **Browse-time reads** — storefront listings, dashboard views, search — MAY use eventual projections (cached read models, event-driven materialized views) when those are added later. v1 may serve these synchronously for simplicity.
- Read tolerance is declared **per-query**. There is no global "everything is eventual" or "everything is strong" rule.

### 5. Failure handling

- Subscribers are retried with **exponential backoff** on transient failures. Retry parameters (initial delay, multiplier, max attempts) are operational configuration.
- After N attempts, the event is marked **stuck**: stored in a `stuck_events` table (or flagged in the outbox); the subscriber's offset does NOT advance past it.
- **Stuck events surface as metrics and alerts** for ops review. There is no silent drop and no automatic skip.
- An operator either fixes the underlying cause and retries, or explicitly **skips with acknowledgement** (recorded as an audited operator action).

### 6. Inbox (subscriber-side dedup)

- Each subscriber maintains a `processed_events` table keyed by `(subscription_name, event_id)`.
- Before invoking the handler, the framework checks this table. If the event was already processed, the framework returns success without re-invoking — duplicate delivery becomes a no-op.
- This is the consumer-side complement to ADR-0011's outbox. Together they form the **outbox + inbox** pattern.

### 7. Cross-context ports

- A context **never** imports another context's internal types or storage.
- Cross-context calls go through **Application-Layer ports** defined by the **calling** context, implemented by the called context as an adapter.
- In a modular monolith, ports resolve to in-process method calls. In a service-extracted deployment, to RPC/HTTP/gRPC. Calling code is identical either way.
- Follows §2.3 (Dependency Rule) and §2.9 (deployment topology deferred).

**Pattern:**

```
// In Order context (caller defines the port):
interface InventoryPort {
  reserve(items: ItemSet): ReservationResult
  releaseReservation(reservationId): void
  ...
}

// In Inventory context: provides the adapter implementing InventoryPort.
```

## Consequences

What this commits to:

- Cross-context state propagation is **always asynchronous via events**. There is no synchronous "atomic cross-context write."
- The orchestrating context owns responsibility for compensation on failure. There is no generic rollback mechanism; each use case writes its own compensation logic.
- Subscribers are idempotent and dedup via the inbox table.
- Failed events are visible (`stuck_events` + alerts) rather than silently retried forever or silently dropped.
- Calling code across contexts is port-mediated; future service extraction is a deployment change, not a refactor.
- Read patterns are declared per-query: transactional vs. browse.

What this defers:

- **Saga / process-manager framework**, if/when needed (likely with Gap 11).
- **Event-driven read projections** (storefront materialized views). Infrastructure decision; supported by the event substrate.
- **Exact retry parameters and stuck-thresholds** — operational.
- **Cross-context distributed tracing** — operational; helpful but not architectural.
- **Compensation pattern library** — develops organically per use case.

What this makes harder:

- **Operations that need cross-context atomicity.** None in v1, but a future feature that wants this must be redesigned around eventual consistency.
- **"What's the world state right now"** across contexts — requires multiple sync queries; no single global snapshot.
- **Naive coding** that assumes cross-context writes happen together — code reviews must catch this.

## References

- [gaps/resolved/14-cross-context-consistency.md](../gaps/resolved/14-cross-context-consistency.md)
- [ADR-0011](0011-domain-events.md) (events, outbox, at-least-once)
- [design/events.md](../design/events.md)
- CLAUDE.md §2.3 (Dependency Rule), §2.6 (inter-context communication), §2.9 (deployment topology), §5.16 (domain events), §5.17 (this ADR's rules)
- Related gaps: [11](../gaps/11-order-and-fulfillment-phasing.md), [15](../gaps/15-first-party-idempotency.md)
