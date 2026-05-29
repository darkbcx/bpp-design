# ADR-0016: Authorization details — capability naming, decision pattern, denies, denial auditing

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/03-authorization-and-capabilities.md](../gaps/resolved/03-authorization-and-capabilities.md) (fully — supplements [ADR-0002](0002-authorization-tiers-and-matrix.md))
- **Builds on**: [ADR-0002](0002-authorization-tiers-and-matrix.md) (tiers, matrix), [ADR-0011](0011-domain-events.md) (events), [ADR-0014](0014-soft-delete-and-audit.md) (Audit)

## Context

[ADR-0002](0002-authorization-tiers-and-matrix.md) established the three-tier authorization model, the system-defined role catalog, and the System-Admin-managed capability matrix. Four sub-questions remained open in Gap 03:

- **Capability granularity** — per-action or per-resource?
- **Decision exposure** — how do use cases express their capability requirements in code?
- **Negative permissions** — explicit denies, or purely additive grants?
- **Authorization failure auditing** — what happens when a check fails?

This ADR closes those four, fully resolving Gap 03.

## Options considered

### Capability granularity

**A. Resource-level** (`manage_products`, `manage_vouchers`). Rejected — too coarse; can't separate Support's "read" from Admin's "write."

**B. Action-level** (`product.create`, `product.publish`) — chosen. Finer access control, clearer audit attribution.

### Decision exposure

**A. Explicit `requireCapability(name, scope)` at use case start** — chosen. Procedural, explicit, code-readable, testable.

**B. Policy objects** (`ProductPolicy(user).canCreate(store)`). Rejected — indirect; couples policy ownership.

**C. Centralized ABAC service.** Rejected — heavy for this scope.

**D. Annotations** (`@RequiresCapability("product.create")`). Rejected — language/framework-coupled.

### Negative permissions

**A. Purely additive** — chosen. Simple, predictable. Special cases ("admins can do everything except X") become different mappings, not deny overrides.

**B. Deny rules with precedence.** Rejected — subtle bugs, hard to audit.

### Authorization failure logging

**A. All denials logged** (verbose). Rejected — anonymous bot probing swamps the outbox.

**B. Two-tier: authenticated denials as events; anonymous as infrastructure logs** — chosen.

**C. None.** Rejected — loses security investigation signal.

## Decision

### 1. Capability naming: action-level

Capabilities follow the format **`<resource>.<verb>`** — lowercase, snake_case verb. Examples:

| Domain | Capability examples |
|---|---|
| Catalog (Product) | `product.create`, `product.update`, `product.publish`, `product.archive`, `product.restore`, `product.read` |
| Catalog (Variant / Category) | `product.variant.add`, `product.variant.remove`, `category.create` (System Admin) |
| Catalog (Store catalogs) | `catalog.create`, `catalog.rename`, `catalog.delete` |
| Tenancy (Store) | `store.create`, `store.submit_for_activation`, `store.activate`, `store.pause`, `store.unpause`, `store.suspend` |
| Tenancy (Membership) | `member.invite`, `member.remove`, `member.role.change` |
| Promotion | `voucher.create`, `voucher.update`, `voucher.disable`, `voucher.enable`, `voucher.read` |
| Inventory | `inventory.adjust`, `inventory.read` |
| Identity & Access | `user.disable`, `user.reactivate`, `user.scrub`, `user.impersonate` |
| Audit | `audit.read.platform`, `audit.read.store` |
| System Admin | `system.matrix.edit`, `system.category.manage` |

The full **capability catalog** is system-defined (per ADR-0002), maintained alongside the implementation, and surfaced by the matrix UI. The matrix UI never creates capabilities — only assigns them to roles.

### 2. Decision exposure: explicit `requireCapability`

The Tenancy context exposes an Application-Layer port:

```
interface AuthorizationPort {
  // Throws AuthorizationDenied on failure.
  requireCapability(name: string, scope?: { store_id?: string }): void

  // Returns boolean; used for conditional UI / optional checks.
  hasCapability(name: string, scope?: { store_id?: string }): boolean
}
```

Every Application-Layer use case across every context begins with one or more `requireCapability` calls **before** any state mutation or side effect.

