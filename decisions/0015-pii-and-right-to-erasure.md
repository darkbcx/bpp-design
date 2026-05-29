# ADR-0015: PII handling and right-to-erasure

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/17-pii-and-compliance.md](../gaps/resolved/17-pii-and-compliance.md)
- **Builds on**: [ADR-0009](0009-identity-and-external-idp.md) (Identity owns PII), [ADR-0011](0011-domain-events.md) (events carry payloads), [ADR-0014](0014-soft-delete-and-audit.md) (Audit, append-only)
- **Sub-design**: [design/pii.md](../design/pii.md)

## Context

Gap 17 is the policy overlay for personally identifiable information. The PII landing zones are already known from prior ADRs:

- **Identity (User)** — canonical: email, display_name, avatar_url.
- **Identity (Session)** — last_ip, device_label.
- **Tenancy (Invitation)** — invitee email at creation time.
- **Audit** — `source_envelope` may carry snapshot PII from the source event.
- **Order (future, Gap 11)** — buyer name, address, phone.

Compliance regime: **Indonesia's PDP law** (Undang-Undang Pelindungan Data Pribadi), with GDPR compatibility as a sensible architectural target.

The hard architectural challenge: how does **right-to-erasure** reconcile with the no-delete pattern from ADR-0014 and tax-record retention requirements?

## Options considered

### Right-to-erasure mechanism

**A. Hard delete user + cascade.** Rejected — breaks the no-delete pattern (ADR-0014), tax retention, audit completeness.
**B. PII scrubbing in place** — chosen. The record remains; PII fields are replaced with deterministic placeholders.

### PII in event payloads

