# 7. Testing

How tests are structured, what each layer's tests cover, and the patterns that keep them maintainable. This is the testing-side companion to [`04-conventions.md`](04-conventions.md) and makes [§6.1 of handoff](../handoff/06-operational.md) concrete with the v1 stack.

| Section | Topic |
|---|---|
| 7.1 | Reading this section |
| 7.2 | Layer × test type summary |
| 7.3 | Domain Layer tests |
| 7.4 | Application Layer tests (use cases) |
| 7.5 | Infrastructure Layer tests (adapters) |
| 7.6 | Interface Layer tests (admin-api routes) |
| 7.7 | Beckn Bridge tests |
| 7.8 | End-to-end Beckn flow tests |
| 7.9 | Property-based tests for state machines |
| 7.10 | Concurrency tests |
| 7.11 | Test fixtures and factories |
| 7.12 | DI containers in tests |
| 7.13 | Frontend tests (admin UI) |
| 7.14 | Common gotchas |

---

## 7.1 Reading this section

Two flavors of guidance:

- **Required:** what every contributor must do regardless of stack. These survive any framework / library override.
- **Lean:** concrete patterns using the v1 stack (Vitest, Testcontainers, Supertest, awilix, Drizzle). Subject to monorepo override.

Examples assume Vitest + TypeScript. The shapes translate.

> The presence of Beckn payloads in any test outside the Bridge's own test suite is a **red flag** ([§4 of CLAUDE.md](../CLAUDE.md)). The Bridge is the only place wire vocabulary belongs.

---

## 7.2 Layer × test type summary

What each layer of code is tested with, and what's deliberately *not* tested at that layer:

| Layer | Test type | Speed | What it tests | What it skips |
|---|---|---|---|---|
| Domain | Unit | Fast | Entities, value objects, state machines, invariants, domain errors | I/O, frameworks, persistence, Beckn |
| Application | Use case unit | Fast | Use case orchestration with port fakes | Real DB; real adapters |
| Infrastructure | Adapter integration | Medium | Real adapter against real backing system (real Postgres via Testcontainers, real OIDC sandbox, etc.) | Domain logic |
| Interface — admin-api | Contract / integration | Medium | HTTP routes through to DB; capability checks; idempotency middleware | Beckn |
| Interface — Bridge | Integration with real ion-specs payloads | Medium | Inbound parsing, signature verification, mapping registry, outbound projection | Domain logic |
| Cross-layer | E2E Beckn flow against stub BAP | Slow | One end-to-end happy path + key error paths per protocol version | Edge cases handled in unit/integration tests |

### Test budget rough split (v1)

| Test type | Approximate share |
|---|---|
| Domain unit | ~40% (cheap, exhaustive) |
| Application use case | ~30% |
| Infrastructure integration | ~15% |
| Interface (admin-api + Bridge) | ~10% |
| E2E Beckn | ~5% (slow, expensive — keep narrow) |

These aren't quotas; they're the natural distribution when the team follows the layer rules. If you find yourself writing tons of E2E tests, push them down (the failures usually express better as Application or Domain tests).

---

## 7.3 Domain Layer tests

### Required

- **Pure unit tests.** No I/O. No frameworks. No fixtures from outside the test file.
- **Test invariants on construction.** Creating an invalid `Money` throws; creating a `LocalizedText` without the `id` key throws.
- **Test state machine transitions.** Each legal transition; each illegal transition rejected with a typed error.
- **Test value-object equality and identity** semantics.
- **No Beckn payloads.** A domain test that needs one is a wrong test.

### Lean (Vitest)

```ts
// contexts/order/domain/entities/order.test.ts
import { describe, it, expect } from 'vitest';
import { Order } from './order';
import { StateTransitionNotAllowedError } from 'shared/kernel/errors';

describe('Order state machine', () => {
  it('transitions Created -> Initiated when Initiate is called with reservations', () => {
    const order = Order.create({ ... });
    const initiated = order.initiate({ reservationIds, contactSnapshot });
    expect(initiated.status).toBe('Initiated');
  });

  it('rejects Confirm from Created', () => {
    const order = Order.create({ ... });
    expect(() => order.confirm()).toThrow(StateTransitionNotAllowedError);
  });

  it('rejects Confirm when Quote TTL has expired', () => {
    const order = Order.create({ ..., quoteValidUntil: pastTime() });
    expect(() => order.confirm()).toThrow(QuoteExpiredError);
  });
});
```

