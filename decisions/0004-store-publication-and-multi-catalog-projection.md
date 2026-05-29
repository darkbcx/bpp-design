# ADR-0004: Store publication and multi-catalog projection

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/06-store-to-beckn-publication.md](../gaps/resolved/06-store-to-beckn-publication.md)
- **Constrains**: [gaps/07-catalog-modeling-scope.md](../gaps/07-catalog-modeling-scope.md) — introduces `Catalog` as a first-class entity with default vs. additional semantics

## Context

Gap 06 was already mostly resolved by [ADR-0003](0003-store-lifecycle-and-state-machine.md) (lifecycle linkage, revocation semantics, modeling location, default visibility, activation flow direction). Two questions remained: **per-product/per-category visibility** on Beckn (granularity), and **what gates Draft → Active** (compliance criteria).

The resolution of granularity introduces the **Catalog** as a first-class entity in the system — a named container of products within a store — and defines a multi-catalog model that projects to the Beckn network. This crosses into Gap 07 (Catalog modeling scope); this ADR commits only the publication-relevant slice and leaves the full Catalog/Product shape (variants, attributes, media, taxonomies) open.

## Options considered

### Per-product Beckn visibility (Gap 06 Q3)

**A. Store-level only.** When the store is Active, every product surfaces on Beckn. Simplest.
- Cons: no way to keep specific items off-network (in-store-only items, soft launches).

**B. Per-product `beckn_visible` attribute.**
- Cons: adds an attribute to every product; adds UX and projection complexity for a use case not yet validated.

**C. Multi-catalog model — default + additional, all projecting** — chosen.
- Pros: matches Beckn's support for one-or-more catalogs per provider; gives BAPs flexibility to choose what to render; gives store owners a clean primitive (the Default Catalog) for "what's on Beckn" without per-product flags; additional catalogs become useful for organization too.
- Cons: introduces Catalog as a first-class entity earlier than Gap 07 might prefer; requires the Bridge to project multiple catalogs per provider.

### Catalog visibility on Beckn

(a) Default catalog only on Beckn — rejected. Limits BAP-side flexibility.
(b) **All catalogs projected** — chosen. BAPs see each catalog as a distinguishable grouping under the provider.

### Activation criteria (Gap 06 Q7)

**A. Manual review, no coded checklist** — chosen.
- Pros: simple; flexible; new compliance requirements are policy changes, not code changes.
- Cons: relies on reviewer discipline; harder to audit "what was checked."

**B. Coded checklist** — rejected. Inflexible during early operations when policy is still moving.

**C. Coded baseline + reviewer judgment** — rejected. The line between system-enforced and reviewer-judged tends to drift; defer until patterns emerge.

## Decision

### 1. Publication unit and surface

- The **publication unit is the store**. An Active store appears on the Beckn network as exactly one `provider` node (per [ADR-0001](0001-bpp-network-identity.md)).
- A store has one or more **Catalogs**. All of a store's Catalogs project to the Beckn network alongside the provider node. BAPs may render any of them.
- The exact Beckn shape used for multi-catalog projection (`category` blocks under provider, sub-catalog structures, or other groupings as the protocol version offers) lives in the Bridge's **mapping registry**. The domain stays protocol-naive.
- Catalog identifiers on the Beckn wire are derived deterministically from internal catalog identifiers, mirroring the provider-ID rule in [ADR-0001](0001-bpp-network-identity.md).

### 2. Catalog model — publication-relevant slice

- Every store has **exactly one Default Catalog**, auto-created when the store is created. The Default Catalog cannot be deleted.
- **Membership in the Default Catalog is opt-out.** Every product belonging to the store is included in the Default Catalog automatically. Owners can explicitly exclude specific products.
- Stores may create **additional Catalogs** — named, scoped to the store, containing products from that store only. **Membership in additional Catalogs is opt-in**: products must be explicitly added.
- A product may belong to **zero or more** catalogs simultaneously. A product excluded from the Default Catalog and not added to any additional Catalog still exists in the store but is invisible on the network.
- The Catalog entity is owned by the **Catalog context**. Full catalog/product modeling (variants, attributes, media, taxonomies, lifecycle) is deferred to [Gap 07](../gaps/07-catalog-modeling-scope.md).

