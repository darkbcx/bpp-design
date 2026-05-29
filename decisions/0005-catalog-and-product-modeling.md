# ADR-0005: Catalog and Product modeling

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/07-catalog-modeling-scope.md](../gaps/resolved/07-catalog-modeling-scope.md)
- **Builds on**: [ADR-0001](0001-bpp-network-identity.md), [ADR-0004](0004-store-publication-and-multi-catalog-projection.md)
- **Sub-design**: [design/catalog.md](../design/catalog.md)

## Context

[ADR-0004](0004-store-publication-and-multi-catalog-projection.md) established the `Catalog` as a first-class container scoped to a store. This ADR fills in **what's inside** a catalog: Products, their variants, attributes, categories, media, and lifecycle. The Catalog context is the largest single piece of the domain by surface area; modeling errors here propagate to inventory, pricing, orders, search, and the Beckn projection. The decisions captured here become storage and code shape that is expensive to retrofit.

The full entity model — boundary contracts, events, queries, Beckn projection notes — lives in [design/catalog.md](../design/catalog.md). This ADR records the decisions and their reasoning.

## Options considered

### Variant strategy

**A. Matrix model only.** Variants are always generated combinations of attributes (Size × Color). Used uniformly across all products.
- Pros: simple uniformity.
- Cons: rigid — products that genuinely need per-variant SKU tracking get awkward.

**B. Flat model only.** Every SKU is an independent Product; variants are just UX grouping.
- Pros: maximum flexibility; each SKU has full attributes.
- Cons: loses the combinatorial UX for fashion/apparel; verbose for simple cases.

**C. Hybrid per-product** — chosen. The store chooses **Matrix** or **Flat** mode per product based on whether variants need individual SKUs.
- Pros: matches the realities of mixed catalogs (apparel + electronics); the mode rule is explicit and decidable; preserves combinatorial UX for fashion while supporting per-SKU tracking for electronics.
- Cons: two code paths to support; mode is immutable per product (no in-place conversion).

**Decision rule for the store:** if one or more variants require an individual SKU, use Flat mode. Otherwise use Matrix.

### Cross-store product identity

**A. Independent per store** — chosen. Each store owns its own Product entities. No cross-store sharing.
- Pros: clean multi-tenant boundary; matches store autonomy; no platform-side canonical-product workflow needed.
- Cons: same physical item modeled twice if two stores sell it; cross-store deduplication for discovery becomes a search-layer concern.

**B. Platform-shared canonical Products** — rejected. Adds a platform-owned catalog of canonical items that stores reference. Useful at scale but premature; introduces governance overhead.

### Categorization

**A. Platform-defined hierarchical taxonomy** — chosen. The platform owns the Category tree; stores assign their products to platform categories.
- Pros: uniform Beckn vocabulary; consistent discovery surface; aligns with System Admin role from [ADR-0002](0002-authorization-tiers-and-matrix.md).
- Cons: every new category is a System Admin action (no self-service); requires up-front taxonomy seeding.

**B. Store-defined categories** — rejected. Each store building its own tree fragments the discovery surface and complicates Beckn category mapping.

**C. Hybrid (platform taxonomy + store tags)** — rejected. Store-private organization is already served by additional Catalogs from [ADR-0004](0004-store-publication-and-multi-catalog-projection.md); a separate tag concept duplicates that.

### Product lifecycle

**Draft → Active → Archived** — chosen. Mirrors the store lifecycle pattern from [ADR-0003](0003-store-lifecycle-and-state-machine.md). No deletion; Archived retains the record for order history.

Forbidden: Active → Draft (no "unpublish back to draft"); deletion of any product.

### Required vs. optional attributes

Baseline locked: name (required, store-scoped uniqueness), description (required), at least one platform category for `Active` state, SKU (optional, store-scoped uniqueness if set), media (optional ordered list), default price (Gap 09).

## Decision

### Product entity

A **Product** is the unit of "thing for sale," owned by a single store. Cross-store identity is independent — no Product entity is shared across stores.

Entity invariants:
- **Name** — required; unique within store (case-insensitive).
- **Description** — required; freeform text.
- **Mode** — `Matrix` or `Flat`; set at creation; **immutable**.
- **Category assignments** — zero or more `PlatformCategory` references; **at least one is required for the `Active` state** (entity-level guard on the Draft → Active transition).
- **SKU** — optional; unique within store (case-insensitive) if set; nullable.
- **Media** — ordered list of media references; first is primary; may be empty.
- **Default price** — deferred to [Gap 09](../gaps/09-pricing-and-promotions.md).

Lifecycle:

| From → To | Actor |
|---|---|
| (new) → Draft | Owner / Admin |
| Draft → Active | Owner / Admin (requires ≥1 category assignment) |
| Active → Archived | Owner / Admin |
| Archived → Active | Owner / Admin |

Forbidden: Active → Draft, any deletion, mode change.

### Variant strategy — two modes

A Product is in one of two modes, chosen at creation and immutable thereafter.

**Matrix Mode.**
- Product carries one or more **variant-defining attributes** (e.g., `Size: [S, M, L]`, `Color: [Red, Blue]`).
- **Variants** are entities representing chosen combinations of attribute values. Not every Cartesian combination must exist — the store enumerates which exist.
- Variants share the parent Product's SKU (variants have no individual SKU in this mode).
- Variants may have per-variant **media** (overrides parent media) and per-variant **price** (Gap 09).
- Used when per-variant individual SKU tracking is **not** required.

