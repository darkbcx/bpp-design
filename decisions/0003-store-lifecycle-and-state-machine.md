# ADR-0003: Store lifecycle and state machine

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/04-store-lifecycle.md](../gaps/resolved/04-store-lifecycle.md)
- **Partially resolves**: [gaps/06-store-to-beckn-publication.md](../gaps/06-store-to-beckn-publication.md) (questions 1, 2, 4, 5, 6 closed; 3 and 7 remain open)

## Context

`CLAUDE.md` covered store creation and ownership transfer but said nothing about what else can happen to a store. Real stores need moderation (platform-driven), voluntary owner pauses, an activation step that gates first publication, and a stance on deletion that respects audit retention. The lifecycle has Beckn-side consequences too: every state change must be reflected on the network, and the Beckn protocol has **no provider-deletion mechanism** — disabling is the only "remove" gesture available, which forces lifecycle and projection to stay in lockstep.

This ADR also bounds the Moderation context (out of scope for now): suspension *uses* the moderation context for triggers and review workflow, but lifecycle modeling stays in Tenancy.

## Options considered

### State list

**Five states (Draft / Active / Suspended / Paused / Archived).**
- Pros: distinct categories for every disablement mode.
- Cons: Archived and Suspended overlap fully — both platform-set, both reversible, both project to the same Beckn wire state. Redundant. The distinction would live in moderation copy, not in domain behavior.

**Four states (Draft / Active / Suspended / Paused)** — chosen.
- Pros: every state has a distinct *behavioral* signature (visibility, who can transition, who can reverse). No redundancy.
- Cons: owners have no in-band wind-down concept; "wound down" stores stay Paused forever. Acceptable given the no-delete stance.

**Minimal (Active / Inactive).**
- Pros: simplest.
- Cons: no registration-approval semantics, no separation between owner-initiated and platform-initiated disablement, no audit-friendly reason coding.

### Deletion stance

**Hard delete** — rejected. Incompatible with audit retention; destroys order history; lets identifiers be reused in confusing ways.

**Soft delete (Deleted state)** — rejected. Effectively another disabled state on top of Suspended/Paused. Redundant.

**No delete** — chosen. Data is retained for audit and historical purposes; identifiers (slugs, internal IDs) are bound to their stores forever. Stores that wind down stay in Paused.

### Wire-state mapping ownership

**In domain** — rejected. Violates §4: Beckn vocabulary in the domain.

**In Bridge** — chosen. State names are domain-authoritative; the wire projection is owned by the Bridge's mapping registry. The Bridge subscribes to `StoreStatusChanged` domain events and re-projects the provider record.

## Decision

### States

A Store is in exactly one of four states at any time:

- **Draft.** Initial state on creation. Owner and admins build catalog, settings, and metadata. The store is **not visible on the Beckn network**. Activation is a platform action.
- **Active.** Platform-activated. Open for buyers; visible on the network.
- **Suspended.** Set by the platform under the moderation context. Reversible to Active **by the platform only**.
- **Paused.** Set by the owner. Reversible to Active **by the owner only**.

There is no Archived state, no Deleted state, no hard delete. Stores wind down by staying in Paused indefinitely. Identifiers and history are retained forever.

### State machine

The full transition set — all other transitions are forbidden:

| From → To | Actor | Purpose |
|---|---|---|
| Draft → Active | Platform | Activation |
| Active → Paused | Owner | Voluntary pause |
| Paused → Active | Owner | Voluntary unpause |
| Active → Suspended | Platform | Moderation |
| Paused → Suspended | Platform | Moderation while owner-paused |
| Suspended → Active | Platform | Lift suspension |

Explicitly forbidden: Owner-driven reversal of Suspended; owner-driven exit from Draft; any transition into Draft; any transition out of Suspended other than to Active. Suspended → Paused is forbidden — owners cannot launder moderation through a voluntary pause.

Activation criteria for Draft → Active (review checklist, compliance gating) are owned by Gap 06.

### Admin access per state

- **Draft.** Full setup access for owner and admins; store not visible to buyers.
- **Active.** Full admin access.
- **Paused.** Full admin access; store hidden from buyers.
- **Suspended.** Read-mostly for owner and admins. Moderation is in progress; structural changes are not permitted. Reversal is platform-only.

### Order behavior in non-Active states

Uniform rule across **Paused** and **Suspended**:

- In-flight orders complete normally. Trust is preserved.
- No new orders are accepted. The wire state prevents discovery on the network; first-party interfaces enforce this independently.

