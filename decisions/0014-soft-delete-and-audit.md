# ADR-0014: Soft-delete pattern and the Audit context

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/16-soft-delete-and-audit.md](../gaps/resolved/16-soft-delete-and-audit.md)
- **Codifies**: the no-delete pattern already adopted by [ADR-0003](0003-store-lifecycle-and-state-machine.md), [ADR-0005](0005-catalog-and-product-modeling.md), [ADR-0006](0006-inventory-model.md), [ADR-0007](0007-pricing-tax-and-vouchers.md), [ADR-0009](0009-identity-and-external-idp.md), [ADR-0010](0010-invitation-account-reconciliation.md)
- **Builds on**: [ADR-0002](0002-authorization-tiers-and-matrix.md) (actor + impersonator), [ADR-0011](0011-domain-events.md) (events / Audit as subscriber), [ADR-0012](0012-cross-context-consistency.md) (separate retention)
- **Sub-design**: [design/audit.md](../design/audit.md)

## Context

Gap 16 bundled two concerns: a system-wide stance on **soft delete**, and the design of the **audit log**. The first is mostly already decided implicitly — every business entity introduced by prior ADRs follows the same "lifecycle-replaces-delete" pattern. This ADR codifies it explicitly. The second has more open ends: audit record shape, what's audited, who can read, retention, and tamper resistance.

ADR-0011 already declared Audit as a downstream subscriber. ADR-0012 declared Audit has its own (longer) retention. ADR-0002 required actor + impersonator capture in audit attribution. This ADR finishes the picture.

## Options considered

### Soft delete stance

**A. Per-entity ad-hoc decisions** (current state per §3.2). Rejected — leaves the door open to inconsistency.

**B. System-wide pattern: lifecycle-replaces-delete for business entities; hard-delete for operational entities** — chosen. Codifies what every prior ADR already did.

### Audit position

**A. Audit as an Infrastructure cross-cut** (middleware). Rejected — Audit has its own data, rules, retention, and consumers; that's a context.

**B. Audit as a bounded context** — chosen. Joins §2.5 alongside Tenancy, Catalog, etc.

### Audit record granularity

**A. One record per consumed event** — chosen. Simple; replay-friendly; defers higher-level composition.

**B. Higher-level "audit operations" composed from multiple events.** Rejected for v1; can be added as a derived read view later.

### Tamper resistance

**A. Append-only by convention.** Rejected — too easy to violate by mistake.

**B. Append-only at the DB role / permission level** — chosen. Subscriber role has `INSERT` only; cleanup role has `DELETE` only on expired rows.

**C. Cryptographic chaining** (each record hashes the prior). Deferred — only if compliance demands it later.

## Decision

### 1. Soft-delete pattern (codification)

The system divides entities into two categories with different deletion semantics:

**Business entities — never deleted.**

End-of-life is a **state transition to a terminal-but-retained state**, not deletion:

| Entity | Terminal-retained state |
|---|---|
| Store | Suspended / Paused (always reversible per ADR-0003) |
| Product | Archived (ADR-0005) |
| ProductVariant | Removed (retained if referenced by orders) |
| StockLevel | Inactive (ADR-0006) |
| User | Disabled (ADR-0009) |
| Voucher | Disabled, or derived Expired / Exhausted (ADR-0007) |
| Invitation | Accepted / Declined / Revoked / Expired (ADR-0010) |
| Membership | Removed (record retained for audit) |

Data is retained indefinitely (subject to PII compliance — [Gap 17](../gaps/resolved/17-pii-and-compliance.md)). Identifiers (slugs, internal IDs) are bound to their entity for life — no reuse after end-of-life.

**Operational entities — hard-deleted on schedule.**

These exist for operational reasons and carry no business-history value:

| Entity | Deletion mechanism |
|---|---|
| Session | Hard-deleted on sign-out, sign-out-everywhere, or expiry |
| Idempotency records | Hard-deleted after retention window (default 24h, ADR-0013) |
| Inbox `processed_events` | Pruned alongside event-log retention (ADR-0012) |
| Outbox dispatched rows | Pruned after retention (ADR-0011) |
| Stuck events | Hard-deleted after operator skip-with-acknowledgement |

Cleanup is operational (background jobs, DB TTLs). The domain accepts no responsibility for these records beyond their useful window.

**Default for new entities.** If a new entity does not clearly belong to the operational category, it is a business entity — no deletion. Adding a "deleted" status to a business entity (e.g., user-initiated account closure) requires a new ADR.

### 2. Audit is a bounded context

Audit joins the §2.5 bounded-context list. It owns its own data, rules, retention, and read interface. Like every other context, it communicates only through declared channels — events in, read queries out — and never reaches into another context's data.

### 3. Audit record schema

One `AuditRecord` per consumed event:

