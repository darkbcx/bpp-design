# CLAUDE.md — Beckn BPP Architecture Charter

This document is the governing charter for the design and evolution of this project. It defines **how the system is conceived**, not how it is coded. Every contributor — human or AI — must read this before proposing changes.

This is a **design-stage repository**. There is no application code yet. The current goal is to establish the architectural, conceptual, and modeling foundations on which all future code will be built.

---

## 1. AI Role & Behavior

### 1.1 Operating Mode

When working in this repository, Claude operates as a **collaborating architect**, not as a coder. The default mode is *thinking, modeling, and documenting* — not implementing.

### 1.2 Behavioral Rules

* **Planning first.** Before producing any artifact (diagram, model, ADR, schema sketch, code), articulate the problem, the options, the trade-offs, and the chosen direction. Wait for confirmation before deepening.
* **Iterative refinement.** Treat every design decision as provisional until it has been challenged. Prefer multiple shallow passes over a single deep one.
* **One concern at a time.** Do not bundle authentication design with catalog modeling with Beckn mapping. Each subsystem is reasoned about in isolation, then integrated.
* **No premature code.** Even if a question seems to call for code, respond with a model, a contract, or a constraint first. Code is the last step, not the first.
* **Surface assumptions explicitly.** If a decision rests on an assumption (e.g., "stores are single-currency"), state it as an assumption and flag it for the user to confirm.
* **Push back on protocol leakage.** If a request implicitly couples the domain to Beckn (e.g., "let's add a `descriptor` field to Product"), name the leak and propose a protocol-isolated alternative.
* **Defer ambiguity to the user.** When two architectural directions are both defensible, present both with their trade-offs rather than silently picking one.

### 1.3 What Claude Must Not Do (in this phase)

* Generate application code, framework scaffolding, ORM definitions, or API handlers.
* Pick a programming language, framework, or database engine without explicit user direction.
* Copy structures verbatim from `bitemycart` or `ion-specs`.
* Introduce Beckn vocabulary (`descriptor`, `fulfillment`, `provider`, `resource`, `offer`, `contract`, `on_select`, etc.) into domain artifacts.

### 1.4 What Claude Must Do

* Maintain conceptual integrity across documents — if a term is defined once, use it consistently everywhere.
* Keep this `CLAUDE.md` and any subsequent design documents internally consistent. If a new decision invalidates an old one, update both.
* Treat the **domain model** as sacred and the **Beckn integration** as replaceable.

---

## 2. System Architecture

### 2.1 What This System Is

A **Beckn Provider Platform (BPP)** that acts as a **multi-tenant aggregator of small retailers**. Each retailer operates an independent store within the platform. The platform appears on the Beckn network as a **single BPP**; stores project onto the network as Beckn `provider` nodes inside that BPP's responses ([ADR-0001](decisions/0001-bpp-network-identity.md)). Internally, the system has no inherent dependence on Beckn.

### 2.2 Guiding Architectural Principle

> **The domain must be able to exist, evolve, and be tested without Beckn.**
> Beckn is a *consumer* of the domain, not a *definer* of it.

If Beckn were removed tomorrow, the system would still be a coherent multi-tenant marketplace. If Beckn evolves (v2 → v3, new domains, breaking schema changes), only the Beckn Bridge should be impacted.

### 2.3 The Dependency Rule

The system is organized so that **dependencies point inward, toward the domain.** This rule precedes and constrains every other architectural choice in this document.

* The **Domain** depends on nothing.
* The **Application** depends only on the Domain.
* **Infrastructure** and **Interface** components depend on the Application and Domain — never the reverse.
* When an inner layer needs an outward capability (e.g., persistence, sending email, calling an external service), it declares a **port** (an abstract contract) that an outer layer implements.

A change in an outer layer must never force a change in an inner layer. A change in the Domain may legitimately require outer layers to adapt — that is the *correct* direction of pressure.

### 2.4 Layered Architecture

Four logical layers, listed innermost to outermost. These are conceptual boundaries; the physical packaging (modules, services, deployables) is deferred (see 2.7).

1. **Domain Layer** — the *what*.
   * Pure business concepts: User, Store, Membership, Role, Invitation, Product, Catalog, Inventory, Order.
   * Owns invariants, identifiers, value objects, state transitions, and domain events.
   * Knows nothing about HTTP, JSON, Beckn, persistence engines, frameworks, or transport.
   * Testable with no infrastructure and no fixtures from outside.

2. **Application Layer** — the *how*.
   * Use cases that orchestrate the domain (e.g., "create a store", "publish a product", "accept an invitation").
   * Coordinates transactions, authorization checks, and domain-event publication.
   * Speaks only in domain terms. Accepts domain-shaped inputs; returns domain-shaped results. Never returns transport payloads.
   * Defines ports for everything it cannot do itself (persist, notify, enqueue).

3. **Infrastructure Layer** — the *with what*.
   * Adapters that implement the ports declared by inner layers: persistence, messaging, identity providers, external services, file storage, search, email delivery.
   * Substitutable. Swapping a database or identity provider must not ripple inward.
   * Owns all I/O, all framework integration, and all third-party SDKs.

4. **Interface Layer** — the *for whom*.
   * The system's outward-facing surfaces — the entry points by which external callers reach the Application Layer. Three distinct families, each its own adapter:
     * **First-party UI adapters** — admin/owner console, storefront.
     * **First-party API adapters** — internal APIs consumed by our own front-ends.
     * **Beckn Bridge adapter** — receives and emits Beckn protocol messages (detailed in 2.6).
   * Every adapter converts the outside world into Application-Layer calls and converts Application-Layer results back into the outside world's format.
   * No business logic. No persistence. No cross-adapter coupling.

**Flow of a request (conceptual):**

```
External caller → Interface adapter → Application use case → Domain logic
                                            ↓
                                     Infrastructure (ports)
                                            ↓
                                     External systems (DB, queue, ...)
```

Results flow back along the same path in reverse. The Interface adapter is responsible for shaping the final response for its specific audience.

### 2.5 Bounded Contexts

Layers are *horizontal* (technical concerns). Bounded contexts are *vertical* (business concerns). A single context — say, Catalog — has its own Domain, its own Application use cases, and its own Infrastructure adapters. Layering and context decomposition are complementary, not competing.

The initial contexts:

* **Identity & Access** — User identity (mirrored from an external IdP), sessions, active-store binding, profile state. Authentication mechanics live entirely at the IdP. See §5.15.
* **Tenancy** — stores, ownership, memberships, roles, invitations. Owner of the multi-tenancy model.
* **Catalog** — products, variants, attributes, media, taxonomies, pricing.
* **Inventory** — stock and availability for catalog items. See §5.11.
* **Promotion** — vouchers and voucher usage; store-scoped. See §5.13.
* **Audit** — append-only history of state-changing events across all contexts; downstream subscriber of the event stream. See §5.19.
* **Order & Fulfillment** — Order, Quote, line-item snapshots, payment-status and fulfillment-status tracking; end-to-end commerce flow. See §5.21.

The **Beckn Bridge is not a bounded context.** It is an *adapter* sitting in the Interface Layer that translates between the Beckn protocol and Application-Layer use cases. It has no domain of its own. See 2.6.

Each context owns its own model, its own language, and its own persistence. **Cross-context coupling is forbidden at the storage layer** — no foreign keys, no shared tables, no direct joins across context boundaries.

### 2.6 Inter-Context Communication

Contexts collaborate through two explicit channels:

* **Synchronous queries / commands** — a context exposes a use-case-level API (Application Layer) that another context may call. Such calls are *typed in domain terms* and pass *identifiers and value objects*, never internal records.
* **Domain events** — when a context changes state in a way that matters to others (e.g., `MembershipAccepted`, `ProductPublished`), it publishes a named event. Interested contexts subscribe.

Rules:

* A context never reaches into another context's persistence or domain model directly.
* Identifiers crossing context boundaries are treated as opaque references, not as foreign keys.
* Events describe *what happened* in domain terms — never transport, never UI, never Beckn vocabulary.
* The choice of synchronous vs. event-driven for a given interaction is a design decision per integration, documented at the time it is made.

### 2.7 The Beckn Bridge: Position and Boundaries

The Beckn Bridge is a **specialized adapter in the Interface Layer**. Architecturally it is one of several entry points into the Application Layer; conceptually it is the most heavily constrained one, because it is the sole place protocol vocabulary is permitted.

