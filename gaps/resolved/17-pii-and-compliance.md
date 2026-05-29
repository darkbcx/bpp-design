# Gap 17 — PII & compliance stance

> **RESOLVED on 2026-05-28** by [ADR-0015 — PII handling and right-to-erasure](../../decisions/0015-pii-and-right-to-erasure.md). Full PII catalog and scrub registry in [`design/pii.md`](../../design/pii.md). Active rules in `CLAUDE.md` §5.20. This file is preserved for historical context.

## Statement

The charter does not address personal data, regulatory compliance, or data residency. The system inherently stores PII (user emails at minimum; buyer details, addresses, and phone numbers once Beckn order flows land), and Beckn / ONDC operates in regulated environments. A baseline stance is required before infrastructure decisions are made.

## Current charter coverage

* No section. PII, compliance, retention, and data residency are not mentioned.

## Open questions

1. **PII inventory.** What categories of PII does the system handle — user contact info, buyer details from Beckn flows, payment data (if any), location data, behavioral data?
2. **Data residency.** Are there geographic constraints (e.g., India-resident data for ONDC participation)?
3. **Regulatory regime.** Which regulations apply at launch — DPDP (India), GDPR (if EU users), others?
4. **Retention policy.** How long are user records, order records, audit records retained after a user / store / order is closed?
5. **Right to erasure.** How is "delete my data" reconciled with order history, audit trails, and tax-record retention?
6. **Encryption.** Stance on encryption at rest, at transit, and for specific sensitive fields (e.g., column-level encryption for emails)?
7. **Logging discipline.** What PII is *forbidden* in logs (raw emails, tokens, full request bodies)? Where is this rule enforced?
8. **Cross-store data sharing.** Even if data is logically isolated (§5.2), can platform analytics aggregate data across stores? Under what consent model?
9. **Third-party processors.** What's the policy when integrating external IdPs, payment processors, hosted databases, observability vendors?

## Implications

* Many of these answers are binary infrastructure decisions (e.g., where the database lives), and getting them wrong post-launch is expensive.
* The audit model (Gap 16) and right-to-erasure interact.
* Affects every storage and logging decision.

## Dependencies

* Influences: 16 (audit retention), infrastructure choices.
* Largely independent of other domain gaps.
