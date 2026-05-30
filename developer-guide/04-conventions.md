# 4. Conventions

How code is shaped, named, and wired in the BPP package. These conventions are how the architectural rules from [`02-repo-layout.md`](02-repo-layout.md) survive contact with day-to-day implementation.

| Section | Topic |
|---|---|
| 4.1 | Reading this section |
| 4.2 | Naming |
| 4.3 | The use-case envelope |
| 4.4 | Ports and adapters wiring |
| 4.5 | Error handling |
| 4.6 | Logging discipline |
| 4.7 | Domain event authoring |
| 4.8 | Repository pattern |
| 4.9 | HTTP route conventions |
| 4.10 | Cross-context ports |
| 4.11 | Migrations |

---

## 4.1 Reading this section

Conventions here have two flavors:

- **Required:** architecture-imposed. They're how the handoff's cross-cutting rules manifest in code. Apply regardless of stack.
- **Lean:** concrete pattern using the confirmed stack (TypeScript + Hono + awilix + Drizzle + Zod + RHF + shadcn/ui). Subject to monorepo override.

Where examples appear they use the v1 confirmed stack. Same patterns translate to other frameworks; the *shape* is what matters.

---

## 4.2 Naming

A short naming table prevents most future churn. **Apply consistently** — the lint can't catch most violations, but mismatched names create cognitive load that compounds.

### Files and folders

| Subject | Convention | Example |
|---|---|---|
| Bounded context (folder) | lowercase, singular | `contexts/identity/` |
| Source files | kebab-case | `create-store.ts`, `user-repository-port.ts` |
| Test files (co-located) | mirror source, `.test.ts` | `create-store.ts` + `create-store.test.ts` |
| Test files (mirrored tree) | mirror source path | `tests/unit/contexts/identity/...` |
| Schema files | `<entity>.schema.ts` | `user.schema.ts` |
| Migration files | numbered + double-underscore + name | `0042__order_state_machine.sql` |

### Code identifiers

| Subject | Convention | Example |
|---|---|---|
| Entity / aggregate | PascalCase, singular | `User`, `Store`, `Product`, `Order` |
| Value object | PascalCase, singular | `Money`, `LocalizedText`, `BCP47Tag` |
| Use case | PascalCase verb phrase | `CreateStore`, `AcceptInvitation`, `ConfirmOrder` |
| Port (interface) | PascalCase + `Port` suffix | `UserRepositoryPort`, `AuthorizationPort`, `CatalogReadPort` |
| Adapter (implementation) | Descriptive + `Adapter` suffix | `DrizzleUserRepositoryAdapter`, `OidcAuthorizationAdapter` |
| Domain event type | PascalCase past-tense | `UserProvisioned`, `StoreActivated`, `OrderConfirmed` |
| Domain event name (wire) | snake_case namespaced past-tense | `identity.user_provisioned`, `order.confirmed` |
| Typed error | PascalCase + `Error` suffix | `AuthorizationDeniedError`, `InvariantViolationError` |
| Function | camelCase, verb-leading | `validateVoucher`, `releaseReservation` |
| Constant | UPPER_SNAKE_CASE | `DEFAULT_LOCALE`, `QUOTE_TTL_MINUTES` |

### Database

| Subject | Convention | Example |
|---|---|---|
| Table | snake_case, plural | `users`, `stores`, `product_variants`, `voucher_usages` |
| Column | snake_case | `created_at`, `published_price`, `external_subject_id` |
| Index | `idx_<table>_<columns>` | `idx_orders_store_id_status` |
| Foreign key (within context) | `<entity>_id` | `store_id`, `product_id` |
| Cross-context reference | `<context>_<entity>_id` if disambiguation helps | `actor_user_id`, `subject_user_id` |
| Enum value | snake_case | `published`, `pending_review` |

### Capabilities

`<resource>.<verb>` — lowercase, snake-or-dot delimited.

Examples: `product.publish`, `voucher.disable`, `org.create`, `store.activate`, `audit.read.platform`.

The capability catalog lives in `shared/auth/capability-catalog.ts`; treat it as a single source of truth.

---

## 4.3 The use-case envelope

Every mutating use case in the Application Layer follows the same shape. The envelope is a small in-house abstraction that owns the cross-cutting concerns; the use-case body owns only the domain logic.

