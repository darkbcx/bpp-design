# 2. Repo layout

The shape of the BPP **package** within a host monorepo, and how that shape mechanically enforces the architectural rules from the handoff. This section is **stack-agnostic** — the rules apply regardless of language, framework, ORM, build tool, or monorepo orchestration choice.

| Section | Topic |
|---|---|
| 2.1 | Reading this section |
| 2.2 | Required properties of the layout |
| 2.3 | Reference layout (one model) |
| 2.4 | Per-context internal layout |
| 2.5 | The Bridge package |
| 2.6 | Shared packages |
| 2.7 | Cross-context dependencies and ports |
| 2.8 | Lint enforcement |
| 2.9 | Tests, migrations, configuration |
| 2.10 | What lives outside the BPP package |

---

## 2.1 Reading this section

This guide describes the **BPP package** — the unit of code that implements the BPP. Where the package lives in the host monorepo (e.g., `apps/bpp/`, `services/bpp/`, `packages/bpp/`) is a **monorepo convention**, not ours. Adopt the monorepo's existing pattern.

Within the package, the layout must mechanically enforce the rules from the handoff:

- The **dependency rule** ([§2.3 of handoff](../handoff/02-principles.md)) — Domain ← Application ← Infrastructure / Interface.
- **Bounded-context isolation** ([§2.4 of handoff](../handoff/02-principles.md), [§5.3.7](../handoff/05-cross-cutting.md)) — no cross-context internal-type imports.
- **The Bridge is isolatable** ([§2.7 of handoff](../handoff/02-principles.md), [§6.3 of handoff](../handoff/06-operational.md)) — Beckn vocabulary contained in one place.
- **Soft-delete pattern + business / operational entities** ([§5.5 of handoff](../handoff/05-cross-cutting.md)).

If your chosen monorepo has alternative conventions that satisfy these, use them. The required properties are below; the reference layout is one example.

---

## 2.2 Required properties of the layout

A layout that satisfies the architecture has all of these:

1. **One folder per bounded context.** Identity, Tenancy, Catalog, Inventory, Promotion, Order, Audit each get a top-level folder. The seven contexts from [§4 of handoff](../handoff/04-bounded-contexts/README.md) are the *only* contexts in v1.
2. **Each context has internal layer separation.** Domain, Application, Infrastructure, and Interface code is distinguishable (separate folders or clearly marked subtrees).
3. **The Bridge is a sibling of contexts**, not a child of any. It's a top-level Interface-layer adapter, not part of any context. **Beckn vocabulary appears only here.**
4. **Cross-context imports go through Application-Layer ports.** A context never imports another context's Domain or Infrastructure files.
5. **Shared cross-cutting code lives in named shared packages**, not in random utility folders. Each shared package has a single, defensible reason to exist.
6. **The lint configuration enforces the rules.** Boundary violations fail the build at PR time, not at code-review.
7. **Tests sit next to (or directly mirror) the code they cover.** Per-context tests don't reach into other contexts.
8. **Migrations live in a single owned location** with an authoritative ordering.
9. **No `index.ts` re-exports that bypass module boundaries.** A barrel file that re-exports internal types defeats the boundary lint. Be deliberate about what's exported.

These properties are stable across any layout the host monorepo prefers.

---

## 2.3 Reference layout (one model)

A concrete shape that satisfies all the Required properties. Adapt names to monorepo conventions.

