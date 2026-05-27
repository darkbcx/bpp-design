# Gap 08 — Inventory boundary

## Statement

Inventory is listed as a bounded context "initial scope to be decided" (§2.5). Until we decide where Catalog ends and Inventory begins — and what each owns — neither can be modeled. The line between *availability* and *stock* is the central ambiguity.

## Current charter coverage

* §2.5 — "Inventory — availability, stock levels, fulfillment readiness (initial scope to be decided)."
* No further detail.

## Open questions

1. **Stock counting.** Does Inventory track integer stock per variant, or only a boolean "available / unavailable"?
2. **Reservations / holds.** When a buyer initiates a Beckn `init` or starts a checkout, is stock held? For how long? Who releases it?
3. **Availability semantics.** Is "available" derived from stock > 0, from an explicit toggle, or both? Who flips the toggle and why?
4. **Multi-location stock.** Single warehouse per store at launch, or multi-location from day one?
5. **Backorders / preorders.** Allow selling beyond stock with a configured policy, or strict "out-of-stock = unsellable"?
6. **Adjustments and history.** Is stock adjustment a first-class event (receive, sell, return, correction)? Audit trail per movement?
7. **Catalog vs. Inventory ownership.** Catalog owns "what a product is"; Inventory owns "is this purchasable right now." Confirm this carve-out is correct, or revise.
8. **Communication mode.** Does the storefront ask Inventory directly on each page view, or read a materialized snapshot from Catalog? Event-driven sync between the two?

## Implications

* The Bridge's `on_search` response needs to know item availability; that fact must come from somewhere.
* Order placement (Gap 11) must coordinate with Inventory for stock decrement and reservation.
* Decisions here directly affect the consistency model (Gap 14).

## Dependencies

* Depends on: 07 (catalog).
* Blocks: 11 (order/fulfillment).
