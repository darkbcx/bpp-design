# ADR-0006: Inventory model

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/08-inventory-boundary.md](../gaps/resolved/08-inventory-boundary.md)
- **Builds on**: [ADR-0005](0005-catalog-and-product-modeling.md) (defines the anchors Inventory tracks)
- **Resolves deferral from**: [ADR-0005](0005-catalog-and-product-modeling.md) (the `availability_flag` placeholder on `ProductVariant` moves to Inventory's `StockLevel.purchasable`)

## Context

[ADR-0005](0005-catalog-and-product-modeling.md) established the Catalog as owning *what the product is* — Products, Variants, attributes, categories, media. The sibling context — Inventory — owns *is it purchasable right now*: stock, reservations, owner-controlled availability flags. Without this model, the storefront can't answer "can the buyer buy this?", the Bridge can't project accurate availability into catalogs published to CDS or `/on_select` quotes, and order placement ([Gap 11](../gaps/11-order-and-fulfillment-phasing.md)) has no place to commit stock changes.

Inventory was deliberately deferred until Catalog was concrete; the Variant entity from ADR-0005 is the natural anchor in Matrix mode, and Product is the anchor in Flat mode.

## Options considered

### Stock model

**A. Integer stock count + owner-controlled `purchasable` flag** — chosen.
- Effective availability is derived: `purchasable AND (stock_count − active_reservations) > 0`.
- Pros: handles both "out of stock" (stock=0, purchasable=true) and "I have it but I'm not selling it" (stock>0, purchasable=false) without conflation. Maps cleanly to Beckn's `quantity.available.count`.
- Cons: two attributes, two UX surfaces.

**B. Boolean availability only.** Rejected — loses per-item quantity which the Beckn protocol exposes and which real retailers care about.

**C. Tracked-vs-untracked discriminator per item.** Rejected — adds a mode flag; integer-everywhere is simpler. Items without real stock tracking can leave `stock_count` high and toggle `purchasable` as needed.

### Reservations

**A. Timed Reservations created at order init; converted to stock decrement at confirm; released on cancel or expiry** — chosen. Standard pattern; matches the Beckn `init → confirm` flow.

**B. No reservations; decrement at confirm.** Rejected — overselling under concurrency.

### Multi-location

**A. Single store-scoped logical inventory** — chosen. Small-retailer aggregator; multi-warehouse is rare and adds significant complexity (allocation rules, location-aware fulfillment).

**B. Multi-location from day one.** Rejected — premature.

### Backorders / preorders

**Not supported in v1** — chosen. Strict stock-checked sales. A per-item `allow_backorder` flag can be added later without touching the rest of the model.

### Movement history

**Typed event stream** — chosen. Required by [Audit (Gap 16)](../gaps/16-soft-delete-and-audit.md); useful for analytics and reconciliation.

### Cross-context comms

**Synchronous queries from Storefront / Bridge to Inventory** — chosen. Consistent with §2.6's two-channel model (synchronous for queries, events for state propagation). A read-side projection can be added later if load demands.

## Decision

### Inventory anchors

An **inventory item** is the unit a `StockLevel` attaches to:
- **Flat Mode Product** → `StockLevel` per Product.
- **Matrix Mode Product** → `StockLevel` per `ProductVariant`.

Exactly one `StockLevel` per inventory item. The Inventory context references the Catalog entity by opaque ID; no joins (per §2.6, §3.3).

### StockLevel entity

- `id` — internal opaque identifier.
- `store_id` — owning Store; cross-context reference to Tenancy.
- `inventory_item_ref` — opaque reference to either a Product (Flat) or a ProductVariant (Matrix).
- `stock_count` — integer ≥ 0; physical units available.
- `purchasable` — boolean; owner-controlled; default `true`.
- `status` — `Active` | `Inactive` (set to `Inactive` when the underlying Catalog item is archived/removed; retained for history).
- `updated_at` — last change timestamp.

### Effective availability

For any inventory item, computed at read time:

```
active_reservations = sum of quantity over Reservation records in state Active
                      for this StockLevel
available_for_purchase = purchasable AND (stock_count − active_reservations) > 0
available_quantity     = max(0, stock_count − active_reservations) if purchasable
                         else 0
```

The Storefront, Admin UI, and Bridge compute these via synchronous query against Inventory.

### Reservation entity

A `Reservation` is a temporary hold against a `StockLevel`:

- `id` — internal opaque identifier.
- `stock_level_id` — reference.
- `quantity` — integer ≥ 1.
- `held_by_ref` — opaque reference to whatever caused the hold (Order context's transaction or cart identifier).
- `expires_at` — timestamp.
- `state` — `Active` | `Converted` | `Released`.

Lifecycle:

| From → To | Trigger |
|---|---|
| (new) → Active | Order `init` (Beckn) or first-party checkout step |
| Active → Converted | Order `confirm` — `stock_count` is decremented atomically; reservation record retained |
| Active → Released | Order cancelled, OR `expires_at` passed |

The exact Beckn-flow trigger points are owned by [Gap 11](../gaps/11-order-and-fulfillment-phasing.md); this ADR commits only the primitive and its lifecycle. Reservation timeout values are operational configuration, not architectural.

### Stock-movement event log

Every stock-affecting action emits a typed `StockMovement` event:

| Type | When | Stock delta |
|---|---|---|
| `Received` | Owner records incoming stock (manual or supplier integration) | +N |
| `Sold` | Reservation → Converted at order confirm | −N |
| `Returned` | Order return processed (details in Gap 11) | +N |
| `Corrected` | Manual adjustment with reason note (positive or negative) | ±N |
| `Reserved` | Reservation Active | 0 (logical) |
| `Released` | Reservation Released | 0 (logical) |
| `PurchasableToggled` | Owner flipped the `purchasable` flag | 0 |

Each event carries: actor, timestamp, item reference, signed delta (or null for non-stock-affecting), optional free-text reason. The event stream is Inventory's contribution to the audit trail.

### Communication

- **Storefront, Admin UI, Bridge** → query Inventory **synchronously** for current availability of one or more items.
- **Order context** → calls Inventory's Application Layer to create / convert / release Reservations.
- **Inventory** → emits `StockMovement` events. Subscribers include Audit ([Gap 16](../gaps/16-soft-delete-and-audit.md)) and any future read-side projections.
- **Catalog → Inventory** subscription: Inventory subscribes to Catalog's product/variant lifecycle events:
  - `ProductCreated` / `ProductVariantAdded` → Inventory creates a `StockLevel` with `stock_count=0`, `purchasable=true`.
  - `ProductArchived` / `ProductVariantRemoved` → Inventory marks the corresponding `StockLevel` as `Inactive`. The record is retained for history; no new reservations or sales are accepted against it.
  - `ProductRestored` → Inventory restores the `StockLevel` to `Active` (keeping existing `stock_count` and `purchasable`).

### Boundary recap

- **Catalog** owns: entity identity, attributes, variants, categories, media, lifecycle.
- **Inventory** owns: stock count, reservations, purchasable flag, movement history.
- Inventory holds opaque references to Catalog entities; never joins or reads Catalog internals (§2.6).

## Consequences

What this commits to:

- The Inventory context contains: `StockLevel`, `Reservation`, plus the `StockMovement` event log. Three entities; one event stream.
- Inventory is store-scoped — single logical inventory per store.
- Stock decrement happens **only** via Reservation conversion at order confirm. Direct stock writes are limited to `Corrected` (manual adjustment) and `Received` (receiving).
- The Order context ([Gap 11](../gaps/11-order-and-fulfillment-phasing.md)) is the primary consumer of Reservations. The Bridge orchestrates the Beckn-side timing.
- The Bridge queries Inventory synchronously when projecting availability into catalogs (for `/catalog/publish` to CDS) and into `/on_select` quotes; availability values are computed on demand.
- Inventory subscribes to Catalog's product/variant lifecycle events. This is the first context-to-context event subscription declared in the design.
- Audit ([Gap 16](../gaps/16-soft-delete-and-audit.md)) ingests `StockMovement` events.
- The `availability_flag` placeholder on `ProductVariant` from ADR-0005 is **removed** — availability lives in `StockLevel.purchasable`. [`design/catalog.md`](../design/catalog.md) is updated accordingly; ADR-0005's deferral note ("final shape decided in Gap 08") is honored.

What this defers:

- **Multi-location inventory** (Warehouse/Location concept).
- **Backorders / preorders** (`allow_backorder` flag).
- **Low-stock thresholds and alerts** — Storefront/Admin UI concern.
- **Inventory reconciliation flows** (physical count vs. system) — operational; handled via `Corrected` events.
- **Supplier integrations / automated receiving** — out of scope; eventually surfaces as `Received` events from an integration layer.
- **Reservation timeout values** — operational configuration.
- **Reservation contention rules** under concurrency — implementation detail; the model accepts that concurrent `init` requests for the last unit race, and one will fail.
- **Event-driven availability read-side projection** — added later if synchronous queries don't scale.

What this makes harder:

- **Selling beyond stock.** No backorder mechanism; owners cannot accept orders for items they don't have.
- **Cross-store unified inventory.** Each store's inventory is independent; multi-store retailers can't pool stock across their stores.
- **Mixing tracked and untracked inventory** in one store. The model is uniformly tracked; "untracked" is approximated by leaving `stock_count` high and using `purchasable` as the gate.

## References

- [gaps/resolved/08-inventory-boundary.md](../gaps/resolved/08-inventory-boundary.md)
- [ADR-0005](0005-catalog-and-product-modeling.md) — defines the inventory anchors (Product / ProductVariant)
- [design/catalog.md](../design/catalog.md) — updated to reflect availability moving to Inventory
- Related gaps: [09](../gaps/09-pricing-and-promotions.md) (Pricing), [11](../gaps/11-order-and-fulfillment-phasing.md) (Order/Fulfillment — Reservation triggers), [13](../gaps/13-domain-events-design.md) (Domain events), [16](../gaps/16-soft-delete-and-audit.md) (Audit)
- `ion-specs` — Beckn `Item.quantity` concepts
