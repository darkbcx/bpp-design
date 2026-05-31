# 3. Beckn integration — the Bridge

This section describes how the platform connects to the Beckn network. The integration is the responsibility of a single component — the **Beckn Bridge** — and is governed by strict isolation rules so the domain stays Beckn-naive (per [§2.6](02-principles.md)).

> Source: [`/CLAUDE.md`](../CLAUDE.md) §4. Decisions: [ADR-0001](../decisions/0001-bpp-network-identity.md) (network identity — *signing / registry parts superseded by ADR-0022*), [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md) (catalog publishing), [ADR-0017](../decisions/0017-order-and-fulfillment.md) (transactional flow + wire vocabulary — *direct-CDS-publish parts superseded by ADR-0022*), **[ADR-0022](../decisions/0022-onix-protocol-gateway.md) (ONIX protocol gateway — current architecture)**.

## 3.1 What the Bridge is (and isn't)

The Bridge is a **specialized adapter in the Interface Layer** (per [§2.2](02-principles.md)). It is **not** a bounded context — it has no domain. It is **not** a layer of its own — it sits alongside the Admin UI as another adapter (the platform is a pure BPP per [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md); the Bridge is the sole buyer-facing surface).

Per [ADR-0022](../decisions/0022-onix-protocol-gateway.md), the network is intermediated by **ONIX** — a vendor-provided Beckn Protocol Server deployed per-BPP. ONIX handles cryptographic signing, network-side schema validation, Beckn registry interactions, and CDS publishing. The Bridge keeps everything else: domain ↔ wire mapping, inbound signature re-verification (for CounterSignature attestation), ION-XXXX error mapping, Ack / AckNoCallback / Nack construction, correlation, and protocol-boundary idempotency.

| Role | What it does |
|---|---|
| Bridge **is** an adapter | Translates between the Beckn wire format and Application-Layer calls in both directions |
| Bridge **calls** Application use cases | Never substitutes for them, bypasses them, or duplicates their work |
| Bridge **is the sole Beckn-vocabulary zone in the BPP** | The only place Beckn schema, vocabulary, mapping, correlation, and Ack/Nack construction live within the BPP. Cryptographic signing + registry / CDS / BAP-callback network HTTP move to ONIX. |
| Bridge **is event-driven** | Subscribes to domain events (per [§5.2](05-cross-cutting.md), forthcoming) and re-projects on relevant state changes |

**Things the Bridge does not do** (these are defects if found):

- Encode business rules ("an item is available if stock > 0" is a domain rule).
- Read from or write to domain stores directly.
- Leak Beckn vocabulary upward — nothing in Application, Domain, or first-party interfaces imports from the Bridge.
- Call other adapters (no calls to the Admin UI or any first-party adapter from inside the Bridge).
- Rewrite domain semantics to fit the protocol.

No other adapter imports anything from the Bridge. The Application Layer is **unaware** that the Bridge exists — it treats Bridge-originated calls identically to first-party-originated calls.

## 3.2 Network identity — the platform as one BPP (via ONIX)

The platform appears on the Beckn network as **one BPP**, identified by a single `bpp-id` and `bpp-uri` ([ADR-0001](../decisions/0001-bpp-network-identity.md), [ADR-0022](../decisions/0022-onix-protocol-gateway.md)). Stores project onto the network as Beckn `provider` nodes inside that BPP's responses. **ONIX is the network-visible BPP**: signed messages on the wire come from ONIX; the registry knows ONIX; `bpp-uri` resolves to the ONIX endpoint.

| Concern | Resolution |
|---|---|
| BPP cardinality | One BPP for the whole platform (not one BPP per store) |
| Provider ID derivation | Deterministic from the internal store ID; immutable for the store's lifetime — **set by the BPP** (ONIX is transparent on `context` contents) |
| Signing key | One platform-level key. **Held by ONIX.** The BPP does not hold the signing key. |
| `bpp-id` / `bpp-uri` | **`bpp-id` set by the BPP** in outbound `context.bpp_id`. **`bpp-uri` = the ONIX network-facing URL**, set by the BPP in outbound `context.bpp_uri`. |
| Callback endpoint | The ONIX network-facing endpoint is one platform-wide endpoint. ONIX forwards inbound messages to the BPP; the BPP routes them to the originating store using protocol-level identifiers (transaction_id, message_id, provider reference) — never by URL shape |
| Reputation | Platform-owned. Per-store reputation visibility is deferred |
| Registry registration | Operational input. `bpp-id`, `bpp-uri`, signing key material, registry credentials are supplied as **ONIX configuration**, not BPP configuration. The BPP does not own registry lifecycle. |
| BPP ↔ ONIX wire | Full Beckn JSON envelope, plain HTTPS POST, no auth between BPP and ONIX. Network-level isolation is the security model — the BPP-facing endpoint of ONIX is private. |

