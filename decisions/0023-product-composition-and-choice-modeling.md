# ADR-0023: Product composition and choice modeling

- **Status**: Accepted
- **Date**: 2026-06-01
- **Builds on**: [ADR-0005](0005-catalog-and-product-modeling.md), [ADR-0006](0006-inventory-model.md)
- **Partially supersedes**: [ADR-0005](0005-catalog-and-product-modeling.md) — the `Matrix | Flat` mode discriminator is replaced by a four-mode discriminator; the "bundles / kits / composite products — not in v1" deferral is reversed. Everything else in ADR-0005 (cross-store independence, identifier strategy, category taxonomy, lifecycle states, media handling) survives unchanged.
- **Sub-design**: [design/catalog.md](../design/catalog.md) (to be updated)

## Context

ADR-0005 modelled a Product as either **Matrix** (variant-defining attributes generating Variant entities that share the parent SKU) or **Flat** (each SKU as an independent Product, grouped via additional Catalogs). This covered single SKUs and variant pickers — the bread-and-butter of small retail. It explicitly deferred "bundles / kits / composite products" as out of v1 scope.

Two things have surfaced since:

1. **Indonesian small-retailer reality.** *Paket murah*, combo meals, family packs, and modifier menus (ayam geprek + level pedas, nasi goreng + tambah telur, made-to-order custom cakes) are pervasive. A BPP that cannot model these cannot serve a meaningful slice of the addressable market.

2. **Beckn / ION expressivity.** The wire protocol distinguishes five `resourceStructure` values: `PLAIN`, `VARIANT`, `WITH_EXTRAS` (item + optional modifiers), `COMPOSED` (build-your-own with required choices), and `BUNDLE` (composite of multiple SKUs). Today our domain projects cleanly to PLAIN and VARIANT only. The remaining three have no domain home, so the Bridge has nothing to project from.

The architectural constraint from CLAUDE.md §3.1 and §4.1 stands: the domain must use domain language, not Beckn vocabulary. The fact that Beckn names five structures does not mean the domain needs five matching types — it means the domain needs **concepts that can express the underlying business reality**, with the Bridge handling projection mechanically.

The decision below introduces two general-purpose concepts (composition and choice) that compose to express all five wire shapes without naming any of them.

## Options considered

### How to discriminate Product kinds

**A. Per-feature flags on Product (no discriminator).**
- Each Product carries `has_variants: bool`, `has_components: bool`, `has_choices: bool`. Any combination is structurally legal; runtime invariants police the meaningful subsets.
- **Pros.** Maximally flexible; nothing to retrofit if a new combination becomes valid later.
- **Cons.** Loses the "this is a Bundle / this is a Configurator" first-class semantics. Every consumer (UI, Bridge, Inventory) has to derive the kind from feature presence — duplicated logic, drift risk. Invalid combinations are not caught at the entity level, only at use-case boundaries.

**B. Four-mode discriminator** — chosen.
- `Product.mode ∈ { Standalone, Variant, Configurable, Composite }`. Set at creation; immutable (consistent with the existing Matrix/Flat immutability rule).
- Each mode admits a specific set of sub-entities (variants, components, choice groups) and forbids the others. Mutual exclusion is an entity-level invariant, not a runtime check sprinkled across use cases.
- **Pros.** First-class business semantics; one place to look up "what is this Product." Cleanly extends the existing immutable-mode discipline from ADR-0005. The Bridge's projection logic is a small `switch` on `mode`.
- **Cons.** Renames the existing modes (Flat → Standalone, Matrix → Variant); a mechanical change but touches multiple documents.

**C. Per-mode subclass entities (inheritance).**
- `StandaloneProduct`, `VariantProduct`, `ConfigurableProduct`, `CompositeProduct` as separate aggregate types.
- **Pros.** Each mode's invariants are statically enforced by its type; no "if mode is Composite, components must exist" runtime checks.
- **Cons.** Polymorphic identity (`Product.id` resolves to one of four types) leaks into every consumer (Inventory references, Order line items, Bridge projections). Storage becomes either one-table-per-type (joins explode) or single-table-inheritance (loses the static typing benefit). Adds significant friction relative to the gain.

