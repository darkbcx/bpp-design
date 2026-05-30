# ADR-0019: Manual catalog republication

- **Status**: Accepted
- **Date**: 2026-05-30
- **Builds on**: [ADR-0003](0003-store-lifecycle-and-state-machine.md) (state-driven republication), [ADR-0004](0004-store-publication-and-multi-catalog-projection.md) (multi-catalog publishing), [ADR-0011](0011-domain-events.md) (events), [ADR-0017](0017-order-and-fulfillment.md) (CDS publishing surface), [ADR-0018](0018-organization-tenancy.md) (Organization tenancy), [ADR-0013](0013-first-party-idempotency.md) (idempotency)

## Context

[ADR-0003](0003-store-lifecycle-and-state-machine.md) and [ADR-0004](0004-store-publication-and-multi-catalog-projection.md) established that the Bridge **republishes catalogs to CDS automatically** on every relevant state change — store lifecycle transitions, catalog and product mutations, etc. — via domain event subscriptions. This event-driven flow handles all normal state changes correctly and idempotently.

What it doesn't handle by itself:

- **Owner-initiated republish for confidence.** "My store isn't showing up on BAP X — let me force a republish."
- **Owner-initiated recovery.** A stuck publish event surfaces in ops, but the owner can attempt remediation first by triggering a fresh republish.
- **Bulk republish for an Org.** An Org Owner with multiple stores may want to republish all of them at once (after a CDS outage, for example).

This ADR introduces **manual catalog republication** as an *additive* feature. The Bridge stays event-driven (no special-case paths). Manual triggers are just additional event sources that feed the existing re-projection logic.

## Options considered

### D1 — Who can trigger manual republish?

**A. Org Owner + Store Admin (for the relevant store)** — chosen. Closest to "my store's state"; gives agency to those responsible for the store.
**B. Org Owner only.** Rejected — too narrow; Store Admins should be able to recover their assigned stores.
**C. Platform-scoped only.** Rejected — treats it purely as an ops action; doesn't help owners with confidence-recovery.

### D2 — Scope per trigger

**A. Per-store** — chosen (primary).
**B. Per-Org (bulk)** — chosen (additional). Useful for Org Owners managing multiple stores.
**C. Per-catalog within a store.** Rejected — granularity not needed; per-store republishes all of the store's catalogs anyway.

### D3 — Just republish, or staging workflow?

**A. Just republish (no staging)** — chosen. Simplest; what's published is the current live state of the store / catalog.
**B. Staging workflow** — edits accumulate in a draft mode, owner batch-publishes. Rejected for v1; substantial UX feature; can be added later as a separate ADR.

### D4 — Rate limiting

**A. Operational, not architectural** — chosen. The architecture does not impose a limit; default operational guidance documented in §5 below.

## Decision

### 1. Two new Application-Layer use cases

`Store.RequestRepublish(store_id, by_actor, reason?)`:
1. Requires `store.republish` capability against the target `store_id` (per [ADR-0016](0016-authorization-details.md)).
2. Verifies the Store exists and is in `Active` / `Paused` / `Suspended` state (republishing a Draft store is a no-op; the Bridge knows Draft is not published).
3. Emits `tenancy.store_republish_requested` with `{ store_id, actor, reason }`.
4. Returns acknowledgement to the caller (the actual republish happens asynchronously via the Bridge).
5. Supports the standard `idempotency_key` per [ADR-0013](0013-first-party-idempotency.md).

`Org.RequestRepublishAll(org_id, by_actor, reason?)`:
1. Requires `org.republish_all_stores` capability against the target `org_id`.
2. Verifies the Organization is `Active` (not `Suspended`).
3. Emits `tenancy.organization_republish_requested` with `{ org_id, actor, reason }`.
4. Returns acknowledgement.
5. Supports `idempotency_key`.

Neither use case writes domain state beyond the outbox row — they trigger Bridge-side action only.

### 2. Two new capabilities

| Capability | Held by |
|---|---|
| `store.republish` | Org Owner (implicit access to all stores in their Org), Store Admin (for assigned stores within their Active Org), Platform-scoped users (via matrix) |
| `org.republish_all_stores` | Org Owner only (within their Active Org), Platform-scoped users (via matrix) |