**Consequence**: a store's Beckn identity is bound to its store identity for life. "Resetting" Beckn presence requires creating a new store. Quality control at store onboarding is a platform responsibility because all stores share the BPP's reputation.

## 3.3 Four-actor architecture — ONIX-mediated network access

Beckn v2 separates discovery from transaction. After [ADR-0022](../decisions/0022-onix-protocol-gateway.md), four actors interact, with ONIX as the BPP's protocol gateway:

```
                     ┌──────────────────────┐
                     │  Catalog Discovery   │
                     │   Service (CDS)      │
                     └─┬──────────────┬─────┘
                       │              ▲
        /on_discover ──┤              │
                       ▼              │
                ┌───────────┐         │
                │    BAP    │         │
                │  (buyer-  │         │
                │   side)   │         │
                └───────┬───┘         │
                        │             │
                        │ Beckn v2    │ /catalog/publish
                        │ signed      │ (forwarded)
                        ▼             │
                  ┌──────────────────────┐
                  │       ONIX           │
                  │  (Beckn Protocol     │
                  │   Server — vendor    │
                  │   binary, per-BPP)   │
                  └──────────┬───────────┘
                             │
                             │ full Beckn JSON
                             │ plain HTTPS POST
                             │ no auth
                             ▼
                       ┌─────────────┐
                       │   BPP (us)  │
                       │             │
                       └─────────────┘
              /select  /init  /confirm  /status  /cancel
              all flow BAP → ONIX → BPP and back
```

- **BAP** (Buyer App): the buyer-side participant. External to us. Never talks to the BPP directly — only to ONIX.
- **ONIX** (Beckn Protocol Server): vendor-provided binary deployed per-BPP. Network-visible endpoint. Handles signing, schema validation, registry, CDS publish. Transparent on `context` fields; never modifies `transaction_id` / `message_id`.
- **CDS** (Catalog Discovery Service): the network's catalog index. External to us. Reachable only through ONIX from the BPP's side.
- **BPP** (us): publishes catalogs through ONIX (which forwards to CDS); handles the transactional flow via ONIX (which forwards from / to BAP).

**Critically: the BPP does not field `/discover`.** Discovery is the CDS's responsibility. The BPP's job is to **publish** its stores' catalogs through ONIX (`POST <onix-url>/catalog/publish`), and the CDS handles BAP discover queries.

The transactional flow (`/select`, `/init`, `/confirm`, etc.) flows **BAP → ONIX → BPP** inbound and **BPP → ONIX → BAP** outbound. ONIX is in the path on both legs but is transparent on protocol-level identifiers — `transaction_id` and `message_id` are preserved end-to-end.

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

### 3.4.1 Multi-catalog projection — one store, many catalogs

A subtlety in the v2 model worth flagging: **a single store may have multiple internal catalogs, and all of them project to the network**. Per [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md):

- A store has **exactly one Default Catalog**, auto-created with the store. **Mandatory membership** ([ADR-0020](../decisions/0020-default-catalog-mandatory.md)): every Product in the store is automatically and permanently a member; there is no exclusion mechanism. To hide a product from Beckn, change the Product's lifecycle state (Draft or Archived).
- A store may also have **additional Catalogs** — named, scoped to that store (e.g., "Summer Collection", "Bestsellers"). **Opt-in membership** — products are added explicitly.
- Every Product is always in the Default Catalog and may additionally be in zero or more additional Catalogs.
- **All catalogs project to Beckn.** Each appears as a distinct wire `Catalog` object, **each carrying the same `Provider`** (the store). BAPs see distinguishable groupings under the same provider and may render any of them.

Example: a store with three internal catalogs (Default + "Summer Collection" + "Bestsellers") publishes **three wire `Catalog` objects** to CDS — same `Provider` repeated in each, different `resources` and `offers` per catalog. The Bridge's mapping registry handles the per-catalog identifier derivation (deterministic, stable for the catalog's lifetime).

The domain (the Catalog context) owns the structure: which catalogs exist, which products belong to each, the Default vs. additional distinction. The Bridge's job is just to produce one wire Catalog object per internal catalog and post each to CDS. See [§4.3 Catalog](04-bounded-contexts/4.3-catalog.md) for the full domain model.

## 3.5 Handler set in v1

