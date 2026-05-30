# 6. Context playbooks

One playbook per bounded context. Each playbook gives you a concrete starting point for implementing that context: which entities to model, which use cases to write, which ports to declare, which events to emit, and the order to build them in.

Read [`02-repo-layout.md`](../02-repo-layout.md), [`04-conventions.md`](../04-conventions.md), and the relevant handoff section before diving in.

## Reading order (matches implementation phase order)

| Phase | Playbook | Status |
|---|---|---|
| 1 | [6.1 Identity & Access](6.1-identity.md) | Drafted |
| 2 | [6.2 Tenancy](6.2-tenancy.md) | Drafted |
| 3 | [6.3 Catalog](6.3-catalog.md) | Drafted |
| 4 | [6.4 Inventory](6.4-inventory.md) | Drafted |
| 4 | [6.5 Promotion](6.5-promotion.md) | Drafted |
| 5 | [6.6 Order & Fulfillment](6.6-order.md) | Drafted |
| 6 | [6.7 Beckn Bridge](6.7-beckn-bridge.md) | Drafted |
| 7 | [6.8 Audit](6.8-audit.md) | Drafted |

## What a playbook gives you

For each context:

1. **At a glance** — what the context owns, what it depends on, the phase it lands in.
2. **Folder structure** — the concrete tree to create in `contexts/<context>/`.
3. **Entities and value objects** — fields, invariants, state machines.
4. **Ports declared** — interfaces this context defines (its repositories + the cross-context ports it offers).
5. **Use cases** — ordered list, each with the envelope shape.
6. **Events emitted** — wire name, payload shape, category (for Audit retention).
7. **Events subscribed** — events from other contexts this one consumes.
8. **Migrations** — first migration's tables (sketch only).
9. **Cross-context integration** — provides / consumes / events-consumed-by.
10. **Implementation order within the context** — step-by-step.
11. **Test checklist** — required tests per layer (cross-ref to [`07-testing.md`](../07-testing.md)).
12. **Gotchas** — context-specific watch-fors.
13. **References** — handoff section, design doc, ADRs.

## What a playbook does NOT give you

- Line-by-line code. The patterns are in [`04-conventions.md`](../04-conventions.md); the playbook tells you *what to build*, not *how to write a use case*.
- Beckn vocabulary. Only the Bridge playbook ([6.7](6.7-beckn-bridge.md) when drafted) talks Beckn.
- Stack-specific code. Examples lean on the v1 stack (TypeScript + awilix + Drizzle + Zod) but the patterns translate.

## Playbook template

If you add a new context (rare — the seven from [§4 of handoff](../../handoff/04-bounded-contexts/README.md) are the only ones in v1), copy the structure from [6.1 Identity](6.1-identity.md). Keep the headings consistent across playbooks — readers should be able to find "ports declared" or "events emitted" by location alone.

## Relationship to other guide sections

| For | Read |
|---|---|
| What the context is and why | [handoff §4.X](../../handoff/04-bounded-contexts/README.md) |
| Where files go | [`02-repo-layout.md`](../02-repo-layout.md) (per-context internal layout) |
| How to shape use cases / errors / events | [`04-conventions.md`](../04-conventions.md) |
| When this context lands | [`05-phases.md`](../05-phases.md) |
| How to test it | [`07-testing.md`](../07-testing.md) |
| **What to build, in order** | **The relevant playbook here** |

---

> **Next**: pick the playbook for the context you're starting on.
