# Design: Tenancy

- **Status**: Draft
- **Last updated**: 2026-05-30
- **Backed by ADRs**: [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md) (store lifecycle), [ADR-0010](../decisions/0010-invitation-account-reconciliation.md) (invitation reconciliation), [ADR-0018](../decisions/0018-organization-tenancy.md) (organization expansion). Integrates with [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md) / [ADR-0016](../decisions/0016-authorization-details.md) (authorization).

## Purpose

Full design of the Tenancy bounded context — the entities, relationships, lifecycles, invariants, and contracts. Tenancy owns the hierarchical platform-tenant structure: **Organizations** (top level) containing **Stores**; **Users** joining Organizations as Members; Members assigned as Store Admins.

**What this document covers:**
- The Organization, OrganizationMember, OrganizationInvitation entities (introduced by ADR-0018).
- The Store entity (recap; lifecycle in ADR-0003).
- The StoreAdminAssignment entity (introduced by ADR-0018).
- Relationships, hierarchy, lifecycles.
- Invariants.
- Boundary contracts (use cases, events, queries).
- Beckn projection notes (only Store projects; Org is platform-internal).

**What this document does NOT cover:**
- Authorization model details — see [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md), [ADR-0016](../decisions/0016-authorization-details.md).
- Full store lifecycle states / transitions — see [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md).
- Invitation acceptance reconciliation details — see [ADR-0010](../decisions/0010-invitation-account-reconciliation.md).

## Position within the architecture

The Tenancy context owns Organizations, Stores, OrganizationMembers, OrganizationInvitations, and StoreAdminAssignments. Other contexts (Catalog, Inventory, Promotion, Order, Audit) reference Store by ID and User by ID (via [ADR-0009](../decisions/0009-identity-and-external-idp.md) Identity); they never reference Organization directly (the Org is platform-internal).

The Beckn Bridge sees only Store (projects as `provider`); Organization is invisible to the wire.

## Concepts

### Organization

The top-level unit of tenancy. Each Organization contains one or more Stores and has a roster of Members.

| Field | Description |
|---|---|
| `id` | Internal opaque identifier |
| `slug` | URL-safe identifier; **unique within platform** (case-insensitive); **immutable** |
| `name` | `LocalizedText`; mutable |
| `description` | `LocalizedText`; optional; mutable |
| `status` | `Active` \| `Suspended` |
| `supported_locales` | List of BCP 47 tags; defaults to `[id]`; mutable |
| `created_at`, `updated_at` | Timestamps |
| `suspended_at`, `suspend_reason` | Optional; populated on Suspended |

### OrganizationMember

Represents a User's relationship to an Organization.

| Field | Description |
|---|---|
| `id` | Internal opaque identifier |
| `organization_id` | Org reference |
| `user_id` | User reference (Identity context) |
| `role` | `Owner` \| `Member` |
| `joined_at` | When the member joined (creation or acceptance) |
| `status` | `Active` \| `Removed` (retained per no-delete pattern) |
| `removed_at`, `removed_by_user_id` | Optional; populated on removal |

**Invariant:** Each Organization has exactly one `OrganizationMember` with `role = Owner` and `status = Active` at all times.

### OrganizationInvitation

| Field | Description |
|---|---|
| `id` | Internal opaque identifier |
| `organization_id` | Target Org |
| `email` | Intended recipient (case-insensitive lookup) |
| `intended_role` | Always `Member` (Owner is set at Org creation, transferred separately) |
| `issuer_user_id` | The Org Owner (or designate) who sent it |
| `issued_at`, `expires_at` | Timestamps |
| `token` | Opaque, non-guessable, single-use index |
| `state` | `Pending` \| `Accepted` \| `Declined` \| `Revoked` \| `Expired` |
| `accepted_at`, `accepted_by_user_id`, `created_member_id` | Nullable; populated on Accept |
| `declined_at`, `revoked_at`, `revoked_by_user_id` | Nullable per state |

