# Design: Order & Fulfillment

- **Status**: Draft
- **Last updated**: 2026-05-30
- **Backed by ADRs**: [ADR-0017](../decisions/0017-order-and-fulfillment.md) (this); integrates with every prior ADR

## Purpose

Full design of the Order & Fulfillment bounded context — the entities, the state machine, the use cases, the cross-context orchestration patterns, and the Bridge ↔ Order wire mapping for v2 Beckn.

**What this document covers:**
- `Order` entity and embedded `Quote` / `LineItem` / `Buyer` structures.
- The Order state machine with transition guards.
- Cross-context orchestration sequences (`Initiate`, `Confirm`, `Cancel`).
- Beckn-flow walkthroughs (`/select` → `/on_select`, etc.).
- Bridge ↔ Order mapping (domain ↔ wire `Contract`).
- Read-side queries.
- Domain events emitted.

**What it does NOT cover:**
- Refund / return workflows (deferred).
- Payment gateway integration (deferred).
- Platform-managed logistics (deferred).
- `/track`, `/update`, `/rate`, `/support` handlers (deferred).

## Position within the architecture

Order & Fulfillment is a bounded context (CLAUDE.md §2.5). It:

- **Depends on** (calls these contexts' Application-Layer ports per §5.17):
  - **Catalog** — resolve product/variant info, capture LocalizedText snapshots at quote time.
  - **Inventory** — Reserve, Convert, Release reservations ([ADR-0006](../decisions/0006-inventory-model.md)).
  - **Promotion** — ValidateVoucher, RecordVoucherUsage, RevertVoucherUsage ([ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md)).
  - **Identity & Access** — impersonation-aware actor for audit on store-admin actions ([ADR-0009](../decisions/0009-identity-and-external-idp.md), [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md)). Note: buyers are not Users — see [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md).

- **Is consumed by**:
  - **Beckn Bridge** — the sole external entry point. Translates wire messages into Order use case calls; projects Orders back to wire `Contract`.
  - **Admin UI** — Store Admins / Org Owners fulfill / cancel orders.
  - **Audit** — subscribes to Order events.

The platform is a pure BPP per [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md): there is no first-party buyer storefront. All buyer flows arrive via the Bridge from BAPs over the Beckn network.

## Concepts

### Order entity

| Field | Description |
|---|---|
| `id` | Internal opaque identifier; cross-context reference key |
| `store_id` | Reference to the owning Store (Tenancy) |
| `status` | `Created` \| `Initiated` \| `Confirmed` \| `Fulfilled` \| `Cancelled` \| `Expired` |
| `quote` | Embedded `Quote` (see below) — captured at Created, immutable thereafter |
| `buyer` | Beckn `Buyer` — `{ bap_id, transaction_id, contact_snapshot? }`; see below. Always Beckn-typed per [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md). |
| `reservation_ids` | List of `Inventory.Reservation` IDs (populated at Initiated) |
| `voucher_usage_id` | Optional `Promotion.VoucherUsage` ID (set at Confirmed) |
| `payment_status` | `Pending` \| `Authorized` \| `Captured` \| `Refunded` \| `Failed` |
| `payment_external_ref` | Optional string — BAP-supplied or gateway reference |
| `fulfillment_status` | `Pending` \| `Preparing` \| `Shipped` \| `Delivered` |
| `fulfillment_note` | Optional free-text (tracking ref, courier info) |
| `cancel_reason` | Optional, set when `status == Cancelled` |
| `created_at`, `updated_at` | Timestamps |
| `confirmed_at`, `fulfilled_at`, `cancelled_at`, `expired_at` | Optional milestone timestamps |

### Quote (embedded in Order)

| Field | Description |
|---|---|
| `line_items[]` | Each is a `LineItem` (see below) |
| `voucher_applied` | Optional — `{ voucher_id, code, discount_type, discount_value, discount_amount }` |
| `subtotal` | Sum of base prices × quantity, pre-discount |
| `discount_total` | Total applied discount |
| `tax_total` | Sum of computed taxes after discount |
| `grand_total` | Final amount due |
| `currency` | Store's currency (per ADR-0007); same for all line items |
| `valid_until` | Timestamp — Quote expires after this |

### LineItem (embedded in Quote)

| Field | Description |
|---|---|
| `item_ref` | Either `{type: "product", product_id}` (Flat) or `{type: "variant", variant_id}` (Matrix) |
| `quantity` | Integer ≥ 1 |
| `captured_name` | `LocalizedText` snapshot of the Product/Variant name at quote time |
| `captured_description` | `LocalizedText` snapshot |
| `captured_base_price` | `Money` — tax-excluded |
| `captured_tax_rate` | Decimal percentage |
| `captured_tax_amount` | `Money` — derived: `base_price × tax_rate / 100` |
| `captured_published_price` | `Money` — derived: `base_price + tax_amount` |
| `line_subtotal` | `Money` — `published_price × quantity` |

**Snapshots are immutable per ADR-0017 §3.** Catalog price changes after issuance do not propagate.

### Buyer (Beckn-typed value object)

Per [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md), the platform is a pure BPP — there is no first-party storefront and no User-typed buyer. The Buyer attribute is always Beckn-typed.

```
type Buyer = {
  bap_id: string,
  transaction_id: string,
  contact_snapshot?: ContactSnapshot,
}

type ContactSnapshot = {
  name: LocalizedText,
  email: string,        // captured from the BAP's /init payload
  phone: string,
  address: ShippingAddress,
}
```

`contact_snapshot` is `null` at `Created` (per v2's "no PII at /select" rule) and is populated at `Initiated` from the inbound `/init` payload.

### ShippingAddress (value object)

| Field | |
|---|---|
| `recipient_name` | `LocalizedText` |
| `line1`, `line2` | Strings |
| `city`, `region`, `country` | Strings (country = ISO 3166-1 alpha-2) |
| `postal_code` | String |
| `phone` | String |
| `notes` | Optional free-text |

## Relationships

```
Store (Tenancy) ────< Order ─────< embedded Quote ───< embedded LineItem ──→ Product/Variant (Catalog) by ID
                       │
                       ├── Buyer (Beckn — bap_id + transaction_id + optional contact snapshot)
                       ├── reservation_ids[] (Inventory.Reservation IDs)
                       └── voucher_usage_id  (Promotion.VoucherUsage ID, set at Confirmed)
```

Order references all other contexts by ID; never joins or imports their internals (§2.6).

## Order state machine

```
        [new]
          │
          ▼
       Created  ── TTL passes ──►  Expired  (terminal)
          │                            │ (releases reservation if held)
          │ Initiate (Beckn /init)
          ▼
       Initiated  ── TTL passes ──► Expired (terminal; releases reservation)
          │
          │ Confirm
          ▼
       Confirmed  ── MarkPreparing ──► (Confirmed + fulfillment_status=Preparing)
          │   │       MarkShipped  ──► (Confirmed + fulfillment_status=Shipped)
          │   │       MarkFulfilled ──► Fulfilled (terminal)
          │   │
          │   └─── Cancel ──► Cancelled (terminal)
          │                    (revert voucher; mark payment Refunded;
          │                     stock NOT returned — that's the deferred Return flow)
          │
          │ Cancel
          ▼
       Cancelled (terminal; reservation released; voucher unused)
```

**Transition table:**

| From → To | Actor | Guard |
|---|---|---|
| (new) → Created | BAP via Bridge (`/select`) | Store is Active; items are Active and in some Catalog; voucher (if any) usable |
| Created → Initiated | BAP via Bridge (`/init`) | Quote not expired; Inventory.Reserve succeeds; Voucher (if any) validates |
| Created → Expired | System (TTL) | `now >= valid_until` |
| Created → Cancelled | BAP via Bridge (`/cancel`) or Platform | (No reservations held yet — just terminal) |
| Initiated → Confirmed | BAP via Bridge (`/confirm`) | Quote not expired; Reservations still active |
| Initiated → Expired | System (TTL) | `now >= valid_until` (releases reservations) |
| Initiated → Cancelled | BAP via Bridge or Platform | Releases reservations |
| Confirmed → Cancelled | BAP via Bridge or Platform | Reverts voucher usage; marks payment Refunded |
| Confirmed → (Confirmed + Preparing) | Store Admin via Admin UI | capability `order.fulfillment.mark_preparing` |
| Confirmed → (Confirmed + Shipped) | Store Admin via Admin UI | capability `order.fulfillment.mark_shipped` |
| Confirmed → Fulfilled | Store Admin via Admin UI | capability `order.fulfillment.mark_fulfilled` |

Forbidden: any other transition. No Returned state in v1.

## Invariants

- `Order.store_id` is immutable.
- `Order.id` is immutable.
- `Quote` snapshots inside an Order are immutable once Created.
- `Order.buyer.type` is immutable.
- For Beckn buyers: `contact_snapshot` is null at Created, non-null at Initiated and after.
- `reservation_ids` is populated at Initiated; reservations are released on Expired/Cancelled-from-Initiated.
- `voucher_usage_id` is populated at Confirmed iff a Voucher was applied; reverted on Cancel-from-Confirmed.
- `currency` matches the Store's configured currency (ADR-0007).
- `Quote.valid_until > Quote.created_at`.

## Boundary contracts

### Use cases exposed (Application Layer)

**Beckn-driven (invoked by the Bridge from inbound BAP messages — the sole external entry point per [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md)):**
- `Order.CreateQuote(store_id, items, voucher_code?, beckn_buyer_ref) → Order`
- `Order.Initiate(order_id, contact_snapshot) → Order`
- `Order.Confirm(order_id) → Order`
- `Order.GetStatus(order_id) → Order`
- `Order.Cancel(order_id, reason, by_actor) → Order`

**Fulfillment (Store Admin via the Admin UI):**
- `Order.MarkPreparing(order_id, by_actor)`
- `Order.MarkShipped(order_id, by_actor, fulfillment_note?)`
- `Order.MarkFulfilled(order_id, by_actor)`

**Read-side (capability-gated, for the Admin UI):**
- `Order.ListForStore(store_id, filter, acting_user) → [Order]` — Store Admins and Org Owners see their store's orders.

All mutating use cases support optional `idempotency_key` per ADR-0013. All use cases call `AuthorizationPort.requireCapability(...)` per ADR-0016.

### Domain events emitted

| Event | When |
|---|---|
| `order.quote_created` | Order created at Quote stage; carries snapshot |
| `order.initiated` | Inventory reserved, voucher validated, buyer contact present |
| `order.confirmed` | Reservation converted, voucher recorded |
| `order.cancelled` | Pre-fulfillment cancel |
| `order.expired` | Quote TTL passed |
| `order.marked_preparing` | Store admin started preparation |
| `order.marked_shipped` | Store admin shipped |
| `order.fulfilled` | Store admin marked delivered |
| `order.payment_status_changed` | External payment signal updated status |

All events conform to the envelope from ADR-0011 (carry actor with impersonator, correlation_id, etc.). Audit ingests per ADR-0014. PII catalog (`design/pii.md`) covers fields with buyer details.

### Cross-context calls

Order calls (via Application-Layer ports per §5.17):
- `Catalog.ResolveItems(items[]) → ResolvedItems` — fetches Product/Variant info, validates Active state, captures LocalizedText
- `Inventory.Reserve(item_ref, quantity, held_by) → Reservation`
- `Inventory.ConvertReservation(reservation_id)`
- `Inventory.ReleaseReservation(reservation_id)`
- `Promotion.ValidateVoucher(code, store_id, user_id?, cart_total) → ValidatedDiscount`
- `Promotion.RecordVoucherUsage(voucher_id, user_id?, order_id, applied_amount) → VoucherUsage`
- `Promotion.RevertVoucherUsage(voucher_usage_id, reason)`
- `Authorization.requireCapability(...)` / `Authorization.hasCapability(...)`

Promotion gains a new use case `RevertVoucherUsage` (in support of cancellation) and a new event `promotion.voucher_usage_reverted` declared in [`design/events.md`](events.md).

## Cross-context orchestration sequences

### CreateQuote

```
Order.CreateQuote(store_id, items, voucher_code?, beckn_buyer_ref):
  require_capability(order.create_quote, scope=store_id)  // Bridge principal capability (invoked from /select)

  resolved = Catalog.ResolveItems(store_id, items)
  if any item not Active or not in any catalog: raise ItemUnavailable

  // Compute captured pricing per line
  for each item in resolved:
    line.captured_base_price = item.base_price
    line.captured_tax_rate = item.tax_rate
    line.captured_tax_amount = round(base_price × tax_rate / 100)
    line.captured_published_price = base_price + tax_amount
    line.captured_name = item.name  // LocalizedText snapshot
    line.captured_description = item.description
    line.line_subtotal = published_price × quantity

  totals.subtotal = sum(line.captured_base_price × quantity)
  totals.tax_total = sum(line.captured_tax_amount × quantity)
  totals.discount_total = 0
  totals.grand_total = totals.subtotal + totals.tax_total

  if voucher_code:
    discount = Promotion.ValidateVoucher(voucher_code, store_id, buyer.user_id?, totals)
    voucher_applied = { voucher_id, code, type, value, amount: discount }
    totals.discount_total = discount
    // Recompute taxes on discounted base per ADR-0007
    apply discount proportionally to line bases, recompute taxes, recompute totals

  order = new Order(
    store_id, status=Created,
    quote = { line_items: resolved, voucher_applied, totals, valid_until: now + 15min },
    buyer = buyer_ref,
    payment_status = Pending,
    fulfillment_status = Pending,
  )
  persist order; write outbox row: order.quote_created
  return order
```

### Initiate

```
Order.Initiate(order_id, contact_snapshot):
  require_capability(order.initiate, scope=order.store_id)
  load order; assert status == Created; assert now < quote.valid_until

  reservations = []
  for line in quote.line_items:
    r = Inventory.Reserve(line.item_ref, line.quantity, held_by=order_id)
    reservations.append(r)

  if quote.voucher_applied:
    try:
      Promotion.ValidateVoucher(voucher_code, store_id, buyer.user_id?, totals)
    except VoucherInvalid:
      for r in reservations: Inventory.ReleaseReservation(r.id)
      raise

  order.reservation_ids = [r.id for r in reservations]
  order.buyer.contact_snapshot = contact_snapshot
  order.status = Initiated
  persist; write outbox row: order.initiated
  return order
```

### Confirm

```
Order.Confirm(order_id):
  require_capability(order.confirm, scope=order.store_id)
  load order; assert status == Initiated; assert now < quote.valid_until

  for res_id in order.reservation_ids:
    Inventory.ConvertReservation(res_id)

  if quote.voucher_applied:
    usage = Promotion.RecordVoucherUsage(
      voucher_id, buyer.user_id?, order.id, quote.discount_total
    )
    order.voucher_usage_id = usage.id

  order.status = Confirmed
  order.confirmed_at = now
  persist; write outbox row: order.confirmed
  return order
```

### Cancel

```
Order.Cancel(order_id, reason, by_actor):
  require_capability(order.cancel, scope=order.store_id)
  load order; assert status in {Created, Initiated, Confirmed}

  match order.status:
    case Initiated:
      for res_id in order.reservation_ids: Inventory.ReleaseReservation(res_id)
    case Confirmed:
      if order.voucher_usage_id:
        Promotion.RevertVoucherUsage(order.voucher_usage_id, reason)
      if order.payment_status == Captured:
        order.payment_status = Refunded
        // actual refund is external; this is the BPP's record of intent

  order.status = Cancelled
  order.cancel_reason = reason
  order.cancelled_at = now
  persist; write outbox row: order.cancelled
  return order
```

## Beckn flow walkthrough

### `/select` → `on_select`

```
1. BAP POSTs /select with Contract (line items, optional voucher_code)
2. Bridge:
   - Validates Beckn envelope, signature
   - Extracts items, voucher_code, beckn_buyer_ref { bap_id, transaction_id }
   - Calls Order.CreateQuote(store_id, items, voucher_code?, beckn_buyer_ref) → order
   - Maps order → Contract (status=DRAFT, includes priced commitments)
   - Responds 200 Ack
3. Bridge asynchronously POSTs /on_select to BAP with the projected Contract
```

### `/init` → `on_init`

```
1. BAP POSTs /init with Contract carrying consumer details
2. Bridge:
   - Extracts contact_snapshot from Contract.participants[buyer]
   - Identifies order via transaction_id correlation
   - Calls Order.Initiate(order_id, contact_snapshot)
     - Reserves inventory across line items
     - Validates voucher (rejects if no longer valid)
   - Maps order → Contract (still DRAFT but with reservations, confirmed buyer)
   - Responds 200 Ack
3. Bridge asynchronously POSTs /on_init to BAP
```

### `/confirm` → `on_confirm`

```
1. BAP POSTs /confirm with Contract
2. Bridge:
   - Identifies order via transaction_id correlation
   - Calls Order.Confirm(order_id)
     - Converts reservations to sales
     - Records voucher usage
   - Maps order → Contract (status=ACTIVE)
   - Responds 200 Ack
3. Bridge asynchronously POSTs /on_confirm to BAP
```

### `/status` → `on_status`

```
1. BAP POSTs /status referencing transaction
2. Bridge:
   - Order.GetStatus(order_id)
   - Maps current state + fulfillment + payment to Contract
   - Responds 200 Ack
3. Bridge POSTs /on_status to BAP
```

### `/cancel` → `on_cancel`

```
1. BAP POSTs /cancel with reason
2. Bridge:
   - Order.Cancel(order_id, reason, by_actor=bridge_principal)
     - Releases reservation or reverts voucher per state
     - Marks payment Refunded if Captured
   - Maps order → Contract (status=CANCELLED)
   - Responds 200 Ack
3. Bridge POSTs /on_cancel to BAP
```

## Bridge ↔ Order mapping

Beckn v2 wire entities mapped to Order context:

| Wire (v2 `Contract`) | Domain (`Order`) | Notes |
|---|---|---|
| `Contract.id` | `Order.id` (deterministic transform — Bridge mapping registry) | Stable for lifetime |
| `Contract.status` ∈ {DRAFT, ACTIVE, CANCELLED, COMPLETE} | `Order.status` mapping | See ADR-0017 §2 |
| `Contract.commitments[]` | Quote line items (mapped to `Commitment` per line) | Carries Resource references and Offer details |
| `Contract.consideration[]` | `Quote.totals` + discount | Computed via mapping |
| `Contract.participants[buyer]` | `Order.buyer.contact_snapshot` | Populated at /init |
| `Contract.participants[seller]` | `Order.store` (projected as Provider) | From ADR-0001 |
| `Contract.performance[]` | `Order.fulfillment_status` + `fulfillment_note` | Maps to Performance entries |
| `Contract.settlements[]` | `Order.payment_status` | Settlement vocabulary |

Detailed mapping per protocol version lives in the Bridge's mapping registry (per ADR-0001 §4).

## Read-side queries

- `Order.GetById(order_id, acting_user)` — capability-gated; returns full Order or `NotAllowed`.
- `Order.ListForStore(store_id, filter, acting_user)` — store-scoped readers see their own store's orders only.
- `Order.ListForUser(user_id, filter, acting_user)` — users see their own orders; admins see all per capability.

Filter parameters typically include status, date range, payment status, fulfillment status. Pagination is per the implementing infrastructure.

## Audit considerations

Order events carry buyer details (PII) in `source_envelope`. Per ADR-0015 / `design/pii.md`:
- `order.initiated` carries `contact_snapshot` — scrubbable per right-to-erasure.
- `order.cancelled`, `order.confirmed`, etc. reference the buyer.

Audit retention bucket: **financial — ~7 years** (per ADR-0014 §6). Long retention reflects tax and dispute compliance needs.

## PII considerations

Per ADR-0015 / `design/pii.md`, the following Order fields are subject to scrubbing on right-to-erasure for the Beckn buyer (when erasure is requested by an identifiable BAP-side actor — process TBD):

- `Order.buyer.contact_snapshot.name` (LocalizedText) — scrub per `replace_name`
- `Order.buyer.contact_snapshot.email` — scrub per `replace_email`
- `Order.buyer.contact_snapshot.phone` — `nullify`
- `Order.buyer.contact_snapshot.address` — `nullify` whole object

The Order entity (id, store, line items, totals, statuses, timestamps) is retained — only the PII fields are scrubbed. This preserves order history for tax/audit while honoring erasure.

## Open questions (within this design)

- **Right-to-erasure for Beckn buyers** — when only `bap_id` and `transaction_id` are known, how does the platform receive an erasure request? Likely via the BAP, but the protocol mechanism is operational. Out of scope here.
- **Order modification (`/update` handler)** — when added, the state machine gains an "amended" transition. Future work.
- **Refund / Return workflow** — requires a post-Fulfilled state (e.g., `Returned`), inventory re-stocking decisions, and payment-refund coordination. Substantial future ADR.
- **Multi-shipment orders** — currently `fulfillment_status` is per-Order. Splitting would require a sub-entity (Shipment).
- **Subscription / recurring orders** — would require a new entity (Subscription) creating Orders on schedule.

## References

- [ADR-0017](../decisions/0017-order-and-fulfillment.md)
- [ADR-0001](../decisions/0001-bpp-network-identity.md), [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md), [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md), [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md), [ADR-0005](../decisions/0005-catalog-and-product-modeling.md), [ADR-0006](../decisions/0006-inventory-model.md), [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md), [ADR-0008](../decisions/0008-localization-and-localizedtext.md), [ADR-0009](../decisions/0009-identity-and-external-idp.md), [ADR-0010](../decisions/0010-invitation-account-reconciliation.md), [ADR-0011](../decisions/0011-domain-events.md), [ADR-0012](../decisions/0012-cross-context-consistency.md), [ADR-0013](../decisions/0013-first-party-idempotency.md), [ADR-0014](../decisions/0014-soft-delete-and-audit.md), [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md), [ADR-0016](../decisions/0016-authorization-details.md)
- [design/catalog.md](catalog.md), [design/identity.md](identity.md), [design/events.md](events.md), [design/audit.md](audit.md), [design/pii.md](pii.md)
- `ion-specs` v2.0.0 — `Contract`, `Resource`, `Offer`, `Performance`, `Settlement` schemas
- CLAUDE.md §2.5 (bounded contexts), §5.21 (this context)
