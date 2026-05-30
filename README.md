# bpp-design

Architecture and design for a multi-tenant **Beckn Provider Platform (BPP)** — an aggregator of small retailers in Indonesia, exposed to the Beckn network as a single BPP, with each store appearing on the network as a `provider` node. v2 / ION wire vocabulary at the protocol boundary.

> **Status: design complete; ready for implementation.** This is still a design-stage repository — there is no application code here. What's been delivered is the **integrated design package** the implementing team will build from.

## Two audiences

### If you are building this system

Read both, in order:

**1. [`handoff/README.md`](handoff/README.md)** — the **architecture reading view**: a 1–2 hour first read covering what the system is, the architectural rules, each bounded context, the cross-cutting patterns, and the deferred work. Tells you *what* to build and *why*.

```
handoff/
├── README.md                   reader's guide + TOC
├── 01-overview.md              what the system is, glossary
├── 02-principles.md            architectural principles
├── 03-beckn-integration.md     the Beckn Bridge
├── 04-bounded-contexts/        Identity, Tenancy, Catalog, Inventory,
│                                 Promotion, Order, Audit (one file each)
├── 05-cross-cutting.md         authorization, events, consistency,
│                                 idempotency, soft-delete, PII, localization
├── 06-operational.md           testing, observability, deployment
├── 07-open-issues.md           v1 deferrals and known limits
└── 08-references.md            ADR + design-doc index
```

**2. [`developer-guide/README.md`](developer-guide/README.md)** — the **implementation reading view**: tells you *with what* (stack), *when* (phases), and *how* (conventions, testing, runbooks). Stack-agnostic where it can be; v1 leans where it helps.

```
developer-guide/
├── README.md                   reader's guide + conventions
├── 01-stack.md                 chosen technologies (D0-D9 resolved)
├── 02-repo-layout.md           BPP package structure (backend, frontend, contracts)
├── 03-dev-setup.md             local environment + IdP sandbox + seed data
├── 04-conventions.md           naming, use-case envelope, errors, logging
├── 05-phases.md                dependency-ordered build plan (Phase 0–8)
├── 06-context-playbooks/       per-context implementation skeletons (one per bounded context)
├── 07-testing.md               per-layer testing posture
├── 08-deployment.md            standalone-v1 reference deployment
└── 09-runbooks.md              33 operational procedures
```

Both reading views cross-reference each other and the ADRs.

### If you are evolving the design itself

Start at [`CLAUDE.md`](CLAUDE.md) — the **architectural charter**. It defines what must be true. Then:

- [`decisions/`](decisions/) — 21 Architecture Decision Records. **Immutable once accepted**; superseded by new ADRs.
- [`design/`](design/) — sub-design documents (full entity models for Catalog, Identity, Events, Audit, PII, Order, Tenancy).
- [`gaps/`](gaps/) — open design questions (all currently resolved; [`gaps/resolved/`](gaps/resolved/) is the historical record).

The full artifact map and authoring rules are in [§10 of `CLAUDE.md`](CLAUDE.md).

## What's here

| Path | Audience | Role |
|---|---|---|
| [`handoff/`](handoff/) | Implementing team | Architecture reading view — *what* + *why* |
| [`developer-guide/`](developer-guide/) | Implementing team | Implementation reading view — *with what* + *when* + *how* |
| [`CLAUDE.md`](CLAUDE.md) | Design contributors | Architectural charter (rules, principles, working agreement) |
| [`decisions/`](decisions/) | Anyone asking "why?" | 21 ADRs — the historical record of judgment calls |
| [`design/`](design/) | Anyone needing schemas | 7 sub-design docs — full entity models per context |
| [`gaps/`](gaps/) | Design contributors | Open design questions (currently empty) |
| [`gaps/resolved/`](gaps/resolved/) | Historical reference | Resolved gaps with pointers to closing ADRs |

## What this system is (1-paragraph summary)

A Beckn Provider Platform that **aggregates small retailers** as independently-operated stores under a single network identity. Each store has its own catalog, inventory, vouchers, and orders; stores belong to **Organizations** (tenancy boundary). The platform integrates with Beckn v2 / ION via a single **Beckn Bridge** adapter — protocol vocabulary is permitted only there. The domain is otherwise Beckn-naive and can exist, evolve, and be tested without Beckn. Indonesia PDP-compliant, OIDC-delegated identity, localized in Bahasa Indonesia by default. Payment and logistics are external in v1; buyers reach the system only through Beckn (BAPs).

For the long version, read [`handoff/01-overview.md`](handoff/01-overview.md).

## Status

- **Design**: complete. 21 ADRs accepted; all original 17 design gaps resolved; 4 follow-on feature ADRs (Organization tenancy, manual republication, Default Catalog mandatory, Pure-BPP scope) landed on top.
- **Implementation guide**: complete. 9 docs + 8 per-context playbooks; 4 of 10 D-decisions confirmed for v1 (D0 awilix, D1 Drizzle, D2 shadcn/ui, D2a React Hook Form); the rest deferred to the host monorepo when it lands.
- **Code**: not started. The implementing team begins from `handoff/` + `developer-guide/`.
- **Host monorepo**: this BPP is planned to land inside a larger monorepo. The developer-guide is written to survive that merger — concrete tooling is presented as v1 leans, replaceable when the monorepo's conventions arrive.
- **Compliance regime**: Indonesia PDP (UU 27/2022), GDPR-compatible by design.
- **Protocol target**: Beckn v2 / ION; CDS-mediated discovery.

For the deferred-work catalog (what v1 explicitly doesn't ship), see [`handoff/07-open-issues.md`](handoff/07-open-issues.md).

## Reference projects (external)

These are inspiration / consultation references, not parts of this system. Used per [§7 of `CLAUDE.md`](CLAUDE.md):

- **bitemycart** — reference for ecommerce *shape*. Treated as "what problems exist", not as a template.
- **ion-specs** — source of truth for the Beckn / ION protocol. Used only inside the Beckn Bridge.