Its position:

```
Beckn network ←→ [ Beckn Bridge adapter ] ←→ Application use cases ←→ Domain
                  (sole protocol-aware zone)
```

What this implies:

* The Bridge is an **adapter**, not a layer of its own and not a bounded context.
* The Bridge **calls** the Application Layer; it never substitutes for it, bypasses it, or duplicates its work.
* No other adapter (storefront, admin UI, internal API) ever imports anything from the Bridge.
* The Application Layer is **completely unaware** that the Bridge exists. It treats Bridge-originated calls identically to first-party-originated calls.

The full responsibilities and constraints of the Bridge are detailed in section 4.

### 2.8 Beckn Is One Interface Among Several

The Beckn endpoint is *one* surface, alongside the admin UI, the storefront, and our internal APIs. It is not privileged.

Concretely:

* Every store function (create, manage, publish, transact) must be exercisable through first-party interfaces **without any Beckn participant in the loop**.
* New domain features are designed against the domain and the first-party interfaces first. Beckn exposure of those features is a follow-up step performed in the Bridge.
* If a feature is requested that only makes sense in a Beckn context, scrutinize it: it likely indicates protocol leakage or a missing domain concept.

### 2.9 Deployment Topology Is Deferred

This document defines the **logical** architecture. Whether the system ships as a single deployable (modular monolith), as multiple services aligned to bounded contexts, or as something in between is a **downstream** decision driven by operational, team, and scale concerns — not by this charter.

What is non-negotiable regardless of topology:

* Layer boundaries are enforced at the source level, not by network distance.
* Context boundaries are enforced by explicit contracts, not by deployment.
* The Bridge is isolatable — it must be possible to deploy or replace it independently, even if today it lives in the same process as everything else.

---

## 3. Data Modeling Philosophy

### 3.1 Modeling Stance

* Model the **business**, not the **wire format**.
* Names come from the problem domain (e.g., `Store`, `Product`, `Membership`), never from Beckn (`Provider`, `Resource`, `Offer`, `Contract`, `Descriptor`).
* Each entity should have a single, defensible reason to exist. If you cannot describe an entity without referencing Beckn, it does not belong in the domain.

### 3.2 Entity Design Principles

* **Identity is internal and stable.** Every entity has an internally-generated identifier. External identifiers (Beckn IDs, OAuth subject IDs, payment provider IDs) are *attributes*, not primary identities.
* **Value objects over primitives.** Concepts like Money, Email, Slug, Address, Quantity, and SKU should be modeled as value objects with their own invariants — not as raw strings or numbers on entities.
* **State transitions are explicit.** Lifecycle stages (e.g., invitation pending → accepted → revoked) are first-class. Avoid boolean flags that imply hidden state machines.
* **Time is a first-class concept.** Created-at, valid-from, valid-until, accepted-at, etc., are not afterthoughts. Anything with a lifecycle has explicit temporal anchors.
* **Soft deletion follows the system-wide pattern in §5.19**: business entities are never deleted (end-of-life is a state transition); operational entities are hard-deleted on schedule.

### 3.3 Relationship Design Principles

* **Normalize first.** Denormalization is a performance decision made later with evidence, not a starting point.
* **Aggregates have clear boundaries.** Identify what must be modified together transactionally (e.g., a Product and its Variants) vs. what is referenced (e.g., a Product references a Category).
* **No bidirectional intimacy across contexts.** A `Product` in Catalog refers to a `Store` by ID; the Catalog context does not load Store internals.
* **Junction tables represent real concepts.** A `Membership` between a User and a Store is itself an entity with attributes (role, joined-at, status), not a faceless link.

### 3.4 Keeping Models Beckn-Compatible Without Coupling

The test of compatibility is: *can the Beckn Bridge construct a valid Beckn message from the current domain state?* This is achieved by:

* Ensuring the domain captures all **business facts** Beckn cares about — descriptions, prices, categories, media, availability — under domain-native names.
* Allowing the Bridge to *project* the domain into Beckn shape, *enriching* with protocol-specific constants where needed (e.g., schema versions, context fields).
* Refusing to add a field to the domain solely to "match Beckn." If Beckn needs a constant or a derived value, the Bridge computes it.
* Accepting that some Beckn fields will have **no domain origin** (they are protocol metadata) and some domain fields will have **no Beckn destination** (they are internal). This asymmetry is healthy.

### 3.5 Identifier Strategy

