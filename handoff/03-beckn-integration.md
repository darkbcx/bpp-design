# 3. Beckn integration — the Bridge

This section describes how the platform connects to the Beckn network. The integration is the responsibility of a single component — the **Beckn Bridge** — and is governed by strict isolation rules so the domain stays Beckn-naive (per [§2.6](02-principles.md)).

> Source: [`/CLAUDE.md`](../CLAUDE.md) §4. Decisions: [ADR-0001](../decisions/0001-bpp-network-identity.md) (network identity), [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md) (catalog publishing), [ADR-0017](../decisions/0017-order-and-fulfillment.md) (transactional flow + wire vocabulary).

## 3.1 What the Bridge is (and isn't)

The Bridge is a **specialized adapter in the Interface Layer** (per [§2.2](02-principles.md)). It is **not** a bounded context — it has no domain. It is **not** a layer of its own — it sits alongside the storefront and admin UI as another adapter, just one with very specific constraints.

| Role | What it does |
|---|---|
| Bridge **is** an adapter | Translates between the Beckn wire format and Application-Layer calls in both directions |
| Bridge **calls** Application use cases | Never substitutes for them, bypasses them, or duplicates their work |
| Bridge **is the sole protocol-aware zone** | The only place Beckn schema, vocabulary, signing, registry lookups, callback URLs, and message envelopes live |
| Bridge **is event-driven** | Subscribes to domain events (per [§5.2](05-cross-cutting.md), forthcoming) and re-projects on relevant state changes |

**Things the Bridge does not do** (these are defects if found):

- Encode business rules ("an item is available if stock > 0" is a domain rule).
- Read from or write to domain stores directly.
- Leak Beckn vocabulary upward — nothing in Application, Domain, or first-party interfaces imports from the Bridge.
- Call other adapters (no calls to the storefront or admin UI from inside the Bridge).
- Rewrite domain semantics to fit the protocol.

No other adapter imports anything from the Bridge. The Application Layer is **unaware** that the Bridge exists — it treats Bridge-originated calls identically to first-party-originated calls.

## 3.2 Network identity — the platform as one BPP

The platform appears on the Beckn network as **one BPP**, identified by a single `bpp-id` and `bpp-uri` ([ADR-0001](../decisions/0001-bpp-network-identity.md)). Stores project onto the network as Beckn `provider` nodes inside that BPP's responses.

| Concern | Resolution |
|---|---|
| BPP cardinality | One BPP for the whole platform (not one BPP per store) |
| Provider ID derivation | Deterministic from the internal store ID; immutable for the store's lifetime |
| Signing key | One platform-level key for all outbound messages |
| Callback endpoint | One platform-wide endpoint; the Bridge routes inbound messages to the originating store using protocol-level identifiers (transaction_id, message_id, provider reference) — never by URL shape |
| Reputation | Platform-owned. Per-store reputation visibility is deferred |
| Registry registration | Operational input. `bpp-id`, `bpp-uri`, signing key material, registry credentials are supplied as configuration. The Bridge does not own the registry lifecycle |

**Consequence**: a store's Beckn identity is bound to its store identity for life. "Resetting" Beckn presence requires creating a new store. Quality control at store onboarding is a platform responsibility because all stores share the BPP's reputation.

## 3.3 Three-actor architecture — CDS-mediated discovery

Beckn v2 separates discovery from transaction. Three actors interact:

```
                     ┌──────────────────────┐
                     │  Catalog Discovery   │
                     │   Service (CDS)      │
                     └─┬──────────────┬─────┘
                       │              ▲
        /on_discover ──┤              ├── /catalog/publish
                       ▼              │
                ┌───────────┐    ┌─────────────┐
                │    BAP    │    │   BPP (us)  │
                │  (buyer-  │───►│             │
                │   side)   │    │             │
                └───────────┘    └─────────────┘
                           /select  /init  /confirm
                           /status  /cancel
```

- **BAP** (Buyer App): the buyer-side participant. External to us.
- **CDS** (Catalog Discovery Service): the network's catalog index. External to us.
- **BPP** (us): publishes catalogs to CDS; handles the transactional flow directly with BAP.

