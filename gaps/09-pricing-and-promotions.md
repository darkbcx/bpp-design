# Gap 09 — Pricing & promotions

## Statement

Pricing is named as part of Catalog (§2.5) but never modeled. Whether prices are simple per-variant numbers, time-bound, multi-currency, tax-inclusive, or composed with promotions is unresolved. Promotions / vouchers (a feature in `bitemycart`) are not mentioned at all. Pricing is on the path of every checkout and every Beckn `select` / `init` quote — it cannot stay vague.

## Current charter coverage

* §2.5 — Catalog includes "pricing."
* §3.2 — `Money` is listed as a value object.
* No stance on promotions, taxes, currency, or pricing rules.

## Open questions

1. **Storage granularity.** Price on Product, on Variant, on both with override? List price vs. effective price separated?
2. **Time-bound prices.** Built-in support for sale windows / future price changes, or single current price only?
3. **Tax model.** Tax inclusive or exclusive in stored price? Tax rates per region, computed at quote time? Who owns tax rules — Catalog, Order, or external?
4. **Currency.** Single currency per store, per product, or platform-wide? (Tightly linked to Gap 10.)
5. **Promotion mechanism.** Code-based vouchers (like `bitemycart`), automatic rules (e.g., "10% off if cart > X"), or both? Is "Promotion" its own context?
6. **Customer-specific pricing.** Are there segments (B2B, member-only, loyalty tiers) that see different prices?
7. **Quote vs. catalog price.** Should the system distinguish between a catalog-listed price and a quoted price (with line-item adjustments, shipping, taxes) attached to an order?
8. **Rounding / display.** Where is currency rounding applied — at storage, computation, or display?

## Implications

* Beckn `on_select` returns a quote: line items, discounts, taxes, fulfillment charges. The shape of that quote is determined by domain pricing.
* Promotion mechanics are commonly retrofitted badly — picking a stance early avoids painful migrations.
* Tax / currency / locale interact with regulatory requirements.

## Dependencies

* Depends on: 07 (catalog), 10 (currency).
* Blocks: 11 (order quoting).
