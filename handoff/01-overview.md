# 1. Overview

## 1.1 What this platform is

This is a **Beckn Provider Platform (BPP)** — a multi-tenant marketplace that aggregates small retailers and exposes them collectively to a Beckn network. The platform appears on the network as a **single BPP**; retailers operate **stores** within the platform, grouped under platform-internal **Organizations**. Each store projects onto the network as a Beckn `provider`; Organizations are invisible to Beckn.

- **Target audience**: small retailers (Indonesia-focused at v1; the architecture is region-agnostic).
- **Protocol**: Beckn v2 / ION, with discovery mediated by a **Catalog Discovery Service (CDS)**.
- **Authentication**: outsourced to an external **Identity Provider (IdP)** via OIDC.
- **Payment**: external to the BPP in v1 (BAP-side / network-mediated).
- **Fulfillment**: self-fulfilled by each store in v1.

> See: [`/CLAUDE.md`](../CLAUDE.md) §2.1 for the canonical framing.

## 1.2 Goals (what this system does)

- Lets users create **Organizations** — platform-internal containers that group one or more stores under shared ownership and membership.
- Lets the **Org Owner** invite users as **Org Members** and assign them as **Store Admins** of specific stores within the Org.
- Lets users create and operate **stores** (within an Organization) via a first-party admin UI.
- Lets each store **catalog** products with multi-language descriptions, tax-aware pricing, variant or flat SKU models, and multiple internal catalogs.
- Lets stores **manage inventory** with stock counts and timed reservations.
- Lets stores **create vouchers** for promotional discounts.
- Exposes each Active store's catalogs to the Beckn network via the **CDS** (so BAPs can discover and transact).
- Handles the Beckn **transactional flow** (`select`, `init`, `confirm`, `status`, `cancel`) for buyer-initiated orders.
- Maintains a tamper-evident **audit log** of all state changes.
- Honors **right-to-erasure** for personal data while preserving business history.

## 1.3 Non-goals (what this system does NOT do in v1)

- **Run a buyer-facing storefront.** The platform is a **pure BPP** (per [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md)) — buyers always come via BAPs over the Beckn network. The only first-party UI is the Admin UI for store and platform actors.
- **Process payments.** No PCI scope. Payment is external; the BPP receives status signals only.
- **Provide logistics.** Stores fulfill their own orders. No courier integration.
- **Run a CDS.** The BPP publishes catalogs to a CDS — it does not operate one.
- **Field `/discover` queries** from BAPs directly. The CDS does that.
- **Support refunds or returns** beyond pre-fulfillment cancel. Post-fulfillment return workflow is deferred.
- **Implement the full Beckn handler set.** `/track`, `/update`, `/rate`, `/support`, and ION's `/raise` and `/reconcile` are deferred.
- **Manage user passwords or MFA.** The IdP does that.

> See: ADR-0001 (one BPP / CDS-mediated discovery), ADR-0009 (external IdP), ADR-0017 (Order scope and deferrals).

## 1.4 Architecture at a glance

```
            ┌──────────────────────────────────────────────────────────────┐
            │              External actors and infrastructure              │
            ├──────────────────────────────────────────────────────────────┤
            │  ┌──────┐  ┌─────────────────┐  ┌──────────┐  ┌──────────┐   │
            │  │ BAP  │  │ Beckn registry  │  │   CDS    │  │   IdP    │   │
            │  └───┬──┘  └────────┬────────┘  └─────┬────┘  └────┬─────┘   │
            └──────┼──────────────┼─────────────────┼────────────┼─────────┘
                   │              │                 │            │
                   ▼              ▼                 ▼            ▼
            ┌──────────────────────────────────────────────────────────────┐
            │                  Interface Layer (BPP)                       │
            │  ┌──────────────────┐  ┌────────────────────────────────┐    │
            │  │   Admin UI +     │  │  Beckn Bridge                  │    │
            │  │  Internal APIs   │  │  (mapping registry; sole       │    │
            │  │  (first-party)   │  │  protocol-aware adapter; the   │    │
            │  │                  │  │  only buyer-facing surface)    │    │
            │  └─────────┬────────┘  └────────────────┬───────────────┘    │
            └────────────┼────────────────────────────┼────────────────────┘
                         │                            │
                         └────────────┬───────────────┘
                                      ▼
            ┌──────────────────────────────────────────────────────────────┐
            │                    Application Layer                         │
            │                  (use cases per context)                     │
            └──────────────────────────────┬───────────────────────────────┘
                                           ▼
            ┌──────────────────────────────────────────────────────────────┐
            │                       Domain Layer                           │
            │                                                              │
            │   Identity   Tenancy   Catalog   Inventory   Promotion       │
            │   & Access                                                   │
            │                                                              │
            │   Order & Fulfillment        Audit                           │
            └──────────────────────────────────────────────────────────────┘
                                           │
                                           ▼  (ports)
            ┌──────────────────────────────────────────────────────────────┐
            │                  Infrastructure Layer                        │
            │   Persistence · Event dispatcher · OIDC client · Email · …   │
            └──────────────────────────────────────────────────────────────┘
```

