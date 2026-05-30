# ADR-0017: Order, fulfillment, and the v2 transactional flow

- **Status**: Accepted
- **Date**: 2026-05-30
- **Partially superseded by**: [ADR-0021](0021-pure-bpp-no-storefront.md) — §4 (Buyer polymorphism) and §8 (first-party Order use cases including `Order.Place`) are replaced; Buyer is Beckn-typed only, and the Bridge is the sole external entry
- **Resolves gap**: [gaps/resolved/11-order-and-fulfillment-phasing.md](../gaps/resolved/11-order-and-fulfillment-phasing.md)
- **Builds on**: every prior ADR — this is the integration ADR for the commerce track
- **Sub-design**: [design/order.md](../design/order.md)

## Context

Order & Fulfillment is the last remaining bounded context and the largest remaining gap. It pulls together every commerce ADR — Catalog (Products + Variants), Inventory (Reservation), Promotion (Voucher), Pricing (Money, tax), Identity (buyer User in first-party flows), Localization (LocalizedText), Audit (compliance trail) — into end-to-end flows.

This ADR also commits the **v2 wire vocabulary** at the Bridge boundary: `Contract` replaces v1's `Order`; `Resource` replaces `Item`; `Offer` is a separate entity; discovery is **CDS-mediated** (the BPP publishes catalogs to a Catalog Discovery Service; it does **not** field `/discover` queries directly).

## Options considered

### Wire-protocol coverage in v1

**A. Full set** (`select` through `support`). Rejected — overreach for v1.
**B. Minimum useful set** — `select`, `init`, `confirm`, `status`, `cancel` (plus `catalog/publish` outbound). Chosen.
**C. Search-only.** Rejected — BPP doesn't field `/discover` in v2; CDS does.

### Payment integration

**A. External to BPP.** Order has `payment_status` driven by external signals. Chosen.
**B. In-scope payment gateway.** Rejected for v1 — PCI scope, refunds, gateway adapter, heavy.

### Fulfillment model

**A. Self-fulfilled by stores.** Chosen.
**B. Platform-managed logistics.** Deferred.
**C. External Beckn-mediated fulfillment provider.** Deferred.

### Quote lifecycle

**A. Stateless** (recompute each request). Rejected — loses correlation across `select` → `init` → `confirm`.
**B. Stored on the Order with TTL.** Chosen.
**C. Quote as a separate entity.** Rejected — adds an entity without clear benefit; the Order's `Created` state subsumes it.

### Buyer representation

**A. Always a User.** Rejected — Beckn buyers don't have User accounts on our side.
**B. Polymorphic: User (first-party) or Beckn snapshot.** Chosen.

### Cancellation scope

**A. Pre-fulfillment only in v1.** Chosen.
**B. Pre + post-fulfillment refund/return.** Deferred (substantial workflow).

### Multi-step coordination