```
<host-monorepo>/
└── apps/bpp/                                  the BPP backend service package
    ├── src/
    │   ├── contexts/
    │   │   ├── identity/                      §4.1 Identity context
    │   │   ├── tenancy/                       §4.2 Tenancy
    │   │   ├── catalog/                       §4.3 Catalog
    │   │   ├── inventory/                     §4.4 Inventory
    │   │   ├── promotion/                     §4.5 Promotion
    │   │   ├── order/                         §4.6 Order
    │   │   └── audit/                         §4.7 Audit
    │   ├── bridge/                            §3 Beckn Bridge — sole protocol-aware zone
    │   ├── shared/                            cross-context shared code (see §2.6)
    │   │   ├── kernel/                        use-case envelope, base errors, result types
    │   │   ├── value-objects/                 Money, LocalizedText, BCP47Tag, ISO4217Code, …
    │   │   ├── domain-events/                 envelope schema, dispatcher, subscriber base
    │   │   └── auth/                          AuthorizationPort + capability catalog
    │   ├── platform-infra/                    third-party adapters consumed via ports
    │   │   ├── db/                            connection pool, migration runner, base repository
    │   │   ├── outbox-dispatcher/             dispatcher + LISTEN/NOTIFY wake
    │   │   ├── idp/                           OIDC client adapter
    │   │   ├── object-storage/                signed-URL provider
    │   │   ├── email/                         email-sending adapter
    │   │   ├── observability/                 OTel SDK + redacting logger wrapper
    │   │   └── beckn-network/                 signing, registry client, CDS client (used by /bridge only)
    │   ├── admin-api/                         first-party HTTP surface for the Admin UI
    │   ├── composition-root.ts                wires DI container; binds ports to adapters
    │   └── main.ts                            entry point
    ├── migrations/
    │   ├── 0001__initial.sql
    │   ├── 0002__identity.sql
    │   └── …
    ├── tests/
    │   ├── unit/                              domain + application unit tests
    │   ├── integration/                       real-DB integration tests
    │   └── e2e/                               end-to-end Beckn flow tests (Bridge against stub BAP)
    ├── package.json
    └── README.md

apps/bpp-admin/                                Admin UI package (own deploy target)
packages/bpp-contracts/                        Zod schemas shared between bpp and bpp-admin
```

Notes on this model:

- `contexts/` is the **vertical** decomposition (per business concern); inside each context is the **horizontal** layering.
- `shared/` is for code that's genuinely **cross-context kernel** — not a junk drawer. New entries here require justification.
- `platform-infra/` holds **third-party adapters** (DB pool, IdP client, OTel SDK). These are stateless / global and are wired into the DI container at app startup. Each context's own adapters live in its own `infrastructure/` folder (see §2.4).
- `bridge/` is a sibling of `contexts/`. It calls **Application-Layer use cases** from any context but never imports their internal types.
- `admin-api/` is the first-party HTTP surface — also calls Application-Layer use cases. **It cannot import from `bridge/`** and vice versa ([§2.4.4 of handoff](../handoff/02-principles.md)).
- The Admin UI lives in a **separate package** (`apps/bpp-admin/`). Sharing happens through `packages/bpp-contracts/`.

---

## 2.4 Per-context internal layout

Each context in `contexts/` follows the same internal shape:

```
contexts/<context>/
├── domain/                  the *what* — pure business model
│   ├── entities/            aggregate roots and entities
│   ├── value-objects/       context-specific value objects
│   ├── events/              this context's domain event definitions (envelope + payload schemas)
│   ├── invariants/          invariants and state-transition guards
│   └── errors/              domain-specific typed errors
├── application/             the *how* — use cases + ports
│   ├── use-cases/           one file per use case
│   ├── ports/               interfaces this context declares; adapters implement them
│   │   ├── repositories/    persistence ports (this context's aggregates)
│   │   ├── external/        outbound calls to other contexts (Application-Layer port from the callee)
│   │   └── outbox.ts        write-to-outbox port (implementation usually shared)
│   ├── event-subscribers/   handlers for events this context subscribes to
│   └── policies/            optional — authorization policies, allowed-state matrices
├── infrastructure/          the *with what* — adapter implementations for this context's ports
│   ├── repositories/        DB-backed repository adapters
│   ├── adapters/            other concrete adapter implementations
│   └── schema.sql           (or schema.ts) declarations for this context's tables, if not centralized
├── interfaces/              the *for whom* — transport-specific surfaces this context exposes
│   ├── http/                HTTP routes / handlers, if this context publishes a first-party surface
│   └── events/              event publishers (typically just register against outbox + envelope)
└── tests/
    ├── unit/                domain + application unit tests
    └── integration/         repository + adapter integration tests
```

### Rules within a context