* Use opaque, non-sequential identifiers for anything that may be exposed externally (URLs, APIs, Beckn payloads). Sequential keys may exist internally for storage, but should not leak.
* Tenancy-scoped identifiers (e.g., a product's slug) must be unique *within their tenant*, not globally.
* Beckn-facing identifiers are derived deterministically from internal identifiers by the Beckn Bridge — never the reverse.

---

## 4. Beckn Integration Strategy

Section 2.7 establishes *where* the Beckn Bridge sits: a specialized adapter in the Interface Layer, never a bounded context, never an inner layer. This section defines *what it does*, *how it does it*, and *what it must refuse to do*.

### 4.1 Exclusive Responsibilities of the Beckn Bridge

The Beckn Bridge is the **sole** place in the system where any of the following is permitted:

* Awareness of Beckn schema, vocabulary, structure, and envelopes.
* Translation between domain shapes and protocol shapes (in either direction).
* Protocol-version-specific logic and version negotiation.
* Validation of Beckn payloads against `ion-specs`.
* Construction and interpretation of Beckn message envelopes and `context` blocks.
* Protocol-level concerns: signing, signature verification, registry/discovery lookups, callback URLs, transport-level idempotency keys, retry policy.

If any of these concerns appear outside the Bridge, that is a defect — regardless of how convenient the shortcut seemed.

**Network identity ([ADR-0001](decisions/0001-bpp-network-identity.md)).** The platform appears on the Beckn network as a single BPP. The Bridge accordingly holds the single platform signing key, exposes a single platform-wide Beckn callback endpoint, and derives Beckn `provider` IDs deterministically from internal store identifiers. Inbound messages are routed back to the originating store using protocol-level identifiers (transaction ID, message ID, provider reference) — never by URL shape. The `bpp-id`, `bpp-uri`, signing key material, and registry credentials are supplied as configuration; the Bridge does not own the registry-side lifecycle.

### 4.2 Direction of Translation

The Bridge has two translation paths, and they are intentionally asymmetric:

* **Inbound (Beckn → Application).** The Bridge receives a Beckn message, verifies signatures, validates the payload against the schema for the declared protocol version, extracts the business intent, and invokes an Application use case with **domain-shaped** inputs. The Application Layer never sees raw Beckn JSON, never sees Beckn field names, and never knows which protocol version originated the call.
* **Outbound (Application → Beckn).** The Application Layer produces a domain result. The Bridge projects that result into the appropriate Beckn message (`on_select`, `on_init`, `on_confirm`, `on_status`, `on_cancel`, etc.), enriches with protocol-level metadata (context envelope, schema version, signatures), and dispatches it on the network.

The asymmetry is deliberate: outbound is *projection plus enrichment*; inbound is *validation plus extraction*. Neither direction is a mechanical inverse of the other.

### 4.3 Asynchronous Flow and Correlation

Beckn is fundamentally an asynchronous, callback-driven protocol. A request (e.g., `select`) is acknowledged immediately; the substantive response (`on_select`) is delivered later via a callback to the originating participant. This shape is **the Bridge's problem**, not the Application Layer's.

The Bridge owns:

* **Correlation.** Mapping outbound responses back to the original transaction and message IDs.
* **Idempotency at the protocol boundary.** Recognizing duplicate inbound messages by their protocol-level identifiers and short-circuiting them before they reach the Application Layer.
* **Transactional context.** Tracking what state a multi-step Beckn flow (search → select → init → confirm) is in, from the protocol's perspective.

The Application Layer is unaware of message IDs, transaction IDs, callback URLs, or acknowledgement semantics. From its perspective, every interaction is a discrete use-case invocation that returns a domain result. The Bridge translates that synchronous-looking interaction into whatever asynchronous protocol dance is required.

### 4.4 Versioning Strategy

The Beckn protocol evolves. Multiple versions may be in flight on the network simultaneously, and our participants may not all upgrade in lockstep. Versioning is a Bridge concern; the domain must not feel it.

* Each supported protocol version has its own translation set within the Bridge. Versions coexist; they do not replace each other silently.
* The protocol version is negotiated and recorded per inbound message based on the `context` block.
* A protocol upgrade is a **Bridge-only change**. If a Beckn version bump forces a domain change, that is evidence of leakage — investigate before accepting it.
* Deprecating a protocol version is an explicit, announced decision, not an accident of refactoring.

### 4.5 Error Handling Across the Boundary

Errors cross the Bridge in both directions, and in both directions they must be **translated**, not forwarded.

* **Domain → Beckn.** Domain errors are expressed in domain terms (e.g., "store not found", "product unavailable", "membership inactive"). The Bridge maps them to the appropriate Beckn error codes and shapes. The Application Layer never emits a Beckn error code.
* **Beckn → Domain.** Protocol-level errors (schema invalid, signature failed, unknown action) are handled **inside the Bridge** and are never surfaced to the Application Layer as domain errors. Only validated, well-formed, business-meaningful requests reach the Application.
* **Unmappable cases are explicit.** If a domain error has no clean Beckn equivalent, the Bridge maps it to a generic protocol error *and* logs the mismatch. Silent loss of information is forbidden.

### 4.6 Mapping Discipline

Every domain ↔ protocol mapping must be:

* **Explicit.** Declared as a named, documented artifact — not inferred from naming conventions, reflection, or implicit conversion.
* **Testable in isolation.** Each mapping is exercisable with sample payloads from `ion-specs`, independent of the rest of the system.
* **Replaceable.** A new mapping (e.g., for a protocol version bump) can be introduced without rewriting the domain or other mappings.
* **Asymmetric-tolerant.** Some Beckn fields will have no domain origin (protocol metadata, constants); some domain fields will have no Beckn destination (internal concerns). This is healthy and expected. Do not invent domain fields to "match" Beckn, and do not invent Beckn fields to "expose" the domain.
* **Defensive about the unknown.** Unrecognized Beckn fields on inbound messages are logged and ignored, not propagated. Domain projections do not invent Beckn fields the schema does not define.

Mappings are documented as a **mapping registry** — a first-class artifact in the repository — not buried inside transformation code.

### 4.7 What the Bridge Must Not Do

* **No business rules.** "An item is available if stock > 0" is a domain rule. "An order can only be confirmed after init" is a domain rule. The Bridge does not encode such things; it asks the Application Layer.
* **No direct persistence.** The Bridge invokes use cases; it does not read from or write to domain stores.
* **No upward vocabulary leakage.** Nothing in the Application, Domain, or first-party interfaces imports anything from the Bridge. Beckn names stop at the Bridge boundary.
* **No cross-adapter shortcuts.** The Bridge does not call into the storefront, the admin UI, or first-party APIs. All paths into the system go through the Application Layer.
* **No silent rewriting of domain semantics.** If a Beckn field implies a business meaning the domain does not represent, that is a domain-modeling conversation — not a Bridge workaround.

### 4.8 Testing Strategy (Conceptual)

* The **Domain** and **Application** Layers are tested with no Beckn fixtures whatsoever. If a domain test needs a Beckn payload to make sense, the test is wrong.
* The **Bridge** is tested with real Beckn payloads sourced from `ion-specs` examples, verifying both inbound parsing and outbound projection for each supported protocol version.
* **End-to-end Beckn flows** are tested as integration tests that exercise the Bridge against a stubbed network counterpart, confirming asynchronous correlation, error mapping, and version negotiation.
* The presence of Beckn in any test outside the Bridge's own test suite is a red flag worth investigating.

---

## 5. Multi-Tenancy Approach

### 5.1 Tenancy Model

A **Store** is the unit of tenancy. Every business object that belongs to a store (products, inventory, orders, members) is scoped to exactly one store. There is no cross-store data sharing by default.

### 5.2 Isolation Principles

* **Logical isolation is mandatory.** Every query, command, and authorization check carries a store context. There is no "global" listing of tenant-owned data outside of platform-level admin operations.
* **Physical isolation is a deployment choice, not a domain concern.** The domain model assumes logical multi-tenancy; whether tenants share a database, a schema, or have dedicated infrastructure is decided at the infrastructure layer.
* **No implicit tenant.** Code paths that operate on tenant-owned data must require a store identifier explicitly. "Current store" is never inferred from ambient state at the domain level.
* **No tenant leakage in identifiers.** Even when identifiers are globally unique, they must not be guessable across tenants in a way that aids enumeration.

### 5.3 Ownership and Access Model (Conceptual)

* A Store has exactly **one Owner** at any given time. Ownership is transferable but never plural.
* A Store has zero or more **Admins**. Admins are Users granted administrative access via a **Membership**.
* A **Membership** is a first-class concept: it represents a User's relationship to a Store, with a role, a status, and a history.
* A User's identity is global; their *capabilities* are scoped. A User may hold zero or more store Memberships and may additionally hold a platform-scoped role. The full tier model is in §5.4.
* **Authorization questions are always of the form** *"Does this actor have this capability in this scope?"* — never simply *"Is this actor an admin?"*. The "scope" is the active store for store-scoped users, the platform for platform-scoped users, or system-wide for System Admins.

### 5.4 Authorization Tiers

The system has three authorization tiers, in decreasing privilege ([ADR-0002](decisions/0002-authorization-tiers-and-matrix.md)):

* **System Admin** — meta-administrators. Manage the role → capability matrix; configure platform-level settings. Seeded via deploy configuration only. No in-band creation, demotion, or recovery — lifecycle is entirely operational.
* **Platform-scoped** — platform operators (support, compliance, network operations). May act across tenants according to assigned capabilities. Not Members of stores by virtue of platform role.
* **Store-scoped** — Members of one or more stores. May act only on their currently active store; no cross-tenant action is reachable from a store-scoped role.

The **role catalog** is system-defined. Adding a new role is a feature change, not a runtime configuration. Roles within tiers are fixed at design time.

A single User may simultaneously hold a platform-scoped role and Memberships in one or more stores. The two scopes never compose into a single ambient capability set; the user's effective scope at any moment is determined by their **active store** (§5.6).

### 5.5 Capabilities and the Permission Matrix

* The **capability catalog** is system-defined. Capabilities follow the **action-level naming** convention `<resource>.<verb>` — e.g., `product.publish`, `voucher.disable`, `store.activate`, `user.scrub` ([ADR-0016](decisions/0016-authorization-details.md)). New capabilities are added when the features they gate are added.
* The **role → capability matrix** is configured at runtime, exclusively by System Admins. No other tier can edit it. Per-store customization of the matrix is not supported. The Tenancy context owns the matrix.
* **Capability identifiers are single.** A capability has one canonical name; whether it applies cross-tenant or only to the active store is determined by the holder's tier, not by the capability name.
* For store-scoped roles, every authorization decision applies an **active-store filter** in addition to the capability check: the target object must belong to the user's currently active store.
* Capabilities exposed via store-scoped roles are a **subset** of those exposed via platform-scoped roles.
* **The matrix is grants-only.** Absence of a `(role, capability)` entry means denied. No explicit deny entries, no precedence rules. Special cases like "admins can do everything except X" are modeled as the explicit absence of X in that role's grants.
* **Decision exposure.** Every Application-Layer use case begins with one or more `AuthorizationPort.requireCapability(name, scope)` calls before any state mutation. The port is owned by the Tenancy context and consumed by every other context via the cross-context port pattern (§5.17). The call raises `AuthorizationDenied` on failure; the use case never proceeds. A non-throwing `hasCapability(...)` is also exposed for conditional UI.
* **Denial auditing is two-tier.** Authenticated denials emit `identity.authorization_denied` (carrying actor, capability name, scope, typed reason); Audit ingests as a standard record. Anonymous denials (no valid session) are infrastructure-level access logs only — they don't reach the event stream.

### 5.6 Active Store

* "Active store" is a **session-level attribute** identifying which store-scoped Membership is currently in effect.
* When a user holds more than one Membership, the active store is chosen by explicit UI action: set in session, then immediately redirected to a URL-scoped path (`/stores/<slug>/...`). The URL slug must match the session value on every request; mismatch is an authorization failure.
* `active store = none` is a valid state available only to System Admins and platform-scoped users. It is the mode in which they exercise platform-scope authority. Pure store-scoped users never see this state.
* The session storage mechanism itself is deferred to Gap 01 (Identity & Access scope); authorization treats the active store as a logical attribute.

### 5.7 Impersonation

* Impersonation is supported **strictly downward** along the tier hierarchy: System Admin → (Platform | Store); Platform → Store. Same-tier and upward impersonation are forbidden.
* During impersonation, the impersonator's effective capabilities become exactly those of the impersonated user — no more, no less. Destructive actions are allowed.
* Impersonation is initiated by an explicit UI action and is time-limited (timeout is operational configuration). An "End impersonation" control is always visible during an impersonating session.
* **Audit attribution**: every action during impersonation records *both* the real actor and the impersonated user, with the real actor as the responsible party. No action ever appears in audit attributed to the impersonated user alone when impersonation was active.

### 5.8 Store Lifecycle

A Store is in exactly one of four states at any time ([ADR-0003](decisions/0003-store-lifecycle-and-state-machine.md)):

* **Draft** — initial state on creation. Owner and admins build catalog, settings, and metadata; the store is not visible on the Beckn network. Activation is a platform action.
* **Active** — platform-activated. Open for buyers and visible on the network.
* **Suspended** — set by the platform under the moderation context. Reversible to Active by the platform only.
* **Paused** — set by the owner. Reversible to Active by the owner only.

There is no Archived state and no deletion. Stores wind down by remaining in Paused indefinitely; identifiers and history are retained forever.

**State machine.** All other transitions are forbidden:

| From → To | Actor |
|---|---|
| Draft → Active | Platform |
| Active → Paused | Owner |
| Paused → Active | Owner |
| Active → Suspended | Platform |
| Paused → Suspended | Platform |
| Suspended → Active | Platform |

Suspended → Paused is forbidden — owners cannot launder moderation through a voluntary pause. Owners cannot exit Draft; only platform activation does.

**Admin access per state.** Draft and Paused allow full admin access (the store is simply not visible to buyers). Suspended is read-mostly for owner and admins while moderation is in progress — structural changes are not permitted. Active is full access.

**Order behavior in non-Active states.** Uniform across Paused and Suspended: in-flight orders complete to maintain trust; no new orders are accepted. The wire state prevents discovery on the network; first-party interfaces enforce this independently.

**Memberships and invitations** are preserved across every state transition. Reversibility is real — a store returning from Paused or Suspended retains its full Membership roster and pending invitations.

**Republication on transition.** Every transition involving Active, Paused, or Suspended emits a `StoreStatusChanged` domain event. The Bridge subscribes and re-projects the store's catalogs to CDS (via `/catalog/publish`) with the updated wire state. Because Beckn has no provider-deletion mechanism, this re-projection is the only way the network sees a state change. The wire-state mapping (Active → Catalog `isActive: true`; Paused / Suspended → Catalog `isActive: false`; Draft → not published) lives in the Bridge's mapping registry, not in the domain. Beckn v2 does not distinguish owner-pause from platform-suspend at the wire level; the distinction is retained internally.

### 5.9 Store Publication and Catalogs

Publication on the Beckn network is **store-level**: an Active store appears as exactly one `provider` node ([ADR-0004](decisions/0004-store-publication-and-multi-catalog-projection.md)). What appears under that provider is determined by the store's **Catalogs**.

* A store has **exactly one Default Catalog**, auto-created with the store and not deletable. Membership is **opt-out**: every product belonging to the store is included automatically; owners can explicitly exclude specific products.
* A store may create **additional Catalogs** — named, scoped to the store, containing products from that store only. Membership is **opt-in**: products are added explicitly.
* A product may belong to **zero or more** catalogs. A product in no catalog still exists in the store but is invisible on the network.
* **All Catalogs project to Beckn.** BAPs see each catalog as a distinguishable grouping under the provider and may render any of them. The exact Beckn structure used for multi-catalog projection lives in the Bridge's mapping registry, not in the domain.
* Catalog state changes (created, renamed, product added/removed, deleted) emit domain events; the Bridge re-projects the provider node accordingly, mirroring the republication mechanism from §5.8.
* The full **Catalog / Product entity model** (variants, attributes, media, taxonomies, lifecycle) is in §5.10 and [`design/catalog.md`](design/catalog.md).

**Activation flow.** Moving a Draft store to Active is a deliberate two-step:

* The owner sets `submitted_at` on the Draft via an explicit "Submit for activation" action.
* A platform-scoped reviewer with the appropriate capability (per §5.5) sees the Draft in the review queue (filtered to `submitted_at` set) and either activates it (Draft → Active per §5.8) or leaves it pending.
* There is **no coded activation checklist**. Criteria live in operational policy. Basic data validity (name, contact) is an entity-level invariant, not part of activation criteria.

### 5.10 Catalog and Product Model

The Catalog context owns Products and their variants. Detailed entity model in [`design/catalog.md`](design/catalog.md); decisions and reasoning in [ADR-0005](decisions/0005-catalog-and-product-modeling.md). At charter level:

* **Cross-store identity is independent.** A Product is owned by one store; no Product entity is shared across stores. Two stores selling the same physical SKU maintain two independent records. Cross-store deduplication for discovery is a search-layer concern, not a domain concern.
* **Two variant modes, chosen per product, immutable:**
  * **Matrix Mode** — Product has variant-defining attributes (e.g., Size, Color). Variants are explicit entities (combinations of attribute values) that share the Product's SKU. Used when per-variant SKU tracking is not required.
  * **Flat Mode** — Product has no variant relationship; each "variant" is a separate independent Product with its own SKU. Stores group related Flat products via additional Catalogs (§5.9). Used when per-variant SKU tracking is required.
  * Decision rule: if any variant needs its own SKU, use Flat; otherwise use Matrix.
* **Product lifecycle**: Draft → Active → Archived. No deletion; Archived is retained for order history. Active → Draft is forbidden. Mode and store ownership are immutable.
* **Categorization is platform-defined.** System Admins (§5.4) own a hierarchical Category taxonomy; stores assign their products to platform categories. At least one assignment is an entity invariant for `Active` Products. Store-private organization is served by additional Catalogs (§5.9), not by per-store categories. The Bridge derives Beckn category vocabulary from the platform taxonomy via the mapping registry.
* **Media** is an ordered list per Product, with optional per-variant overrides in Matrix Mode. Storage is an Infrastructure concern; the domain knows only references.
* **Baseline Product attributes** — required: name (unique within store), description, ≥1 category assignment for Active, `base_price` and `tax_rate` for Active (§5.12). Optional: SKU (unique within store if set), media, variant-defining attributes (Matrix only).
* **Domain events** are emitted for every Catalog and Product mutation. The Bridge consumes them for republication; Audit consumes them for history. Event schema is owned by Gap 13.

What's explicitly out of scope at this stage: bundles / kits, digital-goods type taxonomy, search/discovery implementation, media storage, localization (Gap 10).

### 5.11 Inventory

The Inventory context owns *is the item purchasable right now* — stock count, reservations, owner-controlled availability — distinct from Catalog's *what is this item* ([ADR-0006](decisions/0006-inventory-model.md)).

* **Anchors.** Each inventory item is either a Flat Mode Product or a Matrix Mode ProductVariant; exactly one `StockLevel` per inventory item. Inventory references Catalog entities by opaque ID — no joins (§2.6).
* **StockLevel** carries `stock_count` (integer ≥ 0) and `purchasable` (boolean; owner-controlled; default `true`).
* **Effective availability** = `purchasable` AND `(stock_count − active reservations) > 0`. Computed on demand from `StockLevel` and active `Reservation` records.
* **Reservations.** A `Reservation` is a temporary hold with `expires_at`. Created at order `init` (Beckn) or the equivalent first-party checkout step; **converted** (stock decremented atomically) at order `confirm`; **released** on cancel or timeout. The exact Beckn-flow trigger points are owned by Gap 11; the Reservation primitive itself is committed here.
* **No multi-location, no backorders in v1.** Single logical inventory per store; cannot sell beyond stock.
* **Stock-movement event log.** Every stock-affecting action emits a typed event (`Received`, `Sold`, `Returned`, `Corrected`, `Reserved`, `Released`, `PurchasableToggled`) carrying actor, timestamp, signed delta, and optional reason. Audit (Gap 16) ingests this stream.
* **Communication.** Storefront, Admin UI, and Bridge query Inventory **synchronously** for availability. Order context calls Inventory's Application Layer to create/convert/release Reservations. Inventory subscribes to Catalog product/variant lifecycle events — creating StockLevels on creation, marking them `Inactive` on archive.
* Stock decrement happens **only** via Reservation conversion at order confirm, or via manual `Corrected` adjustments and `Received` receipts. Direct stock writes outside these paths are forbidden.

### 5.12 Pricing and Tax

The Catalog context owns priced entities ([ADR-0007](decisions/0007-pricing-tax-and-vouchers.md)).

* **Money** is a value object: integer amount in minor units (paise, cents) + ISO 4217 currency code. Floating point is forbidden for monetary values.
* A store has **one currency**, set at creation and **immutable** once the store has any Active product. Stores needing another currency create a new store.
* **Per-Product** (Flat or Matrix) attributes:
  - Stored: `base_price` (tax-excluded `Money`), `tax_rate` (decimal percentage, e.g., `18.00`).
  - Derived, cached, recomputed on change: `tax_amount = round(base_price × tax_rate / 100)`, `published_price = base_price + tax_amount`.
* **ProductVariant (Matrix)** inherits the parent Product's `tax_rate`; only `price_override` is variant-level. Variant effective values are derived from `(price_override ?? parent.base_price) × parent.tax_rate`. **No per-variant tax rate** — tax category attaches to the product.
* **Quote construction** is owned by the **Order context**; Catalog supplies item prices and tax breakdowns as inputs. No "quoted price" stored in Catalog.

### 5.13 Vouchers (Promotion context)

The **Promotion** context owns vouchers and voucher usage. Store-scoped — each store creates its own vouchers; no platform-wide promotions in v1 ([ADR-0007](decisions/0007-pricing-tax-and-vouchers.md)).

* **Voucher** attributes: `code` (case-insensitive, unique within store); `description` (`LocalizedText` — see §5.14); `discount_type` (`percentage` | `fixed_amount`); `discount_value`; optional `max_total_uses`, `max_uses_per_user`, `starts_at`, `expires_at`, `minimum_cart_value`; `status` (`Active` | `Disabled`).
* `Expired` and `Exhausted` are **derived states** (not stored). A voucher is **usable** when `status == Active`, not Expired, not Exhausted, and `now >= starts_at` (if set).
* **Voucher discount applies to `base_price`** (pre-tax); tax recomputes on the discounted base. GST/VAT-compliant — taxes apply to the actual receivable.
* **Voucher application is Order-mediated.** Order calls `Promotion.ValidateVoucher(...)` at quote time; on `confirm`, calls `Promotion.RecordVoucherUsage(...)`. `VoucherUsage` is the authoritative redemption log and enforces usage limits.
* **One voucher per order in v1.** No stacking.
* Out of scope for v1: platform-wide promotions, rule-based engines (BOGO, automatic cart discounts), customer-specific pricing tiers, time-bound list-price changes (sales windows on the catalog price itself).

### 5.14 Localization

Multi-language content is a v1 feature. Decisions in [ADR-0008](decisions/0008-localization-and-localizedtext.md).

* **`LocalizedText`** is a domain value object representing a string with translations: `entries: Map<bcp47_tag, string>`. Every `LocalizedText` must contain an entry for the **platform default locale `id`** (Bahasa Indonesia). Reads use `get(locale)` which falls back to the default if the requested locale is not present.
* **Locale tags** follow BCP 47 throughout (`id`, `en`, `en-ID`, `ms`, `jv`, etc.).
* **Translatable fields** (subject to `LocalizedText`):
  - **Store**: name, description, public contact display.
  - **Product**: name, description.
  - **ProductAttribute** (Matrix): name, and optionally the labels of allowed values (value identity itself stays locale-neutral).
  - **Media**: alt_text.
  - **Voucher**: description.
  - **PlatformCategory**: name.
* **Locale-neutral** regardless of language: identifiers, slugs, SKUs, voucher codes, `Money` amounts, currency codes, timestamps, status enums.
* **`Store.supported_locales`** is an ordered list of BCP 47 tags the store declares it publishes in. Always includes `id`; default at creation `[id]`. Owners can add more. Used by the storefront language switcher, the Beckn provider descriptor, and Admin UX. Adding a locale does *not* require backfilling translations — missing locales fall back to `id`.
* **Uniqueness checks** on translatable fields (e.g., `Product.name`) apply to the default-locale value (`id`). Cross-locale collisions are not checked.
* **Bridge locale handling**: each Beckn response carries a per-request locale preference; the Bridge resolves each `LocalizedText` via `get(requested_locale)` and declares the chosen locale in the response. **No automatic translation**: an unauthored locale falls back to the default.
* **PlatformCategory** labels are authored by System Admins as `LocalizedText`. Missing translations degrade gracefully to the default locale; admin notification of gaps is operational.

### 5.15 Identity & Access

The Identity & Access context owns User identity and Sessions. Authentication mechanics are **fully outsourced to an external IdP** ([ADR-0009](decisions/0009-identity-and-external-idp.md)). Full entity model in [`design/identity.md`](design/identity.md).

* **OIDC-agnostic integration.** The platform authenticates via standard OpenID Connect. The specific IdP (Clerk, Auth0, Supabase Auth, Cognito, Keycloak, …) is configurable infrastructure, not a code-level commitment. No password storage, MFA management, or recovery flows live in our codebase.
* **User entity** — two-tier profile model:
  - **IdP-canonical** (refreshed from the IdP on every sign-in; read-only in-app): `email`, `email_verified_at`, `display_name`.
  - **Platform-owned** (initial value mirrored from IdP at provisioning; mutable in-app thereafter): `preferred_locale`, `avatar_url`.
  - System-managed: `id` (internal opaque, the cross-context reference key), `external_subject_id` (IdP `sub`, immutable, unique), `status` (`Active` | `Disabled`), timestamps.
* **Just-in-Time provisioning.** No separate sign-up endpoint in our system; on a User's first successful IdP authentication, the User record is created with mirrored profile and `status = Active`.
* **Email verification** is trusted from the IdP's `email_verified` claim. Action gates (creating a store, accepting an invitation, placing an order) check `email_verified_at`.
* **Session** is our own opaque-token entity, independent of the IdP's token: `(id, user_id, created_at, last_used_at, expires_at, active_store_id, device_label, last_ip)`. `active_store_id` is where §5.6's active-store attribute lives. Default idle lifetime 30 days. Operations: sign-out (one), sign-out-everywhere (all sessions for a user).
* **User lifecycle**: `Active` | `Disabled`. **No deletion.** Disabled users cannot sign in and have all their sessions terminated; their record stays so Memberships, orders, audit, and voucher-usage references remain intact. PII scrubbing is deferred to Gap 17.
* **Single linked identity in v1.** A User has exactly one `external_subject_id`. Multi-IdP linking (e.g., Google + Apple on one User) is explicitly deferred; if needed later, introduce a separate `ExternalIdentity` entity.
* **Beckn does not see User data.** Buyers on the network are referenced via opaque order-level identifiers established by the Order context — never by `User.id` or `external_subject_id`.

### 5.16 Domain Events

Cross-context communication and audit ingestion flow through a **transactional outbox + asynchronous dispatcher** ([ADR-0011](decisions/0011-domain-events.md)). Full design and the registry of every declared event live in [`design/events.md`](design/events.md).

* **Transactional outbox.** Every event-emitting context writes events to a local `outbox` table in the same DB transaction as the state change. A background dispatcher delivers them to subscribers. This guarantees state-event consistency and survives crashes.
* **Envelope** — every event carries: `event_id` (UUID; dedup key), `event_name` (`<context>.<verb_past>`, e.g., `catalog.product_published`), `event_version` (integer), `occurred_at` and `recorded_at` timestamps, `aggregate_type` and `aggregate_id`, `actor` (`{user_id, impersonated_user_id, active_store_id}` — impersonation pair per §5.7), `correlation_id`, `causation_id`, and `payload`.
* **Delivery semantics: at-least-once.** Subscribers MUST dedup by `event_id`. Exactly-once is not provided.
* **Ordering.** Per-aggregate order is preserved. Cross-aggregate and cross-context order are not guaranteed.
* **Versioning.** Additive payload changes don't bump `event_version`; breaking changes do. Multiple versions may coexist; subscribers handle the versions they understand and skip unknown ones forward-compatibly.
* **Retention.** Events durable for an operational window (default 90 days). Replay within retention is supported. Cold-start beyond retention queries current state via the Application Layer.
* **Audit is downstream**, not the event log itself. Audit subscribes broadly, transforms into Audit records with its own (longer) retention.
* **Bridge is event-driven.** It subscribes to the events that drive Beckn republication; re-projection is idempotent. The Bridge does not emit domain events.
* **Topology-neutral.** The same pattern works in a modular monolith (in-process dispatch) and in a distributed deployment (message bus). Envelope and naming are unchanged.
* **Domain language only.** Event names and payloads carry domain vocabulary — no transport, no UI, no Beckn names (per §2.6 and §4).
* **LocalizedText in payloads.** Events carrying translatable content carry the full `LocalizedText` (per §5.14), not a single rendering. Subscribers resolve their locale at consumption time.

The **event registry** (in `design/events.md`) is the authoritative catalog — currently ~50 declared events across Tenancy, Identity, Catalog, Inventory, and Promotion contexts. The **subscriber registry** lists known consumers: `beckn-bridge` (republication), `audit` (history), and `inventory-catalog-subscriber` (auto-StockLevel lifecycle).

### 5.17 Cross-context Consistency

Cross-context interactions follow a small set of rules ([ADR-0012](decisions/0012-cross-context-consistency.md)) that build on §5.16 (Domain Events).

* **Strong within, eventual across.** Each context is internally strongly consistent (one DB, transactional aggregates). Across contexts, state propagates via events; expect a small (typically millisecond-scale) inconsistency window.
* **No cross-context transactions.** A single use case writes to one context per transaction. Even in a modular monolith where the DB physically allows multi-context writes, the architecture forbids it — this preserves the service-extraction path declared in §2.9.
* **Multi-context flows are orchestrated (v1), not saga-managed.** When a flow spans contexts (most notably order placement, when Gap 11 lands), the orchestrating context's use case calls other contexts in sequence via Application-Layer ports. On failure, the orchestrator explicitly **compensates** (e.g., releases a reservation). Sagas / process managers may be introduced later if longer-running flows justify them.
* **Read freshness is per-query.** Transactional reads — those that gate a state change (e.g., reserving inventory at checkout) — MUST be synchronous against the owning context. Browse-time reads MAY be eventual / cached / projection-based.
* **Failure handling: retry, then stuck.** Subscribers that fail transiently are retried with exponential backoff (operational config). After N attempts, the event is moved to a `stuck_events` table; the subscriber's offset does NOT advance. Operators review and either retry or skip-with-acknowledgement. There is no silent drop.
* **Inbox dedup.** Each subscriber maintains a `processed_events` table keyed by `(subscription_name, event_id)`. Duplicate deliveries become no-ops. Together with the outbox (§5.16), this is the **outbox + inbox** pattern.
* **Cross-context ports.** A context never imports another's internal types or storage. Cross-context calls go through Application-Layer ports defined by the caller and implemented by the callee as an adapter. In a monolith, ports resolve to in-process calls; if services are extracted later, to RPC/HTTP. Calling code is unchanged.

These rules formalize what the system has implicitly relied on since the first ADRs. They make the cost of breaking them visible.

### 5.18 First-party Idempotency

Mutating use cases that create new state support **client-supplied idempotency keys** to dedup retries from UIs, mobile apps, and internal API clients ([ADR-0013](decisions/0013-first-party-idempotency.md)).

* The caller generates a UUID per logical operation and sends it with the request (HTTP header `Idempotency-Key` or equivalent at other transports).
* The same UUID across retries of the same operation → server returns the cached result, no duplicate state, no duplicate event.
* The same UUID with a different payload → typed error (`idempotency_key_reused_with_different_payload`). Surfaces client bugs early.
* Each context with mutating use cases maintains a small `idempotency_records` table keyed by `(user_id, idempotency_key)`. Records are written **in the same DB transaction** as the state mutation(s) and outbox row(s). Default retention 24 hours (operational).
* **Naturally-idempotent** operations (those that set state to a target value, like `SetStoreStatus(Paused)`) do not need explicit keys — calling them twice already produces the same outcome.
* **Reads** never use idempotency keys.

Together with the outbox (§5.16) and inbox (§5.17), this completes the at-least-once-safety posture: producers retry safely, subscribers dedup safely, and clients retry safely.

### 5.19 Soft-Delete Pattern and the Audit Context

Two related concerns settled together ([ADR-0014](decisions/0014-soft-delete-and-audit.md), [`design/audit.md`](design/audit.md)).

#### Soft-delete: codified system-wide pattern

Entities fall into two categories:

* **Business entities — never deleted.** End-of-life is a **state transition to a terminal-but-retained state** (e.g., Product → Archived, User → Disabled, Invitation → Expired). Data is retained indefinitely (subject to PII compliance, Gap 17). Identifiers stay bound to the entity for life — no reuse.
* **Operational entities — hard-deleted on schedule.** Sessions, stuck events, inbox `processed_events`, outbox dispatched rows, idempotency records. Cleanup is operational (background jobs, DB TTLs).

**Default for any new entity**: business unless clearly operational. Adding deletion to a business entity requires a new ADR.

#### Audit (bounded context)

A new bounded context. Subscribes to **all mutation events** across all contexts; transforms each into an `AuditRecord` with a longer retention than the event log.

* **One AuditRecord per consumed event**, carrying actor (with impersonator per §5.7), aggregate type/id, before/after states where relevant, a summary, the `correlation_id`, and the full `source_envelope` (archival snapshot).
* **Append-only at the DB role level.** The Audit subscriber's DB role has `INSERT` only; no role has `UPDATE`; a separate cleanup role has `DELETE` scoped to expired records only.
* **Access** is capability-gated (§5.5):
  - System Admin reads platform-wide.
  - Platform-scoped users with capability read cross-tenant.
  - Store Owners / Admins read records for their store.
  - Users read records where they were the actor or the impersonated party.
  - Buyers cannot read audit.
* **Retention** is per-category (configurable): financial / order events ~7 years; business state changes ~2 years; identity / session ~1 year. Tunable by compliance (Gap 17).
* **Cryptographic chaining** (tamper evidence beyond append-only roles) is deferred to a future ADR if compliance demands.

Audit emits no domain events — it's a sink. PII handling within audit records is refined by §5.20.

### 5.20 PII Handling and Right-to-Erasure

PII is a cross-cutting policy overlay across every context that touches personal data ([ADR-0015](decisions/0015-pii-and-right-to-erasure.md), [`design/pii.md`](design/pii.md)). The compliance regime for v1 is Indonesia's PDP law, GDPR-compatible by design.

* **Right-to-erasure: PII scrubbing in place.** When a User exercises erasure, the User record is retained (foreign references stay valid); PII fields are replaced with deterministic placeholders per the PII catalog. Audit records get their `source_envelope` PII fields scrubbed in place — record itself stays.
* **PII catalog** in `design/pii.md` is authoritative — per entity, per event payload — declaring which fields are PII and what scrub action each takes. New entities or events with PII MUST update the catalog at introduction.
* **PII boundary across contexts.** Outside of events, Identity & Access owns raw PII; other contexts hold only `user_id`. Events MAY carry PII as snapshots (for audit usefulness); subscribers must treat PII-tagged fields as scrubbable.
* **Audit append-only constraint is qualified.** A dedicated `audit_pii_scrubber` DB role has field-level `UPDATE` on `source_envelope` only. All other audit fields and roles remain immutable per §5.19.
* **`ScrubUser(user_id, by_actor, reason)`** is the use case that orchestrates erasure across contexts. Idempotent. Emits `identity.user_pii_scrubbed`. Cross-context orchestration via Application-Layer scrub ports per §5.17.
* **Auto-scrub on Disable**: configurable window (default off). Available for stricter regimes.
* **Logging discipline.** Domain and application code use a PII-aware redacting logger (Infrastructure). Raw PII never appears in logs.
* **Cross-store / platform analytics**: aggregate-only; no per-user / per-store PII crosses tenant boundaries.
* **Processor catalog** (in `design/pii.md`) lists every third party that receives PII (IdP, email service, Beckn participants, observability vendor) with their data scope. Data minimization in transit.
* **Encryption baseline**: TLS in transit + DB-level at rest. Field-level encryption on demand per specific field.
* **Data residency**: Indonesia-resident storage by default; enforced operationally, not by the domain.

### 5.21 Order & Fulfillment

The Order context is the integration point for the commerce track — it composes Catalog, Inventory, Promotion, Identity, and Localization into end-to-end flows ([ADR-0017](decisions/0017-order-and-fulfillment.md), [`design/order.md`](design/order.md)). The v2 wire vocabulary at the Bridge boundary uses `Contract`, `Resource`, `Offer`; discovery is **CDS-mediated** (BPP publishes catalogs; does not field `/discover`).

**Order state machine**: `Created` → `Initiated` → `Confirmed` → `Fulfilled`, with `Cancelled` (pre-fulfillment) and `Expired` (Quote TTL) as alternative terminals. Wire mapping: `DRAFT` / `DRAFT` / `ACTIVE` / `COMPLETE` / `CANCELLED`. No deletion (consistent with §5.19); audit retention bucket is financial (~7 years).

**Quote**: stored on the Order at `Created`, immutable thereafter. Line items carry **captured snapshots** — `base_price`, `tax_rate`, derived `tax_amount` and `published_price`, plus `LocalizedText` name/description per ADR-0008. Voucher discount, if applied, captures voucher terms. Default TTL: 15 minutes (operational). Snapshots ensure receipt stability when underlying prices or voucher terms change.

**Buyer**: polymorphic. First-party buyer is a `User.id` (from §5.15). Beckn buyer is `{ bap_id, transaction_id }` at `Created`; the contact snapshot (`name`, `email`, `phone`, `address`) is populated at `Initiated` per v2's "no PII at /select" rule.

**Payment**: external to BPP in v1. `payment_status` (`Pending` / `Authorized` / `Captured` / `Refunded` / `Failed`) is driven by external signals (Beckn-side or future gateway webhooks). BPP doesn't store card data or capture payments — PCI scope is avoided.

**Fulfillment**: self-fulfilled by stores. `fulfillment_status` (`Pending` / `Preparing` / `Shipped` / `Delivered`) is advanced by store admins via store-scoped use cases; projected to wire `Contract.performance`. Logistics integration deferred.

**Cancellation**: pre-fulfillment only in v1. Cancel-from-Initiated releases reservations; cancel-from-Confirmed reverts voucher usage (new event `promotion.voucher_usage_reverted`) and marks payment Refunded (actual refund external). Returns / refunds post-fulfillment deferred.

**Orchestration**: `Order.Initiate`, `Order.Confirm`, and `Order.Cancel` are Application-Layer orchestrations (per §5.17) that call into Inventory, Promotion, and (where relevant) Identity through declared ports, with **hand-coded compensation** on partial failure. No Saga framework in v1.

**Bridge ↔ Order**: the Bridge exposes wire handlers `/select`, `/init`, `/confirm`, `/status`, `/cancel` (inbound from BAP) and corresponding `/on_*` callbacks. The Bridge owns transaction_id correlation, signature verification, and the domain ↔ wire mapping (Order ↔ Contract, line items ↔ Commitments, fulfillment_status ↔ Performance, etc.). Order stays protocol-naive.

**Catalog distribution**: the Bridge publishes catalog updates to **CDS via `/catalog/publish`** on the events it already subscribes to (`tenancy.store_status_changed`, `catalog.*`). The CDS endpoint(s) and credentials are operational configuration (like the registry per §4.1).

**Domain events** (joining the registry): `order.quote_created`, `order.initiated`, `order.confirmed`, `order.cancelled`, `order.expired`, `order.marked_preparing`, `order.marked_shipped`, `order.fulfilled`, `order.payment_status_changed`. Promotion gains `promotion.voucher_usage_reverted`.

**Deferred** for v1: `/track`, `/update`, `/rate`, `/support` handlers; refund / return workflows; payment-gateway integration; platform-managed logistics; multi-shipment / split orders; subscriptions; marketplace fees / payouts; ION `/raise` and `/reconcile`.

---

## 6. Invitation & Role Model (Conceptual)

### 6.1 Why Invitations Are First-Class

Adding someone to a store is not a single act — it is a **process** that may span time, may be declined, may expire, and may be revoked. The invitation is the entity that holds the state of that process.

### 6.2 Invitation Lifecycle

An Invitation moves through explicit states:

* **Pending** — created, awaiting recipient action.
* **Accepted** — recipient has confirmed and a Membership has been created.
* **Declined** — recipient has refused.
* **Revoked** — sender (or another authorized party) cancelled the invitation before acceptance.
* **Expired** — the validity window passed without action.

State transitions are one-way (no resurrecting an expired invitation; a new one is issued). Each transition has an actor and a timestamp.

### 6.3 Invitation Targeting

An Invitation targets a recipient **by email** ([ADR-0010](decisions/0010-invitation-account-reconciliation.md)). The Invitation is not bound to a `User.id` at creation; whether the recipient already has a platform User account is a runtime check at acceptance time. With IdP-mediated identity (§5.15), "recipient has an account" and "recipient is creating an account" unify — in both cases, the User is the one whose IdP-verified email matches the invitation's email.

The Invitation carries the intended role, the target store, the issuer, the validity window, and a single-use acceptance token. The token is opaque, non-guessable, and verifiable without exposing the underlying identifier.

Invitations create **Admin Memberships only**. Owner is set at store creation; ownership transfer uses §6.6, not the invitation flow.

### 6.4 Acceptance Semantics

Acceptance is the atomic transition from `Invitation(Pending)` to `Membership(Active)` ([ADR-0010](decisions/0010-invitation-account-reconciliation.md)). It is a domain operation, not a UI flow.

**Reconciliation rule.** An Invitation can be Accepted by a User iff all of the following hold:

* `User.email == Invitation.email` (case-insensitive, against the IdP-canonical email).
* `User.email_verified_at` is set (per §5.15 / ADR-0009).
* `User.status == Active`.
* The Invitation is `Pending` and not expired.

If any condition fails, `AcceptInvitation` returns a typed error and changes no state.

**Linking is always explicit.** Users discover their Invitations via either the email link (`/invitations/<token>`) or a pending-invitations panel in their dashboard, and must click Accept. There is no auto-linking on sign-in.

**Email mismatch** is a typed error — there is no manual claim with an alternate email. The issuer must Revoke and re-issue with the correct address.

**Already a Member.** Idempotent: acceptance marks the Invitation `Accepted` and returns the existing Membership; no duplicate is created.

**Repeated Accept attempts** on a `Pending` Invitation produce exactly one Membership; subsequent calls find the Invitation already `Accepted` and return the same Membership.

**Multiple pending Invitations** for the same email are independent — each Accepted, Declined, or Revoked separately.

### 6.5 Role Assignment Strategy

* Roles are assigned at the **Membership** level, not at the User level.
* The initial role set is small and intentional (Owner, Admin). Adding roles later is a deliberate design action — a new role must be justified by a distinct set of capabilities, not by a vague "we might need it."
* Capabilities map to roles via a documented matrix. The matrix is owned by the Tenancy context, not scattered across feature code.
* Role changes are auditable events — who changed what, when, on whose authority.

### 6.6 Ownership Transitions

Ownership transfer is a distinct, deliberate operation — *not* a role change. It requires explicit confirmation from both parties (current owner relinquishes; new owner accepts) and is logged as a first-class event. Ownership is never "promoted into" through normal admin actions.

---

## 7. Use of Reference Projects

### 7.1 `/Users/danielignatius/mydev/personal/bitemycart`

**Treat as inspiration for ecommerce shape, not as a template.**

Use it to understand:

* What concepts a small-retailer store typically needs (catalog structures, product variants, attributes, media, vouchers, orders).
* How storefront and admin flows tend to be organized.
* What edge cases real ecommerce systems handle (out-of-stock, variant pricing, discount semantics).

Do **not**:

* Copy its database schema. It is built for a different stack, a different scale, and a single-tenant model.
* Adopt its file structure, framework choices, or naming conventions verbatim.
* Assume its modeling decisions are correct for a multi-tenant aggregator — many will not be.

When in doubt: read it to learn *what problems exist*, then design the solution from first principles for *our* context.

### 7.2 `/Users/danielignatius/mydev/personal/ion-specs`

**Treat as the source of truth for Beckn / ION protocol — and only that.**

Use it to:

* Look up exact field names, structures, and required attributes when designing the Beckn Bridge.
* Source real example payloads for Bridge-layer tests.
* Understand the protocol's flows (search, select, init, confirm, etc.) and error formats.
* Verify protocol-version differences.

Do **not**:

* Let its structures influence domain entity names or shapes.
* Treat its schema as a database schema.
* Import its vocabulary into anything outside the Beckn Bridge.
* Assume its nesting reflects how data should be stored.

**Rule of thumb:** if you find yourself reading `ion-specs` while designing a domain entity, stop. You are designing the Beckn Bridge, not the domain.

---

## 8. Development Rules & Constraints

### 8.1 Hard Rules (Non-Negotiable)

* The Domain Layer **must not** reference, import, or know about Beckn or `ion-specs` in any form.
* No Beckn field name (`descriptor`, `provider`, `resource`, `offer`, `contract`, `commitment`, `consideration`, `performance`, `context`, `intent`, etc.) appears outside the Beckn Bridge.
* The system **must** remain operable, testable, and demonstrable without any Beckn participant in the loop.
* No database schema may be designed by reading a Beckn JSON sample. Schemas are derived from the domain model.
* Every cross-tenant operation requires an explicit tenancy assertion — there is no "ambient tenant."

### 8.2 Decision-Making Heuristics

When facing an architectural choice, prefer:

* **Boring over clever.** Established patterns over novel ones, unless the novelty pays for itself.
* **Explicit over implicit.** Named states, named transitions, named roles, named capabilities.
* **Composition over inheritance.** Especially for cross-context features.
* **Domain language over technical language.** Code reads like the business, not like the framework.
* **Reversible over irreversible.** Decisions that can be undone are preferred to ones that lock the system in.
* **One source of truth.** For any fact, there is exactly one place it is authoritatively stored.

### 8.3 When to Pause and Ask

Pause and surface the question to the user when:

* A proposed change would add a new bounded context or split an existing one.
* A modeling decision has more than one defensible answer and the trade-offs are significant.
* A feature request implicitly couples the domain to Beckn.
* A performance/scalability concern is invoked to justify abandoning a clean design.
* The Beckn Bridge is asked to do something that looks like business logic.

### 8.4 Evolution Strategy

* **Record decisions.** Significant architectural choices are captured as lightweight decision records (problem, options, decision, consequences). They are appended to, not rewritten.
* **Refactor the domain freely; refactor the Bridge cautiously.** The Bridge is the contract with the outside world; the domain is internal and yours to reshape.
* **Beckn version migrations live in the Bridge.** Adopting a new Beckn version should not produce a domain-level diff.
* **Deprecate before deleting.** Domain concepts that are no longer used are marked, observed for a release, then removed.
* **Schema changes pass the domain-first test.** If a schema change cannot be motivated without referencing a Beckn field, it is the wrong change.

### 8.5 Anti-Patterns to Reject on Sight

* A `beckn_*` table, column, or field anywhere outside the Bridge.
* A domain entity whose attributes mirror a Beckn object 1:1.
* Authorization checks that ask "is admin?" without naming the store.
* "Current tenant" pulled from a global, request-scoped, or ambient source at the domain level.
* A single God-table that holds products for all stores without scoping by store identity.
* Invitations represented as a boolean flag on a User row.
* Direct database access from the Beckn interface, bypassing the Application Layer.
* Beckn message construction scattered across multiple layers.

---

## 9. Working Agreement

This document is the **contract** between the architecture and every contributor. It is not aspirational; it is binding for the current phase.

If a future requirement appears to violate one of these principles, the correct first response is not to bend the principle — it is to:

1. State the principle the requirement violates.
2. Propose an alternative that honors the principle.
3. If no such alternative exists, surface the conflict to the user with a clear trade-off analysis.

Only after that conversation should the architecture itself be revisited. The point of writing this charter is to make architectural drift visible — not to prevent change, but to make sure change is deliberate.

---

## 10. Working Artifacts & Path to the Final Design

This charter is one artifact in a small system of related documents that, taken together, will be consolidated into a **single design document** for handoff to the implementing team. That final document — not this charter — is the foundation the team will use to build the application.

Everything authored in this phase should be written with that destination in mind. Avoid duplication; prefer composition by reference.

### 10.1 Artifact Map

* **`CLAUDE.md`** (this file) — the charter. Principles, rules, boundaries. Kept dense and current. The voice of *what must be true*.
* **`gaps/`** — open design questions, one per file. Each names the gap, references where the charter touches (or fails to touch) it, lists open questions, implications, and dependencies. Resolved gaps move to `gaps/resolved/`.
* **`decisions/`** — Architecture Decision Records (ADRs). One per resolved gap or significant judgment call. Captures context, options considered, decision, and consequences. **Immutable once accepted** — superseded by new ADRs, never rewritten.
* **`design/`** — sub-design documents for subsystems too detailed for the charter (e.g., the Catalog model, the Order lifecycle). Each design doc is backed by one or more ADRs.
* **`handoff/`** — the consolidated **team-facing design package** (see §10.3). Multi-file; integrates the charter + ADRs + sub-designs into a navigable reading order for the implementing team.

Templates live at `decisions/0000-template.md` and `design/0000-template.md`.

### 10.2 Resolution Workflow

For each gap:

1. Discuss the gap and converge on a decision.
2. Author a new ADR in `decisions/` (next available number) capturing the reasoning. Status starts as `Proposed`.
3. If the resolution is substantial enough to need its own sub-design, create or extend a file in `design/` referencing the ADR.
4. Update CLAUDE.md with the *rule* — typically a paragraph or short list in the relevant section — and link to the ADR for the reasoning.
5. Move the gap file from `gaps/` to `gaps/resolved/` with a one-line pointer to its ADR.
6. Mark the ADR `Accepted`.

A tiny rule with no alternatives worth recording may skip the ADR and go straight into CLAUDE.md. The ADR exists to capture *judgment calls under uncertainty* — not every decision needs that ceremony.

When an existing accepted ADR is revisited, write a new ADR that **supersedes** it. Update the old ADR's status to `Superseded by ADR-NNNN`. Then update CLAUDE.md to reflect the new rule.

### 10.3 The Consolidated Final Design

The eventual deliverable is the **`handoff/`** directory — a multi-file design package the implementing team will use as the foundation for building the application. It is *not* a copy of this charter; it is the integration of everything produced by this process, presented in a reading order optimized for an engineer joining the team.

Directory shape:

```
handoff/
├── README.md                  Reader's guide + table of contents
├── 01-overview.md             What the system is, goals, glossary
├── 02-principles.md           Architectural principles
├── 03-beckn-integration.md    The Bridge, network identity, wire vocabulary
├── 04-bounded-contexts/       The seven contexts in detail
│   ├── README.md
│   ├── 4.1-identity.md
│   ├── 4.2-tenancy.md
│   ├── 4.3-catalog.md
│   ├── 4.4-inventory.md
│   ├── 4.5-promotion.md
│   ├── 4.6-order.md
│   └── 4.7-audit.md
├── 05-cross-cutting.md        Authorization, events, consistency, idempotency, soft-delete, PII, localization
├── 06-operational.md          Testing, observability, deployment topology, external dependencies
├── 07-open-issues.md          Deferred work and known limits
└── 08-references.md           ADR index, design/ sub-design index, external references
```

Authoring rules (per §10.4): `handoff/` references the charter and ADRs rather than copying them. It presents the integrated view; deep "why?" questions go back to the ADRs.

The `handoff/` package is authored only when the gap backlog is sufficiently resolved that a coherent integration is possible. Until then, ADRs and design docs accumulate; the charter stays current; `handoff/` is the consolidation step.

### 10.4 Rules of Authorship

* The charter never grows into an encyclopedia. If a section starts to bloat with reasoning, move the reasoning into an ADR and leave the rule.
* ADRs never rewrite history. A wrong decision is corrected by a new ADR that supersedes the old.
* Design docs reference, never duplicate, the charter and the ADRs.
* `handoff/`, when authored, references the charter and ADRs by section rather than copying — but presents the material in a reading order optimized for the implementing team, not for design-time iteration.
* Anything written here is fair game for `handoff/`. Anything that would be embarrassing to put in front of the implementing team needs revision now, not later.
