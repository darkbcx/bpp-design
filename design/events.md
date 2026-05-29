# Design: Domain Events

- **Status**: Draft
- **Last updated**: 2026-05-28
- **Backed by ADRs**: [ADR-0011](../decisions/0011-domain-events.md) (this), plus every ADR that emits or consumes events ([ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md), [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md), [ADR-0005](../decisions/0005-catalog-and-product-modeling.md), [ADR-0006](../decisions/0006-inventory-model.md), [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md), [ADR-0009](../decisions/0009-identity-and-external-idp.md), [ADR-0010](../decisions/0010-invitation-account-reconciliation.md))

## Purpose

Full design of the domain-event system: envelope, outbox pattern, dispatcher behavior, subscriber model, naming/versioning rules, **and the registry of every event type currently declared across all contexts.**

**What this document covers:**
- The envelope schema in detail.
- The `outbox` table shape.
- Dispatcher behavior and failure handling.
- Subscriber contract.
- Naming and versioning rules.
- The **event registry** — the authoritative list of every event the system uses.
- The **subscriber registry** — known subscribers and what they consume.

**What this document does NOT cover:**
- Audit record schema — [Gap 16](../gaps/16-soft-delete-and-audit.md).
- Saga / process-manager patterns for multi-event flows — [Gap 14](../gaps/14-cross-context-consistency.md).
- Specific message-bus product (Kafka vs. NATS vs. …) — operational.
- Beckn projection logic — lives in the Bridge's mapping registry.

## Position within the architecture

Domain events are emitted by **Application-Layer use cases** in each context (after state changes). They are written transactionally to a context-local `outbox` table. A dispatcher (Infrastructure Layer) reads the outbox and delivers events to subscribers.

```
[Application Layer of context]
    │ writes state + writes outbox row (same TX)
    ▼
[outbox table]
    │
    ▼
[Dispatcher (Infrastructure)]
    │ delivers
    ▼
[Subscribers: Bridge / Audit / Inventory / future read-models / future services]
```

## The envelope

Every event carries this envelope, regardless of dispatch mechanism:

| Field | Type | Description |
|---|---|---|
| `event_id` | UUID | Unique per event instance. Dedup key. |
| `event_name` | string | `<context>.<verb_past>`, e.g., `catalog.product_published`. |
| `event_version` | integer | Schema version of the payload. Starts at 1. |
| `occurred_at` | ISO-8601 timestamp | Domain time — when the fact became true. |
| `recorded_at` | ISO-8601 timestamp | When the event was written to the outbox. |
| `aggregate_type` | string | E.g., `Product`, `Store`, `Order`, `User`, `Session`, `StockLevel`. |
| `aggregate_id` | string | Opaque identifier of the affected aggregate. |
| `actor` | object | `{ user_id, impersonated_user_id, active_store_id }`. Each field nullable; all-null indicates a system action. |
| `correlation_id` | string \| null | Traces causally-linked events from a single operation across contexts. |
| `causation_id` | UUID \| null | The `event_id` of the event that directly caused this one. |
| `payload` | object | Event-type-specific data; schema versioned per `event_version`. |

### Actor field

Carries the impersonation pair from [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md):

- `user_id` — the real actor (the human or service that initiated the action).
- `impersonated_user_id` — non-null when an admin acted as another user; the user whose identity was assumed.
- `active_store_id` — the session's active store at the time, when relevant.

Audit relies on this structure to attribute actions correctly.

## Outbox table shape

Each context owns a local `outbox` table:

| Column | Type | Description |
|---|---|---|
| `id` | bigserial / sequence | Primary key (for ordering by insertion) |
| `event_id` | UUID | Globally unique; matches envelope |
| `event_name` | string | |
| `event_version` | integer | |
| `occurred_at` | timestamp | |
| `recorded_at` | timestamp | Set on insert by the DB |
| `aggregate_type` | string | |
| `aggregate_id` | string | |
| `actor` | JSON | |
| `correlation_id` | string \| null | |
| `causation_id` | UUID \| null | |
| `payload` | JSON | |
| `dispatched_at` | timestamp \| null | NULL until dispatcher succeeds |
| `dispatch_attempts` | integer | Default 0; incremented on each retry |
| `last_dispatch_error` | text \| null | Most recent failure message, for ops |

