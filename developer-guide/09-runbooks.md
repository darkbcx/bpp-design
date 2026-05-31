# 9. Runbooks

Operational procedures for running the BPP. Each runbook is a **what-to-do** for a specific situation — when to use it, who can run it, the procedure, how to verify, and how to roll back.

These runbooks are **stack-agnostic** — they describe operations, not specific cloud commands. When the host monorepo's tooling lands, fill in the concrete commands per environment.

| Section | Topic |
|---|---|
| 9.1 | Reading this section |
| 9.2 | Runbook template |
| 9.3 | Identity & Access |
| 9.4 | Tenancy |
| 9.5 | Beckn Bridge |
| 9.6 | Events / consistency |
| 9.7 | Audit |
| 9.8 | Catalog / Inventory |
| 9.9 | Order & Fulfillment |
| 9.10 | Database / Infrastructure |
| 9.11 | Cross-cutting |
| 9.12 | Severity levels and on-call |
| 9.13 | Runbook authoring rules |

---

## 9.1 Reading this section

Runbooks complement architectural docs — they live longer because situations recur. A good runbook is:

- **Specific to a situation** ("the dispatcher is stuck"; not "investigate dispatcher").
- **Authoritative** — the team agrees this is the procedure to run, not one of several.
- **Up to date** — when the system changes, runbooks that reference it get updated in the same change.

When an operational task isn't in this list, write one before doing the task — and add it here so the next person doesn't start from scratch.

---

## 9.2 Runbook template

Use this shape for every runbook. Each entry below in §9.3–§9.11 follows it.

```
### <Title>

**When**: trigger condition (alert name, observed symptom, scheduled cadence)
**Who**: role(s) authorized to run this
**Severity**: paging | high | medium | low
**Estimated duration**: rough time budget

**Procedure**:
1. ordered steps
2. each step is a verifiable action
3. when in doubt, log it

**Verification**: how to confirm the procedure succeeded
**Rollback**: how to undo if needed; "n/a" if irreversible (and note that)
**Related**: links to other runbooks, handoff sections, ADRs
```

---

## 9.3 Identity & Access runbooks

### Provision a System Admin (deploy-time)

**When**: bootstrap of a new environment; promotion of an operator.
**Who**: deployment automation OR an existing System Admin via the seed mechanism.
**Severity**: medium (deploy-time)
**Estimated duration**: minutes

**Procedure**:
1. Confirm the operator's IdP-canonical email is verified (per [§4.1 of handoff](../handoff/04-bounded-contexts/4.1-identity.md)).
2. Add their `external_subject_id` to the System Admin seed list (deploy config or secrets manager — never in-band).
3. Deploy / re-run seed.
4. Notify the operator via secure channel.
5. Have the operator sign in once to confirm.

**Verification**: operator can access the System Admin matrix UI; their actions appear in `audit_records` with `actor.user_id` set.
**Rollback**: remove from the System Admin seed list; redeploy. (Their `User` record persists per soft-delete.)
**Related**: [§5.1.1 of handoff](../handoff/05-cross-cutting.md), [`06-context-playbooks/6.1-identity.md`](06-context-playbooks/6.1-identity.md).

---

### Disable a User

**When**: account compromise; off-boarding; abuse / TOS violation.
**Who**: System Admin (capability `user.disable`).
**Severity**: paging if compromise; medium otherwise.
**Estimated duration**: minutes

**Procedure**:
1. Confirm the basis for disabling (incident ticket; HR notice; security report).
2. Run `DisableUser(user_id, reason)` via the admin UI or via the documented admin script.
3. Verify the user's sessions are terminated (the use case does this; verify in `sessions` table where `user_id = X AND terminated_at IS NULL` returns empty).
4. If the user was Org Owner, **immediately initiate `TransferOrgOwnership`** to a designated successor (the system rejects further `org.*` actions by the disabled user).
5. Record the action in the incident ticket.

**Verification**: `User.status = 'disabled'`; user cannot sign in (`/auth/sign-in` returns user_disabled error); `identity.user_disabled` event present in audit.
**Rollback**: `ReactivateUser(user_id)` reverses the status; sessions are NOT restored (user must sign in again).
**Related**: [`06-context-playbooks/6.1-identity.md`](06-context-playbooks/6.1-identity.md).

---

### Process a right-to-erasure (`ScrubUser`)

**When**: PDP / GDPR data-subject request; required compliance window per jurisdiction.
**Who**: System Admin (capability `user.scrub`).
**Severity**: high (compliance SLA)
**Estimated duration**: tens of minutes to hours depending on data volume