Reconciliation rules (email match, IdP-verified email, active user, valid invitation) follow [ADR-0010](../decisions/0010-invitation-account-reconciliation.md).

### Store

(Lifecycle details in [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md); only Tenancy-relevant fields shown.)

| Field | Description |
|---|---|
| `id` | Internal opaque identifier |
| `organization_id` | **Owning Organization; immutable** |
| `slug` | URL-safe; **unique within Organization** (case-insensitive); immutable |
| `name` | `LocalizedText` |
| `description` | `LocalizedText` |
| `currency` | ISO 4217; immutable after first Active product (per ADR-0007) |
| `supported_locales` | BCP 47 list |
| `status` | `Draft` \| `Active` \| `Suspended` \| `Paused` (per ADR-0003) |
| `submitted_at` | Owner action — store is ready for platform activation review |

**Change from prior model**: there is **no `owner_user_id`** field on Store. Ownership is at the Organization level. The Org Owner has implicit ultimate authority over every store in the Org.

### StoreAdminAssignment

The link between an OrganizationMember and a specific Store, granting Store Admin role for that store.

| Field | Description |
|---|---|
| `id` | Internal opaque identifier |
| `store_id` | Store reference |
| `user_id` | The Org Member's user_id (must be an Active OrganizationMember of the Store's Org) |
| `assigned_at` | Timestamp |
| `assigned_by_user_id` | Who made the assignment (Org Owner, typically) |
| `status` | `Active` \| `Removed` (retained per no-delete) |
| `removed_at`, `removed_by_user_id` | Optional |

The Org Owner does **not** need a StoreAdminAssignment for any store — their Owner role grants implicit access.

## Relationships

```
Organization ─────< OrganizationMember >────── User (Identity)
     │                                              │
     │                                              │
     └─< Store ─────< StoreAdminAssignment >────────┘
                    (Org Member assigned to a Store)
```