### 3. Republication on catalog change

Catalog state changes — catalog created, renamed, products added or removed, catalog deleted — emit domain events. The Bridge subscribes and re-projects the provider node and its catalogs accordingly. This mirrors the `StoreStatusChanged` republication mechanism from [ADR-0003](0003-store-lifecycle-and-state-machine.md). Because Beckn has no resource-deletion mechanism, "removing" a catalog or product requires the Bridge to project the absence (e.g., empty or hidden category) per its mapping discipline.

### 4. Activation flow

- A Draft store has a `submitted_at` attribute. The owner sets this via an explicit **"Submit for activation"** action.
- The platform's **reviewer queue** is the filtered view of Drafts where `submitted_at` is set. Drafts without `submitted_at` are not queue-visible.
- A platform-scoped user with the appropriate capability (per [ADR-0002](0002-authorization-tiers-and-matrix.md)) inspects the Draft and either:
  - Activates it (Draft → Active per [ADR-0003](0003-store-lifecycle-and-state-machine.md)), or
  - Leaves it in Draft. A feedback / rejection-reason channel back to the owner is deferred (operational; possibly a Moderation concern).
- There is **no coded activation checklist**. Activation criteria live in operational policy, not in code.
- Basic data validity (store name, contact, etc.) remains an **entity-level invariant** of the Store entity, not part of activation criteria. A Draft is *valid* on creation; whether it's *activatable* is editorial.

## Consequences

What this commits the system to:

- The **Catalog** is a first-class entity in the Catalog context. Catalog modeling (Gap 07) must accommodate: Default vs. additional distinction, opt-out vs. opt-in membership semantics, store-scoping, and a stable identifier suitable for deterministic Bridge projection.
- The Bridge gains responsibility for **multi-catalog projection** per provider. The mapping registry holds the catalog → wire mapping alongside the provider-ID mapping.
- The Store entity carries a `submitted_at` attribute used during the Draft phase. Audit (Gap 16) must record the owner action that sets it and the platform action that activates.
- A platform reviewer queue UI surface exists, scoped by capability per ADR-0002.
- No coded compliance gating. Onboarding quality depends on reviewer discipline and external policy.
- Catalog state changes generate domain events; the Bridge consumes them for republication.

What this defers:

- Full Catalog / Product entity model (variants, attributes, media, taxonomies, lifecycle states) — [Gap 07](../gaps/07-catalog-modeling-scope.md).
- Reviewer feedback / rejection-reason flow — operational; out of scope here.
- Per-Catalog publication toggle (a store with three catalogs that wants only two on Beckn) — not in scope. If needed later, add a `published` flag per catalog without altering the rest of this ADR.
- Exact Beckn projection shape for multi-catalog providers — Bridge mapping registry; may evolve with protocol version.
- Notification mechanics around activation, exclusion, etc. — operational.

What this makes harder:

- **Piecemeal review.** Reviewers evaluate the whole store at once; per-product or per-catalog approval is not a built-in concept.
- **Catalog deletion on the network.** Because catalogs project to Beckn and Beckn has no deletion gesture, removing a catalog requires the Bridge to project the absence (empty or hidden category). The mapping discipline accepts this; the operational consequence is that catalogs linger on the wire even after domain removal.

## References

- [gaps/resolved/06-store-to-beckn-publication.md](../gaps/resolved/06-store-to-beckn-publication.md)
- [ADR-0001](0001-bpp-network-identity.md) — BPP identity, provider derivation
- [ADR-0002](0002-authorization-tiers-and-matrix.md) — reviewer capability
- [ADR-0003](0003-store-lifecycle-and-state-machine.md) — store states and republication mechanism
- Constrains: [Gap 07](../gaps/07-catalog-modeling-scope.md)
- `ion-specs` — provider, catalog, category projection
