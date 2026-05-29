# Design: PII Handling

- **Status**: Draft
- **Last updated**: 2026-05-28
- **Backed by ADRs**: [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md) (this), [ADR-0009](../decisions/0009-identity-and-external-idp.md) (Identity), [ADR-0014](../decisions/0014-soft-delete-and-audit.md) (Audit)

## Purpose

The authoritative **PII catalog** (per entity, per event payload), the **scrub-action registry**, and the **third-party processor catalog**.

**What this document covers:**
- Field-level PII catalog across every entity and event type that carries PII.
- Scrub-action types and what each PII field becomes after scrubbing.
- The `ScrubUser` use-case orchestration across contexts.
- Third-party processor catalog.
- Logging, residency, and encryption baselines.

**What it does NOT cover:**
- Specific encryption product choices — operational.
- Specific logger / masking patterns — operational.
- Compliance certification mechanics — operational.
- Per-jurisdiction retention values (audit retention lives in `design/audit.md`).

## Position within the architecture

PII handling is a **cross-cutting policy layer** overlaid on:

- Identity & Access (canonical PII)
- Audit (PII snapshots in `source_envelope`)
- Tenancy (Invitation invitee email; Membership references)
- Order & Fulfillment (future; buyer PII at order time)
- All contexts via event payloads (snapshot PII per event type)

It is enforced via:
- Per-entity and per-event-type catalog (this document)
- The `ScrubUser` use case
- DB role separation in Audit
- The redacting logger in Infrastructure

## PII catalog

### Identity & Access context

#### User entity

| Field | PII type | Scrub action |
|---|---|---|
| `email` | Email | `replace_email` |
| `display_name` | Name | `replace_name` |
| `avatar_url` | URL (potentially identifying) | `nullify` |
| `external_subject_id` | Identifier (links to IdP) | `replace_external_subject` |
| `preferred_locale` | Not PII | (untouched) |
| `status` | Not PII | Set to `Disabled` if not already |
| `id`, timestamps | Not PII | (untouched; needed for foreign references) |

#### Session entity

| Field | PII type | Scrub action |
|---|---|---|
| `last_ip` | IP address | `nullify` |
| `device_label` | Free-text (may carry name) | `nullify` |
| Other fields | Not PII | (untouched) |

Sessions are operational entities (ADR-0014) — on user Disable, all of a user's sessions are hard-deleted. PII fields are also scrubbed in partial-cleanup scenarios for safety.

### Tenancy context

#### Invitation entity

| Field | PII type | Scrub action |
|---|---|---|
| `email` | Email | `replace_invitation_email` (deterministic on invitation_id) |
| Other fields | Not PII | (untouched) |

Note: Invitations typically reach a terminal state (Accepted / Declined / Revoked / Expired) before the recipient's right-to-erasure would meaningfully apply. Accepted invitations link to the User; that User's scrubbing handles the personal data trace.

### Audit context

#### AuditRecord entity

| Field | PII type | Scrub action |
|---|---|---|
| `source_envelope` | Contains snapshot PII per source event type | Walk per the source event's PII catalog (below); update affected fields in place |
| All other fields | Not PII (IDs, timestamps, summary) | (untouched) |

Only the `audit_pii_scrubber` DB role can `UPDATE` `source_envelope` field-by-field.

### Event payload catalog

Only events that carry PII are listed. Subscribers must be aware that these fields are scrubbable.

#### `identity.user_created`
| Path | PII type | Scrub action |
|---|---|---|
| `payload.email` | Email | `replace_email` |
| `payload.display_name` | Name | `replace_name` |

#### `identity.user_signed_in`
| Path | PII type | Scrub action |
|---|---|---|
| `payload.email` | Email | `replace_email` |
| `payload.ip` | IP | `nullify` |

#### `identity.user_profile_synced`
| Path | PII type | Scrub action |
|---|---|---|
| `payload.before.email` | Email | `replace_email` |
| `payload.after.email` | Email | `replace_email` |
| `payload.before.display_name` | Name | `replace_name` |
| `payload.after.display_name` | Name | `replace_name` |

#### `identity.user_avatar_changed`
| Path | PII type | Scrub action |
|---|---|---|
| `payload.before` | URL | `nullify` |
| `payload.after` | URL | `nullify` |

#### `identity.session_created`
| Path | PII type | Scrub action |
|---|---|---|
| `payload.ip` | IP | `nullify` |
| `payload.device_label` | Free-text | `nullify` |

#### `tenancy.invitation_created`
| Path | PII type | Scrub action |
|---|---|---|
| `payload.email` | Email | `replace_invitation_email` |
| `payload.issuer_email` | Email | `replace_email` (per issuer user) |