- A User can be an OrganizationMember of multiple Organizations.
- Within an Organization, a user is either Owner or Member (mutually exclusive at any given time).
- A Member (not Owner) may have zero or more StoreAdminAssignments within that Org.
- A Store belongs to exactly one Organization (immutable for the Store's lifetime).

## Lifecycles

### Organization lifecycle

States: `Active`, `Suspended`. No `Draft`, no `Paused`, no deletion.

| From → To | Actor | Notes |
|---|---|---|
| (new) → Active | Any authenticated, email-verified User | Becomes the Owner |
| Active → Suspended | Platform (capability `org.suspend`) | Cascade: all stores in Org → Suspended |
| Suspended → Active | Platform (capability `org.reactivate`) | Stores remain Suspended; explicit per-store reactivation needed |

### OrganizationMember lifecycle

States: `Active`, `Removed`.

| From → To | Actor | Notes |
|---|---|---|
| (new) → Active (Owner) | At Org creation | Created with the Org |
| (new) → Active (Member) | Via Org Invitation acceptance | Per ADR-0010 reconciliation |
| Active (Member) → Removed | Org Owner (capability `org.remove_member`) | Cascade: all of that user's `StoreAdminAssignment`s in this Org → Removed |
| Active (Owner) → Active (Member) | Via ownership transfer | See below |
| Active (Member) → Active (Owner) | Via ownership transfer | See below |

The Owner role can **only** transition via the explicit ownership-transfer flow:

1. Current Owner calls `Org.InitiateOwnershipTransfer(org_id, new_owner_user_id)`. Target must be an existing Active Member.
2. Target Member calls `Org.AcceptOwnershipTransfer(transfer_id)`. Atomic swap: target becomes Owner, current Owner becomes Member.

### OrganizationInvitation lifecycle

`Pending` → `Accepted` | `Declined` | `Revoked` | `Expired`. One-way transitions. Email reconciliation per ADR-0010.

### Store lifecycle

Per [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md) — unchanged. The only Tenancy-relevant addition: a Store's `organization_id` is immutable. On Org Suspension, the store cascades to Suspended.

### StoreAdminAssignment lifecycle

States: `Active`, `Removed`.

| From → To | Actor | Notes |
|---|---|---|
| (new) → Active | Org Owner (capability `org.assign_store_admin`) | User must be Active Member of the Org |
| Active → Removed | Org Owner (capability `org.unassign_store_admin`) | Normal unassignment |
| Active → Removed (cascade) | System | When the user's OrganizationMember becomes Removed |
| Active → Removed (cascade) | System | When the User is Disabled (per ADR-0009) |

## Invariants

- Each Organization has exactly one `OrganizationMember` with `role = Owner` and `status = Active`. Never zero, never two.
- Each Store has exactly one `organization_id`, immutable.
- A `StoreAdminAssignment`'s `user_id` must correspond to an Active `OrganizationMember` of the Store's Organization.
- An Org Owner does not need (and cannot have) a `StoreAdminAssignment` — their Owner role grants implicit access to all stores in the Org.
- An Organization Owner cannot be Removed via `org.remove_member` — they must first transfer ownership.
- Organization `slug` is unique within the platform (case-insensitive).
- Store `slug` is unique within its Organization (case-insensitive), not platform-wide.
- An Organization cannot be Suspended while a transfer is in flight; transfer must complete or be revoked first.

## Boundary contracts

### Use cases exposed (Application Layer)

All mutating use cases call `requireCapability(...)` per ADR-0016 and accept an optional `idempotency_key` per ADR-0013.

**Organization management:**
- `Org.Create(slug, name, by_user) → Organization` — creator becomes Owner. Capability: `org.create`.
- `Org.Update(org_id, fields, by_actor) → Organization` — Capability: `org.update`.
- `Org.GetById(org_id, by_actor) → Organization` — Capability: `org.view`.
- `Org.Suspend(org_id, reason, by_actor)` — Capability: `org.suspend` (platform).
- `Org.Reactivate(org_id, by_actor)` — Capability: `org.reactivate` (platform).
- `Org.InitiateOwnershipTransfer(org_id, new_owner_user_id, by_actor) → Transfer` — Capability: `org.transfer_ownership`.
- `Org.AcceptOwnershipTransfer(transfer_id, by_actor)` — Capability: implicit (target user).
- `Org.RevokeOwnershipTransfer(transfer_id, by_actor)` — Capability: `org.transfer_ownership` (the initiator may revoke).

**Member management:**
- `Org.InviteMember(org_id, email, by_actor) → OrganizationInvitation` — Capability: `org.invite_member`.
- `Org.AcceptInvitation(token, by_actor) → OrganizationMember` — by the invited user; ADR-0010 reconciliation rules apply.
- `Org.DeclineInvitation(token, by_actor)`
- `Org.RevokeInvitation(invitation_id, by_actor)` — Capability: `org.invite_member`.
- `Org.RemoveMember(org_id, member_user_id, by_actor)` — Capability: `org.remove_member`.

**Store creation and admin assignment:**
- `Org.CreateStore(org_id, store_slug, name, currency, by_actor) → Store` — Capability: `org.create_store`. Creates a Store in `Draft` state owned by the given Org.
- `Org.AssignStoreAdmin(store_id, user_id, by_actor) → StoreAdminAssignment` — Capability: `org.assign_store_admin`. User must be an Active Org Member.
- `Org.UnassignStoreAdmin(store_id, user_id, by_actor)` — Capability: `org.unassign_store_admin`.

**Manual catalog republication** ([ADR-0019](../decisions/0019-manual-catalog-republication.md)):
- `Store.RequestRepublish(store_id, by_actor, reason?)` — Capability: `store.republish`. Emits `tenancy.store_republish_requested`; the Bridge re-projects the store's catalogs to CDS.
- `Org.RequestRepublishAll(org_id, by_actor, reason?)` — Capability: `org.republish_all_stores` (Org Owner only). Emits `tenancy.organization_republish_requested`; the Bridge fans out to all stores in the Org.

**Read-side:**
- `Org.ListMyOrganizations(user_id) → [Organization]` — drives the Org switcher UI.
- `Org.ListMembers(org_id, by_actor) → [OrganizationMember]`
- `Org.ListStores(org_id, by_actor) → [Store]`
- `Org.ListStoreAdmins(store_id, by_actor) → [User]`
- `Org.GetOwnershipTransfers(org_id, by_actor) → [Transfer]`

Store-specific lifecycle use cases (`Store.SubmitForActivation`, `Store.Pause`, etc.) live in their context (per ADR-0003).

### Domain events emitted

(Full registry in [`design/events.md`](events.md).)

- `tenancy.organization_created`
- `tenancy.organization_updated`
- `tenancy.organization_suspended`
- `tenancy.organization_reactivated`
- `tenancy.organization_member_added` (from invitation acceptance OR from Org creation for the Owner)
- `tenancy.organization_member_removed`
- `tenancy.organization_ownership_transfer_initiated`
- `tenancy.organization_ownership_transferred` (when accepted)
- `tenancy.organization_ownership_transfer_revoked`
- `tenancy.store_admin_assigned`
- `tenancy.store_admin_unassigned`

Existing Store events from ADR-0003 are unchanged. Existing Invitation events from ADR-0010 now apply to `OrganizationInvitation` (they were not Store-specific in their payload).

### Cross-context references

Tenancy holds:
- `user_id` (Identity reference) on OrganizationMember, OrganizationInvitation, StoreAdminAssignment, and various event payloads.
- No direct cross-context joins. Other contexts (Catalog, Inventory, Promotion, Order, Audit) reference Store by `store_id`; none reference Organization.

## Authorization notes

Per ADR-0016, every use case begins with `AuthorizationPort.requireCapability(name, scope)`:

- **Org-level operations**: `scope = { org_id: <active_org_id> }`. The check verifies the user holds the capability *for this Org* (Org Owner role, typically).
- **Store-level operations within Tenancy**: `scope = { store_id: <store_id> }`. The check verifies the user is either:
  - Org Owner of the Store's Org (implicit access), OR
  - Has an Active `StoreAdminAssignment` for that store.

The Active Org (`active_org_id`) is verified to contain the target object. The Active Store (`active_store_id`), if specified in the scope, must belong to the Active Org.

## Beckn projection notes

**The Organization is invisible to Beckn.** The Bridge has no knowledge of Organization. Stores within an Org project as Beckn `provider` nodes (per [ADR-0001](../decisions/0001-bpp-network-identity.md)). Each store's catalogs publish independently to CDS (per [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md)).

Two stores in the same Org are no more related on the wire than two stores from different Orgs. There is no Org-level Beckn presence, no aggregate catalog, no "this BPP has these Orgs" hint.

## Open questions (within this design)

- **Single-store retailer UX**: the platform could auto-create a one-person, one-store Org on a new user's first action. Operational / UI concern; not architectural.
- **Cross-Org store transfer** (e.g., a store sold to another Org) — not supported in v1.
- **Per-Store granular roles** (e.g., "catalog editor but not pricing") — would expand the role catalog; not in v1.
- **Org-level shared catalog templates** — share product definitions across stores within an Org. Future.

## References

- [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md), [ADR-0010](../decisions/0010-invitation-account-reconciliation.md), [ADR-0018](../decisions/0018-organization-tenancy.md) — primary backing ADRs
- [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md), [ADR-0016](../decisions/0016-authorization-details.md) — authorization model
- [design/events.md](events.md) — event registry
- CLAUDE.md §5.1, §5.3–§5.7, §6 — charter sections affected by ADR-0018
