# ADR-0025: Optional per-variant SKU; Pattern B is the sole Variant-mode projection

- **Status**: Accepted
- **Date**: 2026-06-01
- **Builds on**: [ADR-0005](0005-catalog-and-product-modeling.md), [ADR-0023](0023-product-composition-and-choice-modeling.md)
- **Partially supersedes**: [ADR-0005](0005-catalog-and-product-modeling.md) — the rule "Variants share the parent Product's SKU (variants have no individual SKU in this mode)" is replaced; variants may now carry an optional own SKU. The decision rule for choosing Variant vs. Standalone mode also shifts (see below). Everything else in ADR-0005 survives.

## Context

[ADR-0005](0005-catalog-and-product-modeling.md) established the original Matrix-mode constraint: variants share the parent Product's SKU; if per-variant SKU is needed, use Flat mode. [ADR-0023](0023-product-composition-and-choice-modeling.md) renamed the modes (Matrix → Variant; Flat → Standalone) but preserved the shared-SKU rule. The Beckn projection in ADR-0023 was set as Pattern B: PLAIN parent + sibling VARIANT Resources per `ProductVariant`.

Two pressures converge to revisit this:

1. **Real retailers expect per-variant SKU/GTIN.** Phones in color × storage; t-shirt SKUs that track per-size inventory and barcode separately. Forcing those into Standalone mode keeps the data right but loses the variant-axis grouping (the parent → variant relationship, the picker UX, the family identity in the catalog).

2. **Beckn's VARIANT Resource shape is built for own SKUs.** The Pattern B cheatsheet explicitly says "Each combo has own GTIN/stock/policies." With our domain projecting as Pattern B but withholding per-variant SKU, the wire shape carries less than the schema expects.

These pressures point in the same direction. The user direction was explicit: make per-variant SKU optional, and keep Pattern B as the only projection for Variant mode — no Pattern A (`variantMatrix`) optimization in v1.

## Options considered

### SKU semantics for ProductVariant

**A. Shared with parent (status quo, ADR-0005).**
- Variants have no own SKU; the parent's SKU is the only one.
- **Pros.** Simplest. One SKU per family.
- **Cons.** Forces "I need per-variant SKU" into Standalone mode, losing variant-axis grouping.

**B. Required per-variant SKU.**
- Every variant must declare its own SKU.
- **Pros.** Uniformity; every Variant on the wire has full identity.
- **Cons.** Many stores genuinely don't want to assign per-variant SKUs (the t-shirt-sizes case). Mandatory makes the simple case heavier.

**C. Optional per-variant SKU** — chosen.
- `ProductVariant.sku` is optional and nullable. If set, unique within the store across all `Product.sku` and `ProductVariant.sku` values (single namespace).
- **Pros.** Owners pick what fits — assign per-variant SKUs where they matter, leave them blank where they don't. Aligns with Beckn's VARIANT Resource shape, which treats SKU/GTIN as optional fields.
- **Cons.** Two SKU sources to look at (parent's, variant's) when resolving a variant's identity. Mitigated by a clear precedence: if the variant has its own SKU, that's the variant's SKU on the wire; otherwise the variant inherits the parent's (or, if the parent has none either, the wire SKU is empty).

### Projection of Variant-mode Products

**A. Always Pattern B (PLAIN parent + sibling VARIANT Resources)** — chosen.
- Every `ProductVariant` projects as its own VARIANT Resource regardless of whether it carries an own SKU, price_override, or media.
- **Pros.** One projection rule. The Bridge mapping registry is simpler. Each Variant has a stable wire Resource ID — BAPs can reference variants directly across `/select`, `/init`, `/confirm`.
- **Cons.** A 24-combination t-shirt produces 25 Resources on the wire (1 parent + 24 siblings) even when nothing distinguishes them. Theoretical wire bloat; not a real concern at our target scale.

**B. Bridge-level optimization to Pattern A (`variantMatrix`) when uniform.**
- Domain unchanged. Bridge detects "uniform" variants (full Cartesian product, no own SKU / price_override / media) and projects as `variantMatrix` instead of sibling Resources.
- Rejected for v1. Two projection paths add complexity that the wire-bloat saving doesn't yet justify. **May be added later** as a pure Bridge optimization if real catalogs prove the concern.

**C. Author-declared projection hint on the Product.**
- Add a `variant_projection: 'siblings' | 'matrix' | 'auto'` flag.
- Rejected. The flag exists *only* for protocol projection — borderline protocol leakage per CLAUDE.md §3.4 (the domain test: "can you describe this entity without referencing Beckn?" → here, no). Avoid putting Beckn-shaped knobs on domain entities.

### What changes the Variant-vs-Standalone decision rule

ADR-0005's rule: "if any variant needs its own SKU, use Flat; otherwise use Matrix."

With per-variant SKU now optional in Variant mode, that rule no longer applies. The new rule must justify Standalone (formerly Flat) on different grounds — see Decision.

## Decision

### ProductVariant gains optional `sku`

- `ProductVariant.sku` — optional, nullable, string. Default `null`.
- **Uniqueness:** if set, unique within the store across the combined namespace of `Product.sku` and `ProductVariant.sku`. Case-insensitive.
- **Resolution at projection / quote time:** the variant's effective SKU is `variant.sku ?? parent.sku ?? null`. Both the wire (Bridge) and Order line snapshots use this resolved value.
- **Authoring:** `AddProductVariant(product_id, attribute_values, sku?)` and `UpdateProductVariant(variant_id, ..., sku?)` accept the optional value. Setting to `null` clears it.

