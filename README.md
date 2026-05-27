# bpp-design

Architecture and design charter for a multi-tenant **Beckn Provider Platform (BPP)** — an aggregator of small retailers exposed to the Beckn network as a single BPP, with each store appearing on the network as a `provider`.

> This is a **design-stage repository**. There is no application code yet. The current goal is to establish the architectural, conceptual, and modeling foundations on which all future code will be built. The eventual deliverable is a consolidated design document for handoff to the implementing team.

## What's here

| Path | Purpose |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | The **architecture charter**. Principles, rules, boundaries. The voice of *what must be true*. Kept dense and current. |
| [`gaps/`](gaps/) | Open design questions, one per file. Each names the gap, lists open questions, and notes dependencies. See [`gaps/INDEX.md`](gaps/INDEX.md). |
| [`gaps/resolved/`](gaps/resolved/) | Gaps that have been answered. Each carries a banner pointing to the ADR that closed it. |
| [`decisions/`](decisions/) | **Architecture Decision Records (ADRs)** — one per resolved gap or significant judgment call. Captures context, options weighed, decision, and consequences. Immutable once accepted. |
| [`design/`](design/) | Sub-design documents for subsystems too detailed for the charter (e.g., the Catalog model, the Order lifecycle). Backed by ADRs. |

## How to read this repo

1. Start with [`CLAUDE.md`](CLAUDE.md) for the architectural principles, layering, the Beckn Bridge model, multi-tenancy, and the working agreement.
2. Browse [`decisions/`](decisions/) for the decisions made so far and the reasoning behind them.
3. Check [`gaps/INDEX.md`](gaps/INDEX.md) to see what's still open.

## Working process

Resolution flow: a gap is discussed and converged on → an ADR is authored capturing context, options, decision, consequences → the rule lands in `CLAUDE.md` → the gap file moves to `gaps/resolved/`. See [§10 of `CLAUDE.md`](CLAUDE.md) for the full artifact map and authoring rules.

## Status

ADRs accepted to date:

- [ADR-0001 — BPP network identity](decisions/0001-bpp-network-identity.md) — *one BPP for the whole platform; stores project as Beckn `provider` nodes*
- [ADR-0002 — Authorization tiers and the capability matrix](decisions/0002-authorization-tiers-and-matrix.md) — *three-tier model (System Admin / Platform-scoped / Store-scoped), system-admin-managed permission matrix, downward-only impersonation*
- [ADR-0003 — Store lifecycle and state machine](decisions/0003-store-lifecycle-and-state-machine.md) — *four states (Draft / Active / Suspended / Paused), no deletion, Bridge republishes on every transition*

Open gaps: 14 (3 fully resolved, 2 partially resolved out of 17 originally identified).

## Reference projects

This design draws from two external reference projects (not included in this repo):

- **bitemycart** — inspiration for ecommerce flows and store UX patterns. Treated as *what problems exist*, not as a template.
- **ion-specs** — source of truth for the Beckn / ION protocol and wire format. Used only inside the Beckn Bridge.

The discipline around these references is in [`CLAUDE.md` §7](CLAUDE.md).
