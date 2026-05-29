# Design: Audit

- **Status**: Draft
- **Last updated**: 2026-05-28
- **Backed by ADRs**: [ADR-0014](../decisions/0014-soft-delete-and-audit.md) (this), [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md) (actor + impersonator), [ADR-0011](../decisions/0011-domain-events.md) (events / subscriber pattern), [ADR-0012](../decisions/0012-cross-context-consistency.md) (retention separation)

## Purpose

Full design of the Audit bounded context — the `AuditRecord` entity, the subscription model, access rules, retention policy, and the system's posture on tamper resistance.

**What this document covers:**
- `AuditRecord` entity and invariants.
- Subscription configuration and filtering.
- Use cases (write side = ingestion; read side = query).
- Access control matrix.
- Retention policy.
- Tamper-resistance design.

**What it does NOT cover:**
- Audit search / query UX — Interface Layer concern.
- Cryptographic chaining (deferred per ADR-0014).
- PII scrubbing mechanism — [Gap 17](../gaps/17-pii-and-compliance.md).
- Compliance-specific retention values — operational.

## Position within the architecture

Audit is a bounded context (CLAUDE.md §2.5), alongside Tenancy, Catalog, Inventory, Promotion, Identity & Access.

It depends on:
- The event stream — subscribes via [ADR-0011](../decisions/0011-domain-events.md)'s outbox + dispatcher.
- The authorization model — read access enforced per [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md).
- PII conventions — [Gap 17](../gaps/17-pii-and-compliance.md) (forthcoming).

It is consumed by:
- **System Admins** — platform-wide history (incidents, debugging, compliance reports).
- **Platform-scoped operators** with the `audit.read.platform` capability.
- **Store Owners / Admins** — records concerning their store.
- **Users** — their own activity history (and impersonation history per ADR-0002).
- **External auditors** — via System Admin–mediated reports.

## Concepts

### AuditRecord

| Field | Description |
|---|---|
| `audit_id` | Opaque internal identifier |
| `event_id` | From the source event (dedup key) |
| `event_name` | E.g., `tenancy.store_status_changed` |
| `event_version` | From the source event |
| `recorded_at` | When the Audit subscriber ingested the event |
| `occurred_at` | From the source event |
| `actor` | `{ user_id, impersonated_user_id, active_store_id }` |
| `aggregate_type` | E.g., `Store`, `Product`, `Order`, `User` |
| `aggregate_id` | Identifier of the affected aggregate |
| `before_state` | Optional; previous state of a change event |
| `after_state` | Optional; new state of a change event |
| `summary` | Human-readable rendering of the event |
| `correlation_id` | For cross-event tracing |
| `source_envelope` | Full original event envelope (archival snapshot) |

`AuditRecord` is **immutable** once written. The only state transition is *expiry-and-deletion* by the retention cleanup role.

### Subscription configuration

Audit subscribes to events declared in the [event registry](events.md). The default includes **all mutation events**. Specific event types can be excluded by config (e.g., `identity.session_active_store_set` — high-volume, low-value).

The subscription respects the consumer contract from ADR-0011:
- Idempotent (dedup by `event_id` via inbox `processed_events`).
- Forward-compatible (unknown future `event_version` is logged and skipped).
- Tracks its own offset; replay-safe.

### RetentionPolicy

Per-`event_name` (or category) retention windows. Default values:

| Category | Default | Examples |
|---|---|---|
| Business state changes | 2 years | `tenancy.store_status_changed`, `catalog.product_published` |
| Identity & Session | 1 year | `identity.user_signed_in`, `identity.session_*` |
| Financial / Order / Voucher | 7 years | `promotion.voucher_redeemed`, `order.*` (when Gap 11 lands) |
| Inventory adjustments | 3 years | `inventory.stock_sold`, `inventory.stock_corrected` |
| Catalog-routine events | 1 year | `catalog.product_media_updated`, etc. |

Records older than their category's window are pruned by a scheduled job.

## Lifecycle

`AuditRecord` lifecycle: created → (retained) → deleted-on-expiry. No other transitions. No `UPDATE`.

## Invariants

- Audit records are **append-only at the DB role level**. The Audit subscriber's DB role has `INSERT` only on the audit table.
- No role has `UPDATE` on audit records.
- Only the retention-cleanup role has `DELETE`, scoped to expired records: `WHERE recorded_at + retention < now()`.
- One `AuditRecord` per `(subscription_name = audit, event_id)`. Duplicate deliveries no-op.
- `actor.user_id` and `actor.impersonated_user_id` are opaque references (never embed User identity beyond IDs).
- `source_envelope` is stored as-received; never re-serialized to a different format.

