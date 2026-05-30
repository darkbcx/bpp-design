# Developer Guide

The **implementation-side companion** to the [handoff package](../handoff/README.md). Where the handoff says *what* the system is and *why* it's that way, this guide says *with what*, *when*, and *how*.

## Who this is for

You're an engineer on (or joining) the implementing team. You've read the handoff, understand the bounded contexts and cross-cutting patterns, and now you need to actually build the thing.

## Monorepo context

The BPP described in this repository is planned to land **inside a larger monorepo** that hosts other applications too. The monorepo doesn't currently dictate a specific framework or tool for the BPP, so this guide describes:

- **Architecture-imposed requirements** — characteristics the implementation MUST have, derived from the handoff and the ADRs. These are stable across any monorepo conventions that arrive later.
- **v1 reference leans** — concrete library / framework choices that satisfy the requirements for a standalone v1. They are **proposals**, not commitments — when the merger happens, monorepo-wide conventions may supersede them.

In each section, look for **Required:** and **Lean (v1):** labels. Required items don't move; Leans do.

## How this relates to the rest of the repo

| Doc | Layer | Stability |
|---|---|---|
| [`/CLAUDE.md`](../CLAUDE.md) | Architectural charter (governance) | Stable; rarely changes |
| [`/handoff/`](../handoff/) | Architecture (what + why) | Stable; rarely changes |
| [`/decisions/`](../decisions/) | Historical record of judgment calls | Immutable once accepted |
| **`/developer-guide/`** (this dir) | **Implementation (with what + when + how)** | **Living — updated as the codebase evolves** |

This guide quotes / references the handoff and ADRs; it doesn't restate the architecture. If a fact in this guide contradicts the handoff, **the handoff wins** — fix this guide.

## Reading order

1. [`01-stack.md`](01-stack.md) — chosen technologies, layer by layer, with rationale and open decisions.
2. [`02-repo-layout.md`](02-repo-layout.md) — monorepo structure; module boundaries per [§2.4](../handoff/02-principles.md) of the handoff.
3. [`03-dev-setup.md`](03-dev-setup.md) — local environment, IdP sandbox, seed data, common dev tasks.
4. [`04-conventions.md`](04-conventions.md) — code conventions (naming, errors, logging, ports + adapters wiring).
5. [`05-phases.md`](05-phases.md) — dependency-ordered implementation roadmap (build order for v1).
6. [`06-context-playbooks/`](06-context-playbooks/) — per-context "how to implement" guides, one per bounded context.
7. [`07-testing.md`](07-testing.md) — per-layer testing posture (the handoff's [§6.1](../handoff/06-operational.md) made concrete).
8. [`08-deployment.md`](08-deployment.md) — CI/CD, environments, secrets, infra.
9. [`09-runbooks.md`](09-runbooks.md) — operational tasks per handoff [§6.7](../handoff/06-operational.md).

## Drafting status

| Doc | Status |
|---|---|
| README (this file) | Drafted |
| 01-stack.md | Drafted (Required + Lean structure; open decisions marked **D0–D9**) |
| 02-repo-layout.md | Drafted |
| 03-dev-setup.md | Drafted |
| 04-conventions.md | Drafted |
| 05-phases.md | Drafted |
| 06-context-playbooks/ | Drafted (README + 6.1–6.8) |
| 07-testing.md | Drafted |
| 08-deployment.md | Pending |
| 09-runbooks.md | Pending |

## Conventions in this guide

- **Required:** — architecture-imposed need (from the handoff / ADRs). Doesn't move regardless of stack.
- **Lean (v1):** — proposed concrete tool / library for v1 standalone. May be superseded by monorepo conventions.
- **Open (Dn):** — proposed lean + alternatives, numbered for cross-reference, awaiting your confirmation.
- **Open (deferred):** — not deciding yet; revisit at the phase that needs it.

When a Lean is overridden by monorepo conventions, replace it inline and note the source. When an Open (Dn) is confirmed, fold the lean into Required or Lean as appropriate.