**A. Forbid PII in events; carry only IDs; subscribers fetch at consumption time.** Rejected — audit becomes thin (can't show "their email was X at the time of this action"); incident investigation suffers.
**B. Allow PII as event snapshots; scrub on erasure** — chosen. Trade-off: more useful audit; more involved scrubbing logic.

### Auto-scrub on Disable

**A. Configurable window; default off** — chosen. Explicit erasure is the regulatory primitive; auto-scrub is an option for stricter regimes.
**B. Mandatory auto-scrub after Disable.** Rejected — over-prescriptive for v1.

## Decision

### 1. Right-to-erasure: PII scrubbing in place

When a User exercises right-to-erasure (or is auto-scrubbed):

- The **User record is retained**. `id` and `external_subject_id` stay; foreign references throughout the system remain valid.
- **PII fields are replaced** with deterministic placeholders per the [PII catalog](../design/pii.md).
- The User's `status` transitions to `Disabled` if not already.
- A `ScrubUser(user_id, by_actor, reason)` use case in Identity & Access initiates and orchestrates the scrub across contexts.
- The scrub operation emits `identity.user_pii_scrubbed`; Audit ingests it as a standard record.

### 2. PII catalog

A formal registry — [`design/pii.md`](../design/pii.md) — declares which fields across which entities and event payloads contain PII, and what scrub action each takes.

Each catalog entry declares:
- **Field path** (entity.attribute, or event.payload.path)
- **PII type** (email, name, phone, address, IP, free-text)
- **Scrub action** (e.g., `replace_email`, `replace_name`, `nullify`, `hash`)

When new entities or events with PII are introduced, the catalog MUST be updated at the same time.

### 3. PII boundary across contexts

- **Outside of events**: Identity & Access owns raw PII. Other contexts hold only `user_id`. (Already the rule from ADR-0009; formalized here.)
- **Inside events**: PII MAY be carried as a snapshot for audit usefulness, subject to the catalog. Subscribers must treat PII-tagged fields as scrubbable.
- **Logs, search indices, analytics outputs**: by-reference only — raw PII does not appear here unless that consumer is the legitimate recipient (email service receiving the email it must send).

### 4. Audit append-only constraint relaxes for scrubbing

ADR-0014 declared Audit records append-only at the DB role level. PII scrubbing requires field-level `UPDATE` on `source_envelope`. The resolution:

- A dedicated **`audit_pii_scrubber`** DB role has field-level `UPDATE` on `source_envelope` **only**. All other audit fields remain immutable.
- The role is used exclusively by the scrub use case via the Audit context's scrub port.
- All other audit roles (subscriber, reader, cleanup) retain their original constraints from ADR-0014.

### 5. Auto-scrub on Disable

- **Default: off.** Explicit erasure is the regulatory baseline.
- Operationally enabled per environment / compliance need.
- When enabled, an auto-scrub job processes Users in `Disabled` state where `disabled_at + auto_scrub_window < now()`.
- Default window if enabled: 1 year (operational config).
- Auto-scrub uses the same `ScrubUser` use case and emits the same event.

### 6. Logging discipline

Domain and application code MUST use the **PII-aware redacting logger** (Infrastructure Layer). Raw PII never appears in logs.

Specific masking patterns are operational. The **architectural rule** is that:
- All structured logging passes through the redacting logger.
- Domain code never writes to stdout / generic loggers directly.
- The redacting logger consumes the PII catalog to know what to mask.

### 7. Cross-store / platform-wide analytics

- **Aggregate analytics** (counts, distributions, geographic heatmaps without per-store identification) — permitted.
- **Per-user or per-store data** shared across tenants — forbidden without explicit consent.
- Platform-wide analytics queries operate on aggregated views; raw PII never crosses tenant boundaries.

### 8. Third-party processor catalog

The system maintains a **processor catalog** (in [`design/pii.md`](../design/pii.md)) listing every third party that receives PII:

| Processor | PII received | Purpose |
|---|---|---|
| IdP (Clerk / Auth0 / Supabase Auth / etc.) | email, name, locale, picture | Authentication; canonical user profile |
| Email service | recipient email + name | Transactional email |
| Beckn network participants | buyer order info (Gap 11) | Order fulfillment |
| Observability vendor | redacted logs only | Operational visibility |

**Data minimization**: each processor receives only what it needs. New processors update the catalog at design time.

### 9. Encryption baseline

- **In transit**: TLS for all external communication. Baseline.
- **At rest**: DB-level encryption. Baseline.
- **Field-level encryption** for highly sensitive specific fields (e.g., national IDs if ever collected): on demand per field; not blanket.

### 10. Data residency

- **Indonesia-resident storage** by default (PDP compliance).
- Multi-region storage is not in v1.
- Residency enforcement is an operational deployment concern; the domain imposes no constraint.

## Consequences

What this commits to:

- A `ScrubUser` use case in Identity & Access, with cross-context PII-scrubbing orchestration.
- A PII catalog in `design/pii.md`, kept current with the entity and event registries.
- A dedicated `audit_pii_scrubber` DB role; the Audit append-only constraint is qualified accordingly.
- A PII-aware redacting logger is required infrastructure; all domain / application logging routes through it.
- Platform analytics work on aggregated views only; PII does not cross tenant boundaries.
- A processor catalog is maintained at design time; data minimization in transit.
- Encryption at rest and in transit is baseline; field-level encryption per-field on demand.

What this defers:

- **Field-level encryption** for specific sensitive fields — when needed.
- **Specific masking patterns** for the redacting logger — operational.
- **Data residency enforcement mechanisms** — operational.
- **Compliance certification** — operational, legal-driven.
- **Order-context PII** specifics — Gap 11.
- **Cryptographic chaining for tamper-evidence** — remains deferred per ADR-0014.

What this makes harder:

- **Cross-context scrubbing** — a single erasure touches multiple contexts. Orchestrated by `ScrubUser`; eventually consistent across contexts per ADR-0012. Idempotent.
- **Audit append-only contract** — qualified to permit `source_envelope` field updates only via the dedicated role. Reviewers must police this boundary carefully.
- **Provability of erasure** without cryptographic chaining — relies on operational audit logs of scrub actions. Acceptable for v1.
- **Adding new entities or events with PII** — must update the catalog at the same time; not after.

## References

- [gaps/resolved/17-pii-and-compliance.md](../gaps/resolved/17-pii-and-compliance.md)
- [design/pii.md](../design/pii.md) — PII catalog, scrub actions, processor catalog
- [ADR-0009](0009-identity-and-external-idp.md), [ADR-0011](0011-domain-events.md), [ADR-0014](0014-soft-delete-and-audit.md)
- Indonesia PDP law — Undang-Undang Pelindungan Data Pribadi (UU PDP)
- GDPR — compatibility reference
