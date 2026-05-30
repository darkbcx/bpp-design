# Design: Identity & Access

- **Status**: Draft
- **Last updated**: 2026-05-28
- **Backed by ADRs**: [ADR-0009](../decisions/0009-identity-and-external-idp.md) (this gap), [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md) (active store), [ADR-0008](../decisions/0008-localization-and-localizedtext.md) (locale), [ADR-0018](../decisions/0018-organization-tenancy.md) (active org), [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md) (User scope: tenant/platform only; buyers are not Users)

## Purpose

Full design of the Identity & Access context — the entities, relationships, lifecycle, the integration with an external IdP, and the contracts other contexts consume.

**What this document covers:**
- `User` and `Session` entities, in full.
- The IdP integration adapter shape (OIDC).
- Sign-in / Just-in-Time provisioning flow.
- Profile sync mechanics.
- Session lifecycle and operations.
- Boundary contracts (use cases, events, queries) exposed to other contexts.

**What this document does NOT cover:**
- Specific IdP product selection — operational.
- Right-to-erasure / PII scrubbing — [Gap 17](../gaps/17-pii-and-compliance.md).
- Audit log shape — [Gap 16](../gaps/16-soft-delete-and-audit.md).
- Multi-IdP linking — deferred per ADR-0009.
- Anonymous / guest checkout — [Gap 11](../gaps/11-order-and-fulfillment-phasing.md).

## Position within the architecture

The Identity & Access context sits in its own bounded context (CLAUDE.md §2.5). The Domain Layer holds `User` and `Session`; the Application Layer exposes use cases; the Infrastructure Layer holds the IdP integration adapter.

Other contexts that consume Identity:
- **Tenancy** — references `User.id` from Memberships and audit attribution.
- **Catalog**, **Inventory**, **Promotion**, **Order** — actor identity for use-case authorization.
- **Beckn Bridge** — does **not** project User data to the network; buyers on the wire are referenced by opaque order-level identifiers.

## Concepts

### User

A platform-level identity, mirrored from the configured IdP via the `external_subject_id` link.

Attributes:

| Field | Source | Mutability | Notes |
|---|---|---|---|
| `id` | Internal opaque | Immutable | Cross-context reference key |
| `external_subject_id` | IdP `sub` | Immutable | **Unique** within platform |
| `email` | IdP `email` | **IdP-canonical** | Refreshed every sign-in; unique within platform (case-insensitive) |
| `email_verified_at` | IdP `email_verified` | **IdP-canonical** | Nullable; set when IdP last reported verified |
| `display_name` | IdP `name` | **IdP-canonical** | Single string (not LocalizedText) |
| `preferred_locale` | IdP `locale` (at provisioning) | **Platform-owned, mutable** | BCP 47; defaults to platform default `id` |
| `avatar_url` | IdP `picture` (at provisioning) | **Platform-owned, mutable** | URL; nullable |
| `status` | System | `Active` \| `Disabled` | No deletion |
| `created_at`, `updated_at`, `last_signed_in_at`, `last_profile_synced_at` | System | — | Timestamps |

### Session

A platform-issued session attached to a User. Independent of the IdP token.

Attributes:

| Field | Description |
|---|---|
| `id` | Opaque token (cryptographically random; HTTP-only cookie) |
| `user_id` | Reference to `User` |
| `created_at` | Timestamp |
| `last_used_at` | Refreshed on any authenticated request |
| `expires_at` | Idle timeout; default 30 days from `last_used_at` |
| `active_org_id` | Nullable; per [ADR-0018](../decisions/0018-organization-tenancy.md). Required for tenant-scoped actions; null for System Admin / platform-scoped users in their no-tenant mode |
| `active_store_id` | Nullable; per [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md). When set, must belong to the active Organization |
| `device_label` | Optional human-readable device descriptor (e.g., "iPhone 14, Safari") |
| `last_ip` | Last observed client IP (for security review) |

### IdP Integration (adapter)

Lives in the Infrastructure Layer. Implements OIDC client semantics against the configured IdP. Exposes a port consumed by the Application Layer.