### Rules

- **One test file per entity / aggregate / value object.** Co-located: `order.ts` + `order.test.ts`.
- **Use real time only via an injectable clock.** Never `new Date()` in domain tests directly — pass a `now` factory.
- **Test the public surface.** Don't reach into private state; test through the methods that callers use.

---

## 7.4 Application Layer tests (use cases)

### Required

For each use case, test:

- **Happy path** — preconditions met, expected state change, expected events emitted.
- **Authorization** — `requireCapability` is called before any mutation; absence of the capability raises `AuthorizationDeniedError`.
- **Idempotency** — repeat with same key + same payload returns cached result; same key + different payload raises `IdempotencyKeyReusedWithDifferentPayloadError`.
- **State preconditions** — e.g., `Order.Confirm` rejects when status ≠ `Initiated`.
- **Compensation** — multi-context orchestrations release / revert on failure of any step.
- **Event emission** — the right events get appended to the outbox.

### Patterns: ports as fakes

Use cases run against fake ports. No real DB, no real IdP, no real Postgres outbox.

```ts
// contexts/order/application/use-cases/initiate.test.ts
describe('Order.Initiate', () => {
  it('reserves inventory and persists; emits order.initiated', async () => {
    const fakeReservation = makeFakeInventoryPort({ reserveSucceeds: true });
    const fakeOrderRepo = makeFakeOrderRepo({ orders: [draftOrder] });
    const fakeOutbox = makeFakeOutbox();

    const useCase = makeOrderInitiate({
      ports: { inventory: fakeReservation, orderRepo: fakeOrderRepo, outbox: fakeOutbox },
      actor: testActor,
    });

    const result = await useCase({ orderId: draftOrder.id, contactSnapshot: testContact });

    expect(result.status).toBe('Initiated');
    expect(fakeReservation.reserveCalls).toHaveLength(2);   // two line items
    expect(fakeOutbox.appended).toContainEqual(
      expect.objectContaining({ event_name: 'order.initiated' })
    );
  });

  it('releases reservations when voucher validation fails (compensation)', async () => {
    const fakeReservation = makeFakeInventoryPort({ reserveSucceeds: true, releaseSucceeds: true });
    const fakePromotion = makeFakePromotionPort({ validateThrows: new VoucherExpiredError() });
    // ...
    await expect(useCase(input)).rejects.toThrow(VoucherExpiredError);
    expect(fakeReservation.releaseCalls).toHaveLength(2);   // both reservations released
  });

  it('raises AuthorizationDenied when actor lacks capability', async () => {
    const useCase = makeOrderInitiate({
      ports: { ... },
      actor: actorWithoutCapability,
    });
    await expect(useCase(input)).rejects.toThrow(AuthorizationDeniedError);
  });
});
```

### Idempotency test pattern

```ts
it('returns cached result on repeat with same key + same payload', async () => {
  const useCase = makeCreateStore({ ports, actor });
  const first = await useCase(input, { idempotencyKey: 'k1' });
  const second = await useCase(input, { idempotencyKey: 'k1' });
  expect(first).toEqual(second);
  expect(fakeStoreRepo.insertCalls).toHaveLength(1);   // body executed once
  expect(fakeOutbox.appended).toHaveLength(1);          // event emitted once
});

it('raises typed error on same key + different payload', async () => {
  const useCase = makeCreateStore({ ports, actor });
  await useCase(input1, { idempotencyKey: 'k1' });
  await expect(useCase(input2, { idempotencyKey: 'k1' })).rejects.toThrow(
    IdempotencyKeyReusedWithDifferentPayloadError
  );
});
```

### Rules

- **One test file per use case** (or one describe block per use case if grouped).
- **Build fakes inline or in a context-scoped `__fixtures__` folder.** Don't share fakes across contexts — they should be specific to what each context needs.
- **Assert events by name + key payload fields**, not by exhaustive equality (envelopes carry timestamps).
- **No real DB.** If you find yourself needing one, you're writing an Integration test — move it to §7.5.

---

## 7.5 Infrastructure Layer tests (adapters)

### Required

- **One test suite per adapter.**
- **Real backing system.** Postgres for repositories (via Testcontainers); a sandbox / stub IdP for OIDC adapter; etc.
- **Test the adapter's contract** — round-tripping domain entities through the adapter produces the same entity back.
- **Test transaction boundaries** — adapter receives a `tx` and respects it (rollback works).