#### `tenancy.invitation_accepted`
| Path | PII type | Scrub action |
|---|---|---|
| `payload.user_email` | Email | `replace_email` |

#### `promotion.voucher_redeemed`
| Path | PII type | Scrub action |
|---|---|---|
| `payload.user_email` | Email | `replace_email` (when carried for receipt purposes) |

(Order context's events, when Gap 11 lands, will add buyer-name, address, phone fields here.)

## Scrub-action types

| Action | Definition |
|---|---|
| `replace_email` | Replace with `user-<hash(user_id)>@scrubbed.local` (deterministic per owner) |
| `replace_name` | Replace with the literal string `[Removed user]` |
| `replace_invitation_email` | Replace with `<hash(invitation_id)>@scrubbed.local` (per invitation, not per user) |
| `replace_external_subject` | Replace with `scrubbed:<hash(user_id)>` |
| `nullify` | Set to null |
| `hash` | Replace with a stable hash (used when downstream consumers need stable identity comparison without raw data) |

Hash inputs include a platform-wide salt so hashes are not globally guessable.

## `ScrubUser` orchestration

`Identity.ScrubUser(user_id, by_actor, reason)`:

1. Verify acting actor has the `identity.user.scrub` capability (System Admin or platform-scoped with capability).
2. Look up the User. If status != `Disabled`, transition to `Disabled` first (per ADR-0009).
3. In Identity: apply the User catalog scrub.
4. In Identity: hard-delete all Sessions for the user (operational scrub).
5. In Tenancy: invoke `Tenancy.ScrubUserReferences(user_id)` — scrubs `Invitation.email` for invitations issued to this user, etc.
6. (When Order/Gap 11 lands) In Order: invoke `Order.ScrubUserReferences(user_id)` — scrubs buyer PII in order records.
7. Emit `identity.user_pii_scrubbed` event with actor, reason, and timestamp.
8. Audit subscribes; ingests as a standard `AuditRecord`.
9. **Audit's PII-scrub job** (out-of-band background process using the `audit_pii_scrubber` DB role) walks audit records where:
   - `actor.user_id == user_id`, OR
   - `actor.impersonated_user_id == user_id`, OR
   - `source_envelope` contains references to this user
   
   For each, it applies field-level scrubs per the event payload catalog.

The full scrub is **eventually consistent across contexts** (per ADR-0012). If any cross-context port call fails, the User is in a partial-scrub state until retry. The `ScrubUser` operation is **idempotent** — replays are safe and re-apply what's already in placeholder form (no-op).

## Third-party processor catalog

| Processor | PII received | Purpose | Notes |
|---|---|---|---|
| **IdP** (Clerk / Auth0 / Supabase Auth / Cognito / Keycloak) | email, name, locale, picture | Authentication; canonical user profile | Bound by IdP's own DPA |
| **Email service** | recipient email, recipient name | Transactional email (invitations, notifications) | Data minimization: only send the recipient field needed |
| **Beckn network participants** | buyer name, address, phone (when Order/Gap 11 lands) | Order fulfillment | Bound by Beckn network agreements |
| **Observability vendor** | redacted logs only | Operational visibility | Raw PII MUST NOT reach this processor |

When a new processor is integrated, this catalog MUST be updated.

## Logging discipline

- All domain and application code uses the **PII-aware redacting logger** (Infrastructure).
- Patterns (operational; implementation):
  - Email: masked (`u***@example.com`) or hashed
  - IP: not in plaintext
  - Names: not co-logged with `user_id`
  - Free-text user input: not logged unless explicitly sanitized
- Raw PII never appears in logs.
- The redacting logger consumes this PII catalog to know what to mask.

## Data residency

- **Default**: Indonesia-resident storage (PDP compliance).
- **Multi-region**: not in v1.
- Enforcement is operational deployment; domain imposes no constraint.

## Encryption

- **In transit**: TLS, baseline.
- **At rest**: DB-level encryption, baseline.
- **Field-level encryption**: deferred per field; added only when a specific field demands it (e.g., national IDs if ever collected).

## Open questions (within this design)

- **Cryptographic chaining** for tamper-evidence on audit records — deferred per ADR-0014; revisit when compliance demands.
- **Per-region data residency** with cross-region replication — operational; future.
- **Field-level encryption** triggers per specific field — added per requirement.
- **Customer self-serve erasure UX** — Interface Layer, future.
- **Erasure proof certificate** (formal receipt of erasure for the requesting user) — useful for compliance; future enhancement.

## References

- [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md) (this), [ADR-0009](../decisions/0009-identity-and-external-idp.md), [ADR-0014](../decisions/0014-soft-delete-and-audit.md)
- [design/events.md](events.md) (the event registry this catalog overlays), [design/identity.md](identity.md), [design/audit.md](audit.md)
- CLAUDE.md §5.20
