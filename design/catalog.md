# Design: Catalog and Product Model

- **Status**: Draft
- **Last updated**: 2026-05-28
- **Backed by ADRs**: [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md) (Catalog container), [ADR-0005](../decisions/0005-catalog-and-product-modeling.md) (Product modeling)

## Purpose

This document is the full design of the Catalog context: entities, relationships, lifecycles, invariants, and boundary contracts. It is the reference the implementing team will use to build the Catalog subsystem.

**What this document covers:**
- All entities and their attributes.
- Relationships among them.
- State machines (Product lifecycle in particular).
- Boundary contracts (use cases, domain events, queries) the context exposes.
- Beckn projection notes — what the Bridge needs from this domain.

**What this document does NOT cover:**
- Pricing details — [Gap 09](../gaps/09-pricing-and-promotions.md).
- Inventory / stock tracking — [Gap 08](../gaps/08-inventory-boundary.md).
- Localization — [Gap 10](../gaps/10-localization-and-currency.md).
- Storage / persistence implementation — Infrastructure layer.
- UI flows — Interface Layer, out of scope for design.

## Position within the architecture

The Catalog context lives in its own bounded context (CLAUDE.md §2.5), sitting in the Domain Layer (§2.4) with use cases in the Application Layer.

It depends on:
- **Tenancy context** — references Store by ID; no joins (§2.6).
- **Identity & Access context** — actor identity for authorization (per [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md)).

It is consumed by:
- **Beckn Bridge** (§4) — projects to `provider`, `category`, and `item` on the wire.
- **Storefront** — renders products to buyers.
- **Admin UI** — owners author products and catalogs.
- **Inventory** ([Gap 08](../gaps/08-inventory-boundary.md)) — references Products / Variants for availability.
- **Pricing** ([Gap 09](../gaps/09-pricing-and-promotions.md)) — references Products / Variants for prices.
- **Order context** ([Gap 11](../gaps/11-order-and-fulfillment-phasing.md)) — references Products / Variants at purchase time.

## Concepts

### Catalog *(recap from [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md))*

A named, ordered container of Products within a store.

Two kinds:
- **Default Catalog** — exactly one per store; auto-created when the store is created; not deletable. Membership is **mandatory** ([ADR-0020](../decisions/0020-default-catalog-mandatory.md)): every Product in the store is automatically and permanently a member. There is no exclusion mechanism; to hide a product from Beckn, change the Product's `status` (Draft or Archived).
- **Additional Catalog** — created by store users; scoped to that store's products. Membership is **opt-in**: products must be explicitly added.

A Product may belong to **zero or more** Catalogs simultaneously.

All Catalogs project to the Beckn network. The Bridge owns the projection shape.

### Product

The unit of "thing for sale." Owned by a single Store; **not shared across stores**.