What the BPP exposes and consumes in v1. All handlers pass through ONIX (per [§3.3](#33-four-actor-architecture--onix-mediated-network-access)); the table shows logical direction.

| Logical direction | Handler | Wire path | Purpose |
|---|---|---|---|
| BPP → CDS | `POST /catalog/publish` | BPP → ONIX → CDS | Publish or republish a store's catalogs |
| CDS → BPP | `POST /catalog/on_publish` | CDS → ONIX → BPP | Async per-catalog processing result |
| BAP → BPP | `POST /select` | BAP → ONIX → BPP | BAP indicates items; BPP returns a quoted `Contract` |
| BAP → BPP | `POST /init` | BAP → ONIX → BPP | BAP provides buyer details; BPP reserves inventory |
| BAP → BPP | `POST /confirm` | BAP → ONIX → BPP | BAP commits; BPP converts reservation, records voucher use |
| BAP → BPP | `POST /status` | BAP → ONIX → BPP | BAP queries order status |
| BAP → BPP | `POST /cancel` | BAP → ONIX → BPP | BAP requests cancellation (pre-fulfillment) |
| BPP → BAP | `POST /on_select`, `/on_init`, `/on_confirm`, `/on_status`, `/on_cancel` | BPP → ONIX → BAP | Async callbacks projecting the Contract back to the BAP |

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

## 3.8 Catalog publishing (BPP → ONIX → CDS)

The Bridge publishes catalog state via `POST <onix-url>/catalog/publish`; **ONIX signs and forwards to CDS** (per [ADR-0022](../decisions/0022-onix-protocol-gateway.md)). From the Bridge's perspective this is a single HTTP POST with a full Beckn JSON envelope; ONIX handles the network-side delivery. The Bridge publishes on every domain event that changes what should appear on the network:

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

The CDS responds asynchronously with `/catalog/on_publish` (also routed through ONIX) containing per-catalog processing results. The Bridge handles this callback — recording acknowledgements and surfacing publish failures for ops review (consistent with the failure-handling rule from [§2.5](02-principles.md)).

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

The Bridge owns the asynchronous concerns. ONIX is **transparent on protocol identifiers**: `transaction_id` and `message_id` flow through end-to-end (BAP ↔ ONIX ↔ BPP), so correlation logic is unchanged from the original design.

- **Correlation**: mapping inbound responses back to the original `transaction_id` and `message_id`.
- **Protocol-level idempotency**: recognizing duplicate inbound messages by their protocol identifiers and short-circuiting before they reach the Application Layer. ONIX also dedups at the network door (ION-1004 within a 30-minute window), but the BPP keeps its own `bridge_inbox` dedup as defense in depth.
- **Transactional context**: tracking what state a multi-step Beckn flow is in (search → select → init → confirm) from the protocol's perspective.
- **Outbox retry to ONIX**: outbound POSTs use `bridge_outbox` with exponential backoff. 5xx from ONIX (transient — including ION-9001) → retry. 4xx (structural / policy / signing-side) → mark `dead`, emit event for ops review.

The Application Layer is unaware of message IDs, transaction IDs, callback URLs, or acknowledgement semantics. From its perspective, every interaction is a discrete use-case invocation that returns a domain result. The Bridge translates that synchronous-looking call into the asynchronous protocol dance.

## 3.10 Versioning strategy

The Beckn protocol evolves. Multiple versions may be in flight on the network simultaneously; not all participants upgrade in lockstep. The Bridge handles versioning. ONIX is **transparent on protocol version** — it does not version-translate. The BPP targets Beckn v2.0.0 directly in outbound messages; ONIX just signs and routes.

- Each supported protocol version has its **own translation set** within the Bridge. Versions coexist; they don't replace each other silently.
- The protocol version is **negotiated per inbound message** via the `context` block.
- A **protocol upgrade is a Bridge-only change**. If a Beckn version bump forces a domain change, that's evidence of leakage — investigate before accepting.
- Deprecating a protocol version is **explicit and announced**, not an accident of refactoring.
- Structural / version-mismatch inbound is rejected by ONIX at the network door (ION-8xxx schema / ION-1005 invalid domain). State-dependent rejections happen at the BPP via Nack.

## 3.11 Error handling across the boundary

Errors cross the Bridge in both directions, and in both directions they are **translated**, not forwarded. Per [ADR-0022](../decisions/0022-onix-protocol-gateway.md), the Beckn network uses the **ION-XXXX error code registry** (from `ion-specs/errors/`), which the Bridge consumes in both directions.

### Direction summary

| Direction | Translation |
|---|---|
| Domain → ION-XXXX | Domain errors (e.g., "store not found", "product unavailable", "voucher expired") map to ION-XXXX codes for Nack responses or for failure shape on outbound. The Application Layer never emits a Beckn / ION error code. |
| ONIX → Domain | ONIX failures arrive as `{ errorCode: "ION-XXXX", errorMessage }` with HTTP status per the registry's `http_status` field. 5xx (e.g., ION-9001) → retry from `bridge_outbox`. 4xx (e.g., ION-1001 signature, ION-1002 not registered, ION-1003 TTL, ION-1004 duplicate, ION-7xxx policy) → fatal; mark `dead`, emit event for ops review. |
| ONIX-rejected inbound | Structural / protocol failures (ION-1xxx, ION-8xxx) never reach the BPP — ONIX rejects them at the network door. |
| BPP-rejected inbound (state-dependent) | State that ONIX can't see (catalog availability, contract state, etc.) is rejected at the Bridge with a typed Nack carrying an ION-XXXX code (typically ION-3xxx transactional or ION-7xxx policy the BPP catches). |

**Unmappable cases are explicit**: if a domain error has no clean ION-XXXX equivalent, the Bridge maps it to a generic protocol error *and* logs the mismatch. Silent information loss is forbidden.

### ION-XXXX registry consumption

The Bridge loads `ion-specs/errors/registry.json` at startup. Each registry entry pins `http_status`, `title`, `description`, `affected_field`, `affected_apis`, and `resolution`. The Bridge's `mapping-registry/errors.ts` (see [§3.7](#37-mapping-discipline--the-mapping-registry)) is the domain-error → ION-XXXX projection; reverse mapping (ION-XXXX → domain action) is also registry-backed.

ION code ranges:

| Range | Category |
|---|---|
| ION-1xxx | Transport (signing, identity, TTL, duplicate `messageId`, domain code) |
| ION-2xxx | Catalog |
| ION-3xxx | Transaction |
| ION-4xxx | Fulfillment |
| ION-5xxx | Post-order |
| ION-6xxx | Settlement |
| ION-7xxx | Network policy |
| ION-8xxx | Schema |
| ION-9xxx | System (ION infrastructure; ION-9001 is retryable) |

## 3.11a Inbound signature re-verification

Even though ONIX verifies BAP signatures at the network door (rejecting ION-1001 / ION-1002 failures before forwarding to the BPP), **the BPP must re-verify** inbound signatures. Reason: the BPP's `Ack` / `AckNoCallback` / `Nack` response carries a **CounterSignature** attesting that *the BPP itself* authenticated the inbound. Without a BPP-side re-verification step, the CounterSignature would be an implicit assertion ("ONIX told us this is OK") rather than an explicit BPP attestation.

The Bridge keeps a slim verification module (`bridge/protocol/verification.ts`) that re-verifies the BAP's `Authorization` header on every inbound. ONIX forwards the original envelope including the BAP's `Authorization`, so the data is available.

Two operational follow-ups deferred to Phase 6 (per ADR-0022 open follow-ups):
- **N1 — CounterSignature signing locus**: whether the BPP holds a narrow signing key for CounterSignatures only, or whether ONIX adds the CounterSignature on the synchronous response path (BPP has no key). Default working assumption: ONIX adds it.
- **N2 — BAP public key for re-verification**: whether the BPP runs its own registry client or ONIX includes the BAP's key as forwarded metadata. Default working assumption: ONIX forwards it.

## 3.11b Ack / AckNoCallback / Nack

Every inbound message receives a synchronous typed response with a CounterSignature. Silent drop is forbidden by the protocol.

| Response | When |
|---|---|
| `Ack` | The request is accepted; an async callback (e.g., `/on_select`) will follow. |
| `AckNoCallback` | The request is accepted; no callback will follow (e.g., synchronous `/status` queries that complete in the Ack body). |
| `Nack` | The request is rejected at the BPP level (state-dependent failures ONIX couldn't catch). Carries an ION-XXXX code. |

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
| **The Bridge** | Tested with **real Beckn v2 payloads** from `ion-specs` examples, verifying both inbound parsing and outbound projection for each supported protocol version. ION-XXXX error mapping is registry-driven; tests cover both directions. |
| **End-to-end Beckn flows** | Tested as **integration tests** that exercise the Bridge against a **stub ONIX** (an in-process HTTP server emulating ONIX's request/response shape) and a stubbed BAP. Confirms async correlation, error mapping, version negotiation, retry on 5xx, and Ack/Nack with CounterSignature. |
| **Mapping registry** | Each mapping entry is exercisable in isolation with paired sample inputs and expected outputs. |
| **No real ONIX in tests** | Tests use a stub ONIX with deterministic behavior. The vendor binary is exercised in staging integration, not in CI. |

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