### The shape (Required)

```
For every mutating use case:

1. Parse + validate input          (Zod schema from packages/bpp-contracts)
2. Resolve actor                   (from the per-request DI scope)
3. requireCapability(name, scope)  (raises AuthorizationDenied on fail)
4. Idempotency check               (look up by user_id + Idempotency-Key)
   - if cached: return cached result, skip body
   - if conflict: raise IdempotencyKeyReusedWithDifferentPayload
5. Begin transaction
   a. Read aggregate(s) via repository ports
   b. Apply domain logic (state transitions, invariants)
   c. Persist via repository ports
   d. Append domain events to outbox          (same transaction)
   e. Write idempotency record                (same transaction)
6. Commit
7. Return domain-shaped result
```

Steps 1–4 and the transactional boundary (5 + 6) are the envelope's job. The use-case body only does step 5a–5c (read, apply, persist) and step 5d (declare which events to emit).

### Concrete shape (Lean — TypeScript + awilix + Drizzle)

```ts
// shared/kernel/use-case.ts (illustration)
type UseCase<Input, Output> = (input: Input) => Promise<Output>;

interface UseCaseEnvelope {
  <Input, Output>(spec: {
    name: string;
    inputSchema: ZodSchema<Input>;
    requiredCapability: (input: Input) => { name: string; scope?: Scope };
    body: (ctx: UseCaseCtx, input: Input) => Promise<{
      result: Output;
      events: DomainEvent[];
    }>;
  }): UseCase<Input, Output>;
}
```

```ts
// contexts/tenancy/application/use-cases/create-store.ts
export const CreateStore = useCase({
  name: 'tenancy.create_store',
  inputSchema: CreateStoreInput,                    // from bpp-contracts
  requiredCapability: (input) => ({
    name: 'store.create',
    scope: { org_id: input.orgId },
  }),
  body: async (ctx, input) => {
    const org = await ctx.ports.orgRepo.requireById(input.orgId);
    const store = Store.create({
      org,
      name: input.name,
      currency: input.currency,
    });
    await ctx.ports.storeRepo.insert(store);
    return {
      result: { storeId: store.id },
      events: [StoreCreatedEvent.from(store, ctx.actor)],
    };
  },
});
```

The envelope wraps `body` with the boilerplate (validation, auth, idempotency, transaction, outbox append). The body never opens a transaction itself, never touches the outbox table directly, never calls `requireCapability`.

### Rules

- **Use cases are functions, not classes** (lean) — easier to test, easier to compose, no implicit state.
- **Domain logic stays in the domain**, called from the use-case body. The body should read like a paragraph of business logic.
- **No HTTP / no Beckn vocabulary in use cases.** Validation schemas come from `bpp-contracts`; capability scopes are domain identifiers; events are domain events.
- **Idempotency keys are mandatory on mutating routes**, optional on naturally-idempotent operations (per [§5.4 of handoff](../handoff/05-cross-cutting.md)).

---

## 4.4 Ports and adapters wiring

### Where things live

| Layer | Folder | Naming |
|---|---|---|
| Port (interface) | `contexts/<X>/application/ports/` | `UserRepositoryPort`, `CatalogReadPort` |
| Adapter (implementation) | `contexts/<X>/infrastructure/adapters/` or `infrastructure/repositories/` | `DrizzleUserRepositoryAdapter` |
| DI registration | `composition-root.ts` | one block per context |

### DI registration (Lean — awilix)

```ts
// composition-root.ts (illustration)
import { createContainer, asFunction, asValue, asClass, InjectionMode } from 'awilix';

const container = createContainer({ injectionMode: InjectionMode.PROXY });

container.register({
  // Singleton: platform-infra adapters
  db: asValue(drizzleClient),
  logger: asValue(redactingLogger),
  idpClient: asValue(oidcClient),

  // Per-context repositories
  userRepo: asFunction(DrizzleUserRepository).singleton(),
  storeRepo: asFunction(DrizzleStoreRepository).singleton(),

  // Cross-context ports — adapter implemented by callee
  catalogReadPort: asFunction(CatalogReadPortAdapter).singleton(),
  inventoryReservationPort: asFunction(InventoryReservationPortAdapter).singleton(),

  // Cross-cutting
  authorizationPort: asFunction(MatrixBasedAuthorizationAdapter).singleton(),
  outbox: asFunction(PostgresOutbox).singleton(),
});

// Per-request scope: actor, correlation_id, active_org_id, active_store_id
function makeRequestScope(req: Request) {
  const scope = container.createScope();
  scope.register({
    actor: asValue(extractActor(req)),
    correlationId: asValue(req.headers['x-correlation-id'] ?? newCorrelationId()),
    activeOrgId: asValue(req.session.active_org_id),
    activeStoreId: asValue(req.session.active_store_id),
    idempotencyKey: asValue(req.headers['idempotency-key']),
  });
  return scope;
}
```

