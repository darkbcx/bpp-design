# Gap 07 — Catalog modeling scope

## Statement

The Catalog context is named and listed alongside products, variants, attributes, media, taxonomies, pricing (§2.5), but no stance is taken on the structure of any of these. Catalog is also the largest piece of the domain by surface area, so the cost of getting its modeling wrong is high. The charter says Catalog must be "clean, extensible, and protocol-friendly" — those are constraints, not a design.

## Current charter coverage

* §2.5 — Catalog includes "products, variants, attributes, media, taxonomies, pricing."
* §3 — Generic modeling principles (value objects, normalization, aggregates).
* Project overview — "Inspired by bitemycart. Must NOT be UI-driven. Must be clean, extensible, and protocol-friendly."

## Open questions

1. **Variant strategy.** Matrix model (variants generated from attribute combinations like Size × Color) vs. flat model (variants are independent SKUs with their own attribute values)? Or both?
2. **Attributes vs. variants.** Which attributes define variants (price/SKU-bearing) vs. describe the product (informational, e.g., "fabric care")?
3. **Categorization model.** Hierarchical taxonomy (categories → subcategories), flat tags, or both? Store-defined, platform-defined, or hybrid? How does this map to Beckn category codes (Gap interaction with Bridge)?
4. **Media model.** Multiple images per product, ordered? Per-variant images? Video / 3D? Where are binaries stored (infrastructure decision but with domain implications)? Alt text and accessibility?
5. **Product attribution.** Which fields are required, which optional? Default unit-of-measure? Brand attribution? SKU uniqueness scope (store vs. global)?
6. **Lifecycle / publication state.** Draft → Published → Unpublished → Archived for products? Per-variant?
7. **Search / discovery.** Is full-text search inside Catalog, or a separate read-side index? Faceted filters?
8. **Bundles / kits / composite products.** In scope or out?
9. **Digital vs. physical goods.** Both supported, or physical-only at launch?
10. **Cross-store catalog reuse.** If two stores sell the same SKU, is the product entity shared, or duplicated?

## Implications

* Catalog modeling drives the storage layer's shape more than any other context.
* The mapping to Beckn `Item` / `Provider` shapes lives entirely in the Bridge but depends on the domain having enough information to project (descriptors, prices, media, categories).
* Bad variant modeling tends to propagate everywhere — pricing, inventory, orders, search.

## Dependencies

* Depends on: nothing strictly, but 09 (pricing) and 10 (currency/locale) intertwine.
* Blocks: 08 (inventory), 09 (pricing), most of the storefront.
