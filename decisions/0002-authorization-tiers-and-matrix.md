# ADR-0002: Authorization tiers and the capability matrix

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/02-platform-scoped-roles.md](../gaps/resolved/02-platform-scoped-roles.md)
- **Partially resolves**: [gaps/03-authorization-and-capabilities.md](../gaps/resolved/03-authorization-and-capabilities.md) (questions 2, 3, 4 fully; question 1 partially)

## Context

`CLAUDE.md` §5.4 originally distinguished only two authorization scopes — store-scoped and platform-scoped — without specifying what roles existed in each, who could hold them, what they could do, or how authorization composed when a single user spanned both scopes. Until this is resolved, no Application use case can perform a defensible authorization check, no cross-tenant operation has a legitimate actor, and Gap 03 (capability matrix), Gap 04 (store lifecycle actors), and Gap 06 (publication approvers) all sit blocked.

The system must also accommodate **support workflows**: platform staff acting on behalf of stores to investigate or fix issues. The original gap surfaced impersonation as an open question; resolving it here keeps the audit attribution rules coherent with the rest of the authorization model.

## Options considered

### Tier structure

**Two tiers (store / platform)** — what §5.4 originally drafted.
- Pros: Simplest model.
- Cons: Conflates "operate the platform" (support, content moderation) with "configure the platform" (manage the permission matrix, manage other admins, deploy keys). No structural place to require higher privilege for the most sensitive actions; blast radius of a compromised platform credential is maximal.

**Three tiers (system / platform / store)** — chosen.
- Pros: Separates "configure the system" from "operate the platform" from "operate a store." Allows a small System Admin pool to retain meta-control while a larger platform-tier pool does routine cross-tenant work. Limits blast radius of any single compromised credential.
- Cons: One more concept to teach; one more boundary to police.

**Flat (capabilities without tiers)** — rejected.
- Pros: Maximum flexibility.
- Cons: Every authorization decision becomes ad hoc. "No cross-tenant action for store users" turns into a per-capability convention rather than a structural rule.

### Capability identity across scope

**Single identifier, tier is the qualifier** — chosen.
- Pros: Smaller catalog. The same `product.manage` capability means the same operation, and the tier of the role holding it determines whether it applies cross-tenant.
- Cons: Authorization decisions must apply an active-store filter for store-scoped roles in addition to the capability check.

**Distinct identifiers per scope** (e.g., `product.manage.own` vs. `product.manage.any`) — rejected.
- Pros: Authorization decisions become purely "do you have this exact capability."
- Cons: Doubles the catalog. Subset relationship between store and platform scope becomes implicit and error-prone.

### Impersonation

**None** — rejected. Blocks common support workflows.

**Downward-only tiered impersonation** — chosen. Preserves the tier invariant (lower-tier credentials cannot reach upward); enables support workflows; audit attribution is clean.

**Open impersonation (any admin → any user)** — rejected. Defeats the tier guarantees; any compromised admin account becomes a privilege-escalation lever.

## Decision

### 1. Three authorization tiers

The system has three tiers, in decreasing privilege:

1. **System Admin** — meta-administrators. Manage the role → capability matrix; configure platform-level settings. Seeded via deploy configuration only. **No in-band lifecycle**: no UI or API for creating, demoting, or recovering System Admins. Lost / orphaned System Admin credentials are recovered out-of-band by re-seeding from configuration.
2. **Platform-scoped** — platform operators (support, compliance, network operations, etc.). May act across tenants subject to their assigned capabilities. Not Members of stores by virtue of platform role.
3. **Store-scoped** — Members of one or more stores. May act only on their **currently active store**; no cross-tenant action is reachable from a store-scoped role.

The **role catalog** is system-defined at design time. Adding a new role is a feature change, not a runtime configuration action. The matrix UI never offers role creation.

### 2. Capabilities and the matrix

- The **capability catalog** is system-defined. Each capability corresponds to a concrete domain or platform action; new capabilities are added when the feature they gate is added.
- The **role → capability matrix** is configured at runtime, exclusively by System Admins. No other tier may edit it. Per-store customization of the matrix is **not** supported.
- **Capability identifiers are single.** A capability has one canonical name (e.g., `product.manage`). Whether it applies cross-tenant or only to the active store is determined by the **tier** of the holder's role, not by the capability name.
- For store-scoped roles, every authorization decision applies an **active-store filter** in addition to the capability check: the target object must belong to the user's currently active store.
- Capabilities exposed via store-scoped roles are a **subset** of those exposed via platform-scoped roles.
- The Tenancy context owns the capability catalog, the role catalog, and the matrix.

### 3. Cross-tenant rules