- **`domain/` imports nothing outside its own folder.** Pure types and pure functions only.
- **`application/use-cases/` imports `domain/` + `application/ports/`** of the same context. May call `application/ports/external/` for cross-context calls.
- **`application/ports/` are interfaces only.** No implementations.
- **`infrastructure/` imports `application/ports/` and `domain/`** of the same context. Implements ports. Touches the DB, message bus, IdP client, etc.
- **`interfaces/` imports `application/`** of the same context. Translates external requests into use-case invocations.
- **No internal context imports from `interfaces/` or `infrastructure/` upward.** That violates the Dependency Rule.

### What aggregates a context's surface area

Use cases at the top of `application/use-cases/` are the only thing other contexts (or the Bridge, or `admin-api/`) ever call. Everything below — entities, ports, adapters, schemas — is the context's private interior.

---

## 2.5 The Bridge package

The Bridge is its own top-level folder, structurally separate from any context. It is the **sole** place Beckn vocabulary may appear.

```
bridge/
├── inbound/                 handlers for messages BAP → BPP
│   ├── select.ts            validates + maps + invokes Order.CreateQuote
│   ├── init.ts              validates + maps + invokes Order.Initiate
│   ├── confirm.ts           validates + maps + invokes Order.Confirm
│   ├── status.ts
│   └── cancel.ts
├── outbound/                callbacks BPP → BAP (on_select, on_init, on_confirm, …)
│   ├── on-select.ts
│   ├── on-init.ts
│   └── …
├── catalog-publisher/       publishes catalog updates to CDS on relevant events
├── mapping-registry/        explicit domain ↔ wire mappings (per §4.6 of handoff)
│   ├── order-to-contract.ts
│   ├── line-item-to-commitment.ts
│   ├── product-to-resource.ts
│   ├── store-to-provider.ts
│   ├── catalog-projection.ts
│   ├── voucher-to-offer.ts
│   ├── fulfillment-to-performance.ts
│   ├── locale.ts
│   ├── errors.ts            domain-error ↔ Beckn-error-code mapping
│   └── README.md            the registry index — what maps to what, per protocol version
├── protocol/                Beckn-specific protocol concerns
│   ├── signing.ts
│   ├── verification.ts
│   ├── envelope.ts
│   ├── context-block.ts
│   ├── correlation.ts       transaction_id / message_id correlation
│   └── versions/            per-Beckn-version handler sets (if multiple versions in flight)
│       └── v2/
└── beckn-types/             types reflecting the wire schema; manually or from ion-specs
```

### Rules

- **Imports allowed**: `shared/value-objects/`, `shared/kernel/`, `platform-infra/beckn-network/`, and any `contexts/<context>/application/use-cases/` (their public surface).
- **Imports forbidden**: any context's `domain/`, `infrastructure/`, `interfaces/`. The Bridge invokes use cases by their public surface only.
- **The Application Layer never imports from `bridge/`.** Verified by lint.
- **The Admin UI (`admin-api/`) never imports from `bridge/`** and vice versa. The two adapter families are siblings.
- **`mapping-registry/`** is documented as a first-class artifact ([§4.6 of handoff](../handoff/03-beckn-integration.md)). Per-version mappings live there; protocol-version evolution is a Bridge-only change.

---

## 2.6 Shared packages

`shared/` and `platform-infra/` are the only places code is allowed to live outside `contexts/` and `bridge/`. They have narrow, defensible reasons.

### `shared/kernel/`

The use-case envelope abstraction (validate → authorize → idempotency check → transaction → emit → return), base error types, and the Result / Outcome shape if used. Roughly:

- `use-case.ts` — the wrapper that every use case is built on top of.
- `errors.ts` — `AuthorizationDenied`, `IdempotencyKeyReusedWithDifferentPayload`, `InvariantViolation`, `NotFound`, etc.
- `result.ts` — typed result shape (success / error union), if the team prefers explicit Result over throws.

### `shared/value-objects/`

The domain-language value objects from the handoff:

- `Money` — `{ amount: bigint; currency: ISO4217Code }`.
- `LocalizedText` — `{ entries: Record<BCP47Tag, string> }` with an invariant requiring the default-locale key.
- `BCP47Tag`, `ISO4217Code`, `Email`, `Slug`, `Url` — branded primitive types.
- `Id<TBrand>` — opaque ID factory.

Value objects have **no dependency on any context** and **no I/O**.

### `shared/domain-events/`

The event envelope schema, dispatcher interface, subscriber base. Per [§5.2 of handoff](../handoff/05-cross-cutting.md):

- `envelope.ts` — the universal envelope shape (event_id, event_name, version, actor, etc.).
- `dispatcher.ts` — interface for the outbox-dispatcher (implementation lives in `platform-infra/outbox-dispatcher/`).
- `subscriber.ts` — base class / interface for inbox-dedup-aware subscribers.

### `shared/auth/`

The cross-cutting authorization machinery from [§5.1 of handoff](../handoff/05-cross-cutting.md):

- `authorization-port.ts` — the `AuthorizationPort` interface (`requireCapability`, `hasCapability`).
- `capability-catalog.ts` — the code-side enumeration of all capabilities (`product.publish`, `voucher.disable`, etc.).
- `role-catalog.ts` — role enumeration.

The matrix itself (role → capabilities) is **runtime data**, not code — managed by System Admins via the in-band tool ([§5.1.3 of handoff](../handoff/05-cross-cutting.md)). The catalog files just enumerate what's possible.

### `platform-infra/`

Adapters to third-party systems, consumed via ports declared in contexts:

| Subfolder | What lives here | Consumed by |
|---|---|---|
| `db/` | Connection pool, migration runner, base repository helpers | All contexts' `infrastructure/repositories/` |
| `outbox-dispatcher/` | Outbox dispatcher + LISTEN/NOTIFY wake + inbox-dedup | `shared/domain-events/` consumers |
| `idp/` | OIDC client | Identity context's `infrastructure/` |
| `object-storage/` | Signed-URL provider | Catalog context's `infrastructure/` |
| `email/` | Email-sending adapter | Tenancy context's `infrastructure/` (invitations) |
| `observability/` | OTel SDK + redacting logger | All contexts |
| `beckn-network/` | Signing, registry client, CDS client | `bridge/` only |

`platform-infra/` adapters never import from `contexts/` or `bridge/` — they are upstream from everything else.

### `packages/bpp-contracts/` (separate monorepo package)

Zod schemas shared between `apps/bpp/` (the backend) and `apps/bpp-admin/` (the frontend). One source of truth for request / response shapes.

- Per-use-case input + output schemas.
- Value-object Zod parsers.
- No business logic — schemas only.

---

## 2.7 Cross-context dependencies and ports

### The pattern

A context calls another context only through the **callee's Application-Layer port surface**, which the **caller declares** as a port and the **callee implements** as an adapter.

Concretely:

```
contexts/order/
├── application/
│   └── ports/
│       └── external/
│           ├── catalog-port.ts        interface — Order's view of what it needs from Catalog
│           ├── inventory-port.ts      interface — Order's view of what it needs from Inventory
│           └── promotion-port.ts      interface — Order's view of what it needs from Promotion
…

contexts/catalog/
└── infrastructure/
    └── adapters/
        └── catalog-port-for-order-adapter.ts   implements Order's CatalogPort
                                                using Catalog's own use cases
```

The DI container wires these together at app startup.

### Why this shape

- **Order owns its view of Catalog.** It declares exactly what it needs (e.g., "get the captured-snapshot fields for a product"). Catalog can change its internals without Order knowing.
- **The interface lives with the caller**, so Order's compilation depends only on its own port file. The callee implements the contract.
- **In a future service extraction** (per [§6.3.3 of handoff](../handoff/06-operational.md)), only the adapter changes — from in-process call to RPC. Calling code stays the same.

### Catalog of cross-context ports in v1

Approximately (full list emerges as use cases land):

