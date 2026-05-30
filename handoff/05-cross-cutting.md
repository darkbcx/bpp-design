# 5. Cross-cutting concerns

This section captures the patterns that apply *across* all bounded contexts. Every context — Identity, Tenancy, Catalog, Inventory, Promotion, Order, Audit — uses the rules in this section. If you're implementing in any context, you'll touch these patterns.

The section covers seven concerns:

| # | Topic | Backed by |
|---|---|---|
| 5.1 | Authorization | ADR-0002, ADR-0016, ADR-0018 |
| 5.2 | Domain events | ADR-0011 + `design/events.md` |
| 5.3 | Cross-context consistency | ADR-0012 |
| 5.4 | First-party idempotency | ADR-0013 |
| 5.5 | Soft-delete pattern | ADR-0014 |
| 5.6 | PII handling and right-to-erasure | ADR-0015 + `design/pii.md` |
| 5.7 | Localization | ADR-0008 |

---

## 5.1 Authorization

Every Application-Layer use case begins with an **explicit `requireCapability(name, scope)` call** before any state mutation or side effect. There is no implicit authorization, no shared ambient context where "everyone has access by default."

> Backed by: [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md) (tiers + matrix + active store), [ADR-0016](../decisions/0016-authorization-details.md) (capability naming + decision pattern + denial auditing), [ADR-0018](../decisions/0018-organization-tenancy.md) (Tenant-scoped tier rename).

### 5.1.1 Three tiers

In decreasing privilege:

- **System Admin** — meta-admins. Manage the role → capability matrix; configure platform settings. Seeded via deploy config; no in-band lifecycle.
- **Platform-scoped** — platform operators (support, compliance, network ops). May act across tenants per assigned capabilities. Not Members of any Organization by virtue of platform role.
- **Tenant-scoped** — Members of one or more Organizations. Two sub-roles:
  - **Org Owner** — full authority over the Org and all stores within. Plenary access.
  - **Store Admin** — administrative access to *specific* stores within the active Org via `StoreAdminAssignment` (per [§4.2](04-bounded-contexts/4.2-tenancy.md)).
  - Baseline **Org Member** (no store assignments): holds only `org.view`.

The **role catalog** is system-defined at design time. Adding a new role is a feature change, not runtime configuration.

### 5.1.2 Capability catalog — action-level naming

Capabilities follow `<resource>.<verb>` — lowercase, snake_case verb. Examples:

| Domain | Examples |
|---|---|
| Catalog | `product.create`, `product.publish`, `product.archive`, `product.read` |
| Catalog (Variants / Categories) | `product.variant.add`, `product.variant.remove`, `category.create` (System Admin) |
| Tenancy (Store) | `store.create`, `store.activate`, `store.pause`, `store.suspend`, `store.republish` |
| Tenancy (Org) | `org.create`, `org.invite_member`, `org.remove_member`, `org.transfer_ownership`, `org.create_store`, `org.assign_store_admin`, `org.republish_all_stores` |
| Promotion | `voucher.create`, `voucher.disable`, `voucher.enable`, `voucher.read` |
| Inventory | `inventory.adjust`, `inventory.read` |
| Identity & Access | `user.disable`, `user.reactivate`, `user.scrub`, `user.impersonate` |
| Audit | `audit.read.platform`, `audit.read.store` |
| System Admin | `system.matrix.edit`, `system.category.manage` |

The full capability catalog grows when new features are added — each adds one or more capabilities.

### 5.1.3 The matrix — grants only

The role → capability matrix is configured at runtime, exclusively by System Admins. **Absence of an entry means denied.** No explicit denies, no precedence rules.

Special cases like "admins can do everything except delete users" are modeled as the **explicit absence** of `user.scrub` from the admin role's grants — not as a deny rule.

### 5.1.4 The decision pattern

Every use case begins with:

```
ctx.requireCapability("product.publish", scope={ store_id: <id> })
```

