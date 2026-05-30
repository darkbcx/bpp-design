# ADR-0009: Identity & Access — external IdP integration

- **Status**: Accepted
- **Date**: 2026-05-28
- **Partially superseded by**: [ADR-0021](0021-pure-bpp-no-storefront.md) — the implicit "buyers as Users" accommodation is removed; User scope is tenant/platform only
- **Resolves gap**: [gaps/resolved/01-identity-and-access-scope.md](../gaps/resolved/01-identity-and-access-scope.md)
- **Builds on**: [ADR-0002](0002-authorization-tiers-and-matrix.md) (active store as a session attribute), [ADR-0008](0008-localization-and-localizedtext.md) (locale tags)
- **Sub-design**: [design/identity.md](../design/identity.md)

## Context

Identity & Access has been the largest unresolved foundation gap. Every context that names an actor (Memberships, audit attribution, orders, voucher usage, store activation) depends on a stable User identity. The platform has decided to **outsource authentication entirely to an external Identity Provider (IdP)**, making our system a relying party. Passwords, multi-factor authentication, password recovery, and email verification all move out of our scope.

The design must be IdP-agnostic so the choice of provider (Clerk, Auth0, Supabase Auth, Cognito, Keycloak, etc.) is an operational decision rather than an architectural commitment.

## Options considered

### Auth mechanics

**A. Native password authentication.** Rejected — chosen approach is to delegate.

**B. Native + linked IdPs (hybrid).** Rejected — keeps the password-management burden on our side.

**C. External IdP only, OIDC-agnostic** — chosen. All authentication mechanics live at the IdP.

### Profile ownership