**Critically: the BPP does not field `/discover`.** Discovery is the CDS's responsibility. The BPP's job is to **publish** its stores' catalogs to the CDS (via `/catalog/publish`), and the CDS handles BAP discover queries.

The transactional flow (`/select`, `/init`, `/confirm`, etc.) is between BAP and BPP directly.

## 3.4 Wire vocabulary (Beckn v2)

Key v2 entities the Bridge maps to and from:

| Wire entity | Description |
|---|---|
| `Catalog` | Top-level container the BPP publishes. Has one `Provider`, resources, offers, validity, isActive. One per store-catalog combination. |
| `Provider` | Represents one of our stores on the network. Minimal — just id, descriptor, locations, attributes. **No status field**. |
| `Resource` | A discoverable / referenceable unit of value. Replaces v1's `Item`. Our Products and Variants project here. |
| `Offer` | Pricing, terms, availability of one or more Resources. Separate from Resource (matching our Catalog ↔ Inventory/Pricing split). |
| `Contract` | Generalized transaction object. Replaces v1's `Order`. Has commitments, consideration, participants, performance, settlements. Statuses: `DRAFT` / `ACTIVE` / `CANCELLED` / `COMPLETE`. |
| `Commitment` | One element of a Contract — what's agreed (e.g., a line item). |
| `Consideration` | Value exchanged under a Contract (price, tax, discount). |
| `Performance` | How execution happens — fulfillment representation. |
| `Settlement` | How consideration is discharged — payment representation. |
| `Participant` | A party to a Contract (buyer, seller, agent). |
| `Intent` | What a BAP is looking for (used in `/discover`; not handled by us). |

**Domain stays naive of all these names.** They live in the Bridge's mapping registry (§3.7).

## 3.5 Handler set in v1

What the BPP exposes and consumes in v1:

| Direction | Handler | Purpose |
|---|---|---|
| BPP → CDS | `POST /catalog/publish` | Publish or republish a store's catalogs |
| CDS → BPP | `POST /catalog/on_publish` | Async per-catalog processing result |
| BAP → BPP | `POST /select` | BAP indicates items; BPP returns a quoted `Contract` |
| BAP → BPP | `POST /init` | BAP provides buyer details; BPP reserves inventory |
| BAP → BPP | `POST /confirm` | BAP commits; BPP converts reservation, records voucher use |
| BAP → BPP | `POST /status` | BAP queries order status |
| BAP → BPP | `POST /cancel` | BAP requests cancellation (pre-fulfillment) |
| BPP → BAP | `POST /on_select`, `/on_init`, `/on_confirm`, `/on_status`, `/on_cancel` | Async callbacks projecting the Contract back to the BAP |

**Deferred for v1**:

- `/track`, `/update`, `/rate`, `/support` — transactional handlers (future).
- `/catalog/pull`, `/catalog/subscription`, master-catalog APIs — CDS-territory (out of scope for the BPP).
- ION extensions `/raise`, `/reconcile` — grievance / settlement (out of scope for v1).

## 3.6 Bridge ↔ Order interface (transactional flow)

Each inbound wire message is translated into a domain call. The Bridge owns protocol concerns (signature verification, schema validation, version negotiation, correlation); the Order context (see [§4.6](04-bounded-contexts/4.6-order.md), forthcoming) handles domain semantics.

| Wire | Domain call (Bridge invokes) |
|---|---|
| `/select` (Contract with items + optional voucher) | `Order.CreateQuote(store_id, items, voucher_code?, beckn_buyer_ref)` |
| `/init` (Contract with buyer details) | `Order.Initiate(order_id, contact_snapshot)` |
| `/confirm` (Contract) | `Order.Confirm(order_id)` |
| `/status` | `Order.GetStatus(order_id)` |
| `/cancel` (with reason) | `Order.Cancel(order_id, reason, by_actor=bridge_principal)` |

The Order returned from each domain call is then projected to a wire `Contract` and sent back to the BAP via the corresponding `/on_*` callback.

See [ADR-0017](../decisions/0017-order-and-fulfillment.md) and [`design/order.md`](../design/order.md) for the full Order context model.

## 3.7 Mapping discipline — the mapping registry

The Bridge maintains a **mapping registry** — a first-class artifact, not transformation code buried in handlers. Every domain ↔ wire mapping must be:

- **Explicit** — declared as a named, documented entry, not inferred from naming conventions or reflection.
- **Testable in isolation** — exercisable with sample payloads from `ion-specs`.
- **Replaceable** — a new mapping (e.g., for a protocol-version bump) can be added without rewriting other mappings or the domain.
- **Asymmetric-tolerant** — some Beckn fields have no domain origin (protocol metadata, constants); some domain fields have no Beckn destination (internal concerns). Both are healthy.
- **Defensive about the unknown** — unrecognized inbound Beckn fields are logged and ignored, not propagated. Domain projections do not invent Beckn fields the schema doesn't define.

### Key mappings (overview; details in the registry itself)

| Domain entity | Wire projection |
|---|---|
| Store (Active) | `Provider` inside a `Catalog` published to CDS |
| Store (Paused, Suspended) | `Catalog` published with `isActive: false` (the wire does not distinguish owner-pause from platform-suspend) |
| Store (Draft) | Not published |
| Internal catalog (Default + additional) | Distinct `Catalog` objects per store-catalog (each carrying the same Provider info) |
| Product / Variant | `Resource` inside a Catalog |
| Pricing / availability | `Offer` linked to the Resource |
| PlatformCategory taxonomy | Beckn `category` references on resources |
| Order | `Contract` |
| Order status (Created / Initiated → DRAFT; Confirmed → ACTIVE; Fulfilled → COMPLETE; Cancelled / Expired → CANCELLED) | `Contract.status` |
| Order line items | `Commitment[]` + `Consideration[]` |
| Order fulfillment_status | `Performance[]` |
| Order payment_status | `Settlement[]` |
| Buyer contact snapshot (Beckn) | `Participant` (buyer) |

## 3.8 Catalog publishing (BPP → CDS)

The Bridge publishes catalog state to the CDS via `/catalog/publish` on every domain event that changes what should appear on the network:

| Triggering domain event | Bridge action |
|---|---|
| `tenancy.store_status_changed` (Active ↔ Paused ↔ Suspended) | Republish the store's catalogs with updated `isActive` |
| `tenancy.store_status_changed` (… → Active for the first time) | Initial publish of the store's catalogs |
| `catalog.*` (Product / Variant / Category / Catalog mutations) | Republish affected catalog(s) |
| `promotion.voucher_*` (if Beckn projection includes vouchers) | Republish affected catalog(s) |
| `tenancy.store_republish_requested` (manual; see [ADR-0019](../decisions/0019-manual-catalog-republication.md)) | Republish the named store's catalogs |
| `tenancy.organization_republish_requested` (manual bulk; ADR-0019) | Fan out to all stores in the Org |

**Manual republication** is supported alongside event-driven publication. Store Admins or Org Owners may trigger `Store.RequestRepublish(store_id)` for a single store; Org Owners may trigger `Org.RequestRepublishAll(org_id)` for all their stores. These use cases emit dedicated events that the Bridge subscribes to identically to auto-trigger events — same re-projection code path, same idempotency guarantees. Useful for confidence-recovery ("force my store to republish") and bulk operations after a CDS outage. Rate limiting is enforced at the API layer.

Re-projection is **idempotent**: re-publishing the same state is harmless. This is essential because the event delivery layer (per ADR-0011) is at-least-once.

The CDS responds asynchronously with `/catalog/on_publish` containing per-catalog processing results. The Bridge handles this callback — recording acknowledgements and surfacing publish failures for ops review (consistent with the failure-handling rule from [§2.5](02-principles.md)).

**Why push, not pull?** ADR-0001 + ADR-0004 + ADR-0017 settled on push-publish to CDS as the v2 mechanism. The BPP does not field `/discover` directly; that's the CDS's role. Pull mode (`/catalog/pull`) and subscription mode are CDS-territory APIs out of v1 scope for the BPP.

## 3.9 Asynchronous flow and correlation

Beckn is fundamentally **callback-driven**:

```
BAP                            BPP
 │                              │
 │── POST /select ─────────────►│
 │◄──── 200 ACK ────────────────│   (sync ack — "message received")
 │                              │
 │                              │── (process, persist Order, ...)
 │                              │
 │◄─── POST /on_select ─────────│   (async callback — the actual response)
 │──── 200 ACK ────────────────►│
```