### Composition shape

**A. Recursive components (Composite of Composite of …).**
- A component can itself be a Composite Product.
- **Pros.** Mathematically clean.
- **Cons.** Stock propagation becomes a tree walk; UI rendering of nested bundles is bewildering; matches no real retail scenario in our target market. Beckn's own BPP guidance is explicit: "Components must be PLAIN or VARIANT." Recursive composition is an anti-pattern.

**B. Flat composition — chosen.**
- A Composite's components must be Standalone or Variant Products only. Depth is exactly one level.
- **Pros.** Tractable stock/availability propagation (one-level reduction); matches Beckn's BUNDLE rule; matches real retail.
- **Cons.** Some exotic configurations need workarounds (pre-configure the would-be-nested Composite as a Standalone Product, then include it). Acceptable.

### Choice options and stock

**A. Choices are opaque labels only (no Product reference).**
- A choice ("Extra Cheese") is just a `(label, price_delta)` pair on its group.
- **Pros.** Simplest possible model.
- **Cons.** When the choice depletes inventory (extra cheese costs Rp 8,000 *and* uses cheese stock), the domain has no place to express the inventory linkage. Either the store overlooks it (overselling cheese) or has to model cheese as a Component, which doesn't fit Configurable's semantics.

**B. Choices may optionally reference another Product** — chosen.
- A `Choice` carries `option_product_ref` (nullable) — an opaque reference to another Standalone or Variant inventory anchor. When set, the choice depletes that anchor's stock at order confirm.
- **Pros.** Cleanly handles stocked modifiers without forcing every choice to become a Product. Optional pricing remains on the Choice; identity and stock live on the referenced Product.
- **Cons.** Two places to look (price_delta on Choice, stock semantics via Product ref) when both apply. Acceptable; the alternative is worse.

### Mode combinations

**A. Allow any combination (variant bundles, configurable variants, composite configurables).**
- Beckn supports combinations via Pattern B for variants and Pattern 2 for WITH_EXTRAS siblings.
- **Pros.** Matches some real cases (Family Combo S/M/L; Pizza in 3 sizes with per-size extras).
- **Cons.** Combinatorial complexity in invariants, projection, inventory semantics. Each combination introduces new edge cases (what does "variant of a configurable" mean for the choice groups — shared or per-variant?).

**B. Mutually exclusive modes in v1** — chosen.
- A Product is exactly one of Standalone, Variant, Configurable, Composite. No Variant has its own components or choices; no Composite has variants; etc.
- **Pros.** Drastically narrows the invariant surface; each mode has a clean, single projection. The user explicitly chose this trade-off (alignment Q2).
- **Cons.** "Family Combo S/M/L" can't be expressed natively (workaround: three separate Composite Products, optionally grouped via an additional Catalog per ADR-0004). "Pizza in 3 sizes with extras menu per size" must collapse into one Configurable Product with size + extras as choice groups. Acceptable for v1.

### Naming

**A. Adopt Beckn vocabulary in the domain (`PlainProduct`, `BundleProduct`, …).**
- Rejected — violates CLAUDE.md §3.1 / §3.4 / §8.1. The domain must not borrow Beckn names.

**B. Domain-native vocabulary** — chosen.
- `Standalone` (no choices, no parts) is a generic retail term.
- `Variant` describes what the Product *has*, not what Beckn calls it.
- `Configurable` is standard product-config terminology (used by Square, Toast, every modern POS).
- `Composite` is a generic CS/retail term (also "kit," "bundle," "combo" — Composite is the broadest).
- `Component`, `ChoiceGroup`, `Choice` are all problem-domain terms.

## Decision