### Lean (Vitest + Testcontainers + Drizzle)

```ts
// contexts/tenancy/infrastructure/repositories/drizzle-store-repository.test.ts
import { describe, beforeAll, afterAll, beforeEach, it, expect } from 'vitest';
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { drizzle } from 'drizzle-orm/node-postgres';
import { migrate } from 'drizzle-orm/node-postgres/migrator';
import { Pool } from 'pg';

let pgContainer: StartedPostgreSqlContainer;
let db: ReturnType<typeof drizzle>;

beforeAll(async () => {
  pgContainer = await new PostgreSqlContainer().start();
  const pool = new Pool({ connectionString: pgContainer.getConnectionUri() });
  db = drizzle(pool);
  await migrate(db, { migrationsFolder: 'migrations' });
}, 60_000);

afterAll(async () => {
  await pgContainer.stop();
});

beforeEach(async () => {
  await db.delete(stores);   // per-test cleanup
});

describe('DrizzleStoreRepository', () => {
  it('round-trips a Store entity', async () => {
    const repo = new DrizzleStoreRepository();
    const store = makeStore({ id: 'store-1', currency: 'IDR' });
    await db.transaction(async (tx) => {
      await repo.insert(store, tx);
    });
    const loaded = await db.transaction(async (tx) => repo.requireById(store.id, tx));
    expect(loaded).toEqual(store);
  });

  it('rolls back on transaction failure', async () => {
    const repo = new DrizzleStoreRepository();
    const store = makeStore();
    await expect(db.transaction(async (tx) => {
      await repo.insert(store, tx);
      throw new Error('boom');
    })).rejects.toThrow('boom');
    const result = await db.transaction(async (tx) => repo.findById(store.id, tx));
    expect(result).toBeNull();
  });
});
```

### Rules

- **Share the container across tests in a file** (one `beforeAll` per file). Starting a Postgres container per test is too slow.
- **Truncate (or `delete from`) tables in `beforeEach`** to isolate tests. Faster than restarting the container.
- **Don't test domain logic at this layer.** If a test depends on aggregate behavior, push it to Domain tests; the adapter should care only about persistence shape.
- **Migrations run once per suite.** Don't re-run them between tests.

---

## 7.6 Interface Layer tests (admin-api routes)

### Required

- **Routes are tested at the HTTP layer** — call the route, assert response.
- **Capability middleware** is exercised: unauthorized actors get 403.
- **Idempotency middleware** is exercised: replays return cached.
- **Validation errors** return structured JSON with `code` + `message`.
- **No real Beckn** — admin-api never serves Beckn traffic.

### Lean (Vitest + Supertest, real DB via Testcontainers + fake IdP)

```ts
// admin-api/tests/routes/stores.test.ts
import request from 'supertest';
import { describe, beforeAll, beforeEach, it, expect } from 'vitest';
import { buildApp } from 'admin-api/app';

let app;
beforeAll(async () => {
  app = await buildApp({
    db: testDb,
    idpClient: fakeOidcClient({ activeUser: testUser }),
    // other test deps
  });
});

beforeEach(async () => {
  await resetDatabase();
  await seedOrgAndUser(testUser);
});

describe('POST /orgs/:orgSlug/stores', () => {
  it('creates a store; returns 201 with storeId', async () => {
    const res = await request(app.server)
      .post('/orgs/acme/stores')
      .set('Cookie', sessionCookie(testUser))
      .set('Idempotency-Key', 'k1')
      .send({ name: { id: 'Toko Acme' }, currency: 'IDR' })
      .expect(201);
    expect(res.body).toHaveProperty('storeId');
  });

  it('returns 403 when actor lacks store.create', async () => {
    await request(app.server)
      .post('/orgs/acme/stores')
      .set('Cookie', sessionCookie(memberWithoutCapability))
      .send({ name: { id: 'Toko' }, currency: 'IDR' })
      .expect(403);
  });

  it('returns 400 with validation error on missing default-locale name', async () => {
    const res = await request(app.server)
      .post('/orgs/acme/stores')
      .set('Cookie', sessionCookie(testUser))
      .send({ name: { en: 'Acme Store' }, currency: 'IDR' })   // missing 'id' locale
      .expect(400);
    expect(res.body.error.code).toBe('validation_failed');
  });

  it('returns cached result on idempotency replay', async () => {
    const first = await request(app.server)
      .post('/orgs/acme/stores')
      .set('Cookie', sessionCookie(testUser))
      .set('Idempotency-Key', 'k1')
      .send(input);
    const second = await request(app.server)
      .post('/orgs/acme/stores')
      .set('Cookie', sessionCookie(testUser))
      .set('Idempotency-Key', 'k1')
      .send(input);
    expect(second.body).toEqual(first.body);
  });
});
```