The Tenancy context exposes this port:

```
interface AuthorizationPort {
  // Throws AuthorizationDenied on failure.
  requireCapability(name: string, scope?: { org_id?, store_id? }): void

  // Returns boolean; used for conditional UI.
  hasCapability(name: string, scope?): boolean
}
```

**Scope rules:**

- **Tenant-scoped users**: the `requireCapability` call applies an **active-org filter** (the target must belong to the user's `active_org_id`). For store-specific capabilities, an additional **active-store filter** applies (target store must belong to the active Org).
- **Platform-scoped / System Admin users**: scope is informational (the check verifies tier-level capability).
- **Anonymous calls**: rejected before reaching the use case.

### 5.1.5 Active Org and Active Store (session attributes)

Per [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md) + [ADR-0018](../decisions/0018-organization-tenancy.md), the session carries:

- `active_org_id` — required for tenant-scoped actions; chosen via UI switcher when the user belongs to multiple Orgs.
- `active_store_id` — optional within the Active Org; required for store-specific actions.
- URL paths reflect the hierarchy: `/orgs/<org-slug>/...` for Org-level, `/orgs/<org-slug>/stores/<store-slug>/...` for store-level. The URL must match session values per request.

`active_org_id = null` is valid only for System Admin / Platform-scoped users in their no-tenant mode.

### 5.1.6 Impersonation (downward only)

Strictly downward along the tier hierarchy:

- System Admin → (Platform-scoped | Tenant-scoped)
- Platform-scoped → Tenant-scoped
- Same-tier and upward impersonation **forbidden**.

During impersonation: the impersonator's effective capabilities become **exactly** those of the impersonated user. Destructive actions are allowed. The impersonation session is time-limited (operational config).

**Audit attribution**: every action records both the real actor and the impersonated user, with the real actor as the responsible party.

### 5.1.7 Denial auditing — two-tier

Authenticated denials (a real user attempted an action they lack capability for):
- Use case raises `AuthorizationDenied`.
- Tenancy emits `identity.authorization_denied` event with `{ actor, capability_name, scope, reason }`.
- Audit ingests as a standard record.

Anonymous denials (no session — bot probing):
- Handled at the Interface Layer (HTTP 401/403); logged at the **Infrastructure Layer** only.
- **Not** emitted as domain events (avoids swamping the outbox).

---

## 5.2 Domain events

Cross-context communication and audit ingestion flow through a **transactional outbox + asynchronous dispatcher**. Every event-emitting context writes events to a local `outbox` table in the **same DB transaction** as the state change; a background dispatcher delivers them to subscribers.

> Backed by: [ADR-0011](../decisions/0011-domain-events.md) (event substrate); [`design/events.md`](../design/events.md) (full event registry — currently ~70 declared event types across all contexts).

### 5.2.1 Envelope

Every event carries the same envelope:

| Field | Description |
|---|---|
| `event_id` | UUID; the dedup key |
| `event_name` | `<context>.<verb_past>` (e.g., `catalog.product_published`) |
| `event_version` | Integer; payload schema version |
| `occurred_at` | Domain time |
| `recorded_at` | Outbox write time |
| `aggregate_type` | `Product`, `Store`, `Order`, etc. |
| `aggregate_id` | The aggregate this event pertains to |
| `actor` | `{ user_id, impersonated_user_id, active_org_id, active_store_id }` — impersonation pair per [§5.1.6](#516-impersonation-downward-only) |
| `correlation_id` | Traces causally related events from one operation |
| `causation_id` | The `event_id` of the event that caused this one (for chains) |
| `payload` | Event-type-specific data |

### 5.2.2 Naming

Format: `<context>.<verb_past>` — lowercase, snake_case, past-tense, namespaced by emitting context.

Examples: `catalog.product_published`, `tenancy.store_status_changed`, `inventory.stock_reserved`, `promotion.voucher_redeemed`, `identity.session_active_store_set`.

Forbidden: present-tense (those are commands, not events), cross-context names, wire/Beckn vocabulary in event names.

### 5.2.3 Delivery semantics: at-least-once

- Producers always commit (state + outbox) atomically.
- Dispatcher delivers each subscriber at least once.
- **Subscribers MUST dedup by `event_id`** — using an inbox table (per [§5.3](#53-cross-context-consistency)).
- Exactly-once is not provided.

### 5.2.4 Ordering

- **Per-aggregate**: events for the same `aggregate_id` are delivered in `occurred_at` order.
- **Cross-aggregate** and **cross-context**: not guaranteed.

Subscribers needing cross-aggregate sequencing implement buffering/reordering themselves (rare).

### 5.2.5 Versioning

- Each event type carries `event_version`.
- **Additive** payload changes (new optional field) — no version bump.
- **Breaking** changes (removed field, type change, required-ness) — bump version. Multiple versions may coexist in the stream.
- Subscribers handle versions they understand; unknown future versions are logged and skipped (forward-compatible).
- The event registry in `design/events.md` is the authoritative catalog. **New events / version bumps update the registry.**

### 5.2.6 Retention and replay

- Events are durable for an **operational retention window** (default 90 days).
- Within retention, subscribers can replay from any offset.
- Beyond retention, events are deleted (or archived per compliance).
- Cold-start beyond retention: query current state via Application Layer, then attach live.

Audit retention is **independent and longer** — see [§5.5](#55-soft-delete-pattern) and [§4.7](04-bounded-contexts/4.7-audit.md).

### 5.2.7 Known subscribers (v1)

| Subscriber | Filter | Purpose |
|---|---|---|
| `beckn-bridge` | `tenancy.store_status_changed`, all `catalog.*` mutations, selected `promotion.*` and `tenancy.*_republish_requested` | Re-project resources onto the Beckn network (via CDS publish) |
| `audit` | Broad — most mutation events | Compliance and history ingestion |
| `inventory-catalog-subscriber` | `catalog.product_created`, `_variant_added`, `_archived`, `_variant_removed`, `_restored` | Auto-manage `StockLevel` lifecycle |

The Beckn Bridge does NOT emit domain events (it emits Beckn protocol messages, which are not domain events).

### 5.2.8 Topology neutrality

Same envelope, naming, and semantics work in both:

- **Modular monolith** — dispatcher is an in-process worker calling subscriber handlers directly.
- **Distributed services** — dispatcher publishes to a message bus (Kafka, NATS, SQS, …); subscribers consume from topics.

The deployment choice is operational.

---

## 5.3 Cross-context consistency

The system is **strongly consistent within a context, eventually consistent across contexts**. This shapes how every multi-context operation is built.

> Backed by: [ADR-0012](../decisions/0012-cross-context-consistency.md).

### 5.3.1 The consistency model

- **Within a context**: strong. A transaction commits all writes for that context's aggregates atomically.
- **Across contexts**: eventual via events. Expect a small (typically millisecond-scale) lag.

### 5.3.2 No cross-context transactions

A single Application-Layer use case writes to **one context only** in its transaction. Cross-context state propagation is **always via events**.

Even in a modular monolith — where multiple contexts may share a database — code does not span contexts in a single transaction. This preserves the service-extraction path declared in [§2.7](02-principles.md).

### 5.3.3 Multi-context flow orchestration (no Sagas v1)

Flows spanning multiple contexts (most notably `Order.Initiate`, `Order.Confirm`, `Order.Cancel` — see [§4.6](04-bounded-contexts/4.6-order.md)) are coordinated by **Application-Layer orchestration** with **hand-coded compensation** on failure.

Example pattern (from [§4.6 Order](04-bounded-contexts/4.6-order.md) Initiate):

```
reservations = []
for line in line_items:
  reservations.append( Inventory.Reserve(...) )

try:
  Promotion.ValidateVoucher(...)
except VoucherInvalid:
  for r in reservations: Inventory.ReleaseReservation(r.id)   # explicit compensation
  raise
```

**No formal Saga framework in v1.** All v1 flows complete in seconds and fit in a single request lifecycle. Sagas may be reconsidered if payment-gateway integration introduces real long-running flows.

### 5.3.4 Read freshness — per-query

- **Transactional reads** (those that gate a state change — `Inventory.Reserve` reading `StockLevel`, `Order.Confirm` reading the active reservation, etc.) MUST be synchronous against the owning context.
- **Browse-time reads** (Admin UI dashboards, search) MAY be eventual / cached / projection-based when those are added later. v1 may serve these synchronously for simplicity.

Read tolerance is **declared per-query**, not globally.

### 5.3.5 Failure handling — retry-then-stuck

When a subscriber fails to process an event:

- **Retry with exponential backoff** (operational config: initial delay, multiplier, max attempts).
- After N attempts, the event is **stuck** — flagged in the outbox or moved to a `stuck_events` table. The subscriber's offset does NOT advance.
- Stuck events surface as **metrics and alerts** for ops.
- An operator either fixes the underlying cause and retries, or **explicitly skips with acknowledgement** (audited).
- **No silent drops, no automatic skips.**

### 5.3.6 Inbox dedup

Each subscriber maintains a `processed_events` table keyed by `(subscription_name, event_id)`. Before invoking the handler, the framework checks this table — duplicate deliveries become no-ops.

Together with the outbox (per [§5.2](#52-domain-events)), this is the **outbox + inbox** pattern.

### 5.3.7 Cross-context ports

A context **never** imports another context's internal types or storage. Cross-context calls go through **Application-Layer ports** defined by the **calling** context, implemented by the callee as an adapter.

- In a monolith: ports resolve to in-process method calls.
- In service-extracted deployment: ports resolve to RPC / HTTP / gRPC.
- Calling code is identical either way.

---

## 5.4 First-party idempotency

Mutating use cases that create new state support **client-supplied idempotency keys** to dedup retries from UIs, mobile apps, and internal API clients. Together with the outbox (producer side) and inbox (subscriber side), this completes the at-least-once safety posture.

> Backed by: [ADR-0013](../decisions/0013-first-party-idempotency.md).

### 5.4.1 The pattern

- Caller generates a **fresh UUID per logical operation** and sends it with the request (HTTP header `Idempotency-Key` or equivalent for non-HTTP transports).
- Retries of the same logical operation reuse the same UUID.
- Different operation = different UUID.
- Server does NOT derive keys from request inputs.

### 5.4.2 Conflict semantics

| Scenario | Server behavior |
|---|---|
| Same `(user, key)` arrives again with **same payload** | Returns the cached result; no second state change, no second event. |
| Same `(user, key)` arrives with **different payload** | Returns typed error `idempotency_key_reused_with_different_payload`. Surfaces client bugs. |

### 5.4.3 Storage

Each context with mutating use cases maintains an `idempotency_records` table:

| Field | Description |
|---|---|
| `user_id` | Scoping; different users can use the same key |
| `idempotency_key` | UUID supplied by client |
| `request_fingerprint` | Stable hash of canonical request payload (for mismatch detection) |
| `result_snapshot` | Serialized result of the use case |
| `created_at` | When original execution committed |

Primary key: `(user_id, idempotency_key)`. Retention: **default 24 hours** (operational, tunable).

### 5.4.4 Transactional coherence

The idempotency record is written **in the same DB transaction** as state changes and outbox rows:

```
BEGIN;
  -- state mutation
  -- outbox row(s)
  -- idempotency_records (key, fingerprint, result_snapshot)
COMMIT;
```

Crash before COMMIT → no record, no state, no event. Retry runs cleanly as first execution.
Crash after COMMIT but before responding → retry finds the record; cached result returned. State change and event already happened.

### 5.4.5 Scope rules

- **Mutating create-new-state use cases**: `idempotency_key` supported (default).
- **Naturally-idempotent operations** (set state to target value — `SetStoreStatus(Paused)`, `UpdateProductMedia(...)`): no key needed; calling them twice produces the same outcome.
- **Reads**: never use idempotency keys.

### 5.4.6 The three-plane at-least-once posture

Together with the outbox (producers, [§5.2](#52-domain-events)) and inbox (subscribers, [§5.3.6](#536-inbox-dedup)), idempotency keys give the system safety at every layer:

| Layer | Mechanism |
|---|---|
| Producer (state → event) | Transactional outbox |
| Subscriber (event → handler) | Inbox dedup |
| Client (intent → use case) | Idempotency keys |

All three write **transactionally with state**. Any one of them ensures the system is safe under at-least-once delivery; together they make the system resilient end-to-end.

---

## 5.5 Soft-delete pattern

The system divides entities into two categories with different deletion semantics. **Business entities are never deleted.** End-of-life is a state transition to a terminal-but-retained state. **Operational entities** are hard-deleted on schedule.

> Backed by: [ADR-0014](../decisions/0014-soft-delete-and-audit.md) (codifies the pattern + introduces Audit).

### 5.5.1 Business entities — never deleted

End-of-life is a state transition; data is retained indefinitely (subject to PII compliance, [§5.6](#56-pii-handling-and-right-to-erasure)). Identifiers (slugs, internal IDs) are bound to their entity for life — no reuse.

| Entity | Terminal-retained state |
|---|---|
| Organization | `Suspended` (per [ADR-0018](../decisions/0018-organization-tenancy.md)) |
| Store | `Suspended` / `Paused` (always reversible per [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md)) |
| Product | `Archived` (per [ADR-0005](../decisions/0005-catalog-and-product-modeling.md)) |
| ProductVariant | Removed (retained if referenced by orders) |
| StockLevel | `Inactive` (per [ADR-0006](../decisions/0006-inventory-model.md)) |
| User | `Disabled` (per [ADR-0009](../decisions/0009-identity-and-external-idp.md)) |
| Voucher | `Disabled`, or derived `Expired` / `Exhausted` (per [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md)) |
| Invitation | `Accepted` / `Declined` / `Revoked` / `Expired` (per [ADR-0010](../decisions/0010-invitation-account-reconciliation.md)) |
| OrganizationMember | `Removed` (record retained) |
| StoreAdminAssignment | `Removed` (record retained) |
| Order | `Fulfilled` / `Cancelled` / `Expired` (per [ADR-0017](../decisions/0017-order-and-fulfillment.md)) |

### 5.5.2 Operational entities — hard-deleted on schedule

These exist for operational reasons and carry no business-history value:

| Entity | Deletion mechanism |
|---|---|
| Session | Hard-deleted on sign-out, sign-out-everywhere, or expiry |
| Idempotency records | Hard-deleted after retention window (default 24h, [§5.4](#54-first-party-idempotency)) |
| Inbox `processed_events` | Pruned alongside event-log retention |
| Outbox dispatched rows | Pruned after retention |
| Stuck events | Hard-deleted after operator skip-with-acknowledgement |

Cleanup is operational — background jobs, DB TTLs. The domain accepts no responsibility for these records beyond their useful window.

### 5.5.3 Default for new entities

If a new entity does not clearly belong to the operational category, **it is a business entity — no deletion.** Adding a "deleted" status to a business entity requires a new ADR.

### 5.5.4 Relationship to right-to-erasure

For PII compliance, **scrubbing replaces deletion** (per [§5.6](#56-pii-handling-and-right-to-erasure)): the entity record stays; PII fields are replaced with placeholders. This honors both the no-delete invariant *and* the regulatory right to erasure.

---

## 5.6 PII handling and right-to-erasure

PII is a cross-cutting policy overlay across every context that touches personal data. The compliance regime is **Indonesia's PDP law** (Undang-Undang Pelindungan Data Pribadi), GDPR-compatible by design.

> Backed by: [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md) (PII policy); [`design/pii.md`](../design/pii.md) (PII catalog and scrub-action registry).

### 5.6.1 PII landing zones

| Context | PII held |
|---|---|
| Identity & Access | `email`, `display_name`, `avatar_url` (User); `last_ip`, `device_label` (Session) |
| Tenancy | `email` on Invitation invitee |
| Order & Fulfillment | Buyer `contact_snapshot` (name, email, phone, address) — populated at `/init` |
| Audit | `source_envelope` may contain PII from source events |
| All contexts (via events) | PII may flow in event payloads as snapshots |

**What we don't store** (per [ADR-0009](../decisions/0009-identity-and-external-idp.md)): passwords, MFA secrets, recovery tokens — all at the IdP.

### 5.6.2 Right-to-erasure: PII scrubbing in place

When a User exercises right-to-erasure (or is auto-scrubbed):

- The **User record is retained** (foreign references stay valid).
- **PII fields are replaced with deterministic placeholders** per the PII catalog.
- Audit records' `source_envelope` PII fields are scrubbed in place; the record itself stays.
- The User's `status` transitions to `Disabled` if not already.
- A `ScrubUser(user_id, by_actor, reason)` use case in Identity orchestrates the scrub across contexts.
- Emits `identity.user_pii_scrubbed`.

Cross-context scrubbing is **eventually consistent** (per [§5.3](#53-cross-context-consistency)). Idempotent — retries are safe.

### 5.6.3 PII catalog

A formal registry — [`design/pii.md`](../design/pii.md) — declares which fields contain PII and what scrub action each takes:

| Action | Behavior |
|---|---|
| `replace_email` | Replace with `user-<hash(user_id)>@scrubbed.local` (deterministic) |
| `replace_name` | Replace with `[Removed user]` |
| `nullify` | Set to null |
| `hash` | Replace with stable hash (when downstream consumers need stable identity) |

New entities or events introducing PII MUST update the catalog at introduction.

### 5.6.4 PII boundary across contexts

- **Outside of events**: Identity & Access owns raw PII; other contexts hold only `user_id`. Per [ADR-0009](../decisions/0009-identity-and-external-idp.md).
- **Inside events**: PII MAY be carried as a snapshot (for audit usefulness); subscribers must treat PII-tagged fields as scrubbable.
- **Logs, search indices, analytics**: by-reference only — raw PII does not appear except when that consumer is the legitimate recipient (e.g., email service receiving the email).

### 5.6.5 Audit append-only constraint, qualified

[§5.5](#55-soft-delete-pattern) plus [ADR-0014](../decisions/0014-soft-delete-and-audit.md) made AuditRecord append-only at the DB role level. PII scrubbing requires a narrow exception:

- A dedicated **`audit_pii_scrubber`** DB role has field-level `UPDATE` on `source_envelope` only.
- No other role has UPDATE on audit records.
- The role is used exclusively by `ScrubUser`'s Audit-side step.

### 5.6.6 Logging discipline

All domain and application code uses a **PII-aware redacting logger** (Infrastructure Layer):

- Email masked (e.g., `u***@example.com`).
- IP not in plaintext (or hashed).
- Names not co-logged with `user_id`.
- Free-text user input not logged unless explicitly sanitized.

Raw PII never appears in logs. The redacting logger consumes the PII catalog to know what to mask.

### 5.6.7 Cross-store / platform analytics

- **Aggregate analytics** (counts, distributions, heatmaps without per-store identification): permitted.
- **Per-user or per-store data shared across tenants**: forbidden without explicit consent.
- Platform analytics work on aggregated views; raw PII never crosses tenant boundaries.

### 5.6.8 Third-party processor catalog

Maintained in [`design/pii.md`](../design/pii.md):

| Processor | PII received | Purpose |
|---|---|---|
| IdP | email, name, locale, picture | Authentication |
| Email service | recipient email + name | Transactional email (invitations, etc.) |
| Beckn network participants | buyer name, address, phone (via Bridge) | Order fulfillment |
| Observability vendor | redacted logs only | Operational visibility |

Data minimization in transit: each processor receives only what it needs.

### 5.6.9 Encryption baseline

- TLS in transit (baseline).
- DB-level encryption at rest (baseline).
- Field-level encryption per specific field — on demand only (not blanket).

### 5.6.10 Data residency

- Default: **Indonesia-resident storage** (PDP compliance).
- Multi-region not in v1.
- Residency is operational deployment, not domain.

---

## 5.7 Localization

Multi-language content is a v1 feature. Every translatable field uses a **`LocalizedText` value object**; the platform default locale is `id` (Bahasa Indonesia).

> Backed by: [ADR-0008](../decisions/0008-localization-and-localizedtext.md).

### 5.7.1 `LocalizedText` value object

```
LocalizedText {
  entries: Map<bcp47_tag, string>
  // Invariant: entries always contains an entry for the platform default ("id")
}
```

Reads use `get(locale)` which returns the locale-specific value if present, else falls back to the default. `get_strict(locale)` returns null if absent (no fallback).

### 5.7.2 Locale tags — BCP 47

All locale tags follow BCP 47. Examples: `id`, `en`, `en-US`, `en-ID`, `ms`, `jv`, `su`, `zh-Hans`.

### 5.7.3 Translatable fields

Subject to `LocalizedText`:

| Context | Fields |
|---|---|
| Tenancy (Store) | name, description, public contact display |
| Tenancy (Organization) | name, description |
| Catalog (Product) | name, description |
| Catalog (ProductAttribute, Matrix) | name, value labels |
| Catalog (Media) | alt_text |
| Catalog (PlatformCategory) | name |
| Promotion (Voucher) | description |

**Locale-neutral** (single string or non-string regardless of language):
- Identifiers, slugs, SKUs, voucher codes.
- `Money` amounts, ISO 4217 currency codes, timestamps, dates.
- Boolean flags, enum values, status fields.

### 5.7.4 `Store.supported_locales`

An ordered list of BCP 47 tags the store declares it publishes in. Always includes `id`; default at creation is `[id]`. Owners can add more. Used by:

- The Beckn provider descriptor.
- Admin UX for which language fields to surface.

Adding a locale does NOT require backfilling translations — missing locales fall back to the default.

### 5.7.5 Uniqueness on default locale

Where uniqueness applies to a translatable field (e.g., `Product.name` unique within store), the check uses the **default-locale value** (`id`). Cross-locale collisions are not checked at v1.

### 5.7.6 Bridge locale handling

When the Bridge projects content into Beckn responses:

- The BAP's request includes a language preference via `context.language`.
- For each `LocalizedText` field, the Bridge calls `get(requested_locale)` — present → return; absent → return default.
- The Bridge declares the chosen locale in the response per descriptor block.
- **No automatic translation.** If the BAP wanted `ja` and the store has only `id` + `en`, the BAP receives the `id` text.

### 5.7.7 LocalizedText in event payloads

Per [ADR-0008](../decisions/0008-localization-and-localizedtext.md), events carrying translatable content carry the **full `LocalizedText`** (all locales), not a single rendering. Subscribers resolve their locale at consumption time. This is essential for Audit (records must be locale-agnostic for compliance).

### 5.7.8 Currency vs. locale

- **Currency** is a Store-level attribute (per [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md)): one currency per store, immutable after first Active product.
- **Locale** is per-field (`LocalizedText`) and per-Store-declared (`supported_locales`).

The two are orthogonal. A store can publish content in many locales but uses one currency.

---

> **Next**: [§6 Operational stance](06-operational.md) — testing, observability, deployment topology, external dependencies.
