# 2. Architectural principles

This section captures the rules that hold across the entire system. Every detail in later sections respects these. If something seems to conflict with these principles, look harder — the principle wins.

## 2.1 The Dependency Rule

**Dependencies point inward, toward the domain.**

- The **Domain** depends on nothing.
- The **Application** depends only on the Domain.
- **Infrastructure** and **Interface** depend on Application and Domain — never the reverse.
- When an inner layer needs an outward capability (persistence, sending email, calling an external service), it declares a **port** — an abstract contract — that an outer layer implements.

A change in an outer layer must never force a change in an inner layer. A change in the Domain may legitimately require outer layers to adapt — that is the *correct* direction of pressure.

## 2.2 Four logical layers

Innermost to outermost.

### 2.2.1 Domain Layer — the *what*

- Pure business concepts: `User`, `Store`, `Product`, `Order`, etc.
- Owns invariants, identifiers, value objects, state transitions, and domain events.
- Knows nothing about HTTP, JSON, Beckn, persistence engines, frameworks, or transport.
- Testable with no infrastructure and no external fixtures.

### 2.2.2 Application Layer — the *how*

- Use cases that orchestrate the domain (`Order.CreateQuote`, `Catalog.PublishProduct`, …).
- Coordinates transactions, authorization checks, and domain-event publication.
- Speaks only in domain terms. Accepts domain-shaped inputs; returns domain-shaped results. **Never returns transport payloads.**
- Defines **ports** for everything it cannot do itself (persist, notify, enqueue, call another context).

### 2.2.3 Infrastructure Layer — the *with what*

- Adapters that implement the ports declared by inner layers: persistence, messaging, identity providers, file storage, search indices, email delivery.
- **Substitutable.** Swapping a database or identity provider must not ripple inward.
- Owns all I/O, all framework integration, and all third-party SDKs.

### 2.2.4 Interface Layer — the *for whom*

The system's outward-facing surfaces. Three adapter families:

- **First-party UI adapters** — admin/owner console, storefront.
- **First-party API adapters** — internal APIs consumed by our own front-ends.
- **Beckn Bridge adapter** — receives and emits Beckn protocol messages.

Every adapter converts the outside world into Application-Layer calls and converts Application-Layer results back into the outside world's format. **No business logic. No persistence. No cross-adapter coupling.**

### 2.2.5 Request flow

```
External caller → Interface adapter → Application use case → Domain logic
                                            ↓
                                     Infrastructure (ports)
                                            ↓
                                     External systems (DB, queue, ...)
```

## 2.3 Seven bounded contexts

Layers are *horizontal* (technical concerns). **Bounded contexts** are *vertical* (business concerns). A single context — say, Catalog — has its own Domain, its own Application use cases, and its own Infrastructure adapters. Layering and context decomposition are **complementary**, not competing.

The seven contexts:

| Context | What it owns | Detail |
|---|---|---|
| **Identity & Access** | User identity (mirrored from IdP), Sessions, active-store binding, profile state. | [§4.1](04-bounded-contexts/4.1-identity.md) |
| **Tenancy** | Stores, ownership, memberships, roles, invitations, the role-capability matrix. | [§4.2](04-bounded-contexts/4.2-tenancy.md) |
| **Catalog** | Products, variants, attributes, categories, media, pricing structure, store catalogs. | [§4.3](04-bounded-contexts/4.3-catalog.md) |
| **Inventory** | Stock and availability for catalog items; reservations. | [§4.4](04-bounded-contexts/4.4-inventory.md) |
| **Promotion** | Vouchers and voucher usage. | [§4.5](04-bounded-contexts/4.5-promotion.md) |
| **Order & Fulfillment** | Quote, Order entity, payment-status tracking, fulfillment-status tracking. | [§4.6](04-bounded-contexts/4.6-order.md) |
| **Audit** | Append-only history of all state-changing events. | [§4.7](04-bounded-contexts/4.7-audit.md) |

The **Beckn Bridge is not a bounded context** — it is an adapter in the Interface Layer (see [§3](03-beckn-integration.md)). It has no domain.

**Storage-layer rule**: cross-context coupling is forbidden at the storage layer. No foreign keys across context boundaries; no shared tables; no direct joins across contexts. Cross-context references are by ID only.

> ADRs 0001–0017 collectively establish the seven contexts.

## 2.4 Inter-context communication

Two explicit channels, nothing else.

### 2.4.1 Synchronous queries / commands

A context exposes a use-case-level Application Layer API that other contexts may call. Calls are **typed in domain terms** and pass **identifiers and value objects**, never internal records.

Cross-context calls go through **ports** defined by the caller and implemented by the callee:

- In a monolith, ports resolve to in-process method calls.
- If services are extracted later, ports resolve to RPC / HTTP / gRPC. Calling code is unchanged.

### 2.4.2 Domain events

When a context changes state in a way that matters to others (e.g., `tenancy.invitation_accepted`, `catalog.product_published`), it publishes a named event. Interested contexts subscribe.