### Rules

- **Use a fake IdP**, not a real one. The real IdP integration is tested in §7.5 (Infrastructure).
- **Use a real DB** (Testcontainers Postgres). Mocking the DB at this layer misses real bugs.
- **Test the URL-slug-matches-session rule** ([§5.1.5 of handoff](../handoff/05-cross-cutting.md)) explicitly.
- **No business logic assertions.** "Did Store get created" is the assertion; "is `published_price` correctly derived" belongs in Domain or Application tests.

---

## 7.7 Beckn Bridge tests

The Bridge gets the **heaviest test budget** because it's the only place wire vocabulary lives, and any bug here is visible to the network.

### Required

- **Inbound parsing** for every message type × every supported protocol version. Test against **real payloads from `ion-specs/`** ([§4.8 of CLAUDE.md](../CLAUDE.md)).
- **Signature verification** — valid signatures pass; invalid signatures reject.
- **Schema validation** — payloads not matching the spec are rejected.
- **Mapping registry coverage** — every supported message type has a tested mapping; reverse mappings produce the right wire shape.
- **Correlation** — `transaction_id` / `message_id` flow through correctly to callbacks.
- **Idempotency at the protocol boundary** — duplicate inbound messages by `(transaction_id, message_id)` are short-circuited.
- **Error mapping** — every domain-error type has a Beckn-error-code projection; unmappable cases fall through to a generic error AND log the mismatch.
- **Locale resolution** — `LocalizedText.get(requested_locale)` produces the expected response.
- **CDS publishing** — every event the Bridge subscribes to produces the expected `catalog/publish` payload.

### Lean (Vitest, real ion-specs payloads, fake BAP via in-process HTTP)

```ts
// bridge/tests/inbound/select.test.ts
import { describe, it, expect } from 'vitest';
import { handleSelect } from 'bridge/inbound/select';
import { loadSpecPayload } from 'bridge/tests/helpers/ion-specs';
import { buildBridgeApp } from 'bridge/tests/helpers/app';

describe('Bridge /select', () => {
  it('parses a valid v2 /select payload and invokes Order.CreateQuote', async () => {
    const payload = await loadSpecPayload('v2/select/request.json');
    const fakeOrderCreateQuote = vi.fn().mockResolvedValue({ orderId: 'order-1', quote: testQuote });
    const app = buildBridgeApp({ useCases: { CreateQuote: fakeOrderCreateQuote } });

    const res = await app.post('/select', payload);

    expect(res.status).toBe(200);
    expect(fakeOrderCreateQuote).toHaveBeenCalledWith(
      expect.objectContaining({
        storeId: expect.any(String),
        items: expect.any(Array),
      })
    );
  });

  it('rejects payload with invalid signature', async () => {
    const payload = await loadSpecPayload('v2/select/request.json');
    const tampered = { ...payload, message: { ...payload.message, items: [...] } };
    const res = await app.post('/select', tampered);
    expect(res.status).toBe(401);   // or whatever Beckn says for sig failure
  });

  it('short-circuits duplicate inbound by (transaction_id, message_id)', async () => {
    const payload = await loadSpecPayload('v2/select/request.json');
    const first = await app.post('/select', payload);
    const second = await app.post('/select', payload);
    expect(fakeOrderCreateQuote).toHaveBeenCalledTimes(1);   // body invoked once
    expect(second.status).toBe(200);                          // dedup response
  });
});
```

### Mapping registry tests

Every registry entry gets a round-trip test:

```ts
describe('order-to-contract mapping (v2)', () => {
  it('maps a Confirmed Order to a wire Contract with status=ACTIVE', () => {
    const order = makeConfirmedOrder();
    const contract = mapOrderToContract(order);
    expect(contract.status).toBe('ACTIVE');
    expect(contract.commitments).toHaveLength(order.quote.lineItems.length);
  });

  it('maps Cancelled to CANCELLED', () => {
    const order = makeCancelledOrder();
    expect(mapOrderToContract(order).status).toBe('CANCELLED');
  });
});
```

