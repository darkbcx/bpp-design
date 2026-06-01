# Design: Catalog and Product Model

- **Status**: Draft
- **Last updated**: 2026-06-01
- **Backed by ADRs**: [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md) (Catalog container), [ADR-0005](../decisions/0005-catalog-and-product-modeling.md) (Product modeling — base), [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md) (four-mode discriminator; Composite and Configurable Products)

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
- **Admin UI** — renders the catalog to store owners/admins for management (the platform is a pure BPP with no buyer-facing storefront, per [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md)).
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
- `mode` — `Standalone` | `Variant` | `Configurable` | `Composite`; set at creation; **immutable**. See [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md).
- `sku` — optional; **unique within store** if set; nullable.
- `status` — `Draft` | `Active` | `Archived`.
- `media` — ordered list of `Media` references (may be empty); first is primary.
- `base_price` — tax-excluded `Money` (in the store's currency); required for `Active`. See [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md). For `Configurable`, this is the configurator's starting price (choice `price_delta`s compose at quote time). For `Composite`, this is the bundle's own price (typically discounted vs. component sum; component prices are informational).
- `tax_rate` — decimal percentage (e.g., `18.00`); required for `Active`. Variants inherit the parent's `tax_rate`; Configurable / Composite use the Product's own `tax_rate` (choice `price_delta`s and component prices do not carry separate rates).
- Derived (cached, recomputed on `base_price` or `tax_rate` change): `tax_amount`, `published_price`.
- `variant_attributes` — only in `Variant` mode; ordered list of `ProductAttribute` definitions.
- `category_assignments` — set of `ProductCategoryAssignment`; **at least one is required for `Active`**.

Mode-specific sub-entities (see below for each):

| Mode | Admits | Forbids |
|---|---|---|
| `Standalone` | — | `ProductVariant`, `ProductComponent`, `ChoiceGroup` |
| `Variant` | `ProductVariant` (≥1 for `Active`) | `ProductComponent`, `ChoiceGroup` |
| `Configurable` | `ChoiceGroup` (≥1 for `Active`, each with ≥1 `Choice`) | `ProductVariant`, `ProductComponent` |
| `Composite` | `ProductComponent` (≥1 for `Active`) | `ProductVariant`, `ChoiceGroup` |

Mode is mutually exclusive in v1 — no variant bundles, no configurable variants, no composite configurables. See [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md) §"Mode combinations".

### ProductAttribute *(Variant mode only)*

A variant-defining attribute on a Variant Product (e.g., "Size" / "Ukuran", "Color" / "Warna").

Attributes:
- `name` — **`LocalizedText`** (e.g., `{id: "Ukuran", en: "Size"}`).
- `allowed_values` — ordered list of permissible values. Each value has a locale-neutral identifier and an optional **`LocalizedText`** label (e.g., identifier `"m"` with label `{id: "Sedang", en: "Medium"}`). Variants reference values by identifier; display uses the label.

The Cartesian product of a Product's `variant_attributes` defines the *space* of possible variants. The store enumerates which combinations actually exist as `ProductVariant` entities.

### ProductVariant *(Variant mode only)*

A specific combination of attribute values for a Variant Product.

Attributes:
- `id` — internal opaque identifier.
- `product_id` — parent Product reference; immutable.
- `attribute_values` — one value per `ProductAttribute` of the parent (e.g., `{Size: M, Color: Red}`).
- `media` — optional ordered list; if absent, inherits parent's media.
- `price_override` — optional tax-excluded `Money` (in the store's currency). See [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md). Variants inherit the parent's `tax_rate`; effective `tax_amount` and `published_price` are derived from `(price_override ?? parent.base_price) × parent.tax_rate`.

Variants share their parent's name, description, SKU (if any), `tax_rate`, and category assignments. A variant does not carry its own `ProductComponent` or `ChoiceGroup` (mutual exclusion per [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md)). **Availability and stock are owned by the Inventory context** (see [ADR-0006](../decisions/0006-inventory-model.md), [ADR-0024](../decisions/0024-inventory-for-composite-and-configurable-products.md)) — the Catalog domain knows nothing about whether a Variant is in stock or purchasable.

Invariant: a Product cannot have two variants with the same attribute-value combination.

### ProductComponent *(Composite mode only)*

A part of a Composite Product. See [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md).

Attributes:
- `id` — internal opaque identifier.
- `parent_product_id` — references the Composite Product; immutable.
- `component_anchor_ref` — opaque reference to a `Standalone` Product *or* a `ProductVariant` of a `Variant`-mode Product. Same opaque-ref shape used by Inventory's `StockLevel.inventory_item_ref` (per [ADR-0006](../decisions/0006-inventory-model.md)).
- `quantity` — integer ≥ 1; how many of the referenced anchor are consumed per Composite unit.
- `display_order` — position within the parent's ordered component list.

Invariants:
- A Composite cannot reference another Composite (depth = 1).
- A Composite cannot reference a Configurable.
- The same `component_anchor_ref` may appear at most once in a Composite's component list (use `quantity` to express multiples).
- A Composite must have at least one `ProductComponent` to be `Active`.

### ChoiceGroup *(Configurable mode only)*

A decision point on a Configurable Product. See [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md).

Attributes:
- `id` — internal opaque identifier.
- `product_id` — references the Configurable Product; immutable.
- `name` — **`LocalizedText`**, required.
- `selection_type` — one of:
  - `RequiredSingle` — exactly one option must be chosen.
  - `OptionalSingle` — at most one option may be chosen.
  - `OptionalMultiple` — between `min_selections` and `max_selections` options may be chosen.
- `min_selections` — integer ≥ 0; meaningful only for `OptionalMultiple`; default `0`.
- `max_selections` — integer ≥ 1; meaningful only for `OptionalMultiple`; default unbounded.
- `display_order` — position within the parent's ordered ChoiceGroup list.

Invariants:
- For `OptionalMultiple`: `0 ≤ min_selections ≤ max_selections`.
- Every `ChoiceGroup` must have at least one `Choice` when its parent Product is `Active`.
- A Configurable Product must have at least one `ChoiceGroup` to be `Active`.

### Choice *(within a ChoiceGroup)*

A single selectable option. See [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md).

Attributes:
- `id` — internal opaque identifier.
- `choice_group_id` — references the parent `ChoiceGroup`; immutable.
- `label` — **`LocalizedText`**, required.
- `price_delta` — optional `Money` (in the store's currency, tax-excluded); default zero; may be negative.
- `option_product_ref` — optional opaque reference to a `Standalone` Product *or* a `ProductVariant` of a `Variant`-mode Product. When set, selecting this Choice depletes the referenced anchor's stock at order confirm (per [ADR-0024](../decisions/0024-inventory-for-composite-and-configurable-products.md)).
- `display_order` — position within the parent ChoiceGroup's ordered Choice list.

Invariants:
- `option_product_ref` may not reference a Composite Product.
- `option_product_ref` may not reference a Configurable Product.

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
Store (Tenancy ctx) ─────────< Product ─────────< ProductVariant (Variant mode)
                                  │                       │
                                  │                       └── Media (ordered, optional)
                                  │
                                  ├─< ProductCategoryAssignment >─── PlatformCategory
                                  │                                  (hierarchical, System-admin-owned)
                                  │
                                  ├─ ProductAttribute (Variant mode, variant-defining)
                                  │
                                  ├─< ProductComponent (Composite mode)
                                  │      │
                                  │      └── component_anchor_ref → Standalone Product
                                  │                                  or ProductVariant
                                  │
                                  ├─< ChoiceGroup (Configurable mode) ──< Choice
                                  │                                          │
                                  │                                          └── option_product_ref (optional)
                                  │                                              → Standalone Product or ProductVariant
                                  │
                                  └─ Media (ordered)

Store ─< Catalog ─< CatalogMembership >─ Product
                  (inclusion record for additional Catalogs;
                   Default Catalog membership is implicit and mandatory)
```

## Lifecycles

### Product lifecycle

States: `Draft`, `Active`, `Archived`.

| From → To | Actor | Guard |
|---|---|---|
| (new) → Draft | Owner / Admin | none |
| Draft → Active | Owner / Admin | requires ≥1 category assignment **plus** mode-specific guards (see below) |
| Active → Archived | Owner / Admin | none |
| Archived → Active | Owner / Admin | requires ≥1 (still valid, non-deprecated) category assignment plus mode-specific guards |

**Mode-specific guards for `Active`:**

| Mode | Guard |
|---|---|
| `Standalone` | (none beyond shared guards) |
| `Variant` | at least one `ProductVariant` exists |
| `Configurable` | at least one `ChoiceGroup` exists, each with at least one `Choice` |
| `Composite` | at least one `ProductComponent` exists |

**Forbidden transitions:** Active → Draft; any deletion; mode change.

`Draft` is invisible regardless of catalog membership. `Active` visibility depends on catalog membership *and* the store's lifecycle state (per [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md)). `Archived` is invisible everywhere but the record is retained for order history.

### ProductVariant lifecycle *(Variant mode)*

Variants are owned by their parent Product. No independent state machine. A variant exists or doesn't; deletion is allowed but only when no order references it; otherwise it is *retired* (kept but flagged).

### ProductComponent, ChoiceGroup, Choice lifecycle

These entities are owned by their parent Product and follow its lifecycle — they have no independent state machine. They may be added, removed, or updated freely while the parent is `Draft` or `Active`; archiving the parent freezes the structure. Deletion of an individual component or choice is permitted only when no order line references it; otherwise the entity is retained for history.

### Catalog lifecycle *(recap from ADR-0004)*

- Default Catalog: created with the store, never deleted.
- Additional Catalog: created / renamed / deleted by store users.

### PlatformCategory lifecycle

Managed by System Admins. Transitions: created → Active → Deprecated. Deletion only when no assignments reference it.

## Invariants

- `Product.store_id` is immutable; a Product belongs to one Store forever.
- `Product.mode` is immutable; no in-place conversion between modes.
- `Product.status == Active` ⇒ at least one `ProductCategoryAssignment` **and** the mode-specific guard is satisfied (see Lifecycles).
- `Product.name` unique within store (case-insensitive).
- `Product.sku` unique within store (case-insensitive), if set.
- `ProductVariant`: unique attribute-value combination per parent Product.
- `Catalog`: each Store has **exactly one** Default Catalog. Default Catalog cannot be deleted.
- `PlatformCategory`: cannot be deleted while any assignment references it.

**Mode admit/forbid (mutual exclusion, per [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md)):**
- `Standalone` Products have no `ProductAttribute`, no `ProductVariant`, no `ProductComponent`, no `ChoiceGroup`.
- `Variant` Products have ≥1 `ProductAttribute` and ≥0 `ProductVariant` (≥1 for `Active`); no `ProductComponent`, no `ChoiceGroup`.
- `Configurable` Products have no `ProductAttribute`, no `ProductVariant`, no `ProductComponent`; ≥0 `ChoiceGroup` (≥1 for `Active`).
- `Composite` Products have no `ProductAttribute`, no `ProductVariant`, no `ChoiceGroup`; ≥1 `ProductComponent` (for `Active`).

**Composite-specific:**
- A `ProductComponent.component_anchor_ref` may not reference a Composite Product (no nested composites).
- A `ProductComponent.component_anchor_ref` may not reference a Configurable Product.
- The same `component_anchor_ref` may appear at most once per parent Composite.

**Configurable-specific:**
- For `ChoiceGroup.selection_type == OptionalMultiple`: `0 ≤ min_selections ≤ max_selections`.
- A `Choice.option_product_ref` may not reference a Composite or Configurable Product.
- Every `ChoiceGroup` of an `Active` Configurable Product has ≥1 `Choice`.

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
- `AddVariantAttribute(product_id, attribute_name, allowed_values)` *(Variant mode only)*
- `AddProductVariant(product_id, attribute_values)` *(Variant mode only)*
- `UpdateProductVariant(variant_id, ...)`
- `RemoveProductVariant(variant_id)` *(or retire if referenced by orders)*
- `AddProductComponent(product_id, component_anchor_ref, quantity)` *(Composite only)*
- `UpdateProductComponentQuantity(component_id, quantity)` *(Composite only)*
- `RemoveProductComponent(component_id)` *(Composite only; retire if referenced by orders)*
- `AddChoiceGroup(product_id, name, selection_type, ...)` *(Configurable only)*
- `UpdateChoiceGroup(choice_group_id, ...)` *(Configurable only)*
- `RemoveChoiceGroup(choice_group_id)` *(Configurable only; retire if referenced by orders)*
- `AddChoice(choice_group_id, label, price_delta, option_product_ref?)` *(Configurable only)*
- `UpdateChoice(choice_id, ...)` *(Configurable only)*
- `RemoveChoice(choice_id)` *(Configurable only; retire if referenced by orders)*
- `UpdateProductMedia(product_id, ordered_media)`
- `UpdateVariantMedia(variant_id, ordered_media)` *(Variant mode only)*
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

Variants & attributes *(Variant mode)*:
- `ProductVariantAttributeAdded`, `ProductVariantAdded`, `ProductVariantUpdated`, `ProductVariantRemoved`

Components *(Composite mode)*:
- `ProductComponentAdded`, `ProductComponentRemoved`, `ProductComponentQuantityChanged`

Choice groups & choices *(Configurable mode)*:
- `ChoiceGroupAdded`, `ChoiceGroupUpdated`, `ChoiceGroupRemoved`
- `ChoiceAdded`, `ChoiceUpdated`, `ChoiceRemoved`

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

| Domain entity / shape | Beckn projection (v2 / ION) |
|---|---|
| `Store` (Active) | `provider` node |
| `Catalog` (Default + additional) | catalog / category groupings under provider |
| `PlatformCategory` | `provider.categories[]` entry, referenced from items |
| `Product` mode = `Standalone` | `Resource` with `resourceStructure: PLAIN` |
| `Product` mode = `Variant` (parent + variants) | `PLAIN` parent Resource + sibling `VARIANT` Resources (one per `ProductVariant`) |
| `Product` mode = `Configurable`, all `ChoiceGroup`s Optional | `Resource` with `resourceStructure: WITH_EXTRAS`; `customisationGroups[]` from ChoiceGroups |
| `Product` mode = `Configurable`, any `ChoiceGroup` `RequiredSingle` | `Resource` with `resourceStructure: COMPOSED`, standalone; `customisationGroups[]` from ChoiceGroups |
| `Product` mode = `Composite` | `Resource` with `resourceStructure: BUNDLE` + `bundleComposition[]`; paired with a BUNDLE-type Offer |
| `ChoiceGroup.selection_type` | `customisationGroups[].selectionType`: `RequiredSingle → SINGLE_REQUIRED`, `OptionalSingle → SINGLE_OPTIONAL`, `OptionalMultiple → MULTIPLE_OPTIONAL` |
| `Choice.option_product_ref` | `options[].resourceId` |
| `Media` | `item.descriptor.images[]` (URIs resolved by Bridge) |

What the domain provides to the Bridge (and the Bridge consumes via query / event):
- Stable product / variant / component / choice-group / choice / catalog identifiers.
- Mode discriminator, variant existence and attribute values, component lists, choice-group / choice structure.
- Category assignments.
- Media references (the Bridge resolves to URLs).
- Catalog memberships.
- Effective availability per Product / ProductVariant (queried from Inventory; see [ADR-0024](../decisions/0024-inventory-for-composite-and-configurable-products.md)).

What the domain does NOT know:
- Beckn schema versions or `resourceStructure` vocabulary.
- Beckn category code vocabulary.
- Wire-level item formatting, customisation-group selectionType enums, BUNDLE / WITH_EXTRAS / COMPOSED naming.

## Open questions (within this design)

- **Per-variant inventory and pricing** — [ADR-0006](../decisions/0006-inventory-model.md) (StockLevel per Variant) and [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md) (`price_override` per Variant).
- **Composite and Configurable inventory** — owned by [ADR-0024](../decisions/0024-inventory-for-composite-and-configurable-products.md) (`OwnerPurchasability` entity, reservation fan-out, effective-availability formulas).
- **Variant bundles** (e.g., Family Combo S/M/L as native siblings) — deferred per [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md). v1 workaround: three separate Composite Products, optionally grouped via an additional Catalog.
- **Configurable variants** (e.g., per-size extras menus) — deferred per [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md). v1 workaround: one Configurable Product with size and extras as parallel `ChoiceGroup`s.
- **Nested composites** — deferred per [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md). v1 workaround: pre-configure the inner composite as a Standalone Product.
- **Cross-store dedup for discovery** — explicitly out of the domain; will be a search-layer concern.
- **Bulk import / export** — implementation feature, not design.

## References

- [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md), [ADR-0005](../decisions/0005-catalog-and-product-modeling.md), [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md), [ADR-0024](../decisions/0024-inventory-for-composite-and-configurable-products.md)
- CLAUDE.md §2.5 (bounded contexts), §3 (data modeling), §4 (Bridge), §5.9 (publication and catalogs), §5.10 (catalog and product model)
- Related gaps: [08](../gaps/resolved/08-inventory-boundary.md), [09](../gaps/resolved/09-pricing-and-promotions.md), [10](../gaps/resolved/10-localization-and-currency.md), [13](../gaps/resolved/13-domain-events-design.md), [16](../gaps/resolved/16-soft-delete-and-audit.md)
- `bitemycart` — reference for the original Matrix model pattern
- `ion-specs` — Beckn `provider`, `category`, `Resource`; `my_ref/BPP_RESOURCE_STRUCTURE_CHEATSHEET.md` and the per-structure design docs (PLAIN, VARIANT, WITH_EXTRAS, COMPOSED, BUNDLE)
