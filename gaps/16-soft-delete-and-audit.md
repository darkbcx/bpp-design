# Gap 16 — Soft delete & audit log

## Statement

Two related concerns are mentioned in passing but not designed:

1. **Soft delete.** §3.2 says "soft deletion is a domain decision, not a default. Decide per entity." This defers every decision without giving a default stance, leaving the door open to inconsistent behavior across entities.
2. **Audit log.** §5.3, §6.5, §6.6 reference "auditable events" but never define what audit is, where it lives, or how it relates to domain events (Gap 13).

## Current charter coverage

* §3.2 — "Soft deletion is a domain decision, not a default."
* §5.3, §6.5, §6.6 — Various changes are described as auditable.

## Open questions

### Soft delete

1. **Default stance.** What is the system's *default* for an entity with no explicit decision — hard delete, soft delete, or "never delete"?
2. **Per-entity overrides.** What entities are clearly hard-delete (e.g., session tokens), soft-delete (e.g., products), or never-delete (e.g., orders)?
3. **Cascading.** When a parent is soft-deleted, are children soft-deleted too? Lazily filtered out, or actively flagged?
4. **Visibility rule.** Is "deleted" a status, a timestamp, or a separate table? Are deleted rows visible to admins by default, or hidden?
5. **Identifier reuse.** Can a slug / external identifier be reused after the original is soft-deleted?

### Audit

1. **Concept location.** Is audit a domain concept, an infrastructure cross-cut, or a hybrid?
2. **Relationship to events.** Is the domain event log also the audit log? Or are they separate stores with different retention and access?
3. **What gets audited.** All state changes? Only explicitly tagged "auditable" actions? Authorization decisions?
4. **Access control.** Who can read audit records — only platform admins, also store owners, the affected user?
5. **Retention.** How long are audit records kept? Tenant-controlled or platform-controlled?
6. **Tamper resistance.** Is audit append-only at the storage level (e.g., enforced by infrastructure)?

## Implications

* Affects every entity's persistence shape.
* Audit is often required by compliance — answering this affects regulatory posture (Gap 17).
* Confounding events and audit can save effort but also makes both harder to evolve.

## Dependencies

* Related to: 13 (events), 17 (PII / compliance).