Configuration (deployment config, not code):
- `issuer_url`, `client_id`, `client_secret`
- `redirect_uri`
- Requested scopes (at minimum `openid email profile`)
- JWKS URL (for ID-token signature verification)

## Relationships

```
User (Identity & Access)
  ├── 1..* Session
  ├── 1..* Membership (in Tenancy, referenced by user_id)
  ├── *    audit-actor references (across contexts)
  ├── *    VoucherUsage (in Promotion, referenced by user_id)
  └── *    Order (in Order, referenced by user_id)
```

`User.id` is the only attribute other contexts reference. They never see `external_subject_id`, `email`, or any IdP-specific data.

## Lifecycles

### User lifecycle

States: `Active`, `Disabled`. No deletion.

| From → To | Actor | Trigger |
|---|---|---|
| (new) → Active | System | JIT provisioning at first successful IdP sign-in |
| Active → Disabled | System Admin / Platform-scoped role with capability | Admin action |
| Disabled → Active | System Admin / Platform-scoped role with capability | Admin action |

When a User is `Disabled`:
- All their active Sessions are terminated.
- Subsequent sign-in attempts (even with valid IdP authentication) are rejected with a typed error.
- Their record remains; Memberships, orders, audit references stay attached.
- The IdP account is unaffected.

### Session lifecycle

States: implicit (existence = active; absence = ended).

Operations:
- **Create** — at successful sign-in.
- **Refresh `last_used_at`** — on each authenticated request.
- **Set / clear `active_org_id`** — via the active-Org switcher (per ADR-0018).
- **Set / clear `active_store_id`** — via the active-store switcher within the active Org.
- **End (sign-out, single)** — deletes the session.
- **End-all (sign-out-everywhere)** — deletes all sessions for a User.

Idle expiry: Sessions where `last_used_at + idle_window < now` are considered expired and rejected. A background job MAY prune expired sessions; expired sessions are also rejected lazily on request.

## Sign-in flow (JIT provisioning)

```
1. Visitor clicks "Sign in" → GET /auth/start
2. Our app generates `state` (CSRF token) and `nonce`; stores them server-side keyed
   by a short-lived cookie
3. Redirect to IdP authorization endpoint:
   { client_id, redirect_uri, response_type=code, scope, state, nonce }
4. IdP authenticates (sign-up if first time at IdP; sign-in if returning)
5. IdP redirects → GET /auth/callback?code=...&state=...
6. Our app validates state (CSRF)
7. Our app exchanges code at IdP token endpoint → receives id_token, access_token
8. Our app validates id_token:
   - Signature via JWKS
   - Issuer matches configured `issuer_url`
   - Audience matches `client_id`
   - Expiry not passed
   - Nonce matches the one issued in step 2
9. Our app extracts claims: sub, email, email_verified, name, picture, locale
10. Look up User by external_subject_id:
    a. If found (returning user):
       - Refresh email, email_verified_at, display_name from current claims
       - Update last_signed_in_at, last_profile_synced_at
       - If User.status == Disabled → reject with typed error
    b. If not found (JIT provisioning):
       - Create User with:
         external_subject_id = sub
         email = email
         email_verified_at = now if email_verified else null
         display_name = name
         preferred_locale = locale if valid BCP 47 else platform default "id"
         avatar_url = picture if present
         status = Active
       - Emit `UserCreated` domain event
11. Create Session record:
    - id = new opaque token
    - user_id = user.id
    - created_at = now
    - last_used_at = now
    - expires_at = now + 30d (configurable)
    - active_org_id = null
    - active_store_id = null
    - device_label = derived from User-Agent
    - last_ip = client IP
12. Set HTTP-only cookie with Session id
13. Emit `UserSignedIn`, `SessionCreated`
14. Redirect to the visitor's original destination (or default landing)
```

## Profile sync rules

On every successful sign-in (step 10a above):

- Refresh **IdP-canonical** fields from current claims: `email`, `email_verified_at`, `display_name`.
- Update `last_profile_synced_at`.
- Do **not** touch **platform-owned** fields: `preferred_locale`, `avatar_url`.
- If a refresh would create an `email` collision with another User (rare — possible if IdP re-issues an old email), reject the sign-in with a typed error; manual remediation by platform admin.