### Rules

- **Real `ion-specs` payloads only.** No hand-crafted Beckn JSON in tests except for invalid-payload negative tests.
- **The Application Layer never appears in Bridge tests** beyond mocked use cases. Bridge tests test the Bridge, not the Order context.
- **No `bridge/` import outside `bridge/`** is enforced by lint (per [§2.8 of `02-repo-layout.md`](02-repo-layout.md)).

---

## 7.8 End-to-end Beckn flow tests

### Required

- **One happy-path flow per Beckn message family**: `/search` (CDS), `/select` → `/on_select`, `/init` → `/on_init`, `/confirm` → `/on_confirm`, `/cancel` → `/on_cancel`, `/status` → `/on_status`.
- **At least one cancel-from-Confirmed flow** to exercise voucher revert + payment Refunded.
- **At least one expired-quote flow** to exercise the TTL-driven release.
- **Async correlation verified** — outbound callback arrives with matching `transaction_id`.

### Lean (Vitest + stub BAP + real DB + real Bridge)

```ts
// tests/e2e/beckn-v2/select-init-confirm.test.ts
describe('Beckn v2: select -> init -> confirm', () => {
  it('completes the happy path; reservations converted; voucher recorded', async () => {
    const stubBap = await startStubBap();
    const bpp = await startBpp({ db: testDb, bapCallbackUrl: stubBap.url });

    await stubBap.send('/select', selectPayload);
    const onSelect = await stubBap.waitForCallback('on_select', { timeout: 5000 });
    expect(onSelect.message.quote).toBeDefined();

    await stubBap.send('/init', initPayloadFrom(onSelect));
    const onInit = await stubBap.waitForCallback('on_init');

    await stubBap.send('/confirm', confirmPayloadFrom(onInit));
    const onConfirm = await stubBap.waitForCallback('on_confirm');
    expect(onConfirm.message.contract.status).toBe('ACTIVE');

    // Verify side effects
    const order = await testDb.query.orders.findFirst({ where: ... });
    expect(order.status).toBe('Confirmed');
    expect(await stockLevelFor(testProduct)).toEqual({ stock_count: testInitialStock - testQty });
  });
});
```

### Rules

- **Slow tests; keep narrow.** One happy path + one cancel + one expiry per family is sufficient. Edge cases live in Application or Bridge tests.
- **Real DB** (Testcontainers Postgres). Real outbox dispatcher. Real Bridge.
- **Stub BAP** — a small in-process HTTP server that records callbacks and lets the test issue Beckn requests.
- **Tag these tests** so they run in a separate CI stage (skip on PR; run on merge to main, or run nightly).

---

## 7.9 Property-based tests for state machines

### Required

The state machines worth property-testing (per [§6.1.6 of handoff](../handoff/06-operational.md)):

- **Order state machine** — from any reachable state, only declared transitions are accepted.
- **Store lifecycle** — same.
- **Reservation lifecycle** — Active → Converted | Released; never back to Active.
- **Invitation lifecycle** — Pending → Accepted | Declined | Revoked | Expired; one-way only.

### Lean (Vitest + fast-check)

```ts
// contexts/order/domain/entities/order.property.test.ts
import { describe, it } from 'vitest';
import fc from 'fast-check';
import { Order } from './order';
import { StateTransitionNotAllowedError } from 'shared/kernel/errors';

const orderActionArb = fc.oneof(
  fc.record({ type: fc.constant('Initiate'), ... }),
  fc.record({ type: fc.constant('Confirm') }),
  fc.record({ type: fc.constant('Cancel'), reason: fc.string() }),
  fc.record({ type: fc.constant('MarkShipped') }),
  // ...
);

describe('Order state machine — property', () => {
  it('only legal transitions succeed; all illegal raise typed errors', () => {
    fc.assert(fc.property(
      fc.array(orderActionArb, { minLength: 0, maxLength: 10 }),
      (actions) => {
        let order = Order.create({ ... });
        for (const action of actions) {
          if (isLegalTransition(order.status, action.type)) {
            order = applyAction(order, action);
          } else {
            try { applyAction(order, action); throw new Error('expected typed error'); }
            catch (e) { return e instanceof StateTransitionNotAllowedError; }
          }
        }
        return true;
      }
    ));
  });
});
```