Concrete order-completion semantics depend on the Order context (Gap 11) and will be detailed when that gap resolves. This ADR commits only to the *rule*.

### Memberships and invitations

Preserved across every state transition. Reversibility is real: a store returning from Paused or Suspended retains its full Membership roster and pending invitations. State changes never modify the Tenancy roster.

### Catalog republishing on transition

Every state transition involving Active, Paused, or Suspended emits a `StoreStatusChanged` domain event. The Bridge subscribes to these events and re-projects the store's provider record on the Beckn network with the updated wire state. Because Beckn has no provider-deletion mechanism, this re-projection is the only way the network sees a state change.

Transitions that never touch Active/Paused/Suspended (there are none in the current state machine; Draft only exits to Active) are not Beckn-visible.

### Wire format mapping (Bridge-owned)

| Domain state | Wire signal (Beckn v2) |
|---|---|
| Draft | Catalog not published to CDS |
| Active | Catalog published with `isActive: true` |
| Paused | Catalog published with `isActive: false` |
| Suspended | Catalog published with `isActive: false` |

This mapping lives in the Bridge's mapping registry, not in the domain. The domain never references Beckn enum values.

**Note**: Beckn v2 does **not** distinguish owner-pause from platform-suspend at the wire level — both project as `isActive: false`. The distinction is retained internally (for moderation context, audit, and admin UX) but not visible to BAPs. If a future protocol revision adds a state reason field, the mapping registry can carry it without disturbing this ADR.

## Consequences

What this commits the system to:

- Every Store has a defined state at all times. There is no "inactive without reason" state.
- The Store entity exposes its state and a state-transition history. Each transition records actor, timestamp, and the from/to states. Audit (Gap 16) must capture this faithfully.
- The Tenancy context owns the state machine. Transition guards (who may transition from what to what) are evaluated against the authorization model defined in ADR-0002.
- Suspended transitions reference the moderation context for *why*. The store entity carries the state; the moderation context carries the case.
- Stores accumulate forever. There is no garbage-collection of stores. Long-tail storage cost is accepted; it is the price of audit and reversibility.
- The Bridge subscribes to `StoreStatusChanged` events and re-projects the provider record on the Beckn network on every transition. The Bridge's mapping registry holds the domain-state → wire-state table.
- First-party interfaces (admin UI, storefront) must render state-aware behavior: Draft is invisible to buyers; Paused hides the store but allows admin work; Suspended shows a moderation banner; Active is normal.

What this defers:

- **Activation criteria** for Draft → Active (review checklist, compliance gating, KYC) — [Gap 06](../gaps/06-store-to-beckn-publication.md).
- **Moderation context** — what triggers suspension, the review workflow, suspension reason codes, notification flows. Not yet a gap; will be opened as a separate concern when needed.
- **Per-product or per-category Beckn visibility** within an Active store — [Gap 06](../gaps/06-store-to-beckn-publication.md).
- **Order-completion semantics** in non-Active states — [Gap 11](../gaps/11-order-and-fulfillment-phasing.md).
- **`StoreStatusChanged` event schema and delivery guarantees** — [Gap 13](../gaps/13-domain-events-design.md).
- **Audit schema** for transitions — [Gap 16](../gaps/16-soft-delete-and-audit.md). This ADR establishes the requirement; the schema lives there.

What this makes harder:

- Owner-initiated wind-down. The owner has no in-band action to delete or archive. Paused-indefinitely is the only path. If product feedback ever demands explicit wind-down, this ADR must be superseded.
- Provider-deletion on the Beckn side. Because the protocol cannot delete, even wound-down stores remain as catalogs published to CDS with `isActive: false`. They never disappear from the network.
- Identifier reuse. Slugs and internal store IDs are bound to their stores forever; no recycling.

## References

- [gaps/resolved/04-store-lifecycle.md](../gaps/resolved/04-store-lifecycle.md)
- [ADR-0001](0001-bpp-network-identity.md) — establishes the Bridge as the sole wire-projection owner.
- [ADR-0002](0002-authorization-tiers-and-matrix.md) — establishes platform vs. owner authority used in transition guards.
- CLAUDE.md §4 (Bridge), §5 (Tenancy)
- Related gaps: [06](../gaps/06-store-to-beckn-publication.md), [11](../gaps/11-order-and-fulfillment-phasing.md), [13](../gaps/13-domain-events-design.md), [16](../gaps/16-soft-delete-and-audit.md)
