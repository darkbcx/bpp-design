# 4. Bounded contexts in detail

This section presents the seven bounded contexts that make up the platform's domain. Each context owns its concepts, data, lifecycles, and rules; cross-context references are by identifier only (per [§2.4](../02-principles.md)).

## How to read this section

If you're working on a specific context, start with that context's file. If you're new to the system, read in the order below — each context builds on the earlier ones.

Each context file is a **reading view** — concise concept summaries, lifecycle highlights, and the boundary contracts the team will build against. For full attribute schemas and use case lists, the file points to the matching `design/*` sub-design document.

## The seven contexts

| # | Context | Owns | First read if working on… |
|---|---|---|---|
| 4.1 | [Identity & Access](4.1-identity.md) | User identity, Sessions | Authentication; session management; the active Org / Store mechanic |
| 4.2 | [Tenancy](4.2-tenancy.md) | Organizations, Stores, OrganizationMembers, OrganizationInvitations, StoreAdminAssignments | Multi-tenancy; Org / Store lifecycle; admin assignment; permissions |
| 4.3 | [Catalog](4.3-catalog.md) | Products, Variants, Catalogs (Default + additional), PlatformCategories, Media | Product modeling; catalog structure; what gets published |
| 4.4 | [Inventory](4.4-inventory.md) | StockLevels, Reservations, StockMovements | Stock tracking; reservations; the buy/decrement flow |
| 4.5 | [Promotion](4.5-promotion.md) | Vouchers, VoucherUsages | Discounts; voucher application and rollback |
| 4.6 | [Order & Fulfillment](4.6-order.md) | Orders, Quotes, line items, payment status, fulfillment status | The commerce track; Beckn transactional flow; order state machine |
| 4.7 | [Audit](4.7-audit.md) | AuditRecords (append-only history of all state changes) | Compliance; admin-side history; right-to-erasure support |

## Cross-context dependencies at a glance

```
                    Identity & Access  ◄──────────────┐
                          │                            │ (User.id only)
                          ▼                            │
                    Tenancy ◄────────────┐             │
                          │              │ (Store.id)  │
                          ▼              │             │
                    Catalog ◄────────────┼─── Inventory  (Product/Variant ID)
                                         │
                                         │
                    Promotion ◄──────────┤  (Voucher.id; Store.id)
                                         │
                                         │
                    Order & Fulfillment ─┴───── Catalog (line items)
                                          ───── Inventory (Reserve / Convert / Release)
                                          ───── Promotion (Validate / Record / Revert)
                                          ───── Identity (User.id for store admins)

                    Audit  ◄── all of the above (event subscriber; no direct references)
```

All cross-context references are **by identifier only** — never joins, never imports of internal types (per [§2.4](../02-principles.md) and [§5.3](../05-cross-cutting.md), forthcoming).

## What's not a context

Two things deliberately not in this section:

- **The Beckn Bridge** — it's an Interface Layer adapter, not a bounded context. See [§3](../03-beckn-integration.md).
- **The Catalog Discovery Service (CDS)** — external infrastructure, not part of our system. See [§3.3](../03-beckn-integration.md).

## Source pointers

Each context file references:
- The relevant **ADRs** in [`/decisions/`](../../decisions/) — historical decisions with options weighed.
- The full **sub-design doc** in [`/design/`](../../design/) (where one exists) — full entity model, schemas, use case lists, sequence walkthroughs.

The handoff context file is the entry point; `design/*` is for deep dives.

## Reading order for a first pass

If you're seeing the system fresh, read in order: 4.1 → 4.2 → 4.3 → 4.4 → 4.5 → 4.6 → 4.7. Each builds on the previous; the chain mirrors how a customer order actually flows through the system.
