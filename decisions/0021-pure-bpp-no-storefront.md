# ADR-0021: Platform scope — pure BPP, no first-party storefront

- **Status**: Accepted
- **Date**: 2026-05-30
- **Supersedes**: [ADR-0017](0017-order-and-fulfillment.md) §4 (Buyer polymorphism) and §8 (first-party Order use cases including `Order.Place`); the implicit "buyers-as-Users" accommodation in [ADR-0009](0009-identity-and-external-idp.md).
- **Builds on**: [ADR-0001](0001-bpp-network-identity.md) (BPP network identity)

## Context

The original project brief framed this system as a **Beckn Provider Platform (BPP)** — an aggregator of small retailers exposing them to a Beckn network. Buyers were always expected to come via Buyer Apps (BAPs) over the network.

Across several subsequent ADRs, a **first-party storefront** concept crept in:

- [ADR-0009](0009-identity-and-external-idp.md) (Identity) supported User sign-in via IdP, with implicit accommodation for buyers as Users.
- [ADR-0017](0017-order-and-fulfillment.md) (Order) added a first-party Order entry point (`Order.Place`) alongside the Beckn flow, with a **polymorphic Buyer** (User-typed *or* Beckn-typed).
- Several design and handoff documents referenced a "Storefront" as a first-party UI adapter alongside the Admin UI.

This expansion conflated two distinct product shapes:

- **Pure BPP** — Beckn-only; the system has no direct contact with end buyers.
- **Hybrid BPP + B2C** — the platform also runs its own consumer-facing storefront, with buyers shopping directly.

This ADR confirms the platform is **pure BPP**. The hybrid model is out of scope.

## Decision

The platform is a **pure Beckn Provider Platform**:

- The BPP **has no direct contact with end buyers**.
- All buyer-facing transactions arrive via **BAPs over the Beckn network**.
- The **Beckn Bridge** is the sole protocol-aware boundary for buyer transactions.

### 1. No first-party storefront

The system does not host a consumer-facing shopping site. Stores' products are visible to buyers only through the Beckn network (via CDS-mediated discovery and BAP transactions, per [ADR-0001](0001-bpp-network-identity.md) and [ADR-0017](0017-order-and-fulfillment.md)).

The Interface Layer's first-party UI surface is exclusively the **Admin UI** — used by Org Owners, Org Members, Store Admins, and platform-side actors to manage their organizations, stores, catalogs, inventory, vouchers, and orders.

### 2. User entity scope (clarifies ADR-0009)

The `User` entity exists for **tenant-side and platform-side actors only**:

- Org Owners
- Org Members (including those assigned as Store Admins via `StoreAdminAssignment`)
- System Admins
- Platform-scoped operators

**Buyers are never `User`s in our system.** They are referenced per-order as Beckn buyers — `{ bap_id, transaction_id, contact_snapshot?: { ... } }` — populated by the Bridge from inbound Beckn messages.

### 3. `Order.buyer` simplified (supersedes ADR-0017 §4 polymorphism)

The Buyer attribute on `Order` is **always Beckn-typed**. The polymorphic User-typed variant introduced in ADR-0017 §4 is removed:

```
Order.buyer = {
  bap_id: string,
  transaction_id: string,
  contact_snapshot?: {
    name: LocalizedText,
    email: string,
    phone: string,
    address: ShippingAddress,
  }
}
```

`contact_snapshot` remains nullable at `Created` (per v2's "no PII at /select") and is populated at `Initiated` from the `/init` payload.

### 4. Order entry points (supersedes ADR-0017 §8 first-party use cases)

The Bridge is the **sole entry** for Order use cases. The first-party `Order.Place` use case is removed.

Active Order use cases (all invoked by the Bridge from inbound Beckn messages):

- `Order.CreateQuote(store_id, items, voucher_code?, beckn_buyer_ref)` — from `/select`
- `Order.Initiate(order_id, contact_snapshot)` — from `/init`
- `Order.Confirm(order_id)` — from `/confirm`
- `Order.GetStatus(order_id)` — from `/status`
- `Order.Cancel(order_id, reason, by_actor)` — from `/cancel`

Plus store-side fulfillment use cases (`Order.MarkPreparing`, `Order.MarkShipped`, `Order.MarkFulfilled`) invoked by Store Admins via the Admin UI.

### 5. Interface Layer surfaces (clarification)

The BPP exposes three Interface Layer adapter families. **None is a buyer-facing storefront.**

- **Admin UI** (first-party) — for Org Owners, Org Members, Store Admins to manage organizations, stores, catalogs, inventory, vouchers, orders. Includes platform-scoped operator and System Admin consoles.
- **Internal APIs** (first-party) — consumed by our own front-ends (e.g., the Admin UI's API layer).
- **Beckn Bridge** — the sole external surface for buyer transactions; mediated by BAPs over the Beckn network.

### 6. Glossary clarification

The term **"Storefront"** is not used in the platform. The first-party UI is **Admin UI**. The buyer-facing surface is the Beckn network.

References to "storefront read-side projections" in earlier ADRs and design notes refer to the broader concept of event-driven materialized views — those projections, if introduced, would serve the Admin UI, not a buyer site.

## Consequences

What this commits to:

- **No B2C consumer site.** The platform is pure infrastructure for Beckn participation.
- IdP integration (per [ADR-0009](0009-identity-and-external-idp.md)) is for admin/platform actors only — buyers do not sign in.
- The Order context has **one external entry point**: the Bridge.
- `Order.buyer` simplifies to a single shape (Beckn-typed).
- The `Order.Place` first-party use case from ADR-0017 is removed.
- design/order.md, design/identity.md, design/catalog.md, design/events.md, handoff/*, CLAUDE.md updated to remove storefront concepts.
- ADRs that referenced storefronts as a future or current adapter stand as historical record; this ADR supersedes those references for the active model.

What this defers:

- **A B2C storefront** is explicitly out of scope. If the platform ever adopts a hybrid model (B2C + Beckn), it would require a new ADR superseding this one — reintroducing User-typed buyers, polymorphic `Order.buyer`, first-party Order entry, and a Storefront adapter family.

What this makes harder:

- **A retailer who wants both Beckn presence and a custom storefront** must integrate the storefront via a separate channel (their own storefront product, e.g., Shopify; not this platform).
- **Admin-facing read-side projections** can still be built but are framed as Admin-UI projections, not storefront projections.

## References

- [ADR-0001](0001-bpp-network-identity.md) — BPP network identity (this ADR clarifies the scope implied by 0001)
- [ADR-0009](0009-identity-and-external-idp.md) — User entity (scope clarified here)
- [ADR-0017](0017-order-and-fulfillment.md) — Order context (§4 and §8 superseded)
- `design/order.md`, `design/identity.md`, `design/catalog.md`, `design/events.md` — updated
- `handoff/01-overview.md`, `handoff/03-beckn-integration.md` — updated
- CLAUDE.md — updated