### Rules

- **Ports are interfaces** (TypeScript interfaces or type aliases). No implementation.
- **Adapters implement exactly one port.** No "general utility" adapters.
- **Per-request scope** is mandatory for: actor, correlation_id, active_org_id, active_store_id, idempotency_key, logger (so it carries the request context automatically).
- **Singletons** for: DB pool, IdP client, OTel SDK, outbox dispatcher, AuthorizationPort.
- **No service locator.** Don't reach into the container from inside a use case — receive dependencies through the envelope's `ctx`.

---

## 4.5 Error handling

A three-layer error hierarchy that maps cleanly to HTTP responses.

### Layers

| Layer | Examples | Origin |
|---|---|---|
| **Domain** | `InvariantViolationError`, `StateTransitionNotAllowedError`, `EntityNotFoundError` | Thrown from inside entities and aggregates |
| **Application** | `AuthorizationDeniedError`, `IdempotencyKeyReusedWithDifferentPayloadError`, `QuoteExpiredError`, `VoucherNotApplicableError` | Thrown from the use-case envelope or use-case body |
| **Infrastructure** | `DatabaseUnavailableError`, `IdpTimeoutError`, `SignedUrlGenerationFailedError` | Wrapped at the adapter boundary; never propagated raw |

### Base shape (Lean — TypeScript)

```ts
// shared/kernel/errors.ts (illustration)
export abstract class TypedError extends Error {
  abstract readonly code: string;       // stable string for clients / logs
  abstract readonly httpStatus: number; // mapping for HTTP-layer translation
  readonly meta?: Record<string, unknown>;
}

export class AuthorizationDeniedError extends TypedError {
  readonly code = 'authorization_denied';
  readonly httpStatus = 403;
  constructor(readonly capability: string, readonly scope?: Scope) { super('Authorization denied'); }
}

export class InvariantViolationError extends TypedError {
  readonly code = 'invariant_violation';
  readonly httpStatus = 422;
  constructor(message: string, readonly meta?: Record<string, unknown>) { super(message); }
}
// ... etc.
```

### Stance: throw, don't return

- **Throw typed errors.** Don't use `Result<T, E>` or similar. Throwing keeps the use-case body readable; the envelope catches and translates.
- **One throw site, one error type.** Don't reuse the same error type for unrelated failures.
- **Never throw raw `Error`** from the domain or application. Always a `TypedError` subclass.
- **Infrastructure errors are wrapped at the adapter boundary.** A Postgres connection error becomes `DatabaseUnavailableError`; the use-case body never sees raw driver errors.

### HTTP translation (Bridge / admin-api)

The HTTP boundary catches `TypedError` and emits a structured response:

```json
{
  "error": {
    "code": "authorization_denied",
    "message": "Authorization denied",
    "meta": { "capability": "store.create" }
  }
}
```

Unknown errors (non-`TypedError`) → 500 with generic message. Stack trace logged but never returned to the client.

The **Bridge** translates further into Beckn error codes via the mapping registry (per [§4.5 of handoff](../handoff/03-beckn-integration.md)).

### Rules

- **No silent catch.** Every `catch` either re-throws, wraps, or explicitly handles a specific error case with a comment explaining why.
- **No `console.error`.** Use the logger.
- **Errors carry domain identifiers, not transport concerns.** No HTTP status in domain errors; the HTTP layer maps via the `httpStatus` field on `TypedError`.

---

## 4.6 Logging discipline

### Required