Attributes:
- `id` — internal opaque identifier.
- `store_id` — owning Store reference; immutable.
- `name` — required; **`LocalizedText`**. Uniqueness check applies to the default-locale (`id`) value, case-insensitive. See [ADR-0008](../decisions/0008-localization-and-localizedtext.md).
- `description` — required; **`LocalizedText`**.
- `mode` — `Matrix` or `Flat`; set at creation; **immutable**.
- `sku` — optional; **unique within store** if set; nullable.
- `status` — `Draft` | `Active` | `Archived`.
- `media` — ordered list of `Media` references (may be empty); first is primary.
- `base_price` — tax-excluded `Money` (in the store's currency); required for `Active`. See [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md).
- `tax_rate` — decimal percentage (e.g., `18.00`); required for `Active`. Variants inherit the parent's `tax_rate`.
- Derived (cached, recomputed on `base_price` or `tax_rate` change): `tax_amount`, `published_price`.
- `variant_attributes` — only in Matrix mode; ordered list of `ProductAttribute` definitions.
- `category_assignments` — set of `ProductCategoryAssignment`; **at least one is required for `Active`**.

### ProductAttribute *(Matrix mode only)*

A variant-defining attribute on a Matrix Product (e.g., "Size" / "Ukuran", "Color" / "Warna").

Attributes:
- `name` — **`LocalizedText`** (e.g., `{id: "Ukuran", en: "Size"}`).
- `allowed_values` — ordered list of permissible values. Each value has a locale-neutral identifier and an optional **`LocalizedText`** label (e.g., identifier `"m"` with label `{id: "Sedang", en: "Medium"}`). Variants reference values by identifier; display uses the label.

The Cartesian product of a Product's `variant_attributes` defines the *space* of possible variants. The store enumerates which combinations actually exist as `ProductVariant` entities.

### ProductVariant *(Matrix mode only)*

A specific combination of attribute values for a Matrix Product.

Attributes:
- `id` — internal opaque identifier.
- `product_id` — parent Product reference; immutable.
- `attribute_values` — one value per `ProductAttribute` of the parent (e.g., `{Size: M, Color: Red}`).
- `media` — optional ordered list; if absent, inherits parent's media.
- `price_override` — optional tax-excluded `Money` (in the store's currency). See [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md). Variants inherit the parent's `tax_rate`; effective `tax_amount` and `published_price` are derived from `(price_override ?? parent.base_price) × parent.tax_rate`.

Variants share their parent's name, description, SKU (if any), `tax_rate`, and category assignments. **Availability and stock are owned by the Inventory context** (see [ADR-0006](../decisions/0006-inventory-model.md)) — the Catalog domain knows nothing about whether a Variant is in stock or purchasable.

Invariant: a Product cannot have two variants with the same attribute-value combination.

### PlatformCategory

A node in the platform-wide hierarchical category taxonomy.

Attributes:
- `id` — internal opaque identifier.
- `name` — **`LocalizedText`**; required; **unique among siblings** (uniqueness applied to the default-locale value). See [ADR-0008](../decisions/0008-localization-and-localizedtext.md).
- `parent_id` — optional reference to another `PlatformCategory`; nullable means root.
- `status` — `Active` | `Deprecated`.

Managed by **System Admins** (per [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md)). Store owners cannot create, rename, or delete categories.

`name` is a **`LocalizedText`** with the default locale (`id`) required at creation; other locales may be added by the System Admin (see [ADR-0008](../decisions/0008-localization-and-localizedtext.md)).

A `PlatformCategory` cannot be deleted while any `ProductCategoryAssignment` references it. Deprecated categories remain referenceable for legacy assignments but are not selectable for new ones.

### ProductCategoryAssignment

The many-to-many link between `Product` and `PlatformCategory`.

Attributes:
- `product_id`
- `category_id`
- `assigned_at`, `assigned_by`

### CatalogMembership

Tracks `Product` membership in **additional Catalogs only** ([ADR-0020](../decisions/0020-default-catalog-mandatory.md)):

- **For the Default Catalog**: no row required. Default membership is implicit and mandatory — every store Product is a member.
- **For an additional Catalog**: an *inclusion* record (presence = member). Opt-in.

### Media

A reference to a media file (image; video in the future).

Attributes:
- `id` — internal opaque identifier.
- `uri` — storage reference (URL or storage key; Infrastructure-managed).
- `alt_text` — optional **`LocalizedText`**, accessibility hint.
- `order_index` — position within the parent's ordered list.

## Relationships

```
Store (Tenancy ctx) ─────────< Product ─────────< ProductVariant (Matrix only)
                                  │                       │
                                  │                       └── Media (ordered, optional)
                                  │
                                  ├─< ProductCategoryAssignment >─── PlatformCategory
                                  │                                  (hierarchical, System-admin-owned)
                                  │
                                  ├─ ProductAttribute (Matrix only, variant-defining)
                                  │
                                  └─ Media (ordered)

Store ─< Catalog ─< CatalogMembership >─ Product
                  (exclusion for Default,
                   inclusion for additional)
```

## Lifecycles

### Product lifecycle

States: `Draft`, `Active`, `Archived`.

| From → To | Actor | Guard |
|---|---|---|
| (new) → Draft | Owner / Admin | none |
| Draft → Active | Owner / Admin | requires ≥1 category assignment |
| Active → Archived | Owner / Admin | none |
| Archived → Active | Owner / Admin | requires ≥1 (still valid, non-deprecated) category assignment |

**Forbidden transitions:** Active → Draft; any deletion; mode change.

`Draft` is invisible regardless of catalog membership. `Active` visibility depends on catalog membership *and* the store's lifecycle state (per [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md)). `Archived` is invisible everywhere but the record is retained for order history.

### ProductVariant lifecycle *(Matrix mode)*

Variants are owned by their parent Product. No independent state machine. A variant exists or doesn't; deletion is allowed but only when no order references it; otherwise it is *retired* (kept but flagged).

### Catalog lifecycle *(recap from ADR-0004)*

- Default Catalog: created with the store, never deleted.
- Additional Catalog: created / renamed / deleted by store users.

### PlatformCategory lifecycle

Managed by System Admins. Transitions: created → Active → Deprecated. Deletion only when no assignments reference it.

## Invariants

- `Product.store_id` is immutable; a Product belongs to one Store forever.
- `Product.mode` is immutable; no in-place Matrix ↔ Flat conversion.
- `Product.status == Active` ⇒ at least one `ProductCategoryAssignment`.
- `Product.name` unique within store (case-insensitive).
- `Product.sku` unique within store (case-insensitive), if set.
- `ProductVariant`: unique attribute-value combination per parent Product.
- `Catalog`: each Store has **exactly one** Default Catalog. Default Catalog cannot be deleted.
- `PlatformCategory`: cannot be deleted while any assignment references it.
- A Product in Matrix mode has at least one `ProductAttribute` defined before it can have variants.
- A Product in Flat mode has no `ProductAttribute` entries and no `ProductVariant` entries.

## Boundary contracts

### Use cases exposed by the Application Layer

**Store-scoped actors** (Owner / Admin / Membership roles per [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md)):
- `CreateProduct(store, name, mode, ...)`
- `UpdateProduct(product_id, ...)`
- `PublishProduct(product_id)` — Draft → Active
- `ArchiveProduct(product_id)` — Active → Archived
- `RestoreProduct(product_id)` — Archived → Active
- `AssignProductToCategory(product_id, category_id)`
- `UnassignProductFromCategory(product_id, category_id)`
- `AddVariantAttribute(product_id, attribute_name, allowed_values)` *(Matrix only)*
- `AddProductVariant(product_id, attribute_values)` *(Matrix only)*
- `UpdateProductVariant(variant_id, ...)`
- `RemoveProductVariant(variant_id)` *(or retire if referenced by orders)*
- `UpdateProductMedia(product_id, ordered_media)`
- `UpdateVariantMedia(variant_id, ordered_media)` *(Matrix only)*
- `CreateCatalog(store, name)`
- `RenameCatalog(catalog_id, new_name)`
- `DeleteCatalog(catalog_id)` *(additional Catalog only)*
- `IncludeProductInCatalog(catalog_id, product_id)` *(additional Catalog only; the Default Catalog has implicit, mandatory membership per [ADR-0020](../decisions/0020-default-catalog-mandatory.md))*
- `RemoveProductFromCatalog(catalog_id, product_id)` *(additional Catalog only)*

**System Admin actors:**
- `CreatePlatformCategory(parent_id, name)`
- `RenamePlatformCategory(category_id, new_name)`
- `MovePlatformCategory(category_id, new_parent_id)`
- `DeprecatePlatformCategory(category_id)`
- `DeletePlatformCategory(category_id)` *(only if no assignments)*

**Read-side queries:**
- `ListActiveProductsForStore(store_id) → [Product]`
- `GetProductDetail(product_id) → Product with variants, attributes, categories, media`
- `ListCatalogsForStore(store_id) → [Catalog]`
- `GetCatalogContents(catalog_id) → [Product]`
- `GetPlatformCategoryTree() → tree`

### Domain events emitted

Lifecycle:
- `ProductCreated`, `ProductUpdated`, `ProductPublished`, `ProductArchived`, `ProductRestored`

Variants & attributes:
- `ProductVariantAttributeAdded`, `ProductVariantAdded`, `ProductVariantUpdated`, `ProductVariantRemoved`

Categories:
- `ProductCategoryAssigned`, `ProductCategoryUnassigned`
- `PlatformCategoryCreated`, `PlatformCategoryRenamed`, `PlatformCategoryMoved`, `PlatformCategoryDeprecated`

Catalogs:
- `CatalogCreated`, `CatalogRenamed`, `CatalogDeleted`
- `ProductIncludedInCatalog`, `ProductRemovedFromCatalog` *(additional Catalogs only — the Default Catalog has mandatory implicit membership; no add/remove events per [ADR-0020](../decisions/0020-default-catalog-mandatory.md))*

Media:
- `ProductMediaUpdated`, `ProductVariantMediaUpdated`

All events are consumed at least by:
- The **Beckn Bridge** for republication (mirrors the `StoreStatusChanged` pattern from [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md)).
- **Audit** ([Gap 16](../gaps/16-soft-delete-and-audit.md)) for state history.

Event delivery semantics are owned by [Gap 13](../gaps/13-domain-events-design.md); this design only declares the events.

## Beckn projection notes

The Bridge projects (mapping registry owns the exact shape per protocol version):

| Domain entity | Beckn projection |
|---|---|
| `Store` (Active) | `provider` node |
| `Catalog` (Default + additional) | catalog / category groupings under provider |
| `PlatformCategory` | `provider.categories[]` entry, referenced from items |
| `Product` (Flat or Matrix parent) | `item` |
| `ProductVariant` (Matrix) | child `item` under parent, or `item` with parent reference — protocol-version-dependent |
| `Media` | `item.descriptor.images[]` (URIs resolved by Bridge) |

What the domain provides to the Bridge (and the Bridge consumes via query / event):
- Stable product / variant / catalog identifiers.
- Variant existence and attribute values.
- Category assignments.
- Media references (the Bridge resolves to URLs).
- Catalog memberships.

What the domain does NOT know:
- Beckn schema versions.
- Beckn category code vocabulary.
- Wire-level item formatting.

## Open questions (within this design)

- **Per-variant inventory and pricing** — Gaps 08 and 09 own these. This design exposes the `ProductVariant` entity; those gaps will attach availability and price.
- **Localization** — [Gap 10](../gaps/10-localization-and-currency.md). Current string fields are single-string; may become locale maps later.
- **Cross-store dedup for discovery** — explicitly out of the domain; will be a search-layer concern.
- **Bulk import / export** — implementation feature, not design.

## References

- [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md), [ADR-0005](../decisions/0005-catalog-and-product-modeling.md)
- CLAUDE.md §2.5 (bounded contexts), §3 (data modeling), §4 (Bridge), §5.9 (publication and catalogs), §5.10 (catalog and product model)
- Related gaps: [08](../gaps/08-inventory-boundary.md), [09](../gaps/09-pricing-and-promotions.md), [10](../gaps/10-localization-and-currency.md), [13](../gaps/13-domain-events-design.md), [16](../gaps/16-soft-delete-and-audit.md)
- `bitemycart` — reference for the Matrix model
- `ion-specs` — Beckn `provider`, `category`, `item`