| Field | Description |
|---|---|
| `audit_id` | Opaque internal identifier |
| `event_id` | From the source event (dedup key per ADR-0011) |
| `event_name` | E.g., `tenancy.store_status_changed` |
| `event_version` | From the source event |
| `recorded_at` | When the Audit subscriber ingested it |
| `occurred_at` | From the source event |
| `actor` | `{ user_id, impersonated_user_id, active_store_id }` per ADR-0002 |
| `aggregate_type` | E.g., `Store`, `Product`, `Order` |
| `aggregate_id` | The affected aggregate's identifier |
| `before_state` | Optional; for change events with a previous state |
| `after_state` | Optional; for change events with a new state |
| `summary` | Human-readable rendering of the event |
| `correlation_id` | For tracing across events |
| `source_envelope` | The full original event envelope (archival) |

`source_envelope` preserves the original event even if the event schema evolves over time.

### 4. Subscription scope

Audit subscribes to **all mutation events** across all contexts. Specific noise events can be excluded via configuration (e.g., `identity.session_active_store_set` is high-volume and low-information). The default posture is "audit everything that changes state."

Audit dedups by `event_id` using the inbox pattern (ADR-0012).

### 5. Access control

Per ADR-0002's capability model:

| Reader | Scope |
|---|---|
| **System Admin** | Full platform-wide read |
| **Platform-scoped user** with the `audit.read.platform` capability | Cross-tenant read per capability |
| **Store Owner / Admin** | Records where `aggregate_type` belongs to their store, OR `actor.active_store_id` is their store |
| **User** | Records where `actor.user_id == self.id` OR `actor.impersonated_user_id == self.id` — i.e., events the user was party to (as actor or as impersonated) |

Buyers cannot read audit records.

### 6. Retention

Configurable per `event_name` (or category). Defaults:

| Category | Default retention |
|---|---|
| Business state changes (Store, Product, Membership lifecycle) | 2 years |
| Identity & Session events | 1 year |
| Financial / Order / Voucher redemption events | 7 years |
| Inventory adjustments | 3 years |
| Catalog-routine events (media updates, etc.) | 1 year |

Retention values are infrastructure config. The design supports per-category windows; specific values are tunable by compliance (Gap 17).

### 7. Tamper resistance

- **Append-only at the DB role level.** The Audit subscriber's DB role has `INSERT` only on the audit table. No role has `UPDATE`. A separate cleanup role has `DELETE` permission scoped only to expired rows (`WHERE recorded_at + retention < now()`).
- **Cryptographic chaining** (each record references the hash of the prior) — **deferred**. If compliance demands it later, a new ADR adds it without disturbing the schema.

## Consequences

What this commits to:

- The system-wide soft-delete rule is **codified**: business entities never deleted; operational entities pruned on schedule. New entities are reviewed against this rule.
- **Audit is a bounded context** — joins §2.5.
- Audit ingests all mutation events, transforms them into AuditRecords, and stores them with longer retention than the event log.
- The schema is event-mapped (one-to-one); higher-level operation views can be derived later.
- Audit is **append-only at the DB role level**; tamper resistance is at the role / permission boundary.
- Read access is capability-gated per ADR-0002; store-scoped users see only their own store; users see their own actions.
- `source_envelope` preserves originals; subscribers reading old records must tolerate old event versions.

What this defers:

- **Audit query / search UX** — read-side concern.
- **Per-category retention values** — operational, refined by Gap 17.
- **Cryptographic chaining** — only if compliance requires.
- **Higher-level audit operations** (composed from multiple events) — read-side derivation; future enhancement.
- **PII handling within audit records** — Gap 17 owns the right-to-erasure mechanism (likely PII scrubbing in `source_envelope` rather than record deletion).

What this makes harder:

- **Genuine record deletion** for compliance — solved by scrubbing PII fields in place rather than deleting whole records (Gap 17).
- **Schema evolution of source events** — `source_envelope` keeps the original; subscribers reading old audit records may see old envelopes.
- **High-volume operational events** (storefront chatter, etc.) — if audited unconditionally, storage balloons. Configurable filtering required.

## References

- [gaps/resolved/16-soft-delete-and-audit.md](../gaps/resolved/16-soft-delete-and-audit.md)
- [design/audit.md](../design/audit.md) — full context model
- [ADR-0002](0002-authorization-tiers-and-matrix.md) (actor + impersonator), [ADR-0003](0003-store-lifecycle-and-state-machine.md), [ADR-0005](0005-catalog-and-product-modeling.md), [ADR-0006](0006-inventory-model.md), [ADR-0007](0007-pricing-tax-and-vouchers.md), [ADR-0009](0009-identity-and-external-idp.md), [ADR-0010](0010-invitation-account-reconciliation.md), [ADR-0011](0011-domain-events.md), [ADR-0012](0012-cross-context-consistency.md), [ADR-0013](0013-first-party-idempotency.md)
- Related gap: [17](../gaps/resolved/17-pii-and-compliance.md) (PII / compliance — refines audit retention and PII handling)