## Boundary contracts

### Use cases

**Write side (subscriber-driven, internal):**
- `IngestEvent(event)` — invoked by the dispatcher. Validates schema, transforms event into `AuditRecord`, writes. Idempotent by `event_id`.

**Read side (capability-gated per ADR-0002):**
- `GetAuditRecord(audit_id, acting_user) → AuditRecord | NotAllowed | NotFound`
- `ListRecordsForStore(store_id, filter, acting_user) → [AuditRecord]`
- `ListRecordsForUser(user_id, filter, acting_user) → [AuditRecord]`
- `SearchRecords(query, filter, acting_user) → [AuditRecord]` — platform-scope read.

All read use cases enforce the access matrix below before returning records.

### Domain events emitted

**None.** Audit is a sink, not a source. It consumes events; downstream consumers (analytics, alerting) subscribe to the original event stream, not to Audit.

### Cross-context references

Audit references entities by ID only. The `source_envelope` carries the original event data but is treated as opaque payload — Audit never joins or queries into other contexts based on its contents.

## Access matrix

| Reader | Records visible | Mechanism |
|---|---|---|
| **System Admin** | All | Full read; no filter |
| **Platform-scoped** with `audit.read.platform` | All (or subset per finer capability) | Capability check |
| **Store Owner / Admin** | `aggregate_type` belongs to their store, OR `actor.active_store_id` is their store | Filter applied at query time |
| **User** | `actor.user_id == self.id` OR `actor.impersonated_user_id == self.id` | Filter applied at query time |
| **Buyer** | None | Rejected |

Authorization checks happen in the Application Layer of Audit; the read use cases enforce them before returning.

## Tamper resistance

- DB role separation:
  - **`audit_subscriber`** — `INSERT` only on `audit_records`. No `UPDATE`, no `DELETE`.
  - **`audit_cleanup`** — `DELETE` only on `audit_records`, with row-level filter (`recorded_at + retention < now()`).
  - **`audit_reader`** — `SELECT` only.
  - **Application code** uses the appropriate role for each operation.
- No role has `UPDATE`. Records cannot be modified post-write.
- Audit records do **not** store passwords, MFA secrets, or other authentication credentials (none of which exist in our system anyway per ADR-0009).
- **Cryptographic chaining is out of scope for v1.** If compliance demands it later (financial-grade tamper evidence), a new ADR introduces a `prev_hash` field and a verification job.

## PII considerations (preview)

`AuditRecord.source_envelope` may contain PII when the source event payload does (e.g., `identity.user_signed_in` includes the user's IP; profile-sync events carry an email change before/after). Gap 17 will refine:

- Which fields are scrubbable for right-to-erasure compliance.
- Whether scrubbing replaces with placeholders or removes outright.
- The DB role and use case authorized to perform scrubbing.

For v1, audit records may carry PII per their source events; Gap 17 overlays the compliance mechanism on top.

## Subscriber registry update

The `audit` subscriber (declared in [design/events.md](events.md)) is the concrete implementation of this design:

- **Name**: `audit`
- **Event filter**: all mutation events across all contexts, minus a configurable exclusion list for high-volume noise.
- **Handler**: `IngestEvent` — idempotent via the standard inbox dedup.
- **Persistence**: writes to `audit_records` table; deletion via separate scheduled cleanup using the `audit_cleanup` role.

## Open questions (within this design)

- **Audit query / search UX** — read-side; Interface Layer concern.
- **Higher-level "audit operation" composition** — events grouped into business operations as a derived read; future.
- **Cryptographic chaining** — only if compliance demands.
- **Cross-tenant audit summarization** — analytics surface; not in audit's scope.

## References

- [ADR-0014](../decisions/0014-soft-delete-and-audit.md), [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md), [ADR-0011](../decisions/0011-domain-events.md), [ADR-0012](../decisions/0012-cross-context-consistency.md)
- [design/events.md](events.md) — event registry and subscriber registry
- CLAUDE.md §2.5 (bounded contexts), §5.16 (events), §5.17 (consistency), §5.19 (soft delete + audit)
- Related gaps: [17](../gaps/17-pii-and-compliance.md) (PII / compliance)
