# Gap 04 — Store lifecycle beyond creation

> **RESOLVED on 2026-05-28** by [ADR-0003 — Store lifecycle and state machine](../../decisions/0003-store-lifecycle-and-state-machine.md). The active rule lives in `CLAUDE.md` §5.8. This file is preserved for historical context.

## Statement

The charter covers store creation and ownership transfer (§5.3, §6.6) but says nothing about what else can happen to a store across its life. Stores in real systems are suspended, archived, deleted, restored, transferred, and disputed. Without a defined lifecycle, the storage model and authorization model both have unresolved edges.

## Current charter coverage

* §5.1 — A Store is the unit of tenancy; products, inventory, orders, members are scoped to one store.
* §5.3 — One Owner; transferable.
* §6.6 — Ownership transfer is an explicit two-party operation.
* §3.2 — "Soft deletion is a domain decision, not a default" — deferred per entity. (See also Gap 16.)

## Open questions

1. **Possible states.** What lifecycle states does a Store have? Candidates: `Draft`, `Active`, `Suspended` (by platform), `Paused` (by owner), `Archived`, `Deleted`.
2. **Owner actions.** Can the owner suspend, archive, or delete the store? Are these reversible?
3. **Platform actions.** Can the platform unilaterally suspend a store? Under what triggers (policy violation, payment dispute)? Is suspension a domain concept or a separate moderation context?
4. **Cascading effects.** When a store is suspended / deleted, what happens to: pending orders, published catalog items, Memberships, invitations in flight, Beckn presence?
5. **Restoration.** Is `Suspended` → `Active` reversible without data loss? What about `Archived` and `Deleted`?
6. **Distinction.** What separates `Inactive`, `Suspended`, `Archived`, and `Deleted` semantically? Are they all needed, or can the system collapse to two or three?
7. **Identity reuse.** If a store is deleted, can its slug / external identifier be reused later?
8. **History retention.** Is a deleted store's order/audit history retained for legal/compliance purposes? For how long?

## Implications

* Every query that lists "stores" implicitly assumes a state filter (active vs. all). Without a defined state machine, this filter is ad hoc and leaky.
* Beckn publication (Gap 06) must respect lifecycle — a suspended store should not appear in `on_search`.
* Memberships, invitations, and audit need clear semantics when their store is no longer active.

## Dependencies

* Depends on: 02 (Platform roles — who can suspend).
* Influences: 06 (publication), 16 (soft delete defaults).