**Flat Mode.**
- Product has no variants relationship.
- Each "variant" of a real-world product family is modeled as a **separate, peer Product** with its own SKU.
- Stores group related Flat products via additional Catalogs from [ADR-0004](0004-store-publication-and-multi-catalog-projection.md) (e.g., a "Laptop X family" catalog).
- Used when per-variant individual SKU tracking **is** required.

**Decision rule for the store:** if one or more variants need their own SKU, use Flat mode. Otherwise use Matrix.

Mode is immutable: if a store discovers after going live that variants need SKUs, the path forward is to archive the Matrix product and recreate as Flat products.

### Categorization — platform taxonomy

The platform owns a **hierarchical Category taxonomy** (`PlatformCategory`):
- Tree structure (parent / child); initial depth 1–2 levels with room to grow.
- Categories are created, renamed, moved, and deprecated by **System Admins** (per [ADR-0002](0002-authorization-tiers-and-matrix.md)).
- Categories cannot be deleted while products are assigned to them.

Stores assign their Products to one or more `PlatformCategory` references via `ProductCategoryAssignment`. At least one assignment is an entity invariant on `Active` Products.

The Bridge derives Beckn `provider.categories[]` (or equivalent per protocol version) from the union of category assignments across a store's `Active` products. The mapping from `PlatformCategory` to Beckn category vocabulary lives in the Bridge's mapping registry.

No per-store category model. Store-private organization is served by additional Catalogs from [ADR-0004](0004-store-publication-and-multi-catalog-projection.md).

### Media

- A Product has an **ordered list** of media references; the first is the primary image.
- Variants in Matrix Mode may have their own ordered list; if absent, they inherit the Product's media.
- The domain knows only "media references" — storage, file handling, and CDN concerns live in Infrastructure.
- Recommended (not required at the entity level): at least one image for Active products. Enforced as a UX recommendation, not a domain invariant, to avoid blocking partial setups.

## Consequences

What this commits the system to:

- The Catalog context contains: `Catalog` (ADR-0004), `CatalogMembership`, `Product`, `ProductAttribute` (Matrix variant definitions), `ProductVariant` (Matrix), `PlatformCategory`, `ProductCategoryAssignment`, `Media`.
- **Inventory ([Gap 08](../gaps/08-inventory-boundary.md))** tracks availability per `Product` (Flat) or per `ProductVariant` (Matrix). The Variant entity is the Inventory anchor in Matrix mode.
- **Pricing ([Gap 09](../gaps/09-pricing-and-promotions.md))** can have variant-level overrides in Matrix mode; in Flat mode each Product has its own price.
- The **Bridge** projects Products and (Matrix) Variants as Beckn `item`s under the provider. The exact projection of Matrix variants (nested under parent vs. flat with shared parent metadata) lives in the Bridge mapping registry.
- **System Admin** has a new responsibility area: the platform Category taxonomy. New categories are an admin action; not store self-service.
- **Cross-store deduplication** for discovery is explicitly a search-layer concern, not a domain concern. The domain has no notion of "same product, different store."
- **Domain events** are emitted for all Catalog and Product mutations (full list in [design/catalog.md](../design/catalog.md)). The Bridge and Audit ([Gap 16](../gaps/16-soft-delete-and-audit.md)) consume them.
- Mode is an immutable per-product attribute; the system rejects mode changes.

What this defers:

- **Bundles / kits / composite products** — not in v1.
- **Digital vs. physical goods type taxonomy** — assume physical for v1; add a type discriminator later if digital goods enter scope.
- **Search / discovery implementation** (full-text, facets) — read-side index, infrastructure concern.
- **Media storage and file handling** — Infrastructure layer.
- **Per-variant inventory and pricing details** — [Gaps 08](../gaps/08-inventory-boundary.md), [09](../gaps/09-pricing-and-promotions.md).
- **Localization of strings** — [Gap 10](../gaps/10-localization-and-currency.md). Current entity attributes assume single-string fields.
- **Product reviews / ratings** — out of scope for v1.
- **Bulk import / export** — implementation feature; not a design decision.

What this makes harder:

- **Switching modes mid-life.** Forbidden by design. A Matrix product that later needs per-variant SKUs must be archived and recreated as Flat products.
- **Stock tracking per visible variant attribute in Matrix mode.** Matrix variants share a SKU but can still have their own availability flag — but if numeric stock per attribute combination is needed, the store should use Flat mode instead.
- **Adding new platform categories.** Every addition is a System Admin action; not store self-service. New categories require a platform process.
- **Cross-store product reconciliation.** No domain support; must be handled at the search/discovery layer.

## References

- [gaps/resolved/07-catalog-modeling-scope.md](../gaps/resolved/07-catalog-modeling-scope.md)
- [design/catalog.md](../design/catalog.md) — full entity model and contracts
- [ADR-0001](0001-bpp-network-identity.md), [ADR-0002](0002-authorization-tiers-and-matrix.md), [ADR-0003](0003-store-lifecycle-and-state-machine.md), [ADR-0004](0004-store-publication-and-multi-catalog-projection.md)
- Related gaps: [08](../gaps/08-inventory-boundary.md), [09](../gaps/09-pricing-and-promotions.md), [10](../gaps/10-localization-and-currency.md), [16](../gaps/16-soft-delete-and-audit.md)
- `bitemycart` — reference for the Matrix model pattern (Product / ProductVariant / ProductAttribute / ProductAttributeValue)
- `ion-specs` — Beckn `provider`, `category`, `item` concepts