The Application Layer writes to `outbox` in the **same transaction** as the state change. The dispatcher polls (or listens to a DB notification) for rows where `dispatched_at IS NULL`.

## Dispatcher

Responsibilities:
- Continuously read undispatched rows from each context's outbox.
- Deliver each event to every registered subscriber that filters it in.
- On subscriber success: mark `dispatched_at = now()` (per subscriber, in practice — see below).
- On subscriber failure: increment `dispatch_attempts`, record error, retry with exponential backoff.
- Surface stuck rows (e.g., dispatch_attempts > N) for ops review via metrics/logs.

In a modular monolith, the dispatcher is an in-process background worker. In a distributed deployment, the dispatcher publishes to a message bus and subscriber processes consume independently — each manages its own offset/position.

### Per-subscriber tracking

Because multiple subscribers may consume the same event, tracking is per-subscriber. The outbox itself records initial dispatch; each subscriber maintains its own position table (`subscription_name`, `last_processed_event_id`, `last_processed_position`). This decouples slow subscribers from fast ones.

## Subscriber contract

A subscriber declares:

| Field | Description |
|---|---|
| `name` | Unique identifier (e.g., `beckn-bridge`, `audit`, `inventory-catalog-subscriber`) |
| `event_filter` | List of `(event_name, min_version)` tuples or wildcards (`catalog.*`) |
| `handler` | Idempotent function: `(event) → success | retry | fatal` |
| `concurrency` | Single-threaded or parallel processing options |

### Guarantees expected from subscribers

- **Idempotent** — handling the same `event_id` twice produces the same result.
- **Forward-compatible** — unknown `event_version` is logged and skipped, not crashed on.
- **Replay-safe** — replaying any window of events yields a consistent state.

## Naming rules

Format: `<context>.<verb_past>`

- Lowercase.
- snake_case.
- Past-tense verb (the event is a fact about something that happened).
- Namespaced by the **emitting** context.

Examples:
- `catalog.product_published`
- `tenancy.store_status_changed`
- `inventory.stock_reserved`
- `identity.session_active_store_set`

Forbidden:
- Present-tense (commands, not events): no `catalog.publish_product`.
- Cross-context names: an event belongs to one emitter.
- Wire/UI/Beckn vocabulary in event names (per §2.6, §4).

## Versioning rules

- Each event type carries an integer `event_version`.
- **No-bump changes** (additive, backward-compatible):
  - Adding an optional field to `payload`.
  - Adding an optional field to `actor` (extending the actor schema).
- **Bump-required changes** (breaking):
  - Removing a field.
  - Changing a field's type or required-ness.
  - Renaming a field.
  - Changing the semantic meaning of an existing field.
- During a transition, both old and new versions may appear in the stream. Subscribers either handle both or one.
- A retired version's emission stops only after all subscribers have migrated. Then producers stop emitting it.
- This registry is updated whenever an event type changes.

## Event registry

The full set of currently-declared events. Each is `event_version: 1` unless noted.

### Tenancy context

Store lifecycle ([ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md)):
- `tenancy.store_created`
- `tenancy.store_submitted_for_activation` ([ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md))
- `tenancy.store_status_changed` — payload includes `from_state`, `to_state`
- `tenancy.ownership_transferred` (CLAUDE.md §6.6)

Memberships:
- `tenancy.membership_created`
- `tenancy.membership_role_changed`
- `tenancy.membership_removed`

Invitations ([ADR-0010](../decisions/0010-invitation-account-reconciliation.md)):
- `tenancy.invitation_created`
- `tenancy.invitation_accepted`
- `tenancy.invitation_declined`
- `tenancy.invitation_revoked`
- `tenancy.invitation_expired`

### Identity & Access context

User lifecycle ([ADR-0009](../decisions/0009-identity-and-external-idp.md)):
- `identity.user_created`
- `identity.user_signed_in`
- `identity.user_profile_synced` — when IdP-canonical fields change
- `identity.user_preferred_locale_changed`
- `identity.user_avatar_changed`
- `identity.user_disabled`
- `identity.user_reactivated`