### Pattern B is the sole Variant-mode projection

Every Variant-mode Product projects to Beckn as:

- One `PLAIN` parent Resource carrying name, description, parent SKU (if any), categories, default media, base price.
- One `VARIANT` sibling Resource per `ProductVariant`, each carrying:
  - Own Resource ID (derived from `ProductVariant.id`)
  - Own SKU if `variant.sku` is set; else inherits parent's; else empty
  - Own price (`price_override` if set; else parent's `base_price`)
  - Own media (variant's if set; else parent's)
  - Own availability signal (derived from the variant's `StockLevel`)
  - `variantGroup` + `isDefaultVariant` per Beckn Pattern B requirements (Bridge-managed at projection time)

**Pattern A (`variantMatrix`) is explicitly out of scope in v1.** If wire bloat from high-cardinality variant catalogs becomes a real operational concern, add Pattern A as a Bridge-only optimization in a future ADR — no domain change required.

### Revised decision rule for Variant vs. Standalone mode

With per-variant SKU now available in Variant mode, the choice between Variant and Standalone is no longer about SKU at all. It's about **whether the items belong to one family or are independent peers**:

| Use **Variant** mode when… | Use **Standalone** mode (one or more) when… |
|---|---|
| Items share name, description, and category | Items have distinct names / descriptions / categories |
| Items differ along well-defined axes (size, color, capacity) | Items don't share a clean axis structure |
| You want the family to render as a picker UI | You want each item to stand alone |
| You want shared lifecycle (archive the family together) | You want independent lifecycle per item |

Per-variant SKU is no longer a tie-breaker. A t-shirt with three sizes that need per-size barcodes is Variant mode with three SKUs on three Variants. A laptop family where each tier has its own model number, description, and target audience is multiple Standalone Products grouped via an additional Catalog.

### Lifecycle and event payload changes

- `ProductVariantAdded`, `ProductVariantUpdated` payloads gain an optional `sku` field. Backwards-compatible (additive, no `event_version` bump per [ADR-0011](0011-domain-events.md)).
- No other event semantics change. The Catalog → Inventory subscription still creates one `StockLevel` per `ProductVariant`; the Variant's own SKU has no Inventory impact.

## Consequences

### What this commits the system to

- Schema: `ProductVariant` gains a nullable `sku` column with a uniqueness constraint in the single per-store SKU namespace.
- Use cases: `AddProductVariant` and `UpdateProductVariant` accept and validate the optional SKU.
- Domain documents updated: [`design/catalog.md`](../design/catalog.md), [CLAUDE.md §5.10](../CLAUDE.md), [`handoff/04-bounded-contexts/4.3-catalog.md`](../handoff/04-bounded-contexts/4.3-catalog.md), [`design/events.md`](../design/events.md).
- ADR-0005's "variants share parent's SKU" invariant is removed; the partial-supersession marker on ADR-0005 will be extended.
- The Bridge's Variant-mode projection (Pattern B) continues to be the sole path; the mapping registry uses `variant.sku ?? parent.sku ?? null` when populating wire SKU.

### What this defers

- **Pattern A `variantMatrix` projection** for uniform variant sets. Pure Bridge optimization; can be added later without domain change if wire bloat becomes a real concern.
- **Per-variant overrides beyond SKU / price / media** (e.g., per-variant description, per-variant categories) — not requested; would require revisiting the "variants inherit name/description/categories from parent" rule. Out of scope here.
- **Migration tooling for Products that should switch from Variant to multiple Standalones** (or vice versa) — operational; supported today by archive-and-recreate.

### What this makes harder

- **SKU resolution at read time.** Consumers (admin UI, Order's quote builder, the Bridge) must use the `variant.sku ?? parent.sku ?? null` precedence consistently. A single helper in the Catalog Application Layer (`ResolveVariantSku`) should encapsulate this.
- **Uniqueness checks** now span two tables in the single per-store SKU namespace. Either a unified index on a logical view or careful uniqueness validation at the Application Layer.
- **Admin UX.** Authoring a Variant now offers an optional SKU field; UX must communicate "leave blank to inherit the parent's SKU."

### Open follow-ups

- **`ResolveVariantSku` helper** — a small Application-Layer use case used by Order (for line snapshots), the Bridge (for wire projection), and any read-side rendering. Pure utility; not a separate ADR.
- **Index strategy for unified SKU uniqueness** — operational/storage decision; out of scope here.

## References

- [ADR-0005](0005-catalog-and-product-modeling.md) — original Matrix/Flat decision; SKU-sharing rule partially superseded.
- [ADR-0023](0023-product-composition-and-choice-modeling.md) — Mode rename + Pattern B projection for Variant mode (which this ADR now confirms as exclusive).
- [ADR-0006](0006-inventory-model.md), [ADR-0024](0024-inventory-for-composite-and-configurable-products.md) — `StockLevel` per Variant survives unchanged.
- [`design/catalog.md`](../design/catalog.md) — to be updated.
- CLAUDE.md §3 (data modeling philosophy), §5.10 (Catalog and Product Model — to be updated), §3.4 (Beckn-compatibility without coupling).
- `ion-specs/my_ref/BPP_RESOURCE_VARIANT_DESIGN.md` — Pattern A vs. Pattern B reference.
- `ion-specs/my_ref/BPP_RESOURCE_STRUCTURE_CHEATSHEET.md` — variant-projection decision criteria.