Events are emitted via a **transactional outbox** — written in the same DB transaction as the state change. A dispatcher delivers to subscribers with **at-least-once** semantics. Subscribers deduplicate via an inbox table (see [§5.2](05-cross-cutting.md) when drafted).

### 2.4.3 Rules

- A context **never** reaches into another context's persistence or domain model directly.
- Identifiers crossing context boundaries are **opaque references**, not foreign keys.
- Events describe *what happened* in domain terms — never transport, never UI, never Beckn vocabulary.

> See: [ADR-0011](../decisions/0011-domain-events.md) (events / transactional outbox), [ADR-0012](../decisions/0012-cross-context-consistency.md) (consistency rules, inbox).

## 2.5 Cross-context consistency

**Strong within a context; eventual across contexts.**

- **Within a context**: strong consistency. A transaction commits all writes for that context's aggregates atomically.
- **Across contexts**: eventual consistency. State propagates via events. Expect a small (typically millisecond-scale) lag between contexts.

A single Application-Layer use case writes to **one context** in its transaction. Cross-context state propagation is **always via events**. Even in a modular monolith — where multiple contexts may share a database — code does not span contexts in a single transaction. This preserves the path to future service extraction.

Multi-step flows spanning contexts (e.g., `Order.Initiate` calls Inventory + Promotion) use **Application-Layer orchestration with explicit compensation**. There is **no Saga framework in v1**.

**Failure handling**: subscribers retry with exponential backoff. After N attempts, an event is marked stuck; an operator reviews and either retries or skips with acknowledgement. No silent drops; no automatic skips.

> See: [ADR-0012](../decisions/0012-cross-context-consistency.md).

## 2.6 The domain stays Beckn-naive

The Beckn protocol is a *consumer* of the domain, not a *definer* of it.

- The Domain **must not** reference, import, or know about Beckn or `ion-specs` in any form.
- **No Beckn field name** (`provider`, `resource`, `offer`, `contract`, `descriptor`, `context`, `intent`, `commitment`, `consideration`, `performance`, …) appears outside the Beckn Bridge.
- The system **must be operable, testable, and demonstrable without any Beckn participant in the loop**. Every store function (create, manage, publish, transact) must work via first-party interfaces alone.
- The Beckn endpoint is *one* interface among several — alongside admin UI, storefront, and internal APIs. **It is not privileged.**

If a request would couple the domain to Beckn (e.g., "add a `descriptor` field to Product"), that's a defect. The Bridge is the only place protocol vocabulary lives. See [§3 Beckn integration](03-beckn-integration.md) for the Bridge's full responsibilities.

## 2.7 Deployment topology is deferred

The architecture defines **logical** boundaries. Whether the system ships as a single deployable (modular monolith), as multiple services aligned to bounded contexts, or as something in between is a **downstream** decision driven by operational, team, and scale concerns.

What's **non-negotiable** regardless of topology:

- Layer boundaries are enforced at the **source level**, not by network distance.
- Context boundaries are enforced by **explicit contracts** (ports, events), not by deployment.
- The Beckn Bridge is **isolatable** — it must be possible to deploy or replace it independently.

## 2.8 No-delete pattern for business entities

Business entities are **never deleted** in the domain — end-of-life is a state transition to a terminal-but-retained state. Operational entities (Sessions, idempotency records, inbox/outbox rows, stuck events) are hard-deleted on schedule.

This applies system-wide:

| Entity | Terminal-retained state |
|---|---|
| Store | Suspended / Paused (per ADR-0003) |
| Product | Archived |
| ProductVariant | Removed (retained if referenced by orders) |
| StockLevel | Inactive |
| User | Disabled |
| Voucher | Disabled (Expired / Exhausted derived) |
| Invitation | Accepted / Declined / Revoked / Expired |
| Order | Fulfilled / Cancelled / Expired |
| Membership | Removed (record retained) |

**Default for any new entity**: business unless clearly operational. Adding deletion to a business entity requires a new ADR.

Right-to-erasure is handled via **PII scrubbing in place** (see [§5.6](05-cross-cutting.md) when drafted) — records are retained; PII fields are replaced with placeholders.

> See: [ADR-0014](../decisions/0014-soft-delete-and-audit.md) (soft-delete pattern and Audit), [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md) (PII handling).

## 2.9 Authorization is universal and explicit

Every Application-Layer use case starts with an explicit `requireCapability(name, scope)` call before any state mutation or side effect. There is no implicit authorization, no shared context that "everyone has access by default."

The model has **three tiers** — System Admin, Platform-scoped, Store-scoped — and capabilities are **action-level** (e.g., `product.publish`, not `manage_products`). The matrix is grants-only; absence means denied.

See [§5.1](05-cross-cutting.md) (when drafted) and ADRs 0002 + 0016 for the full model.

---

> **Next**: [§3 Beckn integration](03-beckn-integration.md) — how the Bridge connects the domain to the network.