### Product mode discriminator

Every Product carries a `mode: Standalone | Variant | Configurable | Composite`, set at creation and **immutable**. Mode determines which sub-entities the Product admits.

**Renames** (mechanical, applied throughout the design corpus):

| ADR-0005 term | ADR-0023 term |
|---|---|
| Flat mode | **Standalone** mode |
| Matrix mode | **Variant** mode |

### Mode → admitted features

| Mode | `ProductVariant` | `ProductComponent` | `ChoiceGroup` | Notes |
|---|---|---|---|---|
| **Standalone** | ❌ | ❌ | ❌ | The simplest Product. One SKU, no parts, no choices. |
| **Variant** | ✅ (≥1 attribute defined; ≥0 variants in Draft, ≥1 for Active) | ❌ | ❌ | Existing Matrix-mode behaviour. |
| **Configurable** | ❌ | ❌ | ✅ (≥0 in Draft; ≥1 for Active) | At least one `ChoiceGroup` must exist when Active. May contain any selection types. |
| **Composite** | ❌ | ✅ (≥1 component when Active) | ❌ | Components reference Standalone or Variant anchors; flat hierarchy. |

**Mutual exclusion is an entity invariant.** A Product cannot transition between modes; a Product cannot acquire sub-entities forbidden by its mode.

### New entities

#### `ProductComponent` *(Composite mode only)*

A part of a Composite Product.

- `id` — internal opaque identifier.
- `parent_product_id` — references the Composite Product. Immutable.
- `component_anchor_ref` — opaque reference to a **Standalone Product** or a **`ProductVariant` of a Variant-mode Product**. Same opaque-ref type used by `Inventory.StockLevel.inventory_item_ref` (see ADR-0006 §"Inventory anchors").
- `quantity` — integer ≥ 1; how many of the referenced anchor are consumed per Composite unit.
- `display_order` — position within the parent's ordered component list.

**Invariants:**

- A Composite cannot reference another Composite (depth = 1).
- A Composite cannot reference a Configurable (a choice-resolved item is not a fixed component).
- A Composite must have at least one `ProductComponent` to be `Active`.
- The same anchor may appear at most once in a Composite's component list (use `quantity` to express multiples).

#### `ChoiceGroup` *(Configurable mode only)*

A decision point on a Configurable Product.

- `id` — internal opaque identifier.
- `product_id` — references the Configurable Product. Immutable.
- `name` — `LocalizedText`, required.
- `selection_type` — one of:
  - `RequiredSingle` — exactly one option must be chosen.
  - `OptionalSingle` — at most one option may be chosen.
  - `OptionalMultiple` — between `min_selections` and `max_selections` options may be chosen.
- `min_selections` — integer ≥ 0; meaningful only for `OptionalMultiple`; default `0`.
- `max_selections` — integer ≥ 1; meaningful only for `OptionalMultiple`; default unbounded.
- `display_order` — position within the parent's ordered ChoiceGroup list.

#### `Choice` *(within a ChoiceGroup)*

A single selectable option.

