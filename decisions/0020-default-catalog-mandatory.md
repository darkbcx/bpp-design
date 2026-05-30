# ADR-0020: Default Catalog membership is mandatory

- **Status**: Accepted
- **Date**: 2026-05-30
- **Supersedes**: [ADR-0004](0004-store-publication-and-multi-catalog-projection.md) §2 (Default Catalog opt-out membership rule); related wording in [ADR-0005](0005-catalog-and-product-modeling.md).
- **Builds on**: [ADR-0004](0004-store-publication-and-multi-catalog-projection.md), [ADR-0005](0005-catalog-and-product-modeling.md), [ADR-0011](0011-domain-events.md), [ADR-0017](0017-order-and-fulfillment.md)

## Context

[ADR-0004](0004-store-publication-and-multi-catalog-projection.md) §2 originally established:

> "Membership in the Default Catalog is **opt-out**: every product belonging to the store is included automatically; owners can explicitly exclude specific products."

This phrasing introduced two mechanics:
1. Default Catalog auto-includes every store product (good).
2. Owners can exclude individual products from Default Catalog (under reconsideration).

In practice, the exclusion mechanic complicates the model:

- It creates a **parallel visibility mechanism** (catalog membership) alongside the product lifecycle (`Draft` / `Active` / `Archived`) defined in [ADR-0005](0005-catalog-and-product-modeling.md).
- The state "product in no catalog" becomes possible (excluded from Default + not added to any additional), producing a confusing edge case.
- It adds two domain events (`catalog.product_excluded_from_default_catalog`, `catalog.product_restored_in_default_catalog`) and two use cases that duplicate what the product lifecycle already expresses.

The simpler model: **Default Catalog membership is mandatory**. Every Product in a store is always a member of that store's Default Catalog. Visibility on Beckn is controlled exclusively by the **product lifecycle** (Archive the product to hide it).

## Options considered

**A. Keep opt-out membership** (ADR-0004 §2 as originally written). Rejected — duplicates the product-lifecycle visibility lever, introduces an edge state, and complicates events.

**B. Default Catalog membership is mandatory.** Chosen. Simpler invariant; visibility via product lifecycle is sufficient.

**C. Per-product Beckn-visibility flag.** Considered but rejected — would replace one parallel mechanism with another. If a "show on storefront but not Beckn" use case ever materializes, a future ADR can introduce this flag.

## Decision

**Default Catalog membership is mandatory.** Every Product belonging to a store is automatically and permanently a member of that store's Default Catalog. There is **no exclusion mechanism**.

Consequences for the model:

- A Product is **always in at least one catalog** (the Default Catalog) for its entire lifetime in the store.
- A Product may additionally be in zero or more **additional Catalogs** (opt-in membership, unchanged from ADR-0004).
- A Product **cannot be removed from the Default Catalog**.
- Hiding a product from Beckn is exclusively the **product lifecycle's** responsibility: Archive the Product (or keep it in Draft to suppress publication).

### CatalogMembership simplification

The `CatalogMembership` model in [`design/catalog.md`](../design/catalog.md) simplifies:

- **For the Default Catalog**: implicit. No `CatalogMembership` row is needed; every store Product is a member by default.
- **For additional Catalogs**: explicit inclusion record (presence = member). Unchanged.

### Use cases removed

The following Catalog-context use cases are removed (they no longer have a domain operation):

- `ExcludeProductFromDefaultCatalog(store, product_id)`
- `RestoreProductInDefaultCatalog(store, product_id)`

### Domain events removed from the registry

- `catalog.product_excluded_from_default_catalog`
- `catalog.product_restored_in_default_catalog`

`design/events.md` is updated accordingly.

## Consequences

What this commits to:

- A simpler Default-Catalog membership invariant: every store Product is always in Default. One less concept to teach, one less code path.
- Product visibility on Beckn is controlled exclusively by `Product.status` (`Draft` / `Active` / `Archived`).
- `design/catalog.md`, `design/events.md`, CLAUDE.md §5.9, and `handoff/03-beckn-integration.md` §3.4.1 are all updated.
- Two domain events and two use cases are retired before they're built.

What this defers / makes harder:

- **"Hide a product from Beckn but keep it on the storefront"** — was a (theoretical) use case for Default Catalog exclusion. With mandatory Default membership, Beckn visibility is bound to overall product visibility (controlled by `status`). If a future scenario demands "first-party-visible but Beckn-invisible," it would require a new ADR introducing either:
  - A per-product `beckn_visible` flag, OR
  - A reintroduction of selective Default Catalog exclusion (effectively reversing this ADR).
- **No retroactive migration impact** — pre-launch design phase.

## References

- [ADR-0004](0004-store-publication-and-multi-catalog-projection.md) §2 (superseded by this ADR)
- [ADR-0005](0005-catalog-and-product-modeling.md) (related wording superseded)
- [design/catalog.md](../design/catalog.md), [design/events.md](../design/events.md)
- CLAUDE.md §5.9
- `handoff/03-beckn-integration.md` §3.4.1
