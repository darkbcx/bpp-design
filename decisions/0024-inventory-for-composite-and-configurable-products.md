# ADR-0024: Inventory for Composite and Configurable Products

- **Status**: Accepted
- **Date**: 2026-06-01
- **Builds on**: [ADR-0006](0006-inventory-model.md), [ADR-0023](0023-product-composition-and-choice-modeling.md)
- **Partially supersedes**: [ADR-0006](0006-inventory-model.md) — the set of inventory anchors expands from `{ Standalone Product, ProductVariant }` to additionally cover `{ Composite Product, Configurable Product }` via a new lightweight entity; `Reservation` conversion semantics extend to fan out across components and option-anchored choices. All other ADR-0006 decisions (single store-scoped inventory, integer stock count, timed reservations, stock-movement event log) survive unchanged.

## Context

[ADR-0023](0023-product-composition-and-choice-modeling.md) introduces two new Product modes:

- **Composite** — a Product whose availability and stock effects derive from its `ProductComponent` list. Bundle, combo meal, starter kit.
- **Configurable** — a Product whose effective offer is shaped by `ChoiceGroup` selections at order time. Build-your-own pizza, custom cake. Choices may optionally carry an `option_product_ref` that anchors a stock-consuming option (extra cheese drawn from cheese stock).

[ADR-0006](0006-inventory-model.md) anchors `StockLevel` on either a Standalone Product or a `ProductVariant`. Neither anchor fits the new modes naturally:

- A **Composite Product** has no stock count of its own — its sellability follows from its components' availability ("weakest wins" per the BPP composition guidance). Putting an integer `stock_count` on a Composite would invite owners to set it independently of the underlying components, producing oversold orders or misleading availability signals.
- A **Configurable Product** is a configurator, not a finished good — there is no "count of pizzas" before the customer picks a size. What's stocked are the underlying ingredients (when modelled via `option_product_ref`); the configurator's own state is whether the owner has enabled it.

Both modes still need the **owner's purchasable toggle** (the equivalent of "I have it but I'm not selling it now") — pausing a bundle from sale, pausing the configurator for a service window. That capability must exist without forcing an artificial `stock_count`.

The decision below extends ADR-0006's primitives to cover both modes while preserving the existing model for Standalone and Variant Products entirely unchanged.

## Options considered

### Composite stock representation

**A. Composite gets its own `StockLevel` with independent `stock_count`.**
- Owner sets a count on the Composite directly (e.g., "I made 50 family meals").
- **Pros.** Simple uniformity with Standalone/Variant.
- **Cons.** Two sources of truth for Composite availability — own count vs. components' counts. Owners can set Composite stock = 100 while a component sits at 0; the system must either reconcile or oversell. Beckn's BUNDLE wire guidance is "availability propagates from components"; forcing an own count fights that.