Emit `UserProfileSynced` if any IdP-canonical field changed.

## Invariants

- `User.id` is immutable; assigned at creation.
- `User.external_subject_id` is immutable; unique within platform.
- `User.email` is unique within platform (case-insensitive).
- `Session.id` is opaque, unguessable, single-use as a session identifier (not reusable).
- `Session.user_id` is immutable; sessions belong to one User forever.
- A `Disabled` User has zero active Sessions (transition deletes them).

## Boundary contracts

### Use cases exposed (Application Layer)

**Sign-in / sign-out**
- `BeginSignIn(redirect_after_url) → idp_authorization_url`
- `CompleteSignIn(code, state) → Session` *(handles JIT provisioning, profile sync, disabled check)*
- `SignOut(session_id)`
- `SignOutEverywhere(user_id)`

**Session operations**
- `RefreshSessionActivity(session_id)` — bumps `last_used_at`
- `SetActiveStore(session_id, store_id)` — per [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md)
- `ClearActiveStore(session_id)`

**Profile (platform-owned fields)**
- `UpdatePreferredLocale(user_id, locale)`
- `UpdateAvatarUrl(user_id, url)`

**System Admin actions** (per [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md) capabilities)
- `DisableUser(user_id, by_actor)`
- `ReactivateUser(user_id, by_actor)`

**Read-side**
- `GetUserById(user_id) → User`
- `GetUserByEmail(email) → User | null` *(used by invitation reconciliation, Gap 12)*
- `ResolveActorFromSession(session_id) → { user_id, active_org_id, active_store_id, status } | null` *(used by all contexts during authorization)*
- `ListSessionsForUser(user_id) → [Session]` *(for the "your devices" UX)*

### Domain events emitted

- `UserCreated`
- `UserSignedIn`
- `UserProfileSynced` *(when any IdP-canonical field changed during sync)*
- `UserPreferredLocaleChanged`
- `UserAvatarChanged`
- `UserDisabled`
- `UserReactivated`
- `SessionCreated`
- `SessionEnded`
- `SessionActiveStoreSet`
- `SessionActiveStoreCleared`

Event delivery semantics owned by [Gap 13](../gaps/13-domain-events-design.md).

### How other contexts authenticate requests

Every authenticated request through any first-party interface carries the Session cookie. The Application Layer of any context calls `ResolveActorFromSession(session_id)`; if it returns null or a `Disabled` user, the request is rejected. Otherwise, authorization continues with `(user_id, active_org_id, active_store_id)` per ADR-0002 + ADR-0018.

**Note**: per [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md), Users are exclusively tenant/platform actors (Org Owners, Org Members, Store Admins, System Admins, platform-scoped). Buyers are NOT Users in our system — they're captured per-order with Beckn buyer references in the Order context.

## Beckn projection notes

User data is **not** projected to Beckn. The IdP and Identity context exist entirely within the platform. When buyers act through Beckn flows, they are referenced via opaque order-level identifiers established by the Order context — never by `User.id` or `external_subject_id`.

## Open questions (within this design)

- **Anonymous / guest checkout for Beckn buyers** — Order context concern (Gap 11). May introduce a "guest" subject type that doesn't go through the IdP.
- **Multi-IdP linking** — deferred per ADR-0009.
- **IdP-side webhooks** — e.g., on IdP-side account deletion, should we proactively disable the User? Operational, future.
- **Soft session limits** (max concurrent sessions per user) — UX/security concern, not committed in v1.

## References

- [ADR-0009](../decisions/0009-identity-and-external-idp.md), [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md), [ADR-0008](../decisions/0008-localization-and-localizedtext.md)
- CLAUDE.md §2.5 (bounded contexts), §5.5 (capabilities), §5.6 (active store), §5.15 (Identity)
- OpenID Connect Core 1.0
- Related gaps: [12](../gaps/12-invitation-account-reconciliation.md) (now largely simplified), [16](../gaps/16-soft-delete-and-audit.md), [17](../gaps/17-pii-and-compliance.md)