### Rules

- **Properties express invariants**, not just example checks. "Every reachable state allows only declared transitions" is a property; "Cancel from Confirmed releases the reservation" is an example test.
- **Keep generators bounded** (`maxLength`). State-machine property tests can explode in shape complexity.
- **Don't property-test infrastructure.** Properties are domain-shape tests.

---

## 7.10 Concurrency tests

A few specific concurrency tests are mandatory because their failures don't surface in single-threaded tests:

### Required

- **Outbox dispatcher claim semantics.** Multiple dispatcher workers running against the same outbox process distinct rows (no double-processing).
- **Reservation race.** Concurrent `Reserve(item, 1)` calls on a `StockLevel` with `stock_count = 1` succeed exactly once; the others get `InsufficientStockError`.
- **Inbox dedup under retry.** A subscriber processes the same event delivered twice → handler invoked once, side effects once.

### Lean (Vitest + real Postgres)

```ts
// platform-infra/outbox-dispatcher/tests/claim-skip-locked.test.ts
it('multiple workers claim distinct rows under skip-locked', async () => {
  await seedOutbox(100);   // 100 pending rows
  const worker1 = makeWorker({ db });
  const worker2 = makeWorker({ db });
  const [claims1, claims2] = await Promise.all([
    worker1.claimBatch(50),
    worker2.claimBatch(50),
  ]);
  const ids1 = new Set(claims1.map(c => c.id));
  const ids2 = new Set(claims2.map(c => c.id));
  expect([...ids1].filter(id => ids2.has(id))).toEqual([]);   // no overlap
  expect(ids1.size + ids2.size).toBe(100);
});
```

### Rules

- **Real DB required.** In-memory fakes can't reproduce DB-level concurrency semantics.
- **Use `Promise.all` to invoke concurrently.** Don't `await` between operations meant to race.
- **Concurrency tests can be flaky.** If one is flaky, treat that as a real bug — don't retry-loop.

---

## 7.11 Test fixtures and factories

### Required

- **One factory per aggregate / value object.** `makeStore`, `makeOrder`, `makeMoney`, `makeLocalizedText`.
- **Factories provide defaults + accept overrides.** Most tests need a valid baseline; some tweak one or two fields.
- **Default-locale aware.** `LocalizedText` factories always include `id` — that's the invariant.

### Lean

```ts
// shared/value-objects/__fixtures__/money.ts
export const makeMoney = (overrides: Partial<Money> = {}): Money => ({
  amount: 100_00n,         // 100.00 IDR in minor units
  currency: 'IDR',
  ...overrides,
});

// shared/value-objects/__fixtures__/localized-text.ts
export const makeLocalizedText = (overrides: Partial<{
  id: string;
  en?: string;
  others?: Record<string, string>;
}> = {}): LocalizedText => ({
  entries: {
    id: overrides.id ?? 'Contoh teks',
    ...(overrides.en && { en: overrides.en }),
    ...overrides.others,
  },
});

// contexts/tenancy/__fixtures__/store.ts
export const makeStore = (overrides: Partial<Store> = {}): Store =>
  Store.create({
    id: 'store-1',
    orgId: 'org-1',
    name: makeLocalizedText({ id: 'Toko Test' }),
    currency: 'IDR',
    status: 'Draft',
    ...overrides,
  });
```

### Rules

- **Factories live alongside the code they build.** `contexts/<X>/__fixtures__/` or `<entity>.fixture.ts`.
- **No cross-context fixture imports.** Each context owns its fixtures, just like its code.
- **Factories aren't builders.** Keep them simple functions; if you need composition, compose at the call site.
- **Don't randomize defaults.** Stable values make test failures readable.

---

## 7.12 DI containers in tests

### Required

- **Each test creates its own container scope.** No shared mutable state between tests.
- **Adapter ports are replaced with fakes** at the container level — don't reach into the use case to swap.
- **Per-request context** (actor, correlation_id, idempotency_key) provided per test, not globally.

### Lean (awilix)

