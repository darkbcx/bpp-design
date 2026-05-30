# ADR-0018: Organization tenancy expansion

- **Status**: Accepted
- **Date**: 2026-05-30
- **Builds on**: [ADR-0002](0002-authorization-tiers-and-matrix.md) (authorization), [ADR-0003](0003-store-lifecycle-and-state-machine.md) (store lifecycle), [ADR-0010](0010-invitation-account-reconciliation.md) (invitation), [ADR-0016](0016-authorization-details.md) (authorization details)
- **Partially supersedes**: [ADR-0002](0002-authorization-tiers-and-matrix.md) §1 (tier naming: "Store-scoped" → "Tenant-scoped"); [ADR-0003](0003-store-lifecycle-and-state-machine.md) §1 (Store Owner concept removed); [ADR-0010](0010-invitation-account-reconciliation.md) §8 (invitation scope: now Org-level only)
- **Sub-design**: [design/tenancy.md](../design/tenancy.md) (new)

## Context

The original Tenancy model treated **Store** as the unit of tenancy: each store had one Owner; invitations created Admin Memberships per store. This works for single-store retailers but does not accommodate retailers who operate multiple stores under one organizational umbrella — a holding company, a multi-brand operation, a family business with several outlets.

This ADR introduces **Organization** as a higher-level tenancy container. An Organization groups one or more stores under shared ownership and shared membership. Stores within an Organization share the Org's roster of potential admins; the Org Owner has plenary authority across all of them.

The Organization concept is **platform-internal only**. Beckn is unaware of it — the wire surface (the platform appears as one BPP, stores project as Beckn `provider` nodes) is unchanged.

## Options considered

### D1 — Store Owner role

**A. Merge into Org Owner.** Stores within an Org have no per-store Owner; Org Owner is the implicit ultimate authority. **Chosen.**
**B. Keep Store Owner** as a designated Org Member per store.
**C. Org Owner is automatically Store Owner** of every store; cannot be reassigned.

A is cleanest; matches the user's described model (Org Owner creates stores and assigns admins; never mentions Store Owner).

### D2 — Multi-Org membership

**A. Allowed.** A user may belong to multiple Organizations independently. **Chosen.**
**B. Strictly one Org per user.** Simpler but constrains real-world cases.

### D3 — Authorization tier model

**A. Keep three tiers; rename Store-scoped → Tenant-scoped.** Within Tenant-scope, sub-roles (Org Owner, Store Admin) provide internal hierarchy. **Chosen.**
**B. Four tiers**: System / Platform / Org / Store.

A keeps the tier model from ADR-0002 conceptually intact while accommodating the new hierarchy.

### D4 — Active context

**A. Active Org + optional Active Store.** Active Org is required for tenant-scoped actions; Active Store is required for store-specific actions. **Chosen.**
**B. Single Active Store; Active Org is derived.**

A matches the real-world UX flow (Org dashboard → store dashboard).

### D5 — Invitation flow

**A. Org-level invitations only; Store Admin status is direct assignment.** **Chosen.**
**B. Two flows — Org invitation + Store invitation.**

A is cleaner: an Org Member is already verified; the Org Owner can directly assign them to specific stores without a second consent round.

### D6 — Every store must belong to an Org

**A. Yes, in v1.** All stores are created within an Organization. **Chosen.**
**B. Optional.** Allow standalone stores too.

A is more consistent. Single-store retailers create a one-person Org.

## Decision

### 1. Organization entity

A new entity in the Tenancy context. An Organization groups one or more Stores under shared ownership and shared membership.

Key attributes (full schema in [design/tenancy.md](../design/tenancy.md)):
- `id` — internal opaque
- `slug` — URL-safe identifier; unique within platform; immutable
- `name`, `description` — `LocalizedText`
- `status` — `Active` | `Suspended`
- `supported_locales`, `created_at`, `updated_at`

### 2. Tenancy hierarchy

```
Organization (top-level tenant)
    │
    ├── owns Stores (one or more)
    │
    └── has Members (Users)
              │
              └── can be assigned as Store Admin of specific Stores
```

A Store belongs to exactly one Organization for its lifetime. An Organization owns one or more Stores. A User may be a Member of multiple Organizations.