- **Store-scoped users** can only act on their currently active store. No cross-tenant action is reachable through a store-scoped role.
- **Platform-scoped users** may act on any store consistent with their capabilities. Such actions are recorded in the target store's audit trail with the platform user as the actor. Platform-scoped users do **not** become Members of stores they act on.
- A User may simultaneously hold a platform-scoped role and Memberships in one or more stores. The two scopes never compose into a single ambient capability set; the user's effective scope at any moment is determined by their **active store**.

### 4. Active store

- "Active store" is a **session-level attribute** identifying which store-scoped Membership (if any) is currently in effect.
- The active store is chosen by an explicit UI switcher: setting the active store in the session, followed by an immediate redirect to a URL-scoped path (`/stores/<slug>/...`). The URL slug must match the session value on every request; mismatch is an authorization failure.
- `active store = none` is a valid state, available only to **System Admins and platform-scoped users**. It is the mode in which they exercise platform-scope or system-scope authority. Pure store-scoped users never see this state.
- The session storage mechanism (cookie, JWT, server-side record) is deferred to [Gap 01](../gaps/resolved/01-identity-and-access-scope.md). Authorization treats the active store as a logical session attribute.

### 5. Impersonation

- Impersonation is supported **strictly downward** along the tier hierarchy:
  - System Admin → Platform-scoped or Store-scoped user.
  - Platform-scoped → Store-scoped user.
  - Same-tier and upward impersonation are **forbidden**.
- During impersonation, the impersonator's effective capabilities become **exactly** those of the impersonated user — no more, no less. Destructive actions are allowed.
- Impersonation is initiated by an explicit UI action and is **time-limited**; the timeout value is operational configuration. An "End impersonation" control is always visible during an impersonating session.
- **Audit attribution**: every action performed during an impersonation session records **both** the real actor and the impersonated user, with the real actor as the legally responsible party. No action ever appears in audit attributed to the impersonated user alone when impersonation was active.

## Consequences

What this commits the system to:

- Every Application use case performs an authorization check of the form *"Does this actor have this capability in this scope?"* — never simply *"Is this actor an admin?"*. The "scope" is the active store, the platform, or system-wide.
- The Tenancy context owns the capability catalog, the role catalog, and the role → capability matrix. It exposes authorization decisions as resolved yes/no to other contexts.
- Audit (Gap 16) must support attribution with **both** a real actor and an impersonated-as actor. This is a load-bearing constraint on audit schema even though Gap 16 is unresolved.
- Adding a new role is a feature change. The matrix UI never offers role creation.
- Adding a new capability is a feature change. The matrix UI exposes new capabilities as toggleable per role once they exist in code.
- System Admin lifecycle is operational. The product has no UI for creating, demoting, or recovering System Admins.
- The URL scheme for first-party interfaces includes an explicit store slug (`/stores/<slug>/...`) for any store-scoped operation. The session-stored active store and the URL slug must match per request.

What this defers:

- Impersonation timeout values, session retention, and audit log retention — operational.
- Session storage mechanism — [Gap 01](../gaps/resolved/01-identity-and-access-scope.md).
- Audit schema and storage — [Gap 16](../gaps/resolved/16-soft-delete-and-audit.md). This ADR establishes the attribution requirement; the implementation lives there.
- Decision-exposure pattern (predicate functions, policy objects, ABAC service) — implementation detail; remains open in [Gap 03](../gaps/resolved/03-authorization-and-capabilities.md).
- Negative permissions (explicit denies) — remains open in [Gap 03](../gaps/resolved/03-authorization-and-capabilities.md). Default assumption: model is purely additive unless that ADR overturns it.
- Logging of denied authorization attempts — [Gap 03](../gaps/resolved/03-authorization-and-capabilities.md) and [Gap 16](../gaps/resolved/16-soft-delete-and-audit.md).

What this makes harder:

- In-band System Admin lifecycle management. Onboarding a new System Admin requires a deploy / configuration step. Acceptable trade-off given the rarity and the desire to keep the design simple.
- Self-service role creation by stores. If stores ever demand custom roles (e.g., "Manager who can edit products but not pricing"), that becomes a feature decision in code, not a runtime configuration.

## References

- [gaps/resolved/02-platform-scoped-roles.md](../gaps/resolved/02-platform-scoped-roles.md)
- Related: [Gap 03 (Authorization & capabilities)](../gaps/resolved/03-authorization-and-capabilities.md), [Gap 16 (Audit)](../gaps/resolved/16-soft-delete-and-audit.md), [Gap 01 (Identity)](../gaps/resolved/01-identity-and-access-scope.md)
- CLAUDE.md §5.3, §5.4, §5.5, §5.6, §5.7
