# ADR-0007: Pricing, tax, and vouchers

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/09-pricing-and-promotions.md](../gaps/resolved/09-pricing-and-promotions.md)
- **Builds on**: [ADR-0005](0005-catalog-and-product-modeling.md) (price placeholders), [ADR-0006](0006-inventory-model.md) (boundary precedent)
- **Introduces**: the **Promotion** bounded context

## Context

[ADR-0005](0005-catalog-and-product-modeling.md) declared `default_price` on Product and `price_override` on ProductVariant as deferred placeholders. Gap 09 picks those up and adds:

- The shape of `Money` and currency policy.
- A tax model that stores the breakdown at the item level (not just on the final quote).
- A minimal **voucher** mechanism (a v1 feature; bitemycart-style codes are central to the target retailers).
- A clear seam between Catalog (item prices) and Order (quote construction).

The tax decision also drives Beckn quote construction: `on_select` returns an itemized quote with tax breakdowns; Catalog must give the Order context enough to construct that quote without re-deriving tax logic.

Vouchers introduce a new bounded context — **Promotion** — alongside Catalog, Inventory, Order, etc.

## Options considered

### Money representation

**A. `Money` value object: integer minor units + ISO 4217 currency code** — chosen. Standard pattern; avoids floating-point hazards.

**B. Decimal type with implicit currency.** Rejected — couples currency to context.

### Currency policy

**A. Single currency per store, immutable after first Active product** — chosen. Matches small-retailer market; defers multi-currency until proven needed.

**B. Multi-currency catalog.** Rejected — over-engineered.

**C. Platform-wide currency.** Rejected — restricts geographic scope unnecessarily.

### Tax model

**A. Tax-inclusive prices, tax reported in quote only.** Rejected — owner wanted the breakdown crystallized at the item level for transparency, Beckn quote construction, and accounting.

**B. Tax-excluded `base_price` + `tax_rate`, with derived `tax_amount` and `published_price` cached per item** — chosen.

**C. Configurable inclusive/exclusive per store.** Rejected — adds a mode discriminator without clear value at v1.

### Tax rate granularity

**A. Per-Product, with variants inheriting** — chosen. Tax category attaches to the product, not the variant. Variants of the same product (S/M/L of a shirt) don't have different tax categories.

**B. Per-store default with per-Product override.** Rejected — adds a fallback path; UX helpers can handle bulk entry.

**C. Per-variant.** Rejected — tax category doesn't vary by size/color.

### Promotions

**A. Defer entirely.** Rejected — vouchers are central to bitemycart-pattern retailers; expected as a v1 feature.

**B. Minimal voucher mechanism** — chosen. Code-based, store-scoped, two discount types, usage limits, expiry.

**C. Rule-based engine.** Rejected — overkill for v1.

### Discount × tax interaction

**A. Discount applies to `base_price`; tax recomputes on the discounted base** — chosen. GST/VAT-compliant; matches accounting reality (tax on actual receivable).

**B. Discount applies to `published_price`; breakdown is "after discount".** Rejected — splits the tax math; harder to reconcile.

## Decision

### 1. Money value object

`Money = (amount: integer in minor units, currency: ISO-4217 code)`. All monetary quantities use this type. Display rounding is UI-side; storage is integer.

### 2. Currency

A store has **one currency**, set at creation. The currency is **immutable once the store has any Active product** — to prevent retroactive currency reassignment of priced items. A store seeking another currency creates a new store.

Currency lives on the Store entity (Tenancy context). All `Money` values in the store's Catalog, Promotion, and Order data use that currency.

### 3. Tax model — per-Product

Each **Product** (Flat or Matrix) stores:
- `base_price` — `Money` in the store's currency; tax-excluded.
- `tax_rate` — decimal percentage (e.g., `18.00` for 18%).

Derived attributes, cached at write time, maintained as an entity invariant:
- `tax_amount = round(base_price × tax_rate / 100)` — rounded to the minor units of the store's currency.
- `published_price = base_price + tax_amount` — what the buyer sees and pays.

**Invariant:** `tax_amount` and `published_price` are recomputed and rewritten on every change to `base_price` or `tax_rate`.

