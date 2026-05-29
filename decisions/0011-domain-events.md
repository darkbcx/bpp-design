# ADR-0011: Domain events — transactional outbox, versioned envelope

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/13-domain-events-design.md](../gaps/resolved/13-domain-events-design.md)
- **Builds on**: [ADR-0002](0002-authorization-tiers-and-matrix.md) (actor & impersonation), [ADR-0003](0003-store-lifecycle-and-state-machine.md) (Bridge republication), [ADR-0006](0006-inventory-model.md) (Catalog → Inventory subscription)
- **Affects**: every ADR that emits or consumes events (ADR-0003 through ADR-0010)
- **Sub-design**: [design/events.md](../design/events.md)

## Context

Ten ADRs have committed event semantics without ever defining them. Specifically:

- ADR-0003 declares `StoreStatusChanged` and requires reliable delivery to the Bridge for republication.
- ADR-0004 declares catalog state changes drive republication.
- ADR-0005 declares ~20 Catalog events; design/catalog.md names them.
- ADR-0006 declares the Catalog → Inventory subscription (lifecycle events).
- ADR-0007 declares Voucher events.
- ADR-0008 requires event payloads carrying translatable content to preserve full `LocalizedText` (not a single rendering).
- ADR-0009 declares ~10 Identity events; design/identity.md names them.
- ADR-0010 declares Invitation events.

Plus implicit needs: Audit ingestion (Gap 16), idempotency for the Bridge (re-projection must be safe), reliable cross-context propagation, and a path that survives the modular-monolith → service-extraction evolution declared in §2.9.

The event design must be coherent enough that all current and future contexts produce/consume events without ad-hoc patterns drifting in. Without it, the implementing team would invent local conventions per context.

## Options considered

### Delivery and storage

**A. Transient in-memory bus** — rejected. Loses events on crash; Bridge misses republication; Audit becomes incomplete.

**B. Dedicated event store (Kafka, EventStoreDB)** — rejected for v1. Heavy infrastructure for a system at this scale; works against §2.9's "deployment topology is deferred."

**C. Transactional outbox** — chosen. Events written to a DB table in the same transaction as state changes; a background dispatcher delivers. Works in monolithic and distributed deployments unchanged.

### Naming

**A. Past-tense, context-namespaced (`catalog.product_published`)** — chosen.

**B. Flat past-tense (`ProductPublished`)** — rejected. Ambiguous across contexts (`Created`, `Updated`, etc. would collide).

### Envelope

**A. Standard envelope + per-event payload** — chosen. Envelope carries id, name, version, time, actor, correlation/causation; payload is event-type-specific.

**B. Flat per-event schemas** — rejected. Duplicates envelope concerns; loses tooling consistency.

### Audit relationship

**A. Audit = the event log** — rejected. Couples engineering events to compliance retention.

**B. Audit is a downstream subscriber** — chosen.

### Delivery semantics

**A. At-least-once** — chosen. Subscribers dedup by `event_id`.

**B. Exactly-once** — rejected. Hard, requires distributed coordination.

**C. At-most-once** — rejected. Loses events.

### Ordering

**A. Per-aggregate order preserved; cross-aggregate not guaranteed** — chosen.

**B. Strict global ordering** — rejected. Limits scale, no value here.

## Decision

### 1. Transactional outbox

Every event-emitting context maintains a context-local `outbox` table. Events are written to it **in the same database transaction as the state change**. A background **dispatcher** reads from the outbox, delivers each event to its subscribers, and marks the row dispatched on success.

This guarantees state-event consistency, survives crashes, and works identically in modular-monolith (in-process dispatch) and service-extracted (message-bus dispatch) deployments.

### 2. Envelope

Every event carries this envelope:

| Field | Type | Description |
|---|---|---|
| `event_id` | UUID | Unique per event instance; the dedup key for subscribers |
| `event_name` | string | `<context>.<verb_past>` (e.g., `catalog.product_published`) |
| `event_version` | integer | Schema version of the payload; starts at 1 |
| `occurred_at` | timestamp | Domain time — when the event happened |
| `recorded_at` | timestamp | When the event was written to the outbox |
| `aggregate_type` | string | E.g., `Product`, `Store`, `Order` |
| `aggregate_id` | string | The aggregate this event pertains to |
| `actor` | object | `{ user_id, impersonated_user_id, active_store_id }`; fields nullable for system events |
| `correlation_id` | string \| null | Groups causally related events from one operation |
| `causation_id` | UUID \| null | The `event_id` of the event that caused this one (chains) |
| `payload` | object | Event-type-specific data |

The `actor` field carries the impersonation pair per [ADR-0002](0002-authorization-tiers-and-matrix.md): both the real actor and the impersonated user are recorded.

### 3. Naming

`<context>.<verb_past>` — lowercase, snake_case, past tense, namespaced by emitting context:

- `tenancy.store_status_changed`
- `catalog.product_published`
- `inventory.stock_reserved`
- `promotion.voucher_redeemed`
- `identity.session_active_store_set`

### 4. Versioning

Each event type has an integer `event_version`. Rules:

- **Additive changes** (new optional fields) do **not** bump the version.
- **Breaking changes** (removed fields, changed types, changed required-ness, renamed fields) **bump** the version.
- Multiple versions may coexist in the stream during a transition.
- Subscribers handle the versions they understand; unknown future versions are logged and skipped without halting the subscription.
- Deprecated versions are retired only after all subscribers have migrated.

The **event registry** (in [design/events.md](../design/events.md)) is the authoritative catalog of every event type and its current version.

### 5. Subscribers

A subscriber declares:
- A unique subscription name.
- An event filter (event names + minimum versions).
- Persisted offset / last-processed position.
- A handler that is **idempotent by `event_id`**.

Subscribers may be in-process (modular monolith) or out-of-process (service extraction). The contract is the same.

### 6. Ordering guarantees

- **Per-aggregate**: events for the same `aggregate_id` are delivered in `occurred_at` order.
- **Cross-aggregate**: not guaranteed.
- **Cross-context**: not guaranteed.

Subscribers that need cross-aggregate sequencing implement buffering / reordering themselves (rare in this system).

### 7. Retention and replay

- Events are retained in the outbox/log for an **operational window** (default 90 days; configurable).
- Within retention, subscribers can replay from any offset.
- Beyond retention, events are deleted (or archived to cold storage per compliance — operational).
- A fresh subscriber that needs state older than retention queries current state via the Application Layer, then attaches live.

Audit retention is **independent** of event-log retention; Audit keeps its records per its own (likely longer) policy.

### 8. Audit as subscriber

Audit (Gap 16) is a **downstream subscriber**, not the event log itself. Audit:
- Subscribes broadly across contexts.
- Transforms events into Audit records (richer correlation, longer retention).
- Deduplicates by `event_id`.

This decouples engineering events (which can be ephemeral and evolve) from compliance records (which have legal retention requirements).

### 9. Bridge subscription

The Bridge is a dedicated subscriber subscribing to a curated set of events that affect what's projected onto the Beckn network. On each consumed event, the Bridge **re-projects** the affected resource. Re-projection is idempotent — re-publishing the same state is harmless.

The Bridge does **not** emit domain events. (It emits Beckn protocol messages, but those are not domain events.)

### 10. Deployment topology

The outbox + dispatcher pattern is **topology-neutral**:

- **Modular monolith**: dispatcher is an in-process worker calling subscriber handlers directly.
- **Distributed services**: dispatcher publishes to a message bus (Kafka, NATS, SQS, …); subscribers consume from their topics.

Same envelope, same naming, same versioning either way. The deployment choice is Infrastructure (§2.9).

## Consequences

What this commits to:

- Every event-emitting context has an `outbox` table written transactionally with state changes.
- All events follow the envelope and naming rules; the **event registry** in `design/events.md` is the single source of truth for declared events.
- Versioning is the standard contract-evolution mechanism — breaking changes never happen in-place.
- Audit is **downstream** of events, not coupled to them.
- The Bridge is **event-driven**; no direct calls from domain contexts into the Bridge.
- Subscribers are **idempotent by construction** — this is a domain-level requirement.
- Cross-aggregate ordering, when needed, is the subscriber's problem.
- Event payloads carrying translatable content (per ADR-0008) carry the full `LocalizedText`, not a single rendering.
- Retention is operational config; archival beyond retention is a compliance concern.

What this defers:

- **Sagas / process managers** for multi-step cross-context flows — Gap 14 (Cross-context consistency), Gap 11 (Order) will address.
- **Exact retention window value** — operational; default 90 days.
- **Dispatcher implementation** — Infrastructure (in-process worker vs. distributed bus adapter).
- **Schema-registry tooling** — operational; the design declares schemas; tooling enforces.
- **Cold-storage / archive policy beyond retention** — operational, compliance-driven.
- **Read-side projections** (storefront read models, analytics) — Infrastructure / future work; the event design supports them.

What this makes harder:

- **Strict cross-aggregate ordering** — subscribers pay the cost.
- **Exactly-once processing** — not provided; subscribers must dedup.
- **In-place schema breakage** — forbidden; must introduce a new version.
- **Using the event log as a current-state query target** — the log is the change history, not a projection. Current state must come from Application Layer queries.
- **Event-emission outside the outbox** — domain code that "just fires an event" without the outbox writes is a defect; the consistency guarantee depends on the transactional write.

## References

- [gaps/resolved/13-domain-events-design.md](../gaps/resolved/13-domain-events-design.md)
- [design/events.md](../design/events.md) — full envelope, outbox shape, dispatcher, naming registry, event catalog
- [ADR-0002](0002-authorization-tiers-and-matrix.md) (actor model with impersonation), [ADR-0003](0003-store-lifecycle-and-state-machine.md), [ADR-0004](0004-store-publication-and-multi-catalog-projection.md), [ADR-0005](0005-catalog-and-product-modeling.md), [ADR-0006](0006-inventory-model.md), [ADR-0007](0007-pricing-tax-and-vouchers.md), [ADR-0008](0008-localization-and-localizedtext.md) (LocalizedText in payloads), [ADR-0009](0009-identity-and-external-idp.md), [ADR-0010](0010-invitation-account-reconciliation.md)
- Related gaps: [14](../gaps/14-cross-context-consistency.md), [16](../gaps/16-soft-delete-and-audit.md)