```ts
// shared/kernel/__fixtures__/test-container.ts
export function makeTestContainer(overrides: Partial<TestContainerDeps> = {}) {
  const container = createContainer({ injectionMode: InjectionMode.PROXY });
  container.register({
    db: asValue(overrides.db ?? makeFakeDb()),
    logger: asValue(makeSilentLogger()),
    idempotencyTable: asValue(overrides.idempotencyTable ?? makeFakeIdempotencyTable()),
    outbox: asValue(overrides.outbox ?? makeFakeOutbox()),
    authorizationPort: asValue(overrides.authorizationPort ?? makeFakeAuthorization({ allow: true })),
    // ... cross-context ports as fakes
  });
  return container;
}

// In a test:
const container = makeTestContainer({
  authorizationPort: makeFakeAuthorization({ allow: false }),   // override for one test
});
const scope = container.createScope();
scope.register({ actor: asValue(testActor), correlationId: asValue('test-cid') });
const useCase = scope.resolve('CreateStore');
```

### Rules

- **No global container instance in tests.** Each test gets a fresh one.
- **Fake implementations are scoped to the context that needs them.** A "fake `StoreRepo`" lives next to `Store`-tests; no shared fakes folder.
- **Don't mock framework code.** Mock ports the team owns.

---

## 7.13 Frontend tests (admin UI)

### Required

- **Form validation tests** — Zod schemas from `bpp-contracts` validate inputs; error messages surface to the user.
- **Route guard tests** — active-Org / active-Store guards redirect on mismatch.
- **Component tests** — for any non-trivial feature component (rendering logic, conditional UI based on capability).
- **Smoke E2E tests** — at least one per top-level admin journey (Org creation → Store creation → Product publish).

### Lean (Vitest + React Testing Library + Playwright for E2E)

```tsx
// apps/bpp-admin/src/features/tenancy/components/CreateStoreForm.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen, fireEvent } from '@testing-library/react';
import { CreateStoreForm } from './CreateStoreForm';

describe('CreateStoreForm', () => {
  it('renders error when default-locale name is missing', async () => {
    render(<CreateStoreForm onSubmit={() => {}} />);
    fireEvent.change(screen.getByLabelText('Name (en)'), { target: { value: 'Acme' } });
    fireEvent.click(screen.getByRole('button', { name: 'Create' }));
    expect(await screen.findByText(/Bahasa Indonesia name is required/)).toBeInTheDocument();
  });
});
```

```ts
// e2e/admin/create-store.spec.ts (Playwright)
test('user creates an org and a store', async ({ page }) => {
  await signIn(page, testUser);
  await page.click('text=Create organization');
  await page.fill('input[name="orgName"]', 'Acme');
  await page.click('text=Create');
  await page.click('text=New store');
  // ...
  await expect(page.locator('text=Toko Acme')).toBeVisible();
});
```

### Rules

- **Use Testing Library's user-event** (or similar) — assert from the user's perspective, not by component internals.
- **No mocking React Query / TanStack Router internals.** Mock at the network boundary (MSW or similar) so the rest is real.
- **E2E tests run against a real backend** (the BPP package running locally with a Testcontainers DB). Don't stub the backend.

---

## 7.14 Common gotchas

A non-exhaustive list of mistakes that recur. Watch for these on every PR.

| Gotcha | Where it shows up |
|---|---|
| **Beckn vocabulary in a non-Bridge test** | Any test outside `bridge/` |
| **Test relies on `new Date()` directly** | Domain / Application tests with TTLs |
| **Outbox row written outside the use-case transaction in tests** | Application use-case tests where fakes don't reflect the contract |
| **Real external service used in unit tests** | Integration tests masquerading as unit tests |
| **Shared mutable fixture state across tests** | Suites that pass in isolation but fail when run together |
| **Cross-context internals imported by tests** | Caught by lint, but worth scanning |
| **Property-test generators too wide** | Test runs too slow; flakes from edge cases that aren't worth exploring |
| **E2E test asserts on intermediate DB state** | Should assert on observable outputs, not internal state |
| **Tests duplicate domain logic** | If a test reimplements `published_price = base + tax`, you have two sources of truth |
| **Snapshot tests for anything non-trivial** | Snapshots erode meaning; prefer explicit assertions |
| **Tests of framework code** | "Does TanStack Router parse URLs" is not our concern |

---

> **Next**: [`03-dev-setup.md`](03-dev-setup.md) — local environment, IdP sandbox, seed data. Or [`06-context-playbooks/`](06-context-playbooks/) — per-context implementation skeletons.