**Scope semantics:**
- For **store-scoped users with store-scoped capabilities**: the `scope.store_id` parameter MUST match the user's active store (per ADR-0002). Mismatch is denied.
- For **platform-scoped or System Admin users**: scope is informational (still recorded in the denial event if it occurs).
- For **inherently platform-wide capabilities** (e.g., `audit.read.platform`): scope is unused; the check verifies tier-level capability.

**Active-store filter** (per ADR-0002 §4 / CLAUDE.md §5.6) is applied as part of the `requireCapability` evaluation when the actor is store-scoped — not as a separate check.

Other contexts consume `AuthorizationPort` through the cross-context port pattern from ADR-0012.

### 3. Permissions are purely additive

The role → capability matrix is **grants only**. Absence of a `(role, capability)` entry means the capability is denied for that role.

There are no explicit deny entries, no precedence rules, no overrides. Special cases ("admins can do everything except `user.scrub`") are modeled by not granting `user.scrub` to the admin role — a different mapping, not a deny.

### 4. Authorization failure auditing — two-tier

**Authenticated denials** (the actor has a valid session but the capability check fails):
- `requireCapability` raises `AuthorizationDenied`.
- The Tenancy context emits an `identity.authorization_denied` event.
- Payload includes: `actor` (`{user_id, impersonated_user_id, active_store_id}`), `capability_name`, `scope`, `reason` (typed: `missing_capability` | `wrong_active_store` | `user_disabled` | `tier_mismatch`).
- Audit subscribes and ingests this as a standard `AuditRecord` (per ADR-0014).
- Useful for: spotting UI bugs (legitimate users hitting denied paths they shouldn't have reached), insider-misuse review, user-facing diagnostics ("why was my action denied?").

**Anonymous denials** (the request has no valid session — typical bot probing or unauthenticated path):
- Handled at the **Interface Layer** (request returns 401/403) and logged at the **Infrastructure Layer** (web access logs / WAF / firewall).
- Does **NOT** emit a domain event — avoids swamping the outbox.
- Aggregate metrics (rate of anonymous denials, top targets) surface to ops.

This is the dividing line: domain-level event for *who*-failed-*what*; infrastructure log for unauthenticated noise.

## Consequences

What this commits to:

- The capability catalog is action-level. Adding a feature typically adds one or more capabilities.
- Every Application-Layer use case begins with explicit `requireCapability` calls. The pattern is uniform; code reviews can verify it.
- The matrix is grants-only. The UI never offers "deny this capability" as an option.
- `identity.authorization_denied` joins the event registry (`design/events.md`). Audit ingests it.
- Anonymous denials stay out of the event stream.

What this defers:

- **Per-resource scoping** beyond store_id (e.g., "this admin can manage product X but not product Y"). Not needed for v1; would extend the scope parameter if ever required.
- **Capability hierarchies / wildcards** in the matrix UI (e.g., `product.*` as a shorthand). Operational nicety; can be added in the matrix UI without changing the model.
- **UI-side denial messaging** — Interface Layer concern. Backend returns the typed error; UI decides how to render.
- **Rate-limiting / throttling** for repeated denials — Infrastructure / WAF concern.

What this makes harder:

- **Adding a new capability requires matrix work.** A System Admin must decide which roles get it. Tooling can help (e.g., "grant to System Admin role by default; ask for the rest").
- **"Everyone has X except a special case"** must be modeled as the explicit absence of X from that role — no shortcut via a deny entry.
- **Cross-resource capability** ("approve any invitation across all stores") requires a cross-tenant capability or platform-scope tier — not a single capability spanning resources.

## References

- [gaps/resolved/03-authorization-and-capabilities.md](../gaps/resolved/03-authorization-and-capabilities.md) (now fully resolved)
- [ADR-0002](0002-authorization-tiers-and-matrix.md) (tier model, matrix, active-store filter), [ADR-0011](0011-domain-events.md) (events), [ADR-0012](0012-cross-context-consistency.md) (cross-context ports), [ADR-0014](0014-soft-delete-and-audit.md) (Audit as subscriber)
- [design/events.md](../design/events.md) — adds `identity.authorization_denied`
- CLAUDE.md §5.4–§5.7 (existing authorization sections), §5.5 (updated by this ADR)