- `id` — internal opaque identifier.
- `choice_group_id` — references the parent `ChoiceGroup`. Immutable.
- `label` — `LocalizedText`, required.
- `price_delta` — optional `Money` (in the store's currency, tax-excluded); default zero; may be negative.
- `option_product_ref` — optional opaque reference to a **Standalone Product** or a **`ProductVariant` of a Variant-mode Product**. When set, selecting this Choice depletes the referenced anchor's stock at order confirm (semantics owned by ADR-0024).
- `display_order` — position within the parent ChoiceGroup's ordered Choice list.

**Invariants:**

- `option_product_ref` may not reference a Composite Product (avoid recursive resolution).
- `option_product_ref` may not reference a Configurable Product (avoid "choose-then-choose").
- For `OptionalMultiple`: `0 ≤ min_selections ≤ max_selections`.
- Every `ChoiceGroup` must have at least one `Choice` when its parent Product is `Active`.

### Lifecycle implications

The new sub-entities follow their parent Product's lifecycle — they have no independent lifecycle. Adding, removing, or updating components and choice groups is permitted while the parent is `Draft` or `Active`; archiving the parent freezes the structure.

The existing Product lifecycle (Draft → Active → Archived; no deletion; no Active → Draft) is unchanged.

**Invariant additions for Active state:**

- Configurable: ≥1 `ChoiceGroup`, each with ≥1 `Choice`.
- Composite: ≥1 `ProductComponent`.
- Variant: ≥1 `ProductVariant` (this tightens ADR-0005's wording; Variant-mode Products in Draft may have no variants yet).

### Pricing composition (overview only)

ADR-0007's `base_price` + `tax_rate` model applies to every mode. Composition rules:

- **Standalone / Variant:** as today.
- **Configurable:** `base_price` is the configurator's starting price; each chosen Choice's `price_delta` is added (or subtracted) at quote time.
- **Composite:** `base_price` is the bundle's own price (typically discounted vs. component sum). Component prices are informational only; they are not summed.

Detailed quote construction across modes is owned by the Order context (ADR-0017 + future refinements).

### Domain events

New events emitted by the Catalog context:

- `ProductComponentAdded`, `ProductComponentRemoved`, `ProductComponentQuantityChanged`
- `ChoiceGroupAdded`, `ChoiceGroupUpdated`, `ChoiceGroupRemoved`
- `ChoiceAdded`, `ChoiceUpdated`, `ChoiceRemoved`

All carry the parent `product_id` (and `choice_group_id` where applicable), follow the envelope/versioning rules from ADR-0011, and join the registry in [`design/events.md`](../design/events.md).

Existing `ProductPublished`, `ProductArchived`, `ProductRestored`, `ProductUpdated` events apply uniformly across all four modes.

### Use cases (new)

Application-Layer use cases added to the Catalog context (full list in the design doc update):

- `AddProductComponent(product_id, anchor_ref, quantity)` *(Composite only)*
- `RemoveProductComponent(product_id, component_id)` *(Composite only)*
- `UpdateProductComponentQuantity(component_id, quantity)` *(Composite only)*
- `AddChoiceGroup(product_id, name, selection_type, …)` *(Configurable only)*
- `UpdateChoiceGroup(choice_group_id, …)` *(Configurable only)*
- `RemoveChoiceGroup(choice_group_id)` *(Configurable only)*
- `AddChoice(choice_group_id, label, price_delta, option_product_ref)` *(Configurable only)*
- `UpdateChoice(choice_id, …)` *(Configurable only)*
- `RemoveChoice(choice_id)` *(Configurable only)*

### Beckn projection (Bridge concern, summarised)

The Bridge's mapping registry implements this `switch` based on `Product.mode`. The domain remains entirely unaware.

| Domain | Bridge projection (Beckn v2 / ION) |
|---|---|
| `mode: Standalone` | `resourceStructure: PLAIN` |
| `mode: Variant` | `PLAIN` parent + `VARIANT` sibling Resources (one per `ProductVariant`) |
| `mode: Configurable`, all ChoiceGroups Optional | `resourceStructure: WITH_EXTRAS` with `customisationGroups[]` (soft parent reference per ION x-ion-condition) |
| `mode: Configurable`, any ChoiceGroup `RequiredSingle` | `resourceStructure: COMPOSED`, standalone (no parent), with `customisationGroups[]` |
| `mode: Composite` | `resourceStructure: BUNDLE` + `bundleComposition[]`; paired with a BUNDLE-type Offer |

`ChoiceGroup.selection_type` projects to `customisationGroups[].selectionType`: `RequiredSingle → SINGLE_REQUIRED`, `OptionalSingle → SINGLE_OPTIONAL`, `OptionalMultiple → MULTIPLE_OPTIONAL`. `Choice.option_product_ref` projects to `options[].resourceId`.

## Consequences

### What this commits the system to

- The Catalog context grows by three entity types: `ProductComponent`, `ChoiceGroup`, `Choice`.
- The Product mode discriminator becomes a four-way enum, replacing the prior binary.
- The renames `Flat → Standalone` and `Matrix → Variant` cascade through `design/catalog.md`, CLAUDE.md §5.10, the events registry, and the handoff package.
- Mutual exclusion is enforced at the entity boundary; use cases for component/choice operations reject calls on the wrong mode.
- Inventory's anchors expand to cover Composite Products (handled by ADR-0024, in progress).
- The Bridge's mapping registry gains projection logic for three new Beckn `resourceStructure` values. No change to the Bridge's external interface; this is internal mapping work.
- The Order context's quote construction extends to compose choice price deltas (Configurable) and respect bundle pricing (Composite); detailed semantics are deferred to a follow-up ADR if they require more than the existing primitives.

### What this defers

- **Variant bundles** (Family Combo S/M/L as native siblings). v1 workaround: three separate Composite Products, optionally grouped via an additional Catalog. A future ADR may relax the mutual-exclusion rule.
- **Configurable variants** (per-size extras menus). v1 workaround: one Configurable Product with size and extras as parallel ChoiceGroups.
- **Nested composites** (Composite containing Composite). v1 workaround: pre-configure the inner composite as a Standalone Product.
- **`FIXED` choice group semantics** (Beckn's spec-grey fourth `selectionType`). Out of scope; use `longDesc` text for non-selectable disclosure.
- **Quote construction details for Configurable and Composite** at the Order context level — captured at high level here; full semantics live in the Order context's refinements.

### What this makes harder

- **Schema and storage shape.** Three new entity tables; existing Catalog migrations must accept the renamed mode values without losing referential history.
- **Admin UX.** The store admin now picks among four modes at Product creation, with distinct authoring flows for each. UX complexity scales accordingly.
- **Mode discovery.** Owners of existing Flat or Matrix Products see renamed labels (mechanical UI change). No data loss; existing Products keep their (now-renamed) mode.
- **Mode immutability.** Reinforced: a Product that should have been a Composite but was created Standalone cannot be converted in place; archive and recreate.

### Open follow-ups

- **ADR-0024 — Inventory adaptation** (drafted next): `CompositeStatus`, availability formula per mode, Reservation conversion for Composite and option-anchored Choices.
- **Design doc update** — `design/catalog.md` to reflect the four modes and new entities; events registry update.
- **Charter cascade** — CLAUDE.md §5.10 paragraph replacement.

## References

- [ADR-0005](0005-catalog-and-product-modeling.md) — base Catalog and Product model; partially superseded.
- [ADR-0006](0006-inventory-model.md) — inventory anchors (expanded by ADR-0024).
- [ADR-0007](0007-pricing-tax-and-vouchers.md) — `Money` and `base_price` / `tax_rate` model (applies unchanged per mode).
- [ADR-0008](0008-localization-and-localizedtext.md) — `LocalizedText` semantics (applies to new `name` and `label` fields).
- [ADR-0011](0011-domain-events.md) — event envelope and versioning (applies to new events).
- [ADR-0017](0017-order-and-fulfillment.md) — Order context (quote composition extends per mode).
- CLAUDE.md §3 (data modeling philosophy), §4 (Beckn integration), §5.10 (Catalog and Product model — to be updated).
- `ion-specs` — `schema/extensions/trade/resource/v1/attributes.yaml`; `my_ref/BPP_RESOURCE_STRUCTURE_CHEATSHEET.md`, `BPP_RESOURCE_BUNDLE_DESIGN.md`, `BPP_RESOURCE_WITH_EXTRAS_DESIGN.md`, `BPP_RESOURCE_COMPOSED_DESIGN.md`, `BPP_RESOURCE_VARIANT_DESIGN.md`.