**A. Application-Layer orchestration with hand-coded compensation** (per ADR-0012's deferred-revisit). Chosen.
**B. Formal Saga framework.** Rejected — no flow in v1 is long-running.

## Decision

### 1. Order is a new bounded context

A new **Order & Fulfillment** bounded context — the seventh and last. Joins §2.5 alongside Identity & Access, Tenancy, Catalog, Inventory, Promotion, Audit.

### 2. Order entity and state machine

Single Order entity transitioning through six states:

| State | Entered when | Wire `Contract.status` |
|---|---|---|
| `Created` | `/select` (Beckn) or first-party "view cart" — Quote exists, TTL counting | `DRAFT` |
| `Initiated` | `/init` (Beckn) or first-party "checkout" — Reservation held, buyer contact present | `DRAFT` |
| `Confirmed` | `/confirm` (Beckn) or first-party "place order" — Reservation Converted, Voucher recorded | `ACTIVE` |
| `Fulfilled` | Store marked delivered | `COMPLETE` |
| `Cancelled` | Pre-fulfillment cancel — Reservation released, Voucher usage rolled back | `CANCELLED` |
| `Expired` | Quote TTL passed before Initiated/Confirmed — Reservation released if held | `CANCELLED` (with `reason = quote_expired`) |

No `Returned` state in v1. No deletion (consistent with §5.19). Audit retention bucket: financial (~7 years per ADR-0014). The wire-status mapping lives in the Bridge's mapping registry.

### 3. Quote model

Stored on the Order at `Created`. Captures **snapshots** that don't auto-refresh:

- `line_items[]`: each carries product_id (Flat) or variant_id (Matrix), quantity, `captured_base_price`, `captured_tax_rate`, `captured_tax_amount`, `captured_published_price`, captured `LocalizedText` for name/description.
- `voucher_applied`: optional — voucher_id, discount_type, discount_value, computed `discount_amount`.
- `totals`: subtotal, tax_total, discount_total, grand_total.
- `valid_until`: timestamp. **Default TTL: 15 minutes** (operational).

Captured snapshots are essential for receipt stability and dispute resolution — if Catalog prices or Voucher terms change after the Quote is issued, the Order remains as quoted.

### 4. Buyer (polymorphic)

```
buyer = {
  type: "user" | "beckn",

  // type = user (first-party): always present
  user_id?: string,

  // type = beckn (Beckn flow)
  bap_id?: string,
  transaction_id?: string,
  contact_snapshot?: {
    name: LocalizedText,
    email: string,
    phone: string,
    address: ShippingAddress,
  }
}
```

For **Beckn buyers**: at `Created`, `contact_snapshot` is `null` (per v2's "no PII at /select" rule). At `Initiated`, it's populated from the `/init` payload.

For **First-party buyers**: the User is known from the session (§5.15); `user_id` is set at Created. A contact snapshot may still be captured at checkout for shipping (separate from User profile).

### 5. Payment — external to BPP

Order carries:
- `payment_status`: `Pending` | `Authorized` | `Captured` | `Refunded` | `Failed`
- `payment_external_ref`: optional (BAP-supplied ID, gateway reference, etc.)

Status transitions are driven by external signals:
- **Beckn flows**: BAP/network-side; status updated through inbound message metadata.
- **First-party**: gateway webhooks if/when added — not in v1 scope.

The BPP does **not** store card data, authorize, or capture payments.

### 6. Fulfillment — self-fulfilled

Order carries:
- `fulfillment_status`: `Pending` | `Preparing` | `Shipped` | `Delivered`
- `fulfillment_note`: optional free-text (tracking reference, courier info)

Store admins transition `fulfillment_status` via store-scoped use cases. The Bridge projects to wire `Contract.performance` array.

### 7. Cancellation — pre-fulfillment only

- Owner-initiated cancel (Beckn `/cancel` or first-party "cancel order"): allowed when state ∈ {`Created`, `Initiated`, `Confirmed`}, i.e., before `Fulfilled`.
- Platform-initiated cancel: same state rules, plus capability check (per ADR-0016).
- On cancel:
  - If a Reservation is Active → release it via `Inventory.ReleaseReservation`.
  - If a Voucher has been recorded → roll back via `Promotion.RevertVoucherUsage(voucher_usage_id, reason)`. *This adds a new event in Promotion — see §10 below.*
  - If `payment_status == Captured` → record `Refunded` status (the actual refund is external).
  - Order transitions to `Cancelled`.

Returns / refunds **post-fulfillment** are deferred.

### 8. Use cases (Application Layer)

Beckn-driven (invoked by the Bridge):
- `Order.CreateQuote(store_id, items, voucher_code?, beckn_buyer_ref) → Order`
- `Order.Initiate(order_id, contact_snapshot) → Order`
- `Order.Confirm(order_id) → Order`
- `Order.GetStatus(order_id) → Order`
- `Order.Cancel(order_id, reason, by_actor) → Order`

First-party storefront:
- `Order.CreateQuote(store_id, items, voucher_code?, user_id) → Order`
- `Order.Place(order_id, contact_snapshot?) → Order` — combines Initiate + Confirm in one step (no `/init` round-trip needed for first-party)
- `Order.Cancel(order_id, reason, by_actor) → Order`
- `Order.GetStatus(order_id) → Order`

Fulfillment (store admin):
- `Order.MarkPreparing(order_id, by_actor)`
- `Order.MarkShipped(order_id, by_actor, fulfillment_note?)`
- `Order.MarkFulfilled(order_id, by_actor)`

Read-side (capability-gated per ADR-0016):
- `Order.ListForStore(store_id, filter, acting_user) → [Order]`
- `Order.ListForUser(user_id, filter, acting_user) → [Order]`

All use cases requiring action call `AuthorizationPort.requireCapability(...)` per ADR-0016.

### 9. Cross-context orchestration

`Order.Initiate(order_id, contact_snapshot)`:

```
require_capability(order.initiate, scope=store_id)
load Order; assert status == Created; assert now < valid_until

reservations = []
for line in line_items:
  res = Inventory.Reserve(line.item_ref, line.quantity, held_by=order_id)
  reservations.append(res)

if voucher_applied:
  try:
    discount = Promotion.ValidateVoucher(voucher_code, store, buyer, totals)
  except VoucherInvalid:
    for res in reservations: Inventory.ReleaseReservation(res.id)
    raise

persist:
  order.reservation_ids = [r.id for r in reservations]
  order.contact_snapshot = contact_snapshot
  order.status = Initiated
emit: order.initiated
```

`Order.Confirm(order_id)`:

```
require_capability(order.confirm, scope=store_id)  // or implicit if Bridge-originated
load Order; assert status == Initiated; assert now < valid_until

for res_id in order.reservation_ids:
  Inventory.ConvertReservation(res_id)

if order.voucher_applied:
  Promotion.RecordVoucherUsage(voucher, buyer, order, order.discount_amount)

order.status = Confirmed
emit: order.confirmed
```

`Order.Cancel(order_id, reason, by_actor)`:

```
require_capability(order.cancel, scope=store_id)
load Order; assert status in {Created, Initiated, Confirmed}

if status == Initiated and reservations active:
  for res_id in order.reservation_ids: Inventory.ReleaseReservation(res_id)

if status == Confirmed and voucher_usage exists:
  Promotion.RevertVoucherUsage(order.voucher_usage_id, reason)
  // Note: stock already converted; v1 doesn't return stock on cancel
  //       (cancellation differs from a Return — return is post-fulfillment, deferred)

if payment_status == Captured:
  payment_status = Refunded  // actual refund external
  emit: order.payment_status_changed

order.status = Cancelled
order.cancel_reason = reason
emit: order.cancelled
```

Failure mid-orchestration → use case explicitly compensates already-affected contexts (per ADR-0012). No generic rollback.

### 10. Domain events

New event family for Order (joins the registry):

- `order.quote_created` — at Order creation, carries snapshot
- `order.initiated` — Reservation held, buyer contact present
- `order.confirmed` — Reservation converted, voucher recorded
- `order.cancelled`
- `order.expired`
- `order.marked_preparing`
- `order.marked_shipped`
- `order.fulfilled`
- `order.payment_status_changed`

Plus one new event in Promotion (needed for cancel rollback):
- `promotion.voucher_usage_reverted` — `VoucherUsage` is logically nullified (the redemption no longer counts against `current_total_uses`)

Audit subscribes per ADR-0014's "broad subscription". PII catalog (`design/pii.md`) gains new entries for Order events that carry buyer details.

### 11. Bridge ↔ Order interface

The Bridge translates wire messages into domain calls:

| Wire (BAP → BPP) | Domain call |
|---|---|
| `/select` (Contract) | `Order.CreateQuote(store_id, items, voucher_code?, beckn_buyer_ref)` |
| `/init` (Contract + buyer details) | `Order.Initiate(order_id, contact_snapshot)` |
| `/confirm` (Contract) | `Order.Confirm(order_id)` |
| `/status` | `Order.GetStatus(order_id)` |
| `/cancel` | `Order.Cancel(order_id, reason, by_actor=bridge_principal)` |

Callbacks (BPP → BAP): `/on_select`, `/on_init`, `/on_confirm`, `/on_status`, `/on_cancel` — Bridge projects the Order to a Contract via the mapping registry.

The mapping registry maps:
- Order (domain) ↔ Contract (wire), including state → `Contract.status` (DRAFT/ACTIVE/CANCELLED/COMPLETE)
- Line items ↔ `Commitment` + `Consideration`
- Order.fulfillment_status ↔ `Performance` array
- Buyer ↔ `Participant`
- Voucher / discount ↔ `Consideration` adjustments

Domain stays protocol-naive. Bridge owns transaction_id correlation (per ADR-0011 §4.3).

### 12. Catalog publishing (CDS-side, explicit)

The Bridge publishes catalog updates to CDS via `/catalog/publish` on the domain events it already subscribes to (`tenancy.store_status_changed`, `catalog.*`). This is the wire-level mechanism for ADR-0003 / ADR-0004's "re-projection" — now explicit.

The CDS callback `/catalog/on_publish` is consumed by the Bridge for acknowledgement tracking. Publish-retry / dead-letter is operational (per ADR-0012's failure-handling rule).

The CDS endpoint(s) are operational configuration (like the registry / signing keys per ADR-0001).

## Consequences

What this commits to:

- A new **Order & Fulfillment** bounded context (the seventh and last).
- Order owns Quote (with captured snapshots), payment_status, fulfillment_status.
- Beckn buyers are anonymous-by-design — captured per-transaction; no User record created.
- Payment is external to BPP — PCI scope is avoided.
- Self-fulfillment by stores; no logistics integration in v1.
- Pre-fulfillment cancellation only; returns/refunds deferred.
- **No Saga framework** — Application-Layer orchestration with explicit compensation.
- Promotion gains a `voucher_usage_reverted` event for cancellation rollback.
- The Bridge implements transactional handlers (`/select`, `/init`, `/confirm`, `/status`, `/cancel`) and catalog-publish outbound.
- CDS is an external infrastructure dependency, configured operationally.
- Quote captures snapshots; price/voucher changes after issuance don't propagate retroactively.

What this defers:

- `/track`, `/update`, `/rate`, `/support` handlers.
- Refund / return workflows (post-fulfillment).
- Payment gateway integration (PCI scope when needed).
- Platform-managed logistics, courier integrations.
- Multi-shipment / split orders.
- Subscriptions / recurring orders.
- Marketplace fees / payouts to stores.
- ION `/raise` and `/reconcile` (dispute / settlement extensions).
- Catalog subscription / pull mode (CDS-territory).

What this makes harder:

- **Order modification post-Init** — no `/update` handler in v1. Buyers must cancel and re-quote.
- **Splitting an order across shipments** — Order is one-shot fulfillment.
- **Subscription / recurring orders** — would require new modeling.
- **Adding payment in-scope later** — feasible (extend payment_status with gateway transitions), but a future ADR; not free.

## References

- [gaps/resolved/11-order-and-fulfillment-phasing.md](../gaps/resolved/11-order-and-fulfillment-phasing.md)
- [design/order.md](../design/order.md) — full entity model, state machine, sequence walkthroughs, Bridge mapping
- Every prior ADR (0001–0016) — Order pulls them together
- `ion-specs` v2.0.0 — `Contract`, `Resource`, `Offer`, `Performance` schemas