- **All logs go through the redacting logger** ([§5.6.6 of handoff](../handoff/05-cross-cutting.md)). No `console.*` in production code.
- **Standard fields on every log entry**:
  - `timestamp`, `level`, `service`, `context_name`
  - `correlation_id` (from per-request scope)
  - `user_id` (if authorized to log)
  - `active_org_id`, `active_store_id`
- **PII fields are redacted** via the PII catalog from `design/pii.md`. Email, name, IP — never logged in plain.
- **Structured (JSON) at the wire**, even if pretty-printed locally.

### What to log

| Event | Level | Notes |
|---|---|---|
| Unexpected error caught at HTTP boundary | `error` | Include stack |
| Domain / Application error thrown | `warn` | Code + meta; no stack (these are control flow) |
| Authorization denied | `warn` | Emit domain event too — Audit picks it up |
| Use case completed | `info` | Use-case name, duration; no payload (PII risk) |
| State transition (e.g., Order.confirmed) | `info` | Aggregate type + ID, before/after state |
| Outbox delivery success | `debug` | Don't drown logs |
| Outbox delivery failure | `warn` (transient) or `error` (stuck) | |
| Bridge signature verification failure | `warn` | Page on rate spike |
| Routine reads | (don't log) | Use traces |

### Lean (pino + per-request child logger)

```ts
// platform-infra/observability/logger.ts (illustration)
const baseLogger = pino({
  redact: pinoRedactionPathsFromPiiCatalog(),  // from design/pii.md
  formatters: { level: (label) => ({ level: label }) },
});

// Per-request scope (DI):
container.register({
  logger: asFunction(({ correlationId, actor, activeOrgId, activeStoreId }) =>
    baseLogger.child({
      correlation_id: correlationId,
      user_id: actor?.userId,           // redacting layer scrubs if needed
      active_org_id: activeOrgId,
      active_store_id: activeStoreId,
    })
  ).scoped(),
});
```

### Rules

- **Don't co-log PII with `user_id`**. Names and emails are PII; user IDs are not. The redacting layer masks PII fields by name.
- **Don't log payloads.** Log the use-case name and the result shape (e.g., `{ storeId }`), not the input.
- **Don't log secrets.** Never. Even masked.
- **Don't pre-format messages.** Pass structured fields; let the logger serialize.

---

## 4.7 Domain event authoring

### Naming

- **Type name**: PascalCase past-tense (`UserProvisionedEvent`, `StoreStatusChangedEvent`).
- **Wire name**: `<context>.<verb_past>` (`identity.user_provisioned`, `tenancy.store_status_changed`).
- Always past tense — events describe what *happened*.

### Where it lives

```
contexts/<X>/domain/events/
├── store-created.ts                 type + Zod schema for payload
├── store-status-changed.ts
└── …
```

### Shape (Lean)

```ts
// contexts/tenancy/domain/events/store-status-changed.ts
import { z } from 'zod';
import { defineEvent } from 'shared/kernel/events';
import { StoreId, StoreStatus } from '../value-objects';

export const StoreStatusChangedPayload = z.object({
  storeId: StoreId.schema,
  previousStatus: StoreStatus.schema,
  newStatus: StoreStatus.schema,
});

export const StoreStatusChangedEvent = defineEvent({
  name: 'tenancy.store_status_changed',
  version: 1,
  aggregateType: 'Store',
  payloadSchema: StoreStatusChangedPayload,
  category: 'business_state',         // drives Audit retention bucket
});
```

The `defineEvent` helper produces an envelope-aware factory; the use-case body just calls `StoreStatusChangedEvent.from(store, oldStatus, newStatus, ctx.actor)` and appends to `events`.

### Versioning

- **Version 1 to start.** No bump on additive changes (new optional field).
- **Bump on breaking change** (removed field, changed required-ness, type change).
- **Old versions remain in the event registry** ([§5.2.5 of handoff](../handoff/05-cross-cutting.md)). Subscribers handle what they understand; skip unknown versions forward-compatibly.

### Registry

Every event must be registered in `shared/domain-events/registry.ts` (or equivalent — mirrors [`design/events.md`](../design/events.md)). The registry enumerates valid event names and lets Audit's broad subscription discover new events automatically.

### Rules

- **Domain language only.** Names and payloads use domain vocabulary. No HTTP, no UI, no Beckn.
- **`LocalizedText` carried whole** in event payloads ([§5.7.7 of handoff](../handoff/05-cross-cutting.md)) — never resolved to a single locale at emit time.
- **PII in payloads is tagged**, picked up by the PII catalog, scrubbed on right-to-erasure.
- **Events are immutable.** Never edit a published event's payload.

---

## 4.8 Repository pattern

### Required

- **One repository per aggregate root**, owned by the context.
- **Repository ports declared in `application/ports/repositories/`** as interfaces.
- **Repository adapters implemented in `infrastructure/repositories/`**.
- **Methods accept and return domain types** — `Store`, not `StoreRow`. Mapping happens inside the repository.
- **Transactions are opened by the use-case envelope**, not by repositories. Repositories accept a transaction client passed in.

### Shape (Lean — Drizzle + TypeScript)

```ts
// contexts/tenancy/application/ports/repositories/store-repository-port.ts
export interface StoreRepositoryPort {
  insert(store: Store, tx: TxClient): Promise<void>;
  update(store: Store, tx: TxClient): Promise<void>;
  findById(storeId: StoreId, tx: TxClient): Promise<Store | null>;
  requireById(storeId: StoreId, tx: TxClient): Promise<Store>;  // throws EntityNotFoundError
}
```

```ts
// contexts/tenancy/infrastructure/repositories/drizzle-store-repository.ts
export class DrizzleStoreRepository implements StoreRepositoryPort {
  async insert(store: Store, tx: TxClient): Promise<void> {
    await tx.insert(stores).values(toRow(store));
  }
  async update(store: Store, tx: TxClient): Promise<void> {
    await tx.update(stores).set(toRow(store)).where(eq(stores.id, store.id));
  }
  async findById(storeId: StoreId, tx: TxClient): Promise<Store | null> {
    const row = await tx.select().from(stores).where(eq(stores.id, storeId)).limit(1);
    return row[0] ? fromRow(row[0]) : null;
  }
  async requireById(storeId: StoreId, tx: TxClient): Promise<Store> {
    const store = await this.findById(storeId, tx);
    if (!store) throw new EntityNotFoundError('Store', storeId);
    return store;
  }
}
```

### Schema declaration (Drizzle)

```ts
// contexts/tenancy/infrastructure/schema/stores.schema.ts
export const stores = pgTable('stores', {
  id: text('id').primaryKey(),
  orgId: text('org_id').notNull(),
  status: text('status', { enum: ['draft', 'active', 'suspended', 'paused'] }).notNull(),
  name: jsonb('name').$type<LocalizedTextJson>().notNull(),     // LocalizedText.entries
  currency: text('currency').notNull(),                          // ISO 4217
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
});
```

### Rules

- **`toRow` / `fromRow` mappers live in the repository file.** Keep them next to the queries.
- **No cross-context joins.** A repository never queries another context's tables ([§5.3.7 of handoff](../handoff/05-cross-cutting.md)). To get cross-context data, call the other context's use case via a cross-context port (§4.10).
- **`requireById` throws; `findById` returns null.** Pick the right one at the call site.
- **Repositories never start transactions.** They accept a `tx` client. The use-case envelope owns the transactional boundary.

---

## 4.9 HTTP route conventions

### Required

- **Routes validate input with Zod schemas from `bpp-contracts`.** No hand-written parsing.
- **Routes call use cases by token.** A route's body is: validate → look up use case → invoke → translate result.
- **Mutating routes accept `Idempotency-Key` header.**
- **Routes don't contain business logic.** If a route has a conditional that isn't "did the use case succeed," that logic belongs in a use case.
- **Errors are translated to structured JSON.** Stack traces never reach the client.

### Lean (HTTP framework middleware)

```ts
// contexts/tenancy/interfaces/http/create-store.route.ts
const handler = route({
  method: 'POST',
  path: '/orgs/:orgSlug/stores',
  request: CreateStoreInput,                  // from bpp-contracts
  response: CreateStoreOutput,
  capability: 'store.create',
  handler: async (ctx, input) => {
    return await ctx.useCases.CreateStore(input);
  },
});
```

The middleware composition (capability check, idempotency, error translation) is provided by the chosen framework. Hono lean: middleware composed via `.use(authMiddleware)`, etc.

### Pagination

- **Cursor-based.** No offset/limit. Clients send `cursor` + optional `limit`; server returns `items` + `nextCursor`.
- **Default limit: 50.** Max: 200.
- **Cursor is opaque.** Base64-encoded server-side state.

### URL conventions

Match the org/store hierarchy from [§5.1.5 of handoff](../handoff/05-cross-cutting.md):

```
POST   /orgs/:orgSlug/stores
GET    /orgs/:orgSlug/stores/:storeSlug
POST   /orgs/:orgSlug/stores/:storeSlug/products
PATCH  /orgs/:orgSlug/stores/:storeSlug/products/:productId/publish
```

### Rules

- **The URL slug must match the session's active values.** Mismatch is an authorization failure ([§5.1.5 of handoff](../handoff/05-cross-cutting.md)). The capability-check middleware verifies this.
- **No nested resources beyond two levels deep.** If a URL grows past `/orgs/:o/stores/:s/<thing>`, push the structure into the resource shape, not the path.
- **Bridge routes live in `bridge/inbound/`**, not in `interfaces/http/` of any context. The Bridge is its own HTTP family.

---

## 4.10 Cross-context ports

### Required

- **The caller declares the port.** Order declares `CatalogReadPort` based on what Order needs from Catalog.
- **The callee implements the adapter.** Catalog provides an adapter that satisfies Order's port, internally calling Catalog's own use cases.
- **Adapters never reach into the callee's domain or infrastructure.** They go through use cases.

### Shape (Lean)

```ts
// contexts/order/application/ports/external/catalog-read-port.ts (caller declares)
export interface CatalogReadPort {
  getCapturableSnapshot(productId: ProductId, tx: TxClient): Promise<{
    name: LocalizedText;
    description: LocalizedText;
    basePrice: Money;
    taxRate: number;
  } | null>;
}

// contexts/catalog/infrastructure/adapters/catalog-read-port-for-order-adapter.ts (callee implements)
export class CatalogReadPortAdapter implements CatalogReadPort {
  constructor(private readonly getProductDetail: typeof GetProductDetail) {}
  async getCapturableSnapshot(productId, tx) {
    const product = await this.getProductDetail({ productId }, tx);
    if (!product) return null;
    return {
      name: product.name,
      description: product.description,
      basePrice: product.basePrice,
      taxRate: product.taxRate,
    };
  }
}
```

### Rules

- **Port lives with the caller** (Order owns the file). Catalog has no clue what `CatalogReadPort` is — it just implements whatever interface gets registered.
- **The adapter is the only Catalog-side code that knows about Order's port.** It bridges the two.
- **No reverse imports.** Order never imports anything from Catalog.

See [§2.7 of `02-repo-layout.md`](02-repo-layout.md) for the cross-context port catalog (v1).

---

## 4.11 Migrations

Covered structurally in [§2.9 of `02-repo-layout.md`](02-repo-layout.md). Conventions for authoring:

### Required

- **One migration per logical change.** Don't bundle "create users table" with "add audit table."
- **Numbered + named.** `0042__order_state_machine.sql`. Numbers establish order globally.
- **Reviewable as SQL.** Plain SQL files (or framework-generated SQL files, which are also plain SQL).
- **Never drop business-entity tables.** Operational tables may be hard-deleted ([§5.5 of handoff](../handoff/05-cross-cutting.md)).
- **Migrations are forward-only in production.** Down migrations exist for local dev; in production, roll forward with a new migration.

### Lean (Drizzle)

```bash
# Author schema in contexts/<X>/infrastructure/schema/
# Generate migration from diff:
pnpm drizzle-kit generate --name add_quote_ttl_to_orders

# Review the SQL file in migrations/, edit if needed, commit.

# Apply locally:
pnpm drizzle-kit migrate
```

### Rules

- **Never edit a committed migration.** If it's wrong, add a new migration that fixes it. The migration ledger is append-only in production.
- **Schema changes that need data migration get two migrations**: one for the schema change, one for the data backfill. Keep them small and reviewable separately.
- **Don't reference enums by integer.** Use named enum values (handoff convention — Postgres `enum` type or `text` + check constraint).

---

> **Next**: [`03-dev-setup.md`](03-dev-setup.md) — local environment, IdP sandbox, seed data. Or [`07-testing.md`](07-testing.md) — per-layer testing posture made concrete.