**ProductVariant (Matrix)** stores only `price_override` (optional; `Money` in the store's currency). The variant's effective values are derived on read:
- `effective_base_price = price_override ?? parent.base_price`
- `effective_tax_amount = round(effective_base_price × parent.tax_rate / 100)`
- `effective_published_price = effective_base_price + effective_tax_amount`

Variants do **not** carry their own `tax_rate` — they inherit the parent Product's. Variants do **not** cache their derived tax/published values, since the parent's `tax_rate` may change.

### 4. Quote construction

The **quote** — line items with applied vouchers, taxes, fulfillment charges, and totals — is an **Order context** concept, not Catalog. Catalog provides the inputs (item prices, tax breakdowns); Order constructs the quote at `on_select` (Beckn) or first-party checkout time. No "quoted price" is stored in Catalog.

### 5. Promotion context (new)

A new bounded context — **Promotion** — owns vouchers and voucher usage. It sits alongside Catalog, Inventory, Order, etc.

#### Voucher entity

| Field | Description |
|---|---|
| `id` | Internal opaque identifier |
| `store_id` | Owning Store (cross-context reference to Tenancy) |
| `code` | Case-insensitive; **unique within store** |
| `discount_type` | `percentage` \| `fixed_amount` |
| `discount_value` | Percentage (decimal) or `Money` (in store's currency) |
| `max_total_uses` | Integer, optional; null = unlimited |
| `max_uses_per_user` | Integer, optional; null = unlimited |
| `starts_at` | Timestamp, optional |
| `expires_at` | Timestamp, optional |
| `minimum_cart_value` | `Money`, optional |
| `status` | `Active` \| `Disabled` (owner toggle) |
| `current_total_uses` | Derived from `VoucherUsage`; cached for fast limit checks |

**Derived states** (not stored):
- `Expired` — `expires_at < now`
- `Exhausted` — `current_total_uses >= max_total_uses`

A voucher is **usable** if all hold: `status == Active`, not `Expired`, not `Exhausted`, and `now >= starts_at` (if set).

#### VoucherUsage entity

| Field | Description |
|---|---|
| `id` | Internal opaque identifier |
| `voucher_id` | Reference |
| `user_id` | Redeeming User (Identity & Access reference) |
| `order_id` | Order reference |
| `applied_at` | Timestamp |
| `discount_applied_amount` | `Money`; the actual discount applied after computation |

`VoucherUsage` is the authoritative redemption log; usage limits are enforced via this history.

### 6. Voucher application flow

Voucher application happens in the **Order context** at quote construction time. The flow:

1. Buyer applies a voucher code (via storefront or Beckn `select` / `init`).
2. Order calls `Promotion.ValidateVoucher(code, store, user, cart_total)` → returns a computed discount or a typed error (invalid code, expired, exhausted, limit exceeded, below minimum cart value, etc.).
3. The discount is applied to the **sum of base prices** (pre-tax) in the cart. For percentage discounts, it's proportional across line items; for fixed-amount, it's distributed proportionally to base values.
4. Tax is recomputed on each line item's discounted base; the quote shows the breakdown.
5. At order `confirm`, Order calls `Promotion.RecordVoucherUsage(voucher, user, order, applied_amount)`. The usage record bumps `current_total_uses`.
6. If the order is later cancelled or refunded, Order calls a release/refund flow — exact mechanism deferred to [Gap 11](../gaps/11-order-and-fulfillment-phasing.md).

**One voucher per order in v1.** No stacking.

### 7. Promotion context — exposed surface

**Owner / Admin use cases** (store-scoped, per [ADR-0002](0002-authorization-tiers-and-matrix.md) capabilities):
- `CreateVoucher`, `UpdateVoucher`, `DisableVoucher`, `EnableVoucher`
- `ListVouchers` (with filters), `GetVoucherDetail`, `GetVoucherUsageHistory`

**Order context calls** (cross-context, Application-Layer):
- `ValidateVoucher(code, store, user, cart) → discount-or-error` — query at quote time.
- `RecordVoucherUsage(voucher, user, order, amount)` — write at confirm time.

**Domain events emitted:**
- `VoucherCreated`, `VoucherUpdated`, `VoucherDisabled`, `VoucherEnabled`
- `VoucherRedeemed` (on `RecordVoucherUsage` success)

## Consequences

What this commits to:

- **Money** is a domain value object; floating point is forbidden for monetary values.
- **Currency** is store-scoped, single, and locked at first Active product.
- **Tax** is stored as `base_price` + `tax_rate` per Product, with derived `tax_amount` and `published_price` maintained as an entity invariant.
- **Tax recomputation** is triggered by changes to `base_price` or `tax_rate`. Variants recompute on read.
- **Promotion** is a new bounded context with two entities (`Voucher`, `VoucherUsage`) and a clear cross-context API consumed by Order.
- **Discount math** is GST/VAT-compliant: discount applies to base prices, tax recomputes on the discounted amounts.
- **One voucher per order in v1.** Enforced in Order context at validation time.
- **Voucher application is Order-mediated.** Promotion exposes only validation and usage-recording; Order owns the quote.

What this defers:

- **Platform-wide promotions** — not in v1. Adding later would introduce a `platform_id` discriminator on Voucher.
- **Rule-based promotions** (BOGO, X-for-Y, automatic cart discounts) — not in v1.
- **Customer-specific pricing tiers** (B2B, members, loyalty) — not in v1.
- **Time-bound list-price changes** (sales windows on the catalog price itself) — not in v1; voucher `starts_at` / `expires_at` can approximate sale windows.
- **Multi-currency catalogs** — explicitly out of scope.
- **Voucher behavior during refund/cancel** — Order context concern ([Gap 11](../gaps/11-order-and-fulfillment-phasing.md)).
- **Voucher stacking rules** — default is one per order; future work would relax this.
- **Voucher distribution mechanics** (sharing codes via email, links, etc.) — operational; not domain.

What this makes harder:

- **Per-variant tax rates.** Forbidden — if a store has variants in different tax categories, they must use Flat mode and create separate Products.
- **Mid-life currency changes.** A store that goes Active in one currency cannot switch; must create a new store.
- **Floating list prices.** Without time-bound list prices, sales must be modeled as vouchers, which has different accounting characteristics.

## References

- [gaps/resolved/09-pricing-and-promotions.md](../gaps/resolved/09-pricing-and-promotions.md)
- [ADR-0005](0005-catalog-and-product-modeling.md), [ADR-0006](0006-inventory-model.md)
- [design/catalog.md](../design/catalog.md) — Product / Variant entity reference (updated to point here for pricing attributes)
- Related gaps: [10](../gaps/10-localization-and-currency.md), [11](../gaps/11-order-and-fulfillment-phasing.md), [13](../gaps/13-domain-events-design.md), [16](../gaps/16-soft-delete-and-audit.md)
- `bitemycart` — voucher pattern reference (Voucher, CustomerVoucher, VoucherUsage)
- `ion-specs` — `Item.price`, quote breakdown semantics