Key points:

- **Seven bounded contexts** in the Domain Layer; each owns its data and rules.
- **The Beckn Bridge** is a specialized adapter in the Interface Layer — the only place where Beckn protocol vocabulary lives.
- **CDS, IdP, Beckn registry, BAPs** are external actors; the platform integrates via standard interfaces.
- **Dependency Rule**: dependencies point inward. The Domain depends on nothing.

> Full details: [§2 Architectural principles](02-principles.md), [§3 Beckn integration](03-beckn-integration.md).

## 1.5 Glossary

Terms used throughout this document.

| Term | Definition |
|---|---|
| **BPP** | Beckn Provider Platform — this system. A single platform-wide network identity hosting many stores. |
| **BAP** | Beckn App Platform — buyer-side network participant. External to the BPP. |
| **CDS** | Catalog Discovery Service — the network layer that holds published catalogs and serves BAP discovery queries. External to the BPP. |
| **Beckn** | Open commerce protocol. The BPP targets v2. |
| **ION** | A specialization / extension of Beckn v2. The protocol variant this platform implements. |
| **Bridge** | The specialized adapter in the BPP that owns all Beckn protocol concerns. The only place protocol vocabulary appears. |
| **IdP** | Identity Provider — external authentication service. OIDC-compliant (Clerk, Auth0, Supabase Auth, Cognito, Keycloak, …). |
| **Organization** | The platform-internal top-level tenant. Groups one or more Stores under shared ownership. Invisible to Beckn. |
| **Org Owner** | The User with `Owner` role in an Organization (exactly one per Org). Plenary authority over the Org and all stores within. |
| **Org Member** | A User with `Member` role in an Organization, joined via Org-level invitation. Has only `org.view` by default. |
| **Store Admin** | An Org Member with administrative access to one or more specific stores within the Org via `StoreAdminAssignment`. Direct assignment by Org Owner — no invitation. |
| **StoreAdminAssignment** | The link granting an Org Member Store Admin role on a specific store. Distinct from Org membership. |
| **Store** | A tenant *within* an Organization. Has zero or more Store Admins (assigned by the Org Owner) and one or more Catalogs. Projects to Beckn as a `provider`. |
| **Catalog (domain)** | A named collection of Products within a store. Each store has one Default Catalog + zero or more additional Catalogs. |
| **Catalog (wire)** | The Beckn v2 `Catalog` entity. One per (store, internal-catalog) combination. Has one `Provider`. |
| **Provider (wire)** | The Beckn v2 `Provider` entity — represents one of our stores on the network. |
| **Product** | An item for sale within a store. In one of four modes: `Standalone` (single SKU), `Variant` (attribute-driven variants with own SKUs), `Configurable` (build-your-own with choice groups), `Composite` (bundle of multiple SKUs). Per [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md). |
| **Resource (wire)** | The Beckn v2 `Resource` entity — Products and Variants project to this. |
| **Offer (wire)** | The Beckn v2 `Offer` entity — pricing / availability of a Resource. |
| **Contract (wire)** | The Beckn v2 `Contract` entity — generalized transaction object. Orders project to this. |
| **Order (domain)** | The unit of buyer commitment. State machine: Created → Initiated → Confirmed → Fulfilled, with Cancelled / Expired as alternative terminals. |
| **Quote** | The captured-snapshot pricing within an Order at its `Created` state. TTL: 15 minutes default. |
| **Reservation** | A timed hold on inventory created at `Order.Initiated`. Converted to a sale at `Order.Confirmed`. |
| **Voucher** | A code-based discount; redeemed at `Order.Confirmed`; reversible on cancel. |
| **Active Org** | Session attribute identifying which Organization the user is currently operating within. Required for tenant-scoped actions. |
| **Active Store** | Session attribute identifying a specific store within the Active Org. Required for store-specific actions; optional for Org-level work. |
| **LocalizedText** | Value object — a string with translations. Platform default locale: `id` (Bahasa Indonesia). |
| **Money** | Value object: integer amount in minor units + ISO 4217 currency code. |
| **Capability** | Action-level permission (`product.publish`, `voucher.disable`, …). |
| **Outbox** | Per-context table where domain events are written transactionally with state changes. |
| **Inbox** | Per-subscriber table tracking processed events for dedup. |
| **AuditRecord** | Persistent record of a consumed event, with longer retention than the event log. |
| **Bounded context** | A region of the system with its own model, language, data, and rules. The system has seven. |