The Bridge owns the asynchronous concerns:

- **Correlation**: mapping inbound responses back to the original `transaction_id` and `message_id`.
- **Protocol-level idempotency**: recognizing duplicate inbound messages by their protocol identifiers and short-circuiting before they reach the Application Layer.
- **Transactional context**: tracking what state a multi-step Beckn flow is in (search → select → init → confirm) from the protocol's perspective.

The Application Layer is unaware of message IDs, transaction IDs, callback URLs, or acknowledgement semantics. From its perspective, every interaction is a discrete use-case invocation that returns a domain result. The Bridge translates that synchronous-looking call into the asynchronous protocol dance.

## 3.10 Versioning strategy

The Beckn protocol evolves. Multiple versions may be in flight on the network simultaneously; not all participants upgrade in lockstep. The Bridge handles versioning:

- Each supported protocol version has its **own translation set** within the Bridge. Versions coexist; they don't replace each other silently.
- The protocol version is **negotiated per inbound message** via the `context` block.
- A **protocol upgrade is a Bridge-only change**. If a Beckn version bump forces a domain change, that's evidence of leakage — investigate before accepting.
- Deprecating a protocol version is **explicit and announced**, not an accident of refactoring.

## 3.11 Error handling across the boundary

Errors cross the Bridge in both directions, and in both directions they are **translated**, not forwarded.

| Direction | Translation |
|---|---|
| Domain → Beckn | Domain errors (e.g., "store not found", "product unavailable", "voucher expired") are mapped to appropriate Beckn error codes and shapes. The Application Layer never emits a Beckn error code. |
| Beckn → Domain | Protocol-level errors (schema invalid, signature failed, unknown action, malformed envelope) are handled **inside the Bridge** and never surfaced to the Application Layer as domain errors. Only validated, well-formed, business-meaningful requests reach the Application. |

**Unmappable cases are explicit**: if a domain error has no clean Beckn equivalent, the Bridge maps it to a generic protocol error *and* logs the mismatch. Silent information loss is forbidden.

## 3.12 Event-driven Bridge behavior summary

To make the Bridge's behavior concrete:

- **The Bridge does not emit domain events.** It consumes them (for republication) and emits Beckn messages (which are not domain events).
- The Bridge's subscriber filter includes `tenancy.store_status_changed`, all `catalog.*` mutations, and selected `promotion.*` events if their state affects Beckn projections.
- Each subscribed event triggers a re-projection: the Bridge fetches the current relevant state from the Application Layer and publishes the updated catalog or Contract to the appropriate downstream actor (CDS or BAP).
- Re-projection is idempotent by design — re-running it produces the same wire state, so duplicate event deliveries are harmless.

## 3.13 Testing strategy

| What | How |
|---|---|
| **Domain and Application Layers** | Tested with **no Beckn fixtures**. If a domain test needs a Beckn payload to make sense, the test is wrong. |
| **The Bridge** | Tested with **real Beckn v2 payloads** from `ion-specs` examples, verifying both inbound parsing and outbound projection for each supported protocol version. |
| **End-to-end Beckn flows** | Tested as **integration tests** that exercise the Bridge against a stubbed network counterpart (mock BAP, mock CDS), confirming async correlation, error mapping, and version negotiation. |
| **Mapping registry** | Each mapping entry is exercisable in isolation with paired sample inputs and expected outputs. |

The presence of Beckn fixtures in any test outside the Bridge's own test suite is a red flag worth investigating.

## 3.14 What's explicitly deferred

For clarity to anyone reading later:

- `/track`, `/update`, `/rate`, `/support` handlers — future v1.x or v2.
- ION-specific extensions `/raise`, `/reconcile` (grievance + settlement) — future, when compliance demands.
- Subscription / pull-mode catalog APIs (`/catalog/subscription`, `/catalog/pull`) — CDS-territory; the BPP is the publisher.
- Per-store cryptographic identity (separate signing keys per store) — would supersede ADR-0001.
- Per-store reputation visibility back to store owners — operational / UI concern.
- Cryptographic chaining of audit records (tamper-evidence beyond append-only roles) — only if compliance demands.

> **Next**: [§4 Bounded contexts in detail](04-bounded-contexts/README.md) — per-context concepts, lifecycles, contracts.