### 3. Membership model

OrganizationMember replaces the prior Store-level Membership for new (Org-aware) tenancy:

- One OrganizationMember row per `(user, organization)` pair.
- Roles within an Org: `Owner` | `Member`. Exactly one Owner per Org at all times.
- An Org Member may additionally hold one or more **StoreAdminAssignments** (links to specific stores within the Org).

### 4. Store Owner role removed

ADR-0003's per-store `Owner` is **removed**. Stores belong to Organizations; the Org Owner has implicit ultimate authority over all stores in the Organization. Ownership transfer happens at the **Org level**, not per-store.

Other ADR-0003 store-lifecycle decisions (states, transitions, suspension semantics) are unchanged.

### 5. Authorization tier rename — three tiers retained

The three tiers from [ADR-0002](0002-authorization-tiers-and-matrix.md) remain in count; the third tier is renamed:

- **System Admin** — unchanged.
- **Platform-scoped** — unchanged.
- **Store-scoped** → **Tenant-scoped**.

Within Tenant-scope, two sub-roles:

- **Org Owner** — full authority over the Org and all its stores. Holds all `org.*` capabilities and implicitly all `store.*` capabilities for every store in the Org.
- **Store Admin** — administrative access to assigned stores within the Active Org. Holds `store.*`-level capabilities for assigned stores only.

Plus a baseline:
- **Org Member** — the default state of any user joined to an Org. Holds only `org.view` (limited visibility); does not have store access until promoted via StoreAdminAssignment.

The capability matrix from ADR-0002 grows by these sub-role rows. Capability naming follows [ADR-0016](0016-authorization-details.md): `org.*` for Org-level actions, existing `store.*` / `product.*` / `voucher.*` / etc. for store-level.

### 6. Active context

Session attributes:
- **`active_org_id`** — required for any Tenant-scoped action; chosen via UI switcher when a user belongs to multiple Orgs. Set in session, then redirected to `/orgs/<org-slug>/...`.
- **`active_store_id`** — optional within Active Org; required for store-specific actions. Path: `/orgs/<org-slug>/stores/<store-slug>/...`.

`active_org_id = null` and `active_store_id = null` is a valid state only for System Admins and platform-scoped users. The URL must match the session values on every request; mismatch is an authorization failure.

The Active session attribute story replaces the prior "Active Store" model from [ADR-0002](0002-authorization-tiers-and-matrix.md) §4.

### 7. Invitation flow — Org-level only

Invitations now operate at the **Organization level only**. ADR-0010's reconciliation rules apply unchanged (email match against IdP-verified email, email verification gate, idempotency, etc.) but:

- The invitation targets an **Organization** (not a Store).
- Acceptance creates an **OrganizationMember** with role = `Member`.
- Owner role is set at Org creation; transferred via the ownership-transfer flow (not invitation).
- **Store admin status is direct assignment**, not invitation. The Org Owner calls `Org.AssignStoreAdmin(store_id, user_id)` — the user is already verified via Org membership.

ADR-0010's "Invitations create Admin Memberships only" rule is superseded: invitations now create Org Members.

### 8. Capability catalog additions

Per [ADR-0016](0016-authorization-details.md) action-level naming:

Org-level (Org Owner unless noted):
- `org.create` — held by any authenticated, email-verified User (anyone can create an Org)
- `org.update` — Org Owner
- `org.invite_member`, `org.remove_member`, `org.transfer_ownership` — Org Owner
- `org.create_store` — Org Owner
- `org.assign_store_admin`, `org.unassign_store_admin` — Org Owner
- `org.view` — Org Owner + Org Member (baseline visibility)
- `org.suspend`, `org.reactivate` — platform-scoped users with capability

Store-level capabilities from prior ADRs (e.g., `product.publish`, `voucher.create`) remain — held by Store Admins for assigned stores, and implicitly by Org Owners for all stores in their Org.

### 9. Organization lifecycle