**Procedure**:
1. Verify the request through the documented identity-verification channel (not in-band — usually email to the user's verified address requesting confirmation).
2. Record the compliance ticket reference.
3. Run `ScrubUser(user_id, by_actor, reason)` via the admin UI.
4. Monitor the cross-context scrub progress (the orchestration walks Identity → Tenancy → Order → Audit per [§5.6.2 of handoff](../handoff/05-cross-cutting.md)).
5. Verify `identity.user_pii_scrubbed` emitted; verify audit `source_envelope` fields scrubbed for affected records.
6. Reply to the data subject confirming completion within the compliance SLA.
7. Archive the ticket.

**Verification**:
- `User.pii_scrubbed_at` is set.
- `User.email` etc. are replaced with deterministic placeholders per `design/pii.md`.
- Spot-check `audit_records` where `actor.user_id = X`: `source_envelope` PII fields scrubbed.
- Foreign references (in Tenancy memberships, Order records) still resolve.

**Rollback**: **N/A — irreversible by design.** The whole point of right-to-erasure.
**Related**: [§5.6.2 of handoff](../handoff/05-cross-cutting.md), [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md), [`design/pii.md`](../design/pii.md).

---

### Rotate IdP client credentials

**When**: scheduled rotation (per security policy); suspected credential exposure.
**Who**: Platform-scoped with `platform.idp.rotate` (or System Admin).
**Severity**: high if exposure; medium otherwise.
**Estimated duration**: under 30 minutes

**Procedure**:
1. Generate new client secret at the IdP provider.
2. Store the new secret in the secrets manager (do NOT remove the old yet).
3. Deploy with both old and new accepted (if supported by the OIDC adapter) — or deploy in a tight window.
4. Verify successful sign-ins on the new secret.
5. Remove the old secret from the IdP provider's configuration.
6. Remove the old secret from the secrets manager.
7. Verify no further use of the old secret in logs.

**Verification**: signed-in users continue working; new sign-ins succeed; no `idp_authentication_failed` errors in the period after rollover.
**Rollback**: revert to the old secret if available (within the deletion window).
**Related**: [§6.5.2 of handoff](../handoff/06-operational.md), [`06-context-playbooks/6.1-identity.md`](06-context-playbooks/6.1-identity.md).

---

### Force sign-out a User (all sessions)

**When**: suspected compromise of a single user's session.
**Who**: System Admin OR the user themselves.
**Severity**: paging if compromise; otherwise self-service.
**Estimated duration**: seconds

**Procedure**:
1. Run `SignOutEverywhere(user_id)`.
2. Notify the user via secure channel.
3. If compromise suspected, consider `Disable` (above) until the user re-verifies.

**Verification**: all rows in `sessions` for the user have `terminated_at` set; subsequent API calls with stale cookies return 401.
**Rollback**: N/A. User signs in again to get a new session.

---

## 9.4 Tenancy runbooks

### Onboard a new Organization (operational)

**When**: a customer asks to onboard; needs prep beyond what `CreateOrg` self-service handles.
**Who**: Platform-scoped support (`platform.org.onboard` if defined) OR coordinate with the customer to use self-service.
**Severity**: low (planned)
**Estimated duration**: under an hour

**Procedure**:
1. Confirm the customer has a verified IdP account.
2. Have them run `CreateOrg` themselves via the admin UI (they become Org Owner automatically).
3. Walk through `CreateStore` for their first store.
4. Optionally `AssignStoreAdmin` for additional team members.
5. Hand off the documentation pointer for `/orgs/<slug>/...` URLs.

**Verification**: Org appears in their `/orgs` switcher; Store appears in the Org's store list.
**Rollback**: N/A for the Org (no deletion); customer can leave the Org Paused.
**Related**: [`06-context-playbooks/6.2-tenancy.md`](06-context-playbooks/6.2-tenancy.md).

---

### Suspend a Store (platform moderation)

**When**: TOS violation, fraud report, payment dispute, compliance issue.
**Who**: Platform-scoped (`platform.store.suspend`).
**Severity**: high (visible to buyers)
**Estimated duration**: minutes; resolution timeline varies

**Procedure**:
1. Document the basis (ticket, complaint, compliance flag).
2. Run `SuspendStore(store_id, reason)`.
3. Verify Bridge republishes with `isActive: false` per the wire-state mapping (per [§6.7 of context-playbooks](06-context-playbooks/6.7-beckn-bridge.md)).
4. Notify the Store Owner via the documented channel.
5. Track the resolution window per policy.

**Verification**: `Store.status = 'suspended'`; `tenancy.store_status_changed` event emitted; CDS publish completed (check Bridge logs / `bridge_outbox`).
**Rollback**: `ReactivateStore(store_id)` returns to `Active`. Do NOT use `PauseStore` to undo a suspension — `Suspended → Paused` is forbidden ([§5.8 of handoff](../handoff/05-cross-cutting.md)).
**Related**: [`06-context-playbooks/6.2-tenancy.md`](06-context-playbooks/6.2-tenancy.md), [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md).

---

### Manual Org-wide republish (post-incident catalog refresh)

**When**: CDS was down / out of sync; catalog content drift suspected; large schema migration affecting projection.
**Who**: Org Owner (`org.republish_all_stores`) OR Platform-scoped equivalent.
**Severity**: medium
**Estimated duration**: depends on Org's store count and catalog size

**Procedure**:
1. Confirm Bridge → CDS is operational.
2. Run `RequestOrgRepublishAll(org_id)` via the admin UI.
3. Monitor `bridge_outbox` for the resulting publish entries; verify they reach `state: sent`.
4. Spot-check a BAP sandbox query — confirm refreshed content visible.

**Verification**: all of the Org's Active stores have recent successful publish entries in `bridge_outbox`.
**Rollback**: N/A (publish is idempotent; the next event-driven publish overwrites).
**Related**: [ADR-0019](../decisions/0019-manual-catalog-republication.md), [`06-context-playbooks/6.7-beckn-bridge.md`](06-context-playbooks/6.7-beckn-bridge.md).

---

## 9.5 Beckn Bridge runbooks

### Rotate the platform Beckn signing key (ONIX-side)

**When**: scheduled rotation; suspected key compromise.
**Who**: Platform-scoped, coordinating with ONIX vendor.
**Severity**: paging if compromise; high otherwise (touches network identity).
**Estimated duration**: hours (announcement + network propagation)

**Note**: per [ADR-0022](../decisions/0022-onix-protocol-gateway.md), the signing key lives in **ONIX** (vendor binary). The BPP holds no signing key for outbound. This runbook is now a coordination procedure with the ONIX vendor.

**Procedure**:
1. Open a vendor support ticket / use the documented vendor channel to request key rotation.
2. Vendor generates the new key pair per Beckn-prescribed algorithm and registers the new public key with the Beckn registry while the old one remains active.
3. Wait for registry propagation per network policy (typically minutes to hours).
4. Vendor cuts ONIX over to the new private key for outbound signing; existing inbound verification continues to work since ONIX may verify against multiple keys during overlap.
5. After the transition window (announced to network operators), vendor deregisters the old public key.
6. **BPP-side action**: confirm `bridge_outbox` is delivering successfully against the rotated ONIX (no spike in ION-1001 from BAPs that may have stale keys).

**Verification**: outbound signatures from ONIX verified by stub BAP using the new public key; no spike in inbound ION-1001 failures during the transition.
**Rollback**: vendor-side rollback to the old key from their secrets manager (during the overlap window only).
**Related**: [ADR-0022](../decisions/0022-onix-protocol-gateway.md), [§3.2 of handoff](../handoff/03-beckn-integration.md), [`06-context-playbooks/6.7-beckn-bridge.md`](06-context-playbooks/6.7-beckn-bridge.md).

---

### Re-register with the Beckn registry (ONIX-side)

**When**: registry-side issue invalidated our entry; ONIX-network endpoint migrated.
**Who**: Platform-scoped, coordinating with ONIX vendor.
**Severity**: paging (BPP off-network if not handled)
**Estimated duration**: under an hour

**Note**: per [ADR-0022](../decisions/0022-onix-protocol-gateway.md), registry interactions are ONIX's responsibility. The BPP no longer holds registry credentials.

**Procedure**:
1. Open a vendor support ticket / use the documented vendor channel to request re-registration.
2. Vendor confirms current registry state and submits registration update per the Beckn registry's API.
3. Vendor confirms registry propagation.
4. **BPP-side action**: verify inbound traffic resumes (test with stub BAP via ONIX).
5. Verify outbound callbacks reach BAPs (check `bridge_outbox` `state: sent` rate returns to normal).

**Verification**: stub BAP can complete a `/select → /on_select` round-trip via ONIX; production traffic resumes if previously dropped.
**Rollback**: vendor re-submits with previous configuration if available.

---

### Replay catalog publishes after ONIX or CDS outage

**When**: ONIX or upstream CDS was unreachable for a period; catalogs published during outage are stale.
**Who**: Platform-scoped (`platform.bridge.replay_catalog`).
**Severity**: high (BAP discovery affected)
**Estimated duration**: minutes per store, depends on counts

**Procedure**:
1. Confirm ONIX is reachable from the BPP (`curl <ONIX_ENDPOINT>/health` or vendor-defined).
2. If the outage was upstream (ONIX → CDS), coordinate with the vendor to confirm CDS connectivity is restored.
3. Identify affected stores (those with `tenancy.store_status_changed` or `catalog.*` events emitted during the outage window — check the event log).
4. Trigger republish for each:
   - For an entire Org: `RequestOrgRepublishAll(org_id)`.
   - For a single store: `RequestStoreRepublish(store_id)`.
   - For all platform-wide: run a script that fans out per Active store.
5. Monitor `bridge_outbox` for `state: sent`.

**Verification**: `bridge_outbox` shows recent successful publish entries for affected stores; BAP sandbox queries return current content.
**Rollback**: N/A (republish is idempotent).
**Related**: [ADR-0019](../decisions/0019-manual-catalog-republication.md), [ADR-0022](../decisions/0022-onix-protocol-gateway.md).

---

### Investigate ONIX unreachable

**When**: `bridge.outbox.failure_rate` spike with HTTP errors targeting `ONIX_ENDPOINT`; `/readyz` reports ONIX unreachable.
**Who**: Platform-scoped on-call.
**Severity**: paging (BPP off-network)
**Estimated duration**: minutes to hours depending on root cause

**Procedure**:
1. Confirm scope: is it a single instance failing or all BPP traffic? Check observability for `bridge.outbox.5xx_count` and `bridge.outbox.4xx_count`.
2. Try `curl <ONIX_ENDPOINT>/health` (or vendor-defined healthcheck) from a BPP host.
3. Determine the failure mode:
   - **ONIX process crashed / restarting** → wait for vendor's auto-restart; outbox retries should resume. If extended, open vendor ticket.
   - **Network partition** between BPP and ONIX → infrastructure issue; escalate to the network/platform team.
   - **ONIX configuration drift** (e.g., signing key issue, registry credentials expired) → vendor ticket for ONIX-side investigation.
   - **`ONIX_ENDPOINT` env misconfigured on the BPP** → check secrets manager / deploy config.
4. While ONIX is down, BPP traffic accumulates in `bridge_outbox` with `state: pending` (5xx retries). The BPP is "soft offline" but messages aren't lost.
5. After ONIX recovers, monitor `bridge_outbox` for catch-up.

**Verification**: `bridge.outbox.success_rate` returns to baseline; no `state: pending` rows older than 5 minutes; `/readyz` reports ONIX reachable.
**Rollback**: N/A (this is failure-mode triage, not a change to roll back).
**Related**: [ADR-0022](../decisions/0022-onix-protocol-gateway.md), [§9.10 below](#section-deploy-restart) (ONIX restart).

---

### Investigate a signature-failure spike (BPP-side re-verification)

**When**: `bridge.inbound.signature_failure` alert fires (BPP re-verification failures, per ADR-0022 §6).
**Who**: Platform-scoped on-call.
**Severity**: paging
**Estimated duration**: minutes to hours depending on root cause

**Note**: post-[ADR-0022](../decisions/0022-onix-protocol-gateway.md), ONIX already verifies inbound signatures at the network door (returning ION-1001 to BAPs). Failures at the BPP layer mean: ONIX verified but the BPP's re-verification rejected — usually a configuration / key-availability problem at the BPP, not a BAP issue.

**Procedure**:
1. Check the rate of BPP-side re-verification failures vs. baseline. A handful is noise; a spike is signal.
2. Identify the source BAP (`context.bap_id` from rejected payloads in logs).
3. Determine the failure mode:
   - **BAP key not available to the BPP** (per N2 resolution): if N2=(a), our registry client is failing; if N2=(b), ONIX isn't forwarding the key header. Investigate accordingly.
   - **Our re-verification adapter regression** → check recent deploys; consider rollback.
   - **Drift between ONIX-verified state and BPP-verified state** (rare; suggests ONIX bug or key cache staleness) → vendor ticket.
4. Document the determination in the incident ticket.

**Verification**: failure rate returns to baseline.
**Rollback**: deploy rollback if our adapter caused the regression.

---

### Update ION error registry from ion-specs

**When**: `ion-specs` publishes new ION-XXXX codes (per `ion-specs/errors/README.md`) or updates existing entries; scheduled periodic sync.
**Who**: Engineering, with platform sign-off.
**Severity**: low (planned)
**Estimated duration**: under an hour

**Procedure**:
1. Pull the latest `ion-specs/errors/registry.json`.
2. Run the BPP's mapping-registry test suite against the new registry. Any breaking changes (existing code removed, `http_status` changed for an existing code) will surface as test failures.
3. Review additions — do any new codes need a domain-side mapping (e.g., new ION-3xxx transactional code that maps to a domain error)?
4. Update `bridge/mapping-registry/errors.ts` if domain-side mappings need to be added.
5. Deploy as a standard rollout (not paging).
6. Verify in production by checking `bridge.outbox.4xx_codes` distribution; new codes appear in the histogram if BAPs encounter the new conditions.

**Verification**: BPP test suite green; production observability shows correct mapping behavior post-deploy.
**Rollback**: revert to prior `registry.json` snapshot + revert mapping-registry changes; deploy.
**Related**: [ADR-0022 §10 N3](../decisions/0022-onix-protocol-gateway.md#open-follow-ups) (proposed additions to the upstream registry).

---

### Switch protocol-version mappers (Beckn version migration)

**When**: planned Beckn version migration (e.g., v2 → v3).
**Who**: Platform-scoped + engineering.
**Severity**: high (coordinated change with network)
**Estimated duration**: weeks of planning + minutes of cutover

**Procedure**:
1. Implement v(next) handlers + mappings in `bridge/protocol/versions/v<next>/`.
2. Coexist: both versions accepted inbound. Detect version from `context.core_version`.
3. Outbound versions per BAP capability (per the registry / context block from BAP).
4. Test extensively against ion-specs payloads for v(next).
5. Announce migration window to network operators.
6. Cut over outbound to v(next) for BAPs that support it.
7. After all BAPs migrated, deprecate v(old) inbound (alert on continued v(old) traffic).
8. Eventually remove v(old) handlers (separate change).

**Verification**: v(next) traffic flows; no v(old) errors after deprecation.
**Rollback**: keep v(old) handlers; revert any v(next)-only outbound logic.
**Related**: [§4.4 of handoff](../handoff/03-beckn-integration.md), [`06-context-playbooks/6.7-beckn-bridge.md`](06-context-playbooks/6.7-beckn-bridge.md).

---

## 9.6 Events / consistency runbooks

### Investigate a stuck-events alert

**When**: `stuck_events.count` alert fires.
**Who**: Platform-scoped on-call.
**Severity**: paging
**Estimated duration**: minutes to hours

**Procedure**:
1. Identify the stuck event(s) and the failing subscriber: query `stuck_events` table by `subscription_name`.
2. Inspect the event payload + error message recorded for each.
3. Categorize:
   - **Transient external failure** (DB connection blip, IdP timeout) → retry the event (see "Replay events" below); usually clears.
   - **Bad payload** (event shape doesn't match subscriber's expectations) → investigate the producer; consider hot-fix subscriber to tolerate; or `skip-with-acknowledgement` (below) if the event is genuinely unprocessable.
   - **Subscriber bug** → deploy fix; retry the events.
4. Document determination in incident ticket.

**Verification**: `stuck_events.count` returns to zero; subscriber's offset advances.
**Rollback**: N/A (events stay until processed or explicitly skipped).
**Related**: [§5.3.5 of handoff](../handoff/05-cross-cutting.md).

---

### Skip-with-acknowledgement for an unfixable stuck event

**When**: an event genuinely cannot be processed (corrupted payload from a bug, etc.) and retrying is futile.
**Who**: Platform-scoped (`platform.events.skip` if defined).
**Severity**: high (data integrity decision)
**Estimated duration**: minutes

**Procedure**:
1. Document the reason for skipping: which event, which subscriber, what's wrong, what won't happen because we skip.
2. Get written sign-off from a second operator (two-person rule for irreversible operational decisions).
3. Mark the stuck event as `skipped` with the reason recorded in `stuck_events` (or equivalent table).
4. Manually compensate if needed (e.g., if the missed event would have created an AuditRecord, write a manual AuditRecord noting the gap).
5. Document in the incident postmortem.

**Verification**: subscriber's offset advances past the skipped event; no further attempts on the skipped row.
**Rollback**: **N/A — irreversible.** The system will not process this event again.
**Related**: [§5.3.5 of handoff](../handoff/05-cross-cutting.md).

---

### Replay events for a subscriber

**When**: a subscriber bug was deployed; events processed incorrectly during a window need re-processing; a new subscriber needs to catch up on historical events.
**Who**: Platform-scoped.
**Severity**: medium
**Estimated duration**: minutes (small windows) to hours (full retention sweep)

**Procedure**:
1. Identify the time window or event-ID range to replay.
2. **For an existing subscriber after fix**: clear the relevant rows from the subscriber's `processed_events` table so the inbox dedup doesn't short-circuit; then reset the subscriber's offset to before the window.
3. **For a new subscriber**: register the subscription; backfill processed_events for already-known events if desired (or accept they'll re-process).
4. Restart the dispatcher / subscriber worker.
5. Monitor processing rate and any new errors.

**Verification**: subscriber catches up to current; no new stuck events from the replay.
**Rollback**: if the replay produces bad state, identify the subscriber-side mutations and reverse them; replays are not natively reversible.
**Related**: [§5.2.6 of handoff](../handoff/05-cross-cutting.md), [§5.3.6 of handoff](../handoff/05-cross-cutting.md).

---

### Reset an inbox after a subscriber data migration

**When**: a subscriber's read model schema changed; its `processed_events` no longer correctly reflects what's been applied.
**Who**: Platform-scoped + engineering.
**Severity**: medium
**Estimated duration**: minutes + replay time

**Procedure**:
1. Coordinate with the subscriber team — what window of events needs re-processing?
2. Take a snapshot of `processed_events` before resetting (audit trail).
3. Truncate / partial-delete `processed_events` for the subscription within the window.
4. Trigger replay (see above).

**Verification**: subscriber's read model is consistent with current event stream within the window.
**Rollback**: restore `processed_events` from the snapshot.

---

## 9.7 Audit runbooks

### Generate a compliance report

**When**: regulator request; periodic internal review.
**Who**: System Admin OR Platform-scoped with `audit.read.platform`.
**Severity**: high (compliance SLA)
**Estimated duration**: hours

**Procedure**:
1. Capture the request scope (date range, Org/Store, event categories).
2. Run the documented audit-query against `audit_records` using the `audit_reader` role.
3. Export the result (CSV / JSON per request) — through documented export tooling, not by direct DB dump (PII handling).
4. Have a second operator review before delivery.
5. Deliver via the documented secure channel.
6. Record the request + delivery in the compliance ticket.

**Verification**: receipt confirmation from requester; records of audit reads are themselves audited (System Admin reading audit produces audit entries).
**Rollback**: N/A (read-only).
**Related**: [`06-context-playbooks/6.8-audit.md`](06-context-playbooks/6.8-audit.md), [§5.6 of handoff](../handoff/05-cross-cutting.md).

---

### Retention sweep (scheduled)

**When**: scheduled cadence per `retention-sweep.ts` job; or on-demand if backlog accumulates.
**Who**: automated (scheduled); Platform-scoped if manually triggered.
**Severity**: low (routine)
**Estimated duration**: minutes per category

**Procedure**:
1. The sweep runs automatically per schedule (per [`06-context-playbooks/6.8-audit.md`](06-context-playbooks/6.8-audit.md)).
2. Monitor sweep logs for errors.
3. If manual trigger needed (skipped runs, configuration change), invoke `RetentionSweep(category?)` via the documented admin script.
4. Verify expected row counts deleted.

**Verification**: `audit_records` for the swept category have no rows past retention.
**Rollback**: **N/A — deleted records are gone.** The point of the sweep is irreversible deletion. Take a backup before running a manual sweep if needed.
**Related**: [§4.7 of handoff](../handoff/04-bounded-contexts/4.7-audit.md), [`06-context-playbooks/6.8-audit.md`](06-context-playbooks/6.8-audit.md).

---

### Handle a subpoena / legal data-access request

**When**: legal process compels production of records.
**Who**: System Admin + legal counsel.
**Severity**: high (legal SLA)
**Estimated duration**: days

**Procedure**:
1. Receive the subpoena through documented legal-process channels.
2. Validate scope with legal counsel (don't over- or under-produce).
3. Run targeted audit queries (per the compliance-report runbook above).
4. **Place affected records on litigation hold** — flag in operational config so retention sweeps skip them until the hold is released.
5. Produce records through legal counsel.
6. Document chain of custody.
7. Release hold when legal process completes.

**Verification**: legal counsel confirms production; hold released; records audit-trail-preserved.
**Rollback**: N/A (legal process).

---

## 9.8 Catalog / Inventory runbooks

### Bulk-import a catalog (operational)

**When**: store onboarding with existing catalog from elsewhere; no v1 UI for this.
**Who**: System Admin / operational engineer.
**Severity**: low (planned)
**Estimated duration**: hours

**Procedure**:
1. Convert the source catalog into the documented import format (CSV per template, or JSON per schema in `packages/bpp-contracts/`).
2. Validate the file offline (use the documented validator script).
3. Run the import script against the target store, as a System Admin / Org-Owner impersonation.
4. Spot-check imported products in the admin UI.
5. Verify event flow: each imported product should have generated a `catalog.product_created` event in the audit log.

**Verification**: spot-check Products list in admin UI; spot-check StockLevel auto-creation in `stock_levels` table.
**Rollback**: archive imported products (`ArchiveProduct` each); no hard delete.
**Related**: [§7.2.3 of handoff](../handoff/07-open-issues.md) (v1 deferral note).

---

### Bulk-adjust stock (post physical count)

**When**: physical inventory count completed; reconciliation needed.
**Who**: Store Admin OR Org Owner.
**Severity**: low (planned)
**Estimated duration**: minutes per store

**Procedure**:
1. Capture the physical counts in the documented spreadsheet format.
2. Compare against current `StockLevel.stock_count` per item.
3. For each discrepancy: run `AdjustStock(item_ref, delta, reason='physical_count_reconciliation')` via the admin UI or bulk script.
4. Document the reconciliation in the operational log.

**Verification**: spot-check final counts in admin UI; `inventory.stock_corrected` events visible in audit.
**Rollback**: `AdjustStock` with the inverse delta; document the reversal.
**Related**: [`06-context-playbooks/6.4-inventory.md`](06-context-playbooks/6.4-inventory.md).

---

### Deprecate a PlatformCategory (System Admin)

**When**: taxonomy reorganization; category renamed or split.
**Who**: System Admin (`platform.category.deprecate`).
**Severity**: medium (affects store choices)
**Estimated duration**: planning + minutes to execute

**Procedure**:
1. Announce the deprecation to affected store owners with a window (e.g., 30 days).
2. Optionally pre-create the replacement category and document the mapping.
3. Run `DeprecatePlatformCategory(category_id)` — existing assignments remain valid; new assignments are rejected.
4. Track store owners who reassign their products.
5. After the window, follow up with stores still using the deprecated category.
6. Optionally `DeletePlatformCategory` once no assignments reference (only possible if zero references).

**Verification**: `PlatformCategory.status = 'deprecated'`; admin UI shows it as legacy in the picker.
**Rollback**: there's no `UndoDeprecate` use case; if needed, manually update status (operational; requires written justification).
**Related**: [`06-context-playbooks/6.3-catalog.md`](06-context-playbooks/6.3-catalog.md).

---

## 9.9 Order & Fulfillment runbooks

### Force-cancel a stuck Order (admin)

**When**: an Order is in `Initiated` or `Confirmed` and the buyer / BAP is unresponsive past reasonable thresholds; or known data issue.
**Who**: Store Admin OR Platform-scoped support.
**Severity**: medium
**Estimated duration**: minutes

**Procedure**:
1. Verify the Order's current state and the reason for force-cancelling.
2. Run `Cancel(order_id, reason, by_actor)` via the admin UI.
3. Verify compensation: if Initiated → reservations released; if Confirmed → voucher reverted + payment marked Refunded.
4. Notify the BAP / buyer if appropriate per policy.

**Verification**: `Order.status = 'cancelled'`; reservation rows reflect Released; voucher_usage.reverted_at set if applicable.
**Rollback**: N/A (cancel is one-way per the state machine).
**Related**: [`06-context-playbooks/6.6-order.md`](06-context-playbooks/6.6-order.md).

---

### Reconcile payment status from external signal

**When**: external payment system (BAP-side or gateway) recorded a status change that didn't reach the BPP (webhook dropped, signal-lag, etc.).
**Who**: Platform-scoped support.
**Severity**: medium
**Estimated duration**: minutes per order

**Procedure**:
1. Verify the external status from the source-of-truth system.
2. Confirm the BPP's current `payment_status` and `payment_external_ref`.
3. Run `UpdatePaymentStatus(order_id, new_status, external_ref)` via the admin path.
4. Verify the transition is allowed (e.g., `Pending → Captured` ok; `Captured → Pending` rejected).
5. Document in the operations log.

**Verification**: `Order.payment_status` matches external truth; `order.payment_status_changed` event emitted.
**Rollback**: another `UpdatePaymentStatus` with the prior status (if allowed by state machine).

---

### Investigate "quote won't initiate"

**When**: a BAP reports `/init` fails repeatedly for the same `transaction_id`.
**Who**: Platform-scoped on-call.
**Severity**: high (BAP-visible failure)
**Estimated duration**: minutes

**Procedure**:
1. Look up the Order by `(bap_id, transaction_id)`.
2. Check Order's status and Quote TTL:
   - If `Expired` → quote validity passed; BAP must restart with `/select`.
   - If `Initiated` already → previous `/init` succeeded; this is a BAP retry confusion; BAP should proceed to `/confirm`.
   - If `Created` → check the most recent `/init` attempt's failure mode in logs.
3. Check Inventory availability for line items — `InsufficientStockError` is common.
4. Check voucher status if applied — `VoucherExpiredError` / `VoucherExhaustedError`.
5. Communicate findings to the BAP.

**Verification**: BAP can complete the flow OR gets a clear typed error explaining why not.
**Rollback**: N/A (read-only investigation).

---

## 9.10 Database / Infrastructure runbooks

### Backup

**When**: scheduled (daily at minimum; per compliance / RPO requirements).
**Who**: automated; Platform-scoped if manual.
**Severity**: low (routine); paging if backups fail.
**Estimated duration**: minutes to hours depending on DB size

**Procedure**:
1. Backups run on the configured schedule via the DB hosting provider (D7) or self-managed tooling.
2. Verify the backup completed successfully (check storage; check size sanity).
3. Periodically test restore (see Restore runbook).

**Verification**: backup artifact present in storage; size in expected range; manifest valid.
**Rollback**: N/A (backup is itself the rollback mechanism).

---

### Restore from backup

**When**: data loss; corruption; deliberate point-in-time recovery; restore-test drill.
**Who**: Platform-scoped + engineering.
**Severity**: paging
**Estimated duration**: hours

**Procedure**:
1. **Stop the affected service** to prevent additional writes.
2. Provision a fresh DB instance (do NOT restore into a live DB without coordination).
3. Restore from the chosen backup point.
4. Run integrity checks (row counts vs expected; spot-check key tables).
5. Re-point application to the restored DB (or promote the restored DB to primary).
6. Resume service.
7. Document the restore: what was lost, what was recovered, gap analysis.

**Verification**: application functional; key queries return expected data; no PII breaches in logs during restore.
**Rollback**: revert to the original DB if restore is wrong (typically only possible during the cut-over window).
**Related**: [§6.7 of handoff](../handoff/06-operational.md).

---

### Zero-downtime schema migration

**When**: any production migration that affects existing tables.
**Who**: engineering + Platform-scoped during cutover.
**Severity**: high (live system)
**Estimated duration**: planned per migration

**Procedure**:
1. **Expand**: deploy the migration that adds new columns / tables / indexes (backward-compatible with current code).
2. Wait for the migration to complete (large tables may take time).
3. Deploy the code that uses the new schema (still backward-compatible).
4. Verify in production.
5. **Backfill** any data into new columns (separate job).
6. **Contract**: deploy code that no longer uses the old schema.
7. Deploy a migration that drops the old schema elements.

**Verification**: zero failed writes during the migration; new schema observable in production; no regressions.
**Rollback**: at each step, the previous deploy is rollback. After Contract step, rollback requires forward-fix with a new migration.

---

### Failover

**When**: primary DB unreachable; primary AZ outage.
**Who**: Platform-scoped on-call.
**Severity**: paging
**Estimated duration**: minutes

**Procedure**:
1. Confirm the primary is genuinely down (not a connectivity issue from one client).
2. Trigger failover per the DB hosting provider's documented procedure.
3. Verify the new primary accepts writes.
4. Update connection strings if not auto-resolved.
5. Monitor application recovery.
6. Open an incident; document RTO and any data loss (RPO).

**Verification**: application reads + writes succeed against the new primary.
**Rollback**: failover BACK to the original primary once it's healthy (planned, with cutover window).

---

## 9.11 Cross-cutting runbooks

### Edit the role → capability matrix

**When**: new capability added; permission boundary changes; new role.
**Who**: System Admin only ([§5.1.3 of handoff](../handoff/05-cross-cutting.md)).
**Severity**: medium (affects all authorization)
**Estimated duration**: minutes per change

**Procedure**:
1. Plan the change: which role gains/loses which capability, and why.
2. Use the System Admin matrix UI (when Phase 8 ships) OR the documented admin script.
3. Verify the change took effect: spot-check a user with the affected role; their authorization decisions should now reflect the new grants.
4. Document in the change log.

**Verification**: `requireCapability` calls return expected allow / deny per role; `identity.authorization_denied` events change shape post-change.
**Rollback**: re-edit the matrix to restore the previous grants.
**Related**: [§5.1 of handoff](../handoff/05-cross-cutting.md), [`06-context-playbooks/6.2-tenancy.md`](06-context-playbooks/6.2-tenancy.md).

---

### Investigate authorization denial spike

**When**: `auth.denied` metric spike alert fires.
**Who**: Platform-scoped on-call.
**Severity**: high if widespread; medium if scoped
**Estimated duration**: minutes

**Procedure**:
1. Identify the capability(ies) being denied — query `audit_records` where `event_name = 'identity.authorization_denied'` in the alert window, group by capability.
2. Identify affected actors and scopes.
3. Determine root cause:
   - **Recent matrix change** removed a grant inadvertently.
   - **Application bug** is calling `requireCapability` with the wrong scope.
   - **Real attack** — many actors probing.
4. Fix:
   - Matrix change → revert per "Edit the matrix" runbook.
   - App bug → hotfix.
   - Attack → escalate per security policy.

**Verification**: alert returns to baseline.

---

### Emergency feature flag toggle

**When**: a feature is causing production issues; planned dark-launch transition.
**Who**: Platform-scoped (`platform.feature_flag.toggle`).
**Severity**: high (in production change)
**Estimated duration**: under a minute to take effect

**Procedure**:
1. Identify the flag and the desired state.
2. Toggle via the documented admin path.
3. Verify the change propagated (cache TTL may apply).
4. Monitor the affected feature.
5. Document in the operations log.

**Verification**: behavior matches new flag value.
**Rollback**: toggle back.

---

## 9.12 Severity levels and on-call

| Severity | Definition | Response time | Examples |
|---|---|---|---|
| **Paging** | User-visible outage; data integrity at risk; security incident | Immediate | Signing-key compromise; DB primary down; broad authorization denial spike |
| **High** | Significant degradation; SLA at risk; compliance window | Within 1 hour | Single store suspension under dispute; stuck events backlog; payment reconciliation backlog |
| **Medium** | Localized issue; no immediate SLA risk | Within 1 business day | Single user disable; voucher mis-configuration; non-critical replay |
| **Low** | Planned / routine | Per schedule | Backups; retention sweep; planned migrations |

Page-worthy alerts ([§6.2.2 of handoff](../handoff/06-operational.md)):
- `stuck_events.count > 0` (after grace period)
- `beckn.inbound.signature_failure` spike
- `subscriber.dlq_count` increasing
- DB primary unhealthy
- Beckn registry credentials approaching expiry

---

## 9.13 Runbook authoring rules

When you add a new runbook (or update an existing one):

- **Follow the template** in §9.2. Consistency makes runbooks scannable in an incident.
- **Be specific.** "Investigate the issue" is not a procedure. "Query `audit_records` where X; if Y then Z" is.
- **Include verification.** A runbook without "how do you know it worked" isn't done.
- **Include rollback or state it's N/A.** Irreversible operations need to say so loudly.
- **Keep up to date.** When the system changes, runbooks that reference it get updated in the same PR. Stale runbooks are worse than missing ones.
- **Link related runbooks.** Don't duplicate steps; reference.
- **Don't include credentials.** Reference where to find them in the secrets manager; don't paste them.
- **Rehearse before production.** A runbook that's never been run is a draft. Run it in staging at least once.

---

> **End of developer-guide.** From here, the implementing team is the source of truth.