**B. Composite has no inventory entity at all.**
- Availability derives purely from components; nothing represents the bundle in Inventory.
- **Pros.** Single source of truth.
- **Cons.** No owner-controlled toggle — the only way to pause a bundle is to alter components (which affects standalone sales too) or archive the Composite (heavy-handed; loses the bundle's identity).

**C. Composite gets a lightweight `OwnerPurchasability` entity (purchasable flag only, no count) — chosen.**
- Stores the owner's enable/disable toggle without introducing a phantom count. Effective availability is computed: `OwnerPurchasability.purchasable AND every component's effective availability ≥ component.quantity`.
- **Pros.** Single source of truth for stock (components); first-class owner toggle for the bundle. Aligns with the Beckn BUNDLE rule without imitating it in code. The "purchasable" semantic is identical to the flag on `StockLevel`, just on a different anchor.
- **Cons.** Adds a second anchor type to Inventory. Reservation conversion fans out across components — more work per confirm, but mechanically simple.

### Configurable stock representation

**A. Configurable gets its own `StockLevel`.**
- Same shape as Standalone: count + purchasable.
- **Pros.** Uniform model.
- **Cons.** "Stock count of a configurator" has no natural meaning. Owners would either use it as a daily capacity dial (a different concept) or leave it at a large number, in which case it carries no signal.

**B. Configurable has no inventory entity.**
- The configurator is always sellable unless archived.
- **Pros.** Simplest.
- **Cons.** Removes the owner's pause control entirely. Service windows ("closed for the night") can't be expressed without archiving.

**C. Configurable uses the same `OwnerPurchasability` entity as Composite — chosen.**
- One lightweight entity type shared across both non-stocked modes. Same shape, same semantics, same toggle.
- **Pros.** One concept for "owner toggle without stock count." Effective availability formula differs (Composite gates on components; Configurable gates on Required choice groups completability), but the anchor is identical.
- **Cons.** None substantive.

### Configurable availability — what counts as "available"

**A. Available iff at least one Choice in every Required group is available.**
- Selected — matches the Configurable semantics ("you can complete the order"). Optional groups don't gate availability; they're optional.
- **Cons.** A Required group with 5 choices and 1 unavailable still reads as available. Correct, but the BAP/admin UX should surface per-choice availability separately.

**B. Available iff every Choice in every Required group is available.**
- Rejected. Defeats the Required group's purpose (the buyer can still pick from the available subset).

### Reservation semantics for Composite

**A. One Reservation against the Composite itself.**
- Rejected — Composite has no StockLevel; no anchor to hold against.

**B. Per-component Reservations, fanned out at order init — chosen.**
- At init: create one Active Reservation per component, with `quantity = order_qty × component.quantity` against each component's StockLevel. All reservations carry the same `held_by_ref` (the order).
- At confirm: convert all atomically (all-or-nothing).
- At cancel/expiry: release all atomically.
- **Pros.** Reuses the existing Reservation primitive without modification. Per-component visibility (an owner can see "10 buns reserved by pending orders, 4 by carts in init").
- **Cons.** N reservation rows per Composite confirm; storage cost grows with component count. Acceptable for v1 bundle sizes (≤ ~10 components).

### Reservation semantics for Configurable

**A. Reservation only for choices with `option_product_ref`.**
- Selected. At init: for each chosen Choice that has `option_product_ref`, create a Reservation against the referenced anchor's StockLevel (quantity 1 per chosen option, times `order_qty`). Choices without `option_product_ref` consume no stock.
- This is the only sensible interpretation when the configurator itself has no StockLevel.

### Anchor naming

**A. Extend `StockLevel` to allow null `stock_count`.**
- Rejected. Conflates two distinct concepts ("trackable stock" vs. "owner toggle only") into one schema.

**B. Separate `OwnerPurchasability` entity — chosen.**
- Distinct entity, distinct shape, same purchasable-flag semantics. Two anchor types in Inventory, polymorphic at the use-case boundary, statically distinct in storage.

## Decision

### Inventory anchors (expanded)

The Inventory context now recognises **two anchor types**, both store-scoped, each tied to exactly one Catalog entity:

| Anchor entity | Anchored to | Carries |
|---|---|---|
| `StockLevel` (existing) | Standalone Product, or `ProductVariant` of a Variant-mode Product | `stock_count`, `purchasable`, `status`, `updated_at` |
| `OwnerPurchasability` (new) | Composite Product, or Configurable Product | `purchasable`, `status`, `updated_at` |

Exactly one anchor exists per anchored Catalog entity. No anchor exists for the parent Product of a Variant-mode Product (the variants carry the anchors).

### `OwnerPurchasability` entity

- `id` — internal opaque identifier.
- `store_id` — owning Store; cross-context reference to Tenancy.
- `product_ref` — opaque reference to a Composite Product or a Configurable Product.
- `purchasable` — boolean; owner-controlled; default `true`.
- `status` — `Active` | `Inactive`. Set to `Inactive` when the underlying Product is archived; record retained for history (consistent with `StockLevel.status` from ADR-0006).
- `updated_at` — last change timestamp.

### Effective availability formulas (per mode)

The Storefront, Admin UI, and Bridge compute effective availability via synchronous query against Inventory. Inventory exposes a single use case `GetEffectiveAvailability(product_or_variant_ref)` that internally dispatches by mode:

**Standalone:**
```
let s = StockLevel(product)
effective_availability = s.purchasable AND (s.stock_count − active_reservations) > 0
available_quantity     = max(0, s.stock_count − active_reservations) if s.purchasable else 0
```

**Variant (per ProductVariant):**
```
identical to Standalone, with s = StockLevel(variant)
```

**Composite:**
```
let op = OwnerPurchasability(composite_product)
for each component c in composite_product.components:
    component_avail_qty(c) = available_quantity of c.component_anchor_ref
    component_units(c)     = floor(component_avail_qty(c) / c.quantity)
effective_availability = op.purchasable AND min(component_units(c) for all c) > 0
available_quantity     = min(component_units(c) for all c) if op.purchasable else 0
```

**Configurable:**
```
let op = OwnerPurchasability(configurable_product)
for each ChoiceGroup g with selection_type = RequiredSingle in product.choice_groups:
    group_has_available_choice(g) = ∃ Choice c in g where choice_available(c)
        where choice_available(c) =
            (c.option_product_ref is null)
            OR (effective_availability of c.option_product_ref is true)
effective_availability = op.purchasable
                          AND every RequiredSingle group has group_has_available_choice = true
available_quantity     = unbounded (configurator is not stock-quantified) if effective_availability else 0
```

`OptionalSingle` and `OptionalMultiple` groups never gate the configurator's availability — they're optional by definition.

### Reservation semantics (mode-specific)

`Reservation` (ADR-0006) remains the primitive — a hold against a `StockLevel`. What changes is how many Reservations are created per order line, depending on the line's Product mode.

**Standalone / Variant** — unchanged from ADR-0006. One Reservation per order line.

**Composite** — one Reservation per `ProductComponent`:
- At order **init**: for each component `c`, create one Active Reservation against `c.component_anchor_ref`'s `StockLevel` with `quantity = order_line.quantity × c.quantity`. All reservations carry the same `held_by_ref`.
- At order **confirm**: convert all the line's reservations atomically. If any conversion fails (a component went out of stock between init and confirm), the confirm fails as a whole; already-converted reservations are rolled back.
- At order **cancel** or **timeout**: release all reservations atomically.

**Configurable** — one Reservation per chosen Choice with `option_product_ref`:
- At order **init**: for each chosen Choice `ch` where `ch.option_product_ref` is set, create one Active Reservation against the referenced anchor's `StockLevel` with `quantity = order_line.quantity × 1`. Chosen Choices without `option_product_ref` create no reservation. All reservations carry the same `held_by_ref`.
- At order **confirm**: convert all the line's reservations atomically.
- At order **cancel** or **timeout**: release all atomically.

In both cases, the Order context's compensation rules (per ADR-0017 §"Orchestration") apply uniformly — fan-out doesn't change the all-or-nothing semantics of confirm.

### Stock-movement event log (extended)

Existing event subtypes (`Received`, `Sold`, `Returned`, `Corrected`, `Reserved`, `Released`, `PurchasableToggled`) remain — they fire against the `StockLevel` anchor wherever a stock change occurs. Composite confirms fan out to per-component `Sold` events; Configurable confirms fan out to per-option `Sold` events. The Composite/Configurable Product itself emits no stock-movement events because no stock changes on it.

A new event type fires on `OwnerPurchasability` changes:

| Type | When | Stock delta |
|---|---|---|
| `OwnerPurchasabilityToggled` | Owner flipped the `purchasable` flag on a Composite or Configurable Product | n/a (no stock) |

The event is **not** a `StockMovement` subtype — it carries no delta and has no `StockLevel` reference. It's a separate domain event with its own envelope, owned by Inventory. Audit ingests it the same way it ingests StockMovement events.

### Communication

- **Storefront, Admin UI, Bridge** → query Inventory **synchronously** via `GetEffectiveAvailability`. The use case dispatches by mode internally; callers don't need to know which formula applies.
- **Order context** → calls Inventory's `CreateReservation`, `ConvertReservation`, `ReleaseReservation` use cases. The use cases accept `(anchor_ref, quantity, held_by_ref)`; for Composite and Configurable order lines, the Order context iterates and calls per component/per choice.
- **Catalog → Inventory** subscription expands:
  - `ProductCreated` with `mode ∈ { Standalone, Variant }` → unchanged (StockLevel creation as today).
  - `ProductCreated` with `mode ∈ { Composite, Configurable }` → Inventory creates an `OwnerPurchasability` with `purchasable=true`.
  - `ProductVariantAdded` → unchanged (StockLevel per Variant).
  - `ProductArchived` → Inventory marks the corresponding anchor (StockLevel or OwnerPurchasability) `Inactive`.
  - `ProductRestored` → restore the anchor to `Active`.
  - `ProductComponentAdded`, `ProductComponentRemoved`, `ProductComponentQuantityChanged`, `ChoiceGroupAdded`, `ChoiceGroupRemoved`, `ChoiceAdded`, `ChoiceUpdated`, `ChoiceRemoved` → **no Inventory action**. These affect *computation* of effective availability for Composite/Configurable anchors, but the anchor itself is unchanged. Subscribers that cache derived availability must invalidate on these events.

### Use cases (new and updated)

- `ToggleOwnerPurchasability(product_ref, purchasable: bool)` — new; emits `OwnerPurchasabilityToggled`.
- `GetEffectiveAvailability(anchor_ref)` — extended to dispatch by mode.
- `CreateReservation` / `ConvertReservation` / `ReleaseReservation` — unchanged signature; existing primitives.
- Existing `StockLevel` use cases (`SetStockCount`, `RecordReceived`, `RecordCorrection`, `TogglePurchasable`) — unchanged; apply only to StockLevel anchors. Rejected with a typed error if called against an `OwnerPurchasability` anchor.

### Boundary recap

- **Catalog** owns: Product entities, modes, components, choice groups, choices, lifecycle.
- **Inventory** owns: `StockLevel`, `OwnerPurchasability`, `Reservation`, and the stock-movement event log + the new `OwnerPurchasabilityToggled` event.
- Inventory holds opaque references to Catalog entities; never joins or reads Catalog internals (§2.6).

## Consequences

### What this commits the system to

- Inventory now contains two anchor types: `StockLevel` (existing) and `OwnerPurchasability` (new). Same `Reservation` primitive serves both indirectly (Reservations always anchor to StockLevels).
- The effective-availability formula is mode-specific. The `GetEffectiveAvailability` use case is the single entry point that hides the dispatch.
- Composite and Configurable order lines produce **fan-out reservations** at init: one Reservation per component or per option-anchored choice. The Order context's existing all-or-nothing confirm/release semantics apply unchanged.
- The Bridge queries Inventory for effective availability uniformly across modes when projecting `availability.status` into catalogs and `/on_select` quotes. The mapping from `effective_availability` to wire signals (`IN_STOCK` / `LOW_STOCK` / `OUT_OF_STOCK`) remains in the Bridge's mapping registry.
- Catalog's per-component / per-choice mutation events are consumed by **caching subscribers** (if any), not by Inventory directly. Inventory only reacts to Product lifecycle events for anchor creation/lifecycle.
- Audit consumes `OwnerPurchasabilityToggled` like any other domain event.

### What this defers

- **Per-component reservation policies** (e.g., reserve components from a bundle pool separate from the standalone pool). Not in v1; all reservations come from the same StockLevel.
- **Daily capacity limits for Configurable Products** (e.g., "I can make 30 custom cakes today"). Not modelled in v1; if needed, introduce as a separate `DailyCapacity` entity later.
- **Low-stock signal computation** for Composite (which component triggers LOW_STOCK?) — Bridge concern; resolved when the wire signal mapping is detailed.
- **Variant-bundle inventory** (a single bundle with size variants). Out of scope per ADR-0023's mutual-exclusion rule; v1 workaround is three separate Composite Products.
- **Component swap protocols** (replace one component with another mid-life). Treated as: archive Composite, recreate with new components. Same heaviness as mode-change in ADR-0023.

### What this makes harder

- **Reservation table size.** Composite and Configurable orders multiply Reservation rows by `(component count + option-anchored choice count)`. Acceptable for v1 scale; index strategy is operational.
- **Atomic confirm across many components.** The Order context's confirm becomes a multi-Reservation atomic operation. The ADR-0017 orchestration + hand-coded compensation pattern handles this, but the failure surface widens.
- **Owner mental model.** Owners now distinguish "I have N of this Product" (StockLevel) from "I have this Composite/Configurator enabled" (OwnerPurchasability). Admin UX must surface this clearly.
- **Cross-cutting reads.** "Show me everything paused by the owner" now spans two entity types. Read queries that previously hit only `StockLevel` may need to union `OwnerPurchasability`.

### Open follow-ups

- **Design doc update** — extend `design/catalog.md` (per ADR-0023) and add or extend an Inventory design doc covering both anchor types.
- **Charter cascade** — CLAUDE.md §5.11 paragraph extension to acknowledge the two anchor types.
- **Events registry** — add `inventory.owner_purchasability_toggled` to `design/events.md`.

## References

- [ADR-0006](0006-inventory-model.md) — base Inventory model; partially superseded.
- [ADR-0011](0011-domain-events.md) — event envelope (applies to `OwnerPurchasabilityToggled`).
- [ADR-0012](0012-cross-context-consistency.md) — cross-context consistency rules; per-mode reservation fan-out follows the same orchestration discipline.
- [ADR-0017](0017-order-and-fulfillment.md) — Order context; orchestration of multi-Reservation confirms.
- [ADR-0023](0023-product-composition-and-choice-modeling.md) — defines the Catalog entities (Composite, Configurable, ProductComponent, ChoiceGroup, Choice) anchored by this ADR.
- CLAUDE.md §5.10 (Catalog model — updated by ADR-0023), §5.11 (Inventory — extension target), §2.6 (inter-context communication).
- `ion-specs/my_ref/BPP_RESOURCE_BUNDLE_DESIGN.md` — "bundle availability propagates from components (weakest wins)" guidance.