States: `Active` | `Suspended`. No `Draft` (Orgs aren't on Beckn — nothing to publish). No `Paused` (Orgs have no buyers to hide from). No deletion (consistent with §5.19 / ADR-0014).

State transitions:

| From → To | Actor |
|---|---|
| (new) → Active | Any authenticated, email-verified User (becomes Org Owner) |
| Active → Suspended | Platform (moderation) |
| Suspended → Active | Platform (lift) |

On Org Suspension, all stores within the Org cascade to Suspended state (per ADR-0003's platform-Suspend action). On Org reactivation, stores remain Suspended — explicit per-store reactivation is required.

### 10. Org ownership transfer

Same two-party pattern as the original store ownership transfer (now obsolete):

1. Current Org Owner initiates: `Org.InitiateOwnershipTransfer(org_id, new_owner_user_id)` — target must be an existing Active Org Member.
2. Target Member accepts: `Org.AcceptOwnershipTransfer(transfer_id)` — atomic role swap. Target becomes Owner; old Owner becomes Member.

### 11. Beckn integration — unchanged

The Bridge has **no knowledge** of Organizations. Stores within an Org each project as Beckn `provider` nodes, exactly as defined by [ADR-0001](0001-bpp-network-identity.md). Catalog publication to CDS (per [ADR-0004](0004-store-publication-and-multi-catalog-projection.md)) is store-level, not Org-level. There is no Org-wide Beckn presence.

This is intentional: Beckn is store-centric. Two stores belonging to the same Org are no more related on the wire than two stores from different Orgs.

### 12. Domain events — new entries

Tenancy events added (full registry update in [design/events.md](../design/events.md)):

- `tenancy.organization_created`
- `tenancy.organization_updated`
- `tenancy.organization_suspended`
- `tenancy.organization_reactivated`
- `tenancy.organization_member_added` (from invitation acceptance)
- `tenancy.organization_member_removed`
- `tenancy.organization_ownership_transferred`
- `tenancy.store_admin_assigned`
- `tenancy.store_admin_unassigned`

The existing `tenancy.store_created`, `tenancy.store_status_changed`, `tenancy.invitation_*` events remain. The `tenancy.ownership_transferred` event (formerly per-store) is renamed `tenancy.organization_ownership_transferred` (per-store ownership no longer exists).

## Consequences

What this commits to:

- A new **Organization** entity in the Tenancy context. All stores in v1 belong to one Organization.
- Authorization tier 3 is renamed **Tenant-scoped**; internal sub-roles (Org Owner, Store Admin, baseline Org Member) replace the single "Admin" role.
- The Active session attribute is now hierarchical (`active_org_id` + optional `active_store_id`).
- Org-level invitations replace per-store invitations.
- Store Admin status is a **direct assignment** (not invitation).
- New `design/tenancy.md` documents the full entity model.
- Beckn integration is unchanged — Organizations are invisible to the network.

What this defers:

- **Org-wide promotions / vouchers** — vouchers remain store-scoped (per ADR-0007). Org-wide promotion engine is future.
- **Org-level catalog templates** (share products across stores) — future.
- **Multi-store cart / unified buyer experience across an Org** — future, Beckn-side and storefront concern.
- **Org-level analytics dashboards** — future, UI concern.
- **Sub-roles within Store Admin** (granular per-area access like "catalog editor but not pricing") — future.
- **Cross-Org store transfer** — not supported in v1.

What this makes harder:

- **Single-store retailers** must still create an Org (a one-person, one-store Org). Minor UX friction for architectural consistency.
- **Cross-Org employee operations** — a user with the same email can be in multiple Orgs but wears separate hats per Org; no platform-level "this is the same person" coordination.
- **Promoting an Org Member to Org Owner** — must use the explicit transfer-of-ownership flow; cannot be done via a role change UI.
- **Renaming a tier** (Store-scoped → Tenant-scoped) is a documentation correctness issue across all prior ADRs; the work happens in CLAUDE.md updates and is not retroactively applied to ADRs.

## References

- [ADR-0002](0002-authorization-tiers-and-matrix.md), [ADR-0003](0003-store-lifecycle-and-state-machine.md), [ADR-0010](0010-invitation-account-reconciliation.md), [ADR-0016](0016-authorization-details.md) — affected ADRs
- [design/tenancy.md](../design/tenancy.md) — full entity model
- [design/events.md](../design/events.md) — updated with new Tenancy events
- CLAUDE.md §2.5 (bounded contexts), §5.1, §5.3–§5.7, §6 (affected sections)