| Caller | Port | Implemented by |
|---|---|---|
| Order | `CatalogReadPort` (snapshot lookup) | Catalog |
| Order | `InventoryReservationPort` (reserve / convert / release) | Inventory |
| Order | `PromotionVoucherPort` (validate / record / revert) | Promotion |
| Tenancy | `IdentityUserPort` (lookup user by ID for member listing) | Identity |
| All | `AuthorizationPort` | Tenancy |
| All | `AuditEventPort` (subscribe) | Audit |
| Bridge | every context's Application use case (called as-is) | each context |

### Anti-pattern to reject on sight

```
// contexts/order/application/use-cases/confirm.ts
import { ProductRepository } from '../../../catalog/infrastructure/...'  // FORBIDDEN
```

This reaches into Catalog's internals. The lint catches it (§2.8).

---

## 2.8 Lint enforcement

The bounded-context rules are enforced **mechanically at build time** — not in code review.

### Required rules

Whatever lint plugin is used, the following constraints must be enforceable:

| Rule | Plain English |
|---|---|
| **R1** | Files under `contexts/<X>/domain/` may import only from the same folder (no application, no infra, no other contexts, no `bridge/`). |
| **R2** | Files under `contexts/<X>/application/use-cases/` may import from `contexts/<X>/domain/` and `contexts/<X>/application/ports/`, plus `shared/*` and `packages/bpp-contracts`. They may NOT import from `contexts/<X>/infrastructure/` or any other context's internal subtree. |
| **R3** | Files under `contexts/<X>/infrastructure/` may import from `contexts/<X>/domain/` + `contexts/<X>/application/`, plus `shared/*` and `platform-infra/*`. |
| **R4** | Files under `bridge/` may import from `contexts/<*>/application/use-cases/` (public surface), `shared/*`, and `platform-infra/beckn-network/`. They may NOT import from any context's `domain/` or `infrastructure/`. |
| **R5** | Files outside `bridge/` may NOT import from `bridge/`. |
| **R6** | Files under `admin-api/` may import from `contexts/<*>/application/use-cases/` and `packages/bpp-contracts`. They may NOT import from `bridge/`. |
| **R7** | Files under `platform-infra/` may NOT import from `contexts/` or `bridge/` (they're upstream of everything else). |
| **R8** | Files under `shared/` may NOT import from `contexts/`, `bridge/`, `admin-api/`, or `platform-infra/`. |

### Example with eslint-plugin-boundaries (illustration only)

```jsonc
{
  "settings": {
    "boundaries/elements": [
      { "type": "domain",        "pattern": "src/contexts/*/domain/**" },
      { "type": "application",   "pattern": "src/contexts/*/application/**" },
      { "type": "infra",         "pattern": "src/contexts/*/infrastructure/**" },
      { "type": "context-iface", "pattern": "src/contexts/*/interfaces/**" },
      { "type": "bridge",        "pattern": "src/bridge/**" },
      { "type": "admin-api",     "pattern": "src/admin-api/**" },
      { "type": "shared",        "pattern": "src/shared/**" },
      { "type": "platform",      "pattern": "src/platform-infra/**" }
    ]
  },
  "rules": {
    "boundaries/element-types": ["error", {
      "default": "disallow",
      "rules": [
        { "from": "domain",        "allow": ["domain", "shared"] },
        { "from": "application",   "allow": ["domain", "application", "shared"] },
        { "from": "infra",         "allow": ["domain", "application", "shared", "platform"] },
        { "from": "context-iface", "allow": ["application", "shared"] },
        { "from": "bridge",        "allow": ["application", "shared", "platform"] },
        { "from": "admin-api",     "allow": ["application", "shared"] },
        { "from": "shared",        "allow": ["shared"] },
        { "from": "platform",      "allow": ["platform"] }
      ]
    }]
  }
}
```

`import/no-restricted-paths` from `eslint-plugin-import` is an equivalent alternative; pick whichever is conventional in the host monorepo.

### Sibling discipline: no barrel re-exports across boundaries

A `contexts/<X>/index.ts` that re-exports `domain/` types defeats the lint. Either:

- **No barrel files** at context level (forces direct path imports — most explicit).
- **Tight barrel files** that only re-export `application/use-cases/*` and explicit DTOs needed by callers (callable surface only).

Whichever the monorepo prefers, **the boundary lint must still trip** on a deliberate violation. Add a CI test that fails on a known-bad import.

---

## 2.9 Tests, migrations, configuration

### Tests

Two patterns are both fine — pick one and apply it consistently:

| Pattern | Tradeoff |
|---|---|
| **Co-located**: tests next to the code (`use-case.ts` + `use-case.test.ts`) | Easiest to maintain; deletes auto-clean |
| **Mirrored tree**: `tests/` mirrors `src/` exactly | Cleaner builds; tests packageable separately; common in Java/Go monorepos |

The reference layout above uses **mirrored tree** at the BPP-package level (separating unit / integration / e2e) — that pattern parallelizes well in CI. Co-located unit tests within a context are also fine — both can coexist.

What stays constant: **tests for one context don't import from another context's internals** (same lint rules apply). Tests cross context only through the Application Layer.

### Migrations

- **One owned location.** `migrations/` at the BPP-package root is the lean.
- **Numbered + named.** `0042__order_state_machine.sql` (or framework-equivalent).
- **One migration per logical change.** Don't bundle.
- **Migrations are reviewable artifacts** — not generated on demand at deploy.
- **Each context may have its own subfolder** if migrations get unwieldy: `migrations/identity/`, `migrations/order/`, etc. Order across contexts is established by a top-level manifest or a global numbering scheme.
- **Migrations never drop business-entity tables** ([§5.5 of handoff](../handoff/05-cross-cutting.md)). Operational tables may be hard-deleted.

### Configuration

Per [§6.5 of handoff](../handoff/06-operational.md):

- **Compile-time** — feature flags, role / capability catalog structure.
- **Deploy-time** — per-environment config (DB URLs, IdP URLs, CDS endpoints, etc.). Loaded from environment variables; secrets from the host monorepo's secret-management layer.
- **Runtime** — matrix grants, voucher / catalog content, operational toggles.

Configuration loading lives in `composition-root.ts` (or wherever DI registration happens). **The Domain Layer never sees configuration.**

---

## 2.10 What lives outside the BPP package

Several things are deliberately **not in the BPP package**, because they're consumed by other monorepo apps too or because they're operational concerns.

| Location | Contents | Why outside |
|---|---|---|
| `packages/bpp-contracts/` | Zod schemas shared between bpp and bpp-admin | Cross-package shared types |
| `apps/bpp-admin/` | The Admin UI | Separately deployable; different stack typically |
| Host monorepo's `infrastructure/` (if exists) | Cross-app DB, observability, IdP shared adapters | If multiple apps share these, they sit at the monorepo level |
| Host monorepo's CI workflows | GitHub Actions / equivalent | Monorepo-wide concern |
| Host monorepo's secret-management layer | Vault / cloud secret manager | Monorepo-wide concern |
| `developer-guide/` (this directory) | Implementation guide | Currently in the design repo; may move when the BPP code lands |
| `handoff/`, `decisions/`, `design/`, `CLAUDE.md` | Architecture references | Design repo; the BPP package may keep pointers to these |

The BPP package focuses on **the BPP itself**. Anything that's cross-app, operational, or design-history sits one level up.

---

## Summary

A layout that satisfies the architecture:

```
apps/bpp/src/
├── contexts/<X>/{domain,application,infrastructure,interfaces}/   one folder per bounded context
├── bridge/                                                        sole protocol-aware zone
├── shared/{kernel,value-objects,domain-events,auth}/              cross-context kernel
├── platform-infra/                                                third-party adapters
├── admin-api/                                                     first-party HTTP surface
└── composition-root.ts                                            DI wiring at startup
```

With lint rules enforcing the dependency rule and the context boundaries. Whatever the host monorepo's conventions dictate for filenames and exact folder placement, **the relationships in this section don't move**.

---

> **Next**: [`03-dev-setup.md`](03-dev-setup.md) — local environment, IdP sandbox, seed data. Or [`04-conventions.md`](04-conventions.md) — naming, error handling, logging, ports + adapters wiring patterns.