Sessions:
- `identity.session_created`
- `identity.session_ended`
- `identity.session_active_store_set`
- `identity.session_active_store_cleared`

### Catalog context

Products ([ADR-0005](../decisions/0005-catalog-and-product-modeling.md)):
- `catalog.product_created`
- `catalog.product_updated`
- `catalog.product_published`
- `catalog.product_archived`
- `catalog.product_restored`
- `catalog.product_media_updated`

Variants (Matrix mode):
- `catalog.product_variant_attribute_added`
- `catalog.product_variant_added`
- `catalog.product_variant_updated`
- `catalog.product_variant_removed`
- `catalog.product_variant_media_updated`

Categories:
- `catalog.product_category_assigned`
- `catalog.product_category_unassigned`
- `catalog.platform_category_created`
- `catalog.platform_category_renamed`
- `catalog.platform_category_moved`
- `catalog.platform_category_deprecated`

Store catalogs ([ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md)):
- `catalog.store_catalog_created`
- `catalog.store_catalog_renamed`
- `catalog.store_catalog_deleted`
- `catalog.product_included_in_catalog`
- `catalog.product_removed_from_catalog`
- `catalog.product_excluded_from_default_catalog`
- `catalog.product_restored_in_default_catalog`

### Inventory context

Stock movements ([ADR-0006](../decisions/0006-inventory-model.md)):
- `inventory.stock_level_created`
- `inventory.stock_level_inactivated`
- `inventory.stock_received`
- `inventory.stock_sold`
- `inventory.stock_returned`
- `inventory.stock_corrected`
- `inventory.stock_reserved`
- `inventory.stock_released`
- `inventory.purchasable_toggled`

### Promotion context

Vouchers ([ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md)):
- `promotion.voucher_created`
- `promotion.voucher_updated`
- `promotion.voucher_disabled`
- `promotion.voucher_enabled`
- `promotion.voucher_redeemed`

### Order & Fulfillment context

Deferred to [Gap 11](../gaps/11-order-and-fulfillment-phasing.md).

## Subscriber registry (v1)

| Subscriber | Event filter | Purpose |
|---|---|---|
| **`beckn-bridge`** | `tenancy.store_status_changed`, all `catalog.*` mutations, selected `promotion.*` | Re-project resources onto the Beckn network ([ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md), [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md)) |
| **`audit`** | Broad subscription across all contexts | Compliance and history ingestion ([Gap 16](../gaps/16-soft-delete-and-audit.md)) |
| **`inventory-catalog-subscriber`** | `catalog.product_created`, `catalog.product_variant_added`, `catalog.product_archived`, `catalog.product_variant_removed`, `catalog.product_restored` | Auto-create / inactivate `StockLevel` per [ADR-0006](../decisions/0006-inventory-model.md) |

Future subscribers (not in v1):
- Storefront read-side projections (event-sourced product catalogs for fast queries).
- Analytics ingestion.
- Notification service (email reminders, etc.).

## Localization in event payloads

Per [ADR-0008](../decisions/0008-localization-and-localizedtext.md), payloads carrying translatable content (e.g., a `catalog.product_updated` event with a new `name`) **carry the full `LocalizedText`**, not a single rendering. Subscribers that need a specific locale call `.get(locale)` at consumption time.

## Beckn vocabulary in events

Per §2.6 and §4, domain events MUST NOT carry Beckn / transport / UI vocabulary. The Bridge consumes domain events and translates them into Beckn messages — never the reverse. Events stay in domain language.

## Open questions (within this design)

- **Saga / process-manager pattern** for multi-event flows (e.g., order placement coordinating Inventory + Promotion + Payment) — [Gap 14](../gaps/14-cross-context-consistency.md).
- **Storefront read models** (event-driven projections) — Infrastructure; future work.
- **Exact retention values per context** — operational tuning; default 90 days.

## References

- [ADR-0011](../decisions/0011-domain-events.md) (this)
- All other ADRs that emit/consume events
- CLAUDE.md §2.6 (inter-context communication), §5.16 (domain events)