### 3. Two new domain events

| Event | Payload | Notes |
|---|---|---|
| `tenancy.store_republish_requested` | `{ store_id, actor, reason? }` | Manual trigger for a single store |
| `tenancy.organization_republish_requested` | `{ org_id, actor, reason? }` | Bulk trigger for all stores in an Org |

Both events conform to the standard envelope (per [ADR-0011](0011-domain-events.md)) with full `actor` (including impersonator per ADR-0002).

### 4. Bridge behavior

The Bridge's subscriber filter expands to include the two new events:

- **`tenancy.store_republish_requested`** → the Bridge fetches the current catalog state for the named store and POSTs `/catalog/publish` to CDS. Same code path as auto-trigger events (per ADR-0003, ADR-0004).
- **`tenancy.organization_republish_requested`** → the Bridge enumerates the Org's stores (via Tenancy query) and republishes each. Implementation choice: the Bridge may fan out internally (one event → N publish calls) OR derive per-store events; either is acceptable as long as the result is identical to N individual store-republish requests.

Re-projection remains **idempotent**. A simultaneous manual + auto trigger is harmless — same wire state results either way.

Failure handling (retries, stuck events) follows [ADR-0012](0012-cross-context-consistency.md). A stuck `store_republish_requested` event is reviewed by ops; the owner may also retry the manual action (which creates a new event, distinct by `event_id`).

### 5. Rate limiting (operational)

Recommended defaults (not architectural — tune as needed at the Interface Layer):

- `store.republish`: max **1 per store per 60 seconds** per user.
- `org.republish_all_stores`: max **1 per Org per 5 minutes** per user.

Enforced at the API layer (web request rate-limit middleware), not in the domain. The use case itself stays straightforward and idempotent.

### 6. Audit

Both events are ingested by the Audit context (per [ADR-0014](0014-soft-delete-and-audit.md)'s broad subscription). Each manual republish is recorded with actor (including impersonator), `reason` if supplied, and timestamp. This gives owners and ops a history of who initiated which manual publishes.

## Consequences

What this commits to:

- Two new Tenancy use cases (`Store.RequestRepublish`, `Org.RequestRepublishAll`).
- Two new capabilities in the matrix.
- Two new domain events in the registry (`design/events.md` updated).
- The Bridge's subscriber list expands to include manual-trigger events.
- The Audit history grows by manual-publish records.
- No change to wire protocol — same `/catalog/publish` mechanism.

What this defers:

- **Staging / draft mode for catalog changes** — author edits in a draft state, batch-publish. Substantial UX feature; future ADR.
- **Per-catalog republish** (one of a store's catalogs, not all) — rarely needed; can be added if specific demand emerges.
- **Scheduled republish** (publish at a future time) — future enhancement.
- **Bulk republish at platform scope** — not a new feature; platform-scoped users can already iterate over Orgs/stores via existing tooling.

What this makes harder:

- **Mistake / abuse mitigation** — rate limiting is operational. Without it, a misbehaving client could trigger excessive publishes. Mitigated by idempotency (same state = same wire result), but CDS-side traffic still increases.
- **Reasoning about publish causation** — Audit captures the actor and reason, but inside the Bridge the manual-trigger path looks identical to the auto-trigger path. This is intentional (same code path), but debug logs should include the originating `event_name` for traceability.

## References

- [ADR-0003](0003-store-lifecycle-and-state-machine.md), [ADR-0004](0004-store-publication-and-multi-catalog-projection.md), [ADR-0011](0011-domain-events.md), [ADR-0012](0012-cross-context-consistency.md), [ADR-0013](0013-first-party-idempotency.md), [ADR-0014](0014-soft-delete-and-audit.md), [ADR-0016](0016-authorization-details.md), [ADR-0017](0017-order-and-fulfillment.md), [ADR-0018](0018-organization-tenancy.md)
- [design/events.md](../design/events.md) — registry updated with new events
- [design/tenancy.md](../design/tenancy.md) — new use cases (added)
- CLAUDE.md §5.9 (publication and catalogs)
- handoff/03-beckn-integration.md §3.8 (catalog publishing)