**A. All profile data is IdP-canonical; we cache and never write.** Rejected — `preferred_locale` is platform-specific (IdP doesn't know our locale set), and avatar choice is often a UX preference users want to manage in-app.

**B. All profile data is platform-owned; sync from IdP only at provisioning.** Rejected — keeping `email` and `display_name` in sync with IdP is important.

**C. Split: IdP-canonical (refreshed every sign-in) vs. platform-owned (initial mirror at provisioning, mutable thereafter)** — chosen.

### Multiple linked IdP identities

**A. Single `external_subject_id` per User** — chosen for v1. Simpler; matches typical managed-IdP flows that hide multi-method linking inside the IdP.

**B. Separate `ExternalIdentity` join entity from day one.** Rejected — pays a complexity tax for capability not needed in v1.

### Session model

**A. Pure IdP JWT, no local session.** Rejected — loses the active-store attribute (ADR-0002) and idle timeout control.

**B. Local opaque-token session with active_store_id** — chosen.

### User lifecycle

**A. Active / Disabled, no delete; PII erasure deferred to Gap 17** — chosen. Consistent with ADR-0003 (Store) and other ADRs.

**B. Hard delete with cascade.** Rejected — breaks audit trails and foreign references.

## Decision

### 1. IdP-agnostic OIDC integration

The platform authenticates via a standard **OpenID Connect (OIDC)** flow. The specific IdP is configurable infrastructure (per-environment).

- Uses standard OIDC endpoints: authorization, token, userinfo, JWKS, end-session.
- The integration adapter implementing the OIDC client lives in the Infrastructure Layer; the Application Layer calls it through a port.
- Configuration (issuer URL, client_id, client_secret, redirect URIs, scopes) is deployment config, not code.

### 2. User entity (Identity & Access context)

| Field | Source | Mutability |
|---|---|---|
| `id` | Internal opaque | Immutable |
| `external_subject_id` | IdP `sub` claim | Immutable |
| `email` | IdP `email` claim | **IdP-canonical** (refreshed every sign-in) |
| `email_verified_at` | Derived from IdP `email_verified` claim | **IdP-canonical** (refreshed every sign-in) |
| `display_name` | IdP `name` claim | **IdP-canonical** (refreshed every sign-in) |
| `preferred_locale` | IdP `locale` claim at provisioning; defaults to platform `id` | **Platform-owned, mutable** |
| `avatar_url` | IdP `picture` claim at provisioning; defaults to null | **Platform-owned, mutable** |
| `status` | System | `Active` \| `Disabled` |
| `created_at`, `updated_at`, `last_signed_in_at`, `last_profile_synced_at` | System timestamps | — |

`User.id` is what other contexts reference. `external_subject_id` is the immutable join key with the IdP.

**Uniqueness invariants:**
- `external_subject_id` is unique within the platform.
- `email` is unique within the platform (case-insensitive). Conflicts surface as sign-in errors when the IdP-canonical refresh would create a collision (rare edge case).

### 3. Just-in-Time provisioning

There is **no separate sign-up endpoint in our system**. On a User's first successful IdP authentication, our system creates the User record:

- `external_subject_id` ← IdP `sub`
- `email`, `email_verified_at`, `display_name` ← mirrored from IdP claims
- `preferred_locale` ← IdP `locale` claim if it is a valid BCP 47 tag we support; otherwise the platform default `id`
- `avatar_url` ← IdP `picture` claim if present; otherwise null
- `status` ← `Active`

### 4. Profile sync rules

- **IdP-canonical fields** (`email`, `email_verified_at`, `display_name`) are refreshed from the IdP's userinfo on every sign-in. `last_profile_synced_at` is updated.
- **Platform-owned fields** (`preferred_locale`, `avatar_url`) are NOT touched on sync — they retain whatever the user set in our app.

### 5. Email verification trust

The system trusts the IdP's `email_verified` claim as the authoritative signal. A User is considered email-verified iff `email_verified_at` is set (which it is iff the IdP last reported `email_verified: true`). Action gates that require verification (creating a store, accepting an invitation, placing an order, etc.) check this attribute.

### 6. Session model

Identity & Access owns a `Session` entity, **independent of the IdP's token**. The Session is backed by an HTTP-only cookie carrying an opaque token.

| Field | Description |
|---|---|
| `id` | Opaque token (cryptographically random) |
| `user_id` | User reference |
| `created_at`, `last_used_at`, `expires_at` | Lifecycle timestamps |
| `active_store_id` | Nullable; per [ADR-0002](0002-authorization-tiers-and-matrix.md). Set by the active-store switcher. |
| `device_label` | Optional human-readable device descriptor |
| `last_ip` | Last observed client IP (for security review) |

- Idle lifetime: 30 days (configurable). `last_used_at` refreshes on activity.
- Operations: **sign-out** (single session), **sign-out-everywhere** (all sessions for a User).
- The IdP token is exchanged for a Session at sign-in. We do not need to retain the IdP token beyond the initial exchange — we own the session.

### 7. User lifecycle

States: **`Active`**, **`Disabled`**. **No deletion.**

| From → To | Actor |
|---|---|
| (new) → Active | System (JIT provisioning at first sign-in) |
| Active → Disabled | System Admin / Platform-scoped actor with capability |
| Disabled → Active | System Admin / Platform-scoped actor with capability |

`Disabled` Users cannot sign in (the auth flow rejects after IdP returns; an existing Session is terminated). Existing references (Memberships, orders, audit trails, voucher usage) remain attached to the User record. Disabling in our system has no effect on the user's IdP account.

True right-to-erasure / PII scrubbing is deferred to [Gap 17](../gaps/17-pii-and-compliance.md).

### 8. Multi-IdP linking — deferred

A User has exactly one `external_subject_id` in v1. Multi-IdP linking (a User with both Google and Apple identities) is **not supported** in v1.

If the platform later needs multi-IdP linking, the recommended evolution path is to introduce a separate `ExternalIdentity` entity (`user_id`, `provider`, `provider_subject_id`, `linked_at`) and treat `User.external_subject_id` as a denormalized cache of the primary identity. Until then, `User.external_subject_id` *is* the link.

## Consequences

What this commits to:

- **No password storage, no MFA management, no recovery flows in our codebase.** All of that is the IdP's responsibility.
- The Identity & Access context contains: `User`, `Session`. The OIDC integration adapter lives in Infrastructure.
- `User.id` is the cross-context reference key throughout the system.
- Profile sync on every sign-in refreshes IdP-canonical fields; platform-owned fields are mutable in-app.
- `Session.active_store_id` is the source of truth for ADR-0002's active-store attribute.
- Sign-up is invisible in our system — first IdP authentication creates the User (JIT).
- The Bridge does not project User data to the Beckn network — buyers on the network are referenced by opaque order-level identifiers.
- Disabling a User is reversible; deletion is not supported.
- **Gap 12 (invitation reconciliation) is now trivial**: the invitation's email matches the IdP-verified email at sign-in.

What this defers:

- **Multi-IdP linking** (`ExternalIdentity` entity).
- **In-app push of profile edits back to IdP** — currently `preferred_locale` and `avatar_url` stay local.
- **MFA enforcement requirements** — operational at the IdP layer.
- **Account locking for suspicious activity** — operational.
- **PII scrubbing / right-to-erasure** — [Gap 17](../gaps/17-pii-and-compliance.md).
- **IdP-side webhook integration** (e.g., notify-on-deletion at IdP) — operational; future enhancement.
- **Anonymous / guest checkout** — Order context concern, not Identity.

What this makes harder:

- **Working without an IdP** — local dev and testing need a stub IdP or a containerized one (Keycloak, dex).
- **Email change UX** — users change email at the IdP; our cache catches up on next sign-in. If they don't sign in for a while, our cache is stale.
- **Branding the sign-up flow** — the actual sign-up screen lives at the IdP. Customization is IdP-side.
- **Email-based collision** — if `email` uniqueness invariant trips during a profile sync (rare), the sign-in must fail gracefully with a clear remediation path.

## References

- [gaps/resolved/01-identity-and-access-scope.md](../gaps/resolved/01-identity-and-access-scope.md)
- [design/identity.md](../design/identity.md) — full entity model, flow diagrams, integration details
- [ADR-0002](0002-authorization-tiers-and-matrix.md), [ADR-0008](0008-localization-and-localizedtext.md)
- Constrains: [Gap 12](../gaps/12-invitation-account-reconciliation.md), [Gap 17](../gaps/17-pii-and-compliance.md)
- OpenID Connect Core 1.0
