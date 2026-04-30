# WDP-CHANGE-LOG.md
**Worldpay Dispute Platform — Platform-Level Change Log**
*Started: April 2026*

---

## Purpose

This log captures the **platform-level impact** of every component update. It is
written by per-component chats at the end of Phase 2 and consumed by
**reconciliation sessions** that rebuild derivative documents (WDP-DB.md,
WDP-KAFKA.md, WDP-HANDOVER.md, and others) from the current state of the
component files.

### What belongs here
- Which derivative documents are affected by a component change
- Which rows, sections, or entries inside those documents need to change
- Deviation flag status for each component update
- Candidate new ADRs surfaced by the component update

### What does NOT belong here
- Full component file contents (those live in `WDP-COMP-NN-*.md`)
- Rewritten derivative documents inline (those are produced in reconciliation
  sessions, not here)
- Implementation details, code, or config

---

## How to use this file

### In a per-component chat (Phase 2)
At the end of Phase 2, Claude appends a new block under **Pending Entries**
using the template at the bottom of this file. Nothing above the Pending
Entries header is modified.

### In a reconciliation session
1. Read this file and note the current **Reconciliation Marker**
2. Consume all entries under **Pending Entries**
3. Rebuild the affected derivatives
4. Move the consumed entries from **Pending Entries** to **Reconciled**
5. Update the **Reconciliation Marker** with the new date and the derivatives
   that were rebuilt

---

## Reconciliation Markers

*Each marker records one reconciliation session — what was rebuilt, when, and
against which set of Pending Entries.*

| Date | Derivatives rebuilt | Entries consumed | Notes |
|------|---------------------|------------------|-------|
| 2026-04-25 | WDP-DB.md v2.1, WDP-KAFKA.md v2.1, WDP-HANDOVER.md v3.1, WDP-NFRS.md v2.1, WDP-DECISIONS.md v2.1, WDP-INTEGRATIONS.md v2.1, WDP-ARCHITECTURE.md v2.1, WDP-COMP-INDEX.md v2.1 | 20 entries — see entry list below | First full reconciliation session. All 8 Tier 1/Tier 2 derivatives rebuilt against accumulated Pending Entries from 2026-04-18 through 2026-04-25. **20 of 41 DRAFT component files now carry source-verified status (📝 DRAFT 🔍).** Architect confirmation pending across all 20. |

**Entries consumed in 2026-04-25 reconciliation pass (20 total):**

| Date | Component | Version transition |
|------|-----------|---------------------|
| 2026-04-18 | COMP-07 VisaDisputeBatch | v1.0 → v1.1 |
| 2026-04-18 | COMP-08 FirstChargebackBatch | v1.0 → v2.0 |
| 2026-04-18 | COMP-09 CaseFillingBatch | v1.0 → v2.0 |
| 2026-04-18 | COMP-11 FileProcessor | v1.0 → v1.1 |
| 2026-04-18 | COMP-12 InboundDisputeEventScheduler | v1.0 → v1.1 |
| 2026-04-18 | COMP-13 FileAcknowledgementProcessor | v1.0 → v2.0 |
| 2026-04-18 | COMP-14 CaseCreationConsumer | v1.0 → v2.0 |
| 2026-04-18 | COMP-15 EvidenceConsumer | v1.0 → v1.1 |
| 2026-04-18 | COMP-16 BusinessRulesProcessor | v1.0 → v1.1 |
| 2026-04-18 | COMP-18 NotificationOrchestrator | v1.0 → v2.0 |
| 2026-04-23 | COMP-19 AcceptService | v1.0 → v2.0 |
| 2026-04-23 | COMP-20 ContestService | v1.0 → v2.0 |
| 2026-04-23 | COMP-23 CaseManagementService | v1.0 → v1.1 |
| 2026-04-23 | COMP-24 CaseActionService | v1.0 → v1.1 |
| 2026-04-23 | COMP-37 DocumentManagementService | v1.0 → v1.1 |
| 2026-04-24 | COMP-27 CaseSearchService | v1.0 → v2.0 |
| 2026-04-25 | COMP-17 CaseExpiryUpdateConsumer | v1.0 → v1.1 |
| 2026-04-25 | COMP-21 ChargebackService | v1.0 → v1.1 |
| 2026-04-25 | COMP-41 ThirdPartyNotificationConsumer | v1.0 → v1.1 |
| 2026-04-25 | COMP-43 CoreNotificationConsumer | v1.0 → v2.0 |

**Reconciliation summary:**
- **WDP-NFRS.md v2.1:** 60 new RISK rows added (RISK-024 through RISK-082); RISK-009 withdrawn (underlying table absent from source); RISK-015 extended by RISK-040; RISK-018 refined.
- **WDP-DECISIONS.md v2.1:** Deviation maps for DEC-001, DEC-003, DEC-004, DEC-005, DEC-014, DEC-016, DEC-019, DEC-020, DEC-023 enriched. 22 candidate ADRs surfaced for architect promotion. No new DEC numbers introduced.
- **WDP-DB.md v2.1:** `wdp.dispute_event_change_log` row removed (table doesn't exist); column corrections (I_ACCT_CDH on nap.case + BC.TBC_DM_CASE); writer correction (COMP-18 not a writer of `wdp.outgoing_event_outbox`).
- **WDP-KAFKA.md v2.1:** business-rules topic confirmed 6 publishers (COMP-37 added; COMP-14 candidacy resolved as NOT a publisher); GP_EVENTS channel correction; multiple COMP-12 corrections.
- **WDP-INTEGRATIONS.md v2.1:** JustAI scope split (COMP-21 active inbound vs COMP-41 planned outbound); COMP-21 added as 38-call-site outbound owner; Cert eAPI new section; MCM stage-dependent path detail; AMEX/DISCOVER + MC CHI + NAP-publish split-brain consequence.
- **WDP-ARCHITECTURE.md v2.1:** Minimal topology-level updates — mostly cross-references to component-level findings recorded in NFRS and DECISIONS.
- **WDP-COMP-INDEX.md v2.1:** 20 components marked 📝 DRAFT 🔍 source-verified; 17 component descriptions enriched with audit findings; Reconciliation Tracker section added.
- **WDP-HANDOVER.md v3.1:** Confirmed Architectural Facts massively expanded with 17 component subsections; ~30 currently-open questions documented.

**⚠️ Snapshot note:** The Pending Entries section in this file may not contain all 20 reconciled entries depending on snapshot freshness. The 20-entry list above is authoritative — sourced from WDP-HANDOVER.md v3.1 reconciliation history. Verify your local copy and move all 20 entries to Reconciled if they exist.

---

## Reconciled — 2026-04-25 reconciliation pass

*All entries below were absorbed into the v2.1 derivative documents on
2026-04-25. See Reconciliation Markers above for the full scope summary.*

⚠️ **Snapshot note:** This section contains 9 of the 20 reconciled entries
(all dated 2026-04-18). The remaining 11 entries (COMP-18 from 2026-04-18;
COMP-19, 20, 23, 24, 37 from 2026-04-23; COMP-27 from 2026-04-24; COMP-17,
21, 41, 43 from 2026-04-25) were consumed in this reconciliation pass but
may live in a more current local copy of this file. Move them here when
this file is next reconciled with the local copy.

### 2026-04-18 — COMP-16 BusinessRulesProcessor (source audit confirmed)

**Component:** 16 — BusinessRulesProcessor
**Doc version:** v1.0 DRAFT → v1.1 (architect confirmation still PENDING)
**Trigger:** Direct source audit of `gcp-business-rules-processor` on
2026-04-18. Closed configuration/scaling gaps left open in v1.0 DRAFT;
corrected a material mis-label (FCMG → FCHG); surfaced two new US-only
behaviours and one asymmetry between UK and US paths.

---

**1. Corrections to facts previously in COMP-16 DRAFT**

| Topic | v1.0 DRAFT said | Source shows |
|---|---|---|
| DEC-005 pre-ACK line numbers | line 37 ack / line 40 process | **line 38 ack / line 41 process** (`KafkaConsumer.java:38,41`) |
| FCMG guard | `actionType == FCMG AND prior FCMG without RCAL → set queue to WEXQUE` | **FCHG / RCAL** pairing guard — checks FirstChargeback (`FCHG`) action types paired with `RCAL`. On guard failure, UK code calls `updateCase(...)` at `NAPRulesProcessServiceImpl.java:207`. The "queue to WEXQUE" behaviour is not supported by source and has been removed |
| US DB bean name | `wdpDataSource` | **`wdpdataSource`** (lowercase `d`, `USDataSourceConfig.java:29`) |
| UK transaction manager | Implied `napTransactionManager` | **`ukTransactionManager`** (`UKDataSourceConfig.java:43`) |
| Token cleanup | "in finally block" | **Asymmetric** — UK `tokenService.clear()` INSIDE finally (`NAPRulesProcessServiceImpl.java:157`); US `tokenService.clear()` OUTSIDE finally (`USRulesProcessServiceImpl.java:144`) |
| REST base path | Inferred from artefact naming | **Confirmed** `/merchant/gcp/business-rules-processor` from `SERVER_SERVLET_CONTEXT_PATH` in `resources.yml:62-63` |
| `ErrorLogService` | "commented out" | **Class does not exist in source tree** — autowire and call sites commented out in `KafkaServiceImpl.java:31-32, 66-67` |

**2. New facts added**

- **US-only `DOCUMENT_ATTACHED_TO_OPEN_CASE` recursive fallback** —
  `USRulesProcessServiceImpl.java:170-185`. When issuer-doc check returns
  `issuerDocAddedToCase=true` and the matched rule's `applyRuleGroup` is
  blank, the US path recursively evaluates rules under
  `DOCUMENT_ATTACHED_TO_OPEN_CASE`. UK has no equivalent.
- **US-only action types** — `US_OUTGOING_PRE_ARB` and `MERCHANT_ACCEPT`
  dispatched only on the US path.
- **US-only double `updateCaseAction` call** — US path calls
  `caseService.updateCaseAction()` twice per rule action (pre with
  `isActionUpdate=true` at line 600; post with `isActionUpdate=false` at
  line 739). UK path calls it once.
- **`v-correlation-id` header always null in production** —
  `RestClientInvoker.getHeaders()` reads from `MDC.get(CORRELATION_ID)`,
  but no MDC-populating filter or interceptor exists anywhere in the
  component. All outbound correlation-id propagation is effectively
  broken.
- **`outgoing-events` publish and `internal-integration-events` publish
  can both fire for the same message** — DMT001 publish does not suppress
  the finally-block outgoing publish.
- **`isErrorOccured` flag from outgoing publish is ignored by caller** —
  publish failures surface only as SNOTE via REST; the caller does not
  react to the flag.
- **Audit-log sequences** — `nap.br_case_audit_log_id_seq`,
  `wdp.br_case_audit_log_id_seq`, both `allocationSize=1`. No
  `@Transactional` on `saveAuditLog` — implicit Spring-Data-JPA
  transaction per `saveAll`. No unique constraint on either audit table.

**3. Configuration and scaling gaps closed**

| Parameter | Value | Source |
|---|---|---|
| Memory request / limit | 2048 Mi / 4096 Mi | `resources.yml:58,60` |
| CPU request / limit | Not set (Burstable QoS) | `resources.yml` |
| Rolling update | maxSurge=1, maxUnavailable=0, minReadySeconds=30 | `resources.yml:11-14,25` |
| Topology spread | maxSkew=1, topologyKey=`kubernetes.io/hostname`, whenUnsatisfiable=ScheduleAnyway | `resources.yml:26-32` |
| HPA | None | No `HorizontalPodAutoscaler` in manifest |
| PDB | None | No `PodDisruptionBudget` in manifest |
| Liveness | `GET /livez`, initialDelay 40s, period 10s, timeout 5s, failureThreshold 3 | `resources.yml:45-52` |
| Readiness | `GET /readyz`, initialDelay 30s, period 10s, timeout 5s, failureThreshold 3 | `resources.yml:36-44` |
| Actuator endpoints | `info`, `health`, `prometheus` on port 8082 | `application.yml:35-39` |
| Container port | 8082 | `resources.yml:55` |
| Security | OAuth2 Resource Server (JWT), CSRF disabled, whitelist `/actuator/health`, `/livez`, `/readyz` | `SecurityConfig.java:29-37` |

---

**4. WDP-KAFKA.md updates required**

**Section 2 — DEC-005 Deviations table, row for COMP-16:**
Change "line 37 of KafkaConsumer.java before processRulesEvent()" to
**"line 38 of KafkaConsumer.java before processRulesEvent() on line 41"**.

**Section 2 — DEC-001 Deviations table, row for COMP-16:** No change
(still covers both `outgoing-events` and `internal-integration-events`).

**Section 2 — DEC-003 Deviations table, row for COMP-16:** No change
(caseNumber on both producer topics).

**Section 3 — Topic Registry:**
- `outgoing-events` row: no content change; reference updated to v1.1.
- `internal-integration-events` row: reword COMP-16 note to clarify
  DMT001-only publish is **independent of** the finally-block
  `outgoing-events` publish (both can fire for the same message).
- `business-rules` row: no change.

**Section 4 — Consumer Group Registry, row for
`business-rules-processor-group`:** Update wording to
**"Pre-ACK — MANUAL_IMMEDIATE committed on line 38 of KafkaConsumer.java,
before processRulesEvent() on line 41 (DEC-005 deviation). Concurrency: 1
(Spring default)."**

**Section 5 — Components without Kafka involvement:** No change (COMP-16
is not in this list).

---

**5. WDP-DB.md updates required**

**Section 2 — Tables, `wdp.br_case_audit_log` row:** Append to Notes
**"No unique constraint declared — duplicate rows possible if DEC-005
pre-ACK is ever remediated without adding dedup. Sequence
`wdp.br_case_audit_log_id_seq`, allocationSize=1. No `@Transactional`
on save — implicit per-`saveAll` JPA transaction."**

**Section 2 — Tables, `nap.br_case_audit_log` row:** Same appended note
with UK sequence name.

**Section 2 — `wdp.CASE`, `wdp.ACTION`, `nap.case`, `nap.ACTION`,
`nap.rules`, `nap.rule_criterion`, `nap.rule_action`, `nap.rule_group`,
`wdp.rules`, `wdp.rule_criterion`, `wdp.rule_action`, `wdp.rule_group`
rows:** No ownership change. Confirm COMP-16 listed as reader.

**No new shared table risk introduced** by this revision. The existing
shared-audit-log note (COMP-31 reads these tables for its GET
`/audit-log/case/{caseNumber}` endpoint) remains accurate.

---

**6. Deviation flags — COMP-16 confirmed**

| DEC | Requirement | Actual | Severity |
|---|---|---|---|
| DEC-001 | Transactional outbox for Kafka publish | Not implemented on either producer topic — direct synchronous `.send().get()` | 🔴 HIGH |
| DEC-003 | Partition key = `merchantId` | `caseNumber` on both producer topics (consumer inbound is `merchantId` — upstream-compliant) | 🟡 MEDIUM |
| DEC-004 | PAN encryption before persistence | NOT APPLICABLE — no PAN handled by this component | 🟢 LOW (n/a) |
| DEC-005 | Manual offset commit AFTER processing | Pre-ACK: line 38 ack precedes line 41 process in `KafkaConsumer.java` | 🔴 HIGH |
| DEC-011 | BRE step checkpointing | ⛔ VOID — never implemented; formally voided April 2026 | 🟢 LOW (informational) |
| DEC-014 | Resilience4j circuit breakers | ⛔ VOID — absent platform-wide; formally voided April 2026 | 🟡 MEDIUM (accepted) |
| DEC-017 | BRP reads rules directly from DB | COMPLIES — confirmed | 🟢 LOW (compliant) |
| DEC-019 | Clear PAN on persistent store | NOT APPLICABLE — no PAN handled | 🟢 LOW (n/a) |
| DEC-020 | Full at-least-once idempotency | DEVIATES — `idempotency-key` pass-through only, no dedup store; compounded by DEC-005 pre-ACK | 🔴 HIGH |

---

**7. Remaining gaps — need follow-up**

| Gap | What type of follow-up |
|---|---|
| COMP-14 CaseCreationConsumer — does it publish to `business-rules` topic after case creation? | **Claude Code follow-up** on COMP-14 repo: *"Does `CaseCreationConsumer` publish to the `business-rules` Kafka topic after case creation? If so, include the call site (file:line), the Kafka template bean, and the payload type."* |
| `v-correlation-id` gap — fix or accept? | **Architect decision** — defect (add MDC filter populated from inbound Kafka headers) or accepted limitation? |
| US-path token cleanup outside finally (line 144) — defect or intentional? | **Architect decision** / team confirmation — candidate for symmetry fix with UK path |
| FCHG guard failure → `updateCase(...)` semantic — is this the intended final behaviour, or is there a missing downstream queue-transfer step that was removed? | **Team confirmation** — owning team to validate |
| Admin `PUT /event` endpoint — runbook and production posture | **Team confirmation** — remain enabled in prod, or gate by profile? |
| Exact production values for `max_poll_records`, `session_timeout_ms`, `heartbeat_interval_ms`, `max_poll_interval` env vars | **Environment config / team confirmation** — values live in K8s secrets outside repo |
| Replica count `{{ replicas-gcp-business-rules-processor }}` XL-Deploy substitution | **Environment config / team confirmation** — value outside repo |
| Kafka producer `retries`, `linger.ms`, `batch.size`, `compression.type`, `delivery.timeout.ms`, `request.timeout.ms` — not set, relying on client defaults | **Architect decision** — accept defaults or set explicit values for prod? |
| `rule_group` owning writer for both schemas | **Claude Code follow-up** on COMP-32 RulesService repo (already open question in WDP-HANDOVER) |
---

### 2026-04-18 — COMP-15 EvidenceConsumer · v1.0 DRAFT → v1.1 DRAFT

**Source:** `wdp-evidence-consumer` @ v1.1.2 — source-verified by
Claude Code 2026-04-18. Architect confirmation still pending.

**Nature of change:** Correction pass against source. No functional change
in production; 17 corrections and additions to the documented behaviour of
an existing component. Two findings rise to new 🔴 HIGH risk severity
(V3 silent-ATTACHED data-integrity defect, V3 PATCH ghost-upload). Mermaid
flow rebuilt to show two distinct transaction boundaries and correct
WDP-path-only gating of IDP / field-check / multi-doc.

#### Platform-level impacts

**WDP-DB.md**
- No change to the shared-writer row for `wdp.chbk_outbox_row`,
  `wdp.CASE`, `wdp.ACTION`, `wdp.disputes_questionnaire`, or `wdp.file_evidence` —
  COMP-15 write attribution is unchanged.
- Clarification candidates (not corrections):
    - `wdp.file_evidence` row for COMP-15: add note that
      `appended_file_name` may be `null` on V3 path for
      MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC due to confirmed data-integrity
      defect — downstream readers should not assume this column is
      populated on every ATTACHED row.
    - `wdp.ACTION` shared-writer row: note confirmed — no `@Version`,
      no `@UniqueConstraint`, no `SELECT FOR UPDATE` on any WDP entity
      accessed by COMP-15. Existing 🔴 HIGH severity retained.
    - `wdp.disputes_questionnaire` shared-writer row: confirmed —
      COMP-15 writes only on WDP path RESPDOC. Existing 🔴 HIGH severity
      (COMP-26 POST idempotency gap) unchanged.

**WDP-KAFKA.md**
- Section 2 (Topic Registry) — `case-evidence-events` row: add clarification
  that inbound `RECEIVED_KEY` (caseNumber) is `@Nullable`; ordering
  guarantee by `caseNumber` holds only when upstream sets a non-null key.
- Section 2 (Topic Registry) — `business-rules` row: the COMP-15 producer
  entry is still accurate, but:
    - Add: RESPDOC WDP path emits **two** events per success
      (`ACTION_UPDATED` then `DOCUMENT_UPLOADED`, sorted alphabetically).
    - Add: DSNOTDOC WDP path publishes `source=""` (builder has no
      DSNOTDOC branch — downstream-routing defect).
    - Add: Platform string is uppercased on publish by COMP-15.
    - Add: Compensating BR events emitted by `updateFailedStep` when
      `retry_count > 2` escalates to ERROR and gate conditions met
      (WDP path AND `!isMultiDocPending` AND `documentType≠DSNOTDOC`
      AND `caseLookupResponse` present).
- Section 3 Confirmed DEC-005 Deviations — COMP-15 entry still accurate.
- Section 4 Confirmed DEC-001 Deviations — COMP-15 entry still accurate;
  add note that the publish is `kafkaTemplate.send(...).get()` blocking
  on the future inside `@Transactional`.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: On COMP-15 V3 path, MISCDOC / DRFTDOC / RESPQDOC / ISSRQDOC
  document types silently mark `file_evidence.attachment_status=ATTACHED`
  without any upload occurring — data-integrity defect, not merely an
  "unfinished state" as previously documented.
- Add: On COMP-15 V3 RESPDOC path, a failed ownership-transfer PATCH
  leaves the document uploaded to V3 Core but the WDP DB unmarked —
  ghost-upload failure mode. No automatic reconciliation.
- Add: COMP-15 publishes two BR events per successful RESPDOC on the
  WDP path (`ACTION_UPDATED` then `DOCUMENT_UPLOADED`), sent synchronously
  inside `TXN_WDP` via `kafkaTemplate.send(...).get()`.
- Add: COMP-15 DSNOTDOC WDP events publish with `source=""` — the
  builder has no DSNOTDOC branch. This is a suspected downstream-routing
  bug that requires COMP-16 consumer-side investigation.
- Add: COMP-15 V3 Update Action PATCH mutates the shared `RestTemplate`
  bean's request factory — after the first V3 PATCH call, every
  subsequent REST call on the pod inherits 30s/30s timeouts. Order-
  dependent global state.
- Add: COMP-15 inbound Kafka key and `idempotency-key` header are both
  `@Nullable`. Nulls pass through to DMS upload and to the outbound
  `business-rules` message.
- Add: COMP-15 WDP-path gating is narrower than prior documentation
  implied — IDP fetch, required-field validation, and multi-doc sibling
  scan are all skipped on the V3 legacy path.
- Add: COMP-15 `coreMigrationFlag` is sourced from env var
  `${core_migration_flag}` via K8s secrets / configmap. Fixed at
  container start; no runtime toggle.
- Resolved open questions: COMP-15 Step 6 `@Transactional` boundary
  (closed — `wdpTransactionManager`, `rollbackFor=Exception.class`,
  default propagation, two distinct transaction methods for WDP vs V3);
  COMP-15 compensating-BR behaviour (closed — lives inside
  `updateFailedStep`, gated as above); `isMultiDocPending` scan
  semantics (closed — negation against `NOT_VALID_STATUS = {ATTACHED,
  SKIPPED, ERROR}`); PENDING_DEFERRED re-driver (closed at COMP-15 scope
  — no re-driver in this repo, cross-component follow-up needed).
- New open questions to add to the platform list:
    - OQ: COMP-15 V3-path MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC silently
      mark `file_evidence=ATTACHED` — is the fix an upload path for
      these types, or an explicit skip/reject path? Architect decision.
    - OQ: COMP-15 DSNOTDOC WDP events publish with `source=""` — is
      COMP-16 tolerant of this, or does it misroute? Follow-up Claude
      Code question against `wdp-business-rules-processor`: *"How does
      BusinessRulesProcessor (COMP-16) handle inbound events on the
      `business-rules` topic where the `source` field is the empty
      string?"*
    - OQ: PENDING_DEFERRED re-driver ownership — which WDP component
      polls or re-drives `chbk_outbox_row.status=PENDING_DEFERRED`?
      Follow-up Claude Code scan across all components. If no re-driver
      exists, this is a latent stuck-record source.
    - OQ: COMP-15 `vantiveLicense` YAML/Java key case mismatch — is the
      production path relying on env-var override of `${vantive_license}`
      from K8s secrets? Team confirmation.
    - OQ: COMP-15 has no K8s liveness/readiness/startup probes — is this
      intentional for this workload class, or a deployment template
      gap? Operational confirmation.
    - OQ: COMP-15 `{{ replicas-wdp-evidence-consumer }}` runtime value —
      environment config. Confirm via XL Deploy / Helm.

**WDP-DECISIONS.md · Candidate new ADRs**
- No new ADRs. The DEC-001 deviation map already captures COMP-15.
  Candidate clarifications:
    - DEC-001 COMP-15 entry: add note that the publish is a blocking
      `future.get()` inside `@Transactional` — the ghost-event window
      is narrow but non-zero (Kafka success → later DB write failure in
      the same transaction).
    - DEC-020 deviation map: add COMP-15 — four distinct at-least-once
      gaps (pre-ACK silent loss window; deserialization silent drop;
      no DB-level unique constraint on `file_evidence`; compensating
      BR events require `caseLookupResponse` to be present, so errors
      before case lookup emit no compensating event). Severity 🔴 HIGH.
    - DEC-003 deviation map: COMP-15 entry unchanged — `caseNumber` on
      both inbound and outbound topics.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- Candidate new RISK rows (pending architect decision):
    - "V3-path document-type dispatcher silently marks
      `file_evidence=ATTACHED` without upload for unhandled document
      types (MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC) — silent data-integrity
      defect on CORE legacy path." 🔴 HIGH.
    - "V3 Update Action PATCH failure leaves V3-side document uploaded
      but WDP DB unmarked — ghost-upload requiring manual
      reconciliation." 🔴 HIGH.
    - "Shared-`RestTemplate` mutation in COMP-15: V3 PATCH permanently
      reconfigures the shared bean's request factory, making timeout
      behaviour order-dependent across REST calls on the same pod."
      🟠 MEDIUM-HIGH (latent).
    - "COMP-15 has no Kubernetes liveness/readiness/startup probes —
      hung pods are not evicted by kubelet." 🔴 HIGH (operational).
- Existing RISK-013 (replica constraint has no automated enforcement) —
  not applicable to COMP-15 (no replica=1 constraint; this is a standard
  scaled consumer).

**WDP-INTEGRATIONS.md**
- No change. COMP-15 inbound and outbound contracts are component-
  internal (COMP-11 DB handoff, COMP-12 Kafka handoff, COMP-16 Kafka
  handoff, COMP-37 DMS call). No new external integration surface.

#### Deviation flags for COMP-15

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⛔ DEVIATES | 🔴 HIGH |
| DEC-003 Kafka Partition Key = merchantId | ⛔ DEVIATES | 🟡 MEDIUM |
| DEC-004 PAN Encryption Before Persistence | ✅ NOT APPLICABLE | — |
| DEC-005 Manual Kafka Offset Commit | ⛔ DEVIATES (pre-ACK) | 🔴 HIGH |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL — four gaps | 🔴 HIGH |

**DEC-001 DEVIATES detail:** Business Rules publish is
`kafkaTemplate.send(message).get()` called blocking inside
`TXN_WDP (@Transactional(wdpTransactionManager))`. Failure modes: (a)
Kafka send failure → JPA rollback — recoverable via updateFailedStep.
(b) Kafka success followed by a DB failure at a subsequent write in the
same transaction → ghost BR event in `business-rules` topic with no
corresponding WDP DB state — unrecoverable without manual intervention.

**DEC-003 DEVIATES detail:** Inbound `case-evidence-events` and
outbound `business-rules` both key on `caseNumber`. Consistent with
platform-wide deviation pattern across business-rules publishers.

**DEC-005 DEVIATES detail:** Pre-ACK at Step 1 — offset committed
immediately on receipt, before any DB read, S3 download, case lookup,
upload, or DB write. Any pod death after Step 1 causes permanent
evidence loss for that message.

**DEC-020 PARTIAL detail (severity 🔴 HIGH):** Four concurrent gaps.
(a) Pre-ACK silent loss window — any crash between Step 1 and Step 6
loses the message permanently. (b) Deserialization silent drop — no
audit record anywhere for malformed payloads. (c) No DB-level UNIQUE
constraint on `file_evidence (chbk_outbox_row_id)` — rolling deploy
replica overlap can produce duplicate upload. (d) Compensating BR
events require `caseLookupResponse` to be present, so errors occurring
before Step 4 (IDP, S3 download, field validation) emit no compensating
BR event on escalation.

#### Remaining gaps

- **OQ-COMP-15-1 V3 unhandled document-type defect** — architect
  decision. Do MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC have a legitimate V3
  upload path that is missing, or should they be explicitly rejected?
- **OQ-COMP-15-2 DSNOTDOC BR `source=""`** — follow-up Claude Code
  question against `wdp-business-rules-processor`: *"How does
  BusinessRulesProcessor (COMP-16) handle inbound events on the
  `business-rules` topic where the `source` field is the empty
  string? Is there a default rule group, a silent drop, or an
  error path?"*
- **OQ-COMP-15-3 PENDING_DEFERRED re-driver ownership** — follow-up
  Claude Code scan across all components: *"Which component reads or
  updates `wdp.chbk_outbox_row` rows where status=PENDING_DEFERRED?
  If none, this status is a terminal stuck state."*
- **OQ-COMP-15-4 `vantiveLicense` resolution path** — team
  confirmation. Is `${vantive_license}` env var injected in production
  via K8s Secret, making the YAML/Java key case mismatch harmless,
  or is the YAML path actually dead?
- **OQ-COMP-15-5 Absent K8s health probes** — operational confirmation.
  Is the absence of livenessProbe/readinessProbe/startupProbe intentional
  for this workload class, or a deployment template gap?
- **OQ-COMP-15-6 Production replica count** — environment config
  (`{{ replicas-wdp-evidence-consumer }}`). Not in repo.
- **OQ-COMP-15-7 V3 `${v3_upload_doc_url}` / `${v3_upload_action_url}`
  fully-qualified URLs** — environment config. Not in repo.
- **OQ-COMP-15-8 FileProcessor (COMP-11) write pattern on
  `wdp.file_evidence`** — confirmed from prior COMP-11 audit (see
  WDP-CHANGE-LOG.md 2026-04-18 COMP-11 entry). Cross-reference only.

#### Doc status after this change
- `WDP-COMP-15-EVIDENCE-CONSUMER.md` → `v1.1 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending
---

### 2026-04-18 — COMP-14 CaseCreationConsumer · v1.0 DRAFT → v2.0 DRAFT

**Source:** `gcp-case-creation-consumer` v1.3.7 — source-verified by Claude Code
2026-04-18. Architect confirmation pending.

**Nature of change:** Correction pass against source. No functional change in
production; 11 corrections to the v1.0 DRAFT plus four previously-undocumented
production-risk findings surfaced during the audit. Three findings rise to
🔴 HIGH severity. Closes 20 of 21 enumerated gaps (one gap remains not
determinable from source).

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- Clarify COMP-14's writer scope: status transitions ONLY (`FAILED`, `ERROR`,
  `SUCCESS`, `PENDING_DEFERRED`, `SKIPPED`, and `PENDING` on EVIDENCE_ATTACH
  children). This component does NOT insert new rows.
- Add: "COMP-14 writes are all **auto-commit** — zero `@Transactional`
  annotations anywhere in `gcp-case-creation-consumer` source. Parent SUCCESS
  save and EVIDENCE_ATTACH child `saveAll` are two separate auto-commits. A
  crash between them leaves the outbox in a non-atomic, non-self-healing state
  that the Kafka offset (pre-ACK) will not cause to be redelivered."
- Add: "COMP-14 `retry_count > 2 → ERROR` promotion is Java-side, not a DB
  trigger or CHECK constraint. Any direct-DB status manipulation bypasses the
  promotion logic."

**WDP-DB.md · Section 2 · `wdp.case` row**
- Add COMP-14 to read-only accessor list. Purpose: Capone REQ stage duplicate
  check via JPA method-name derivation over (level1Entity, tr, acctCdhLst,
  purchaseAmt, originalDisputeAmount). Read-only; no `@Query`; no lock.

**WDP-DB.md · Section 4 · Shared Table Risk Register**
- Append to `wdp.chbk_outbox_row` risk entry: "COMP-14 source-verification
  2026-04-18 confirms the Layer-1 prior-chargeback dedup is a 2-column DB
  query (`networkCaseId + cardNetwork`) plus an in-memory Java stream filter
  for status/event-type. No DB lock, no `SELECT ... FOR UPDATE`. Concurrent
  processing of the same `(networkCaseId, cardNetwork)` by two replicas — for
  example during rolling-update overlap or any concurrency-setting increase —
  can produce duplicate case creation. Current safety is entirely operational:
  concurrency = 1 and Kafka consumer-group partition assignment."

**WDP-KAFKA.md · Section 3 Topic Registry · `new-case-events` row**
- Add / confirm: Consumer groups `new-case-events-group` (prod) and
  `new-case-events-group-cert` (cert). The earlier documented `-dev` name is
  incorrect — the cert profile uses `-cert`. Confirm whether a separate dev
  profile and group exists (not in source for this audit pass).
- Add partition-key annotation: compound key logged as
  `keyNetworkCaseCardNetworkId` — consumer side. DEC-003 deviation confirmed
  both sides (producer COMP-12 + consumer COMP-14).

**WDP-KAFKA.md · Section 4 Consumer Group Registry · `new-case-events` row**
- Confirm: AckMode = `MANUAL_IMMEDIATE`, syncCommits = true, pre-ACK before
  `processKafkaNotificationEvent()`. DEC-005 deviation.
- Correct: `auto.offset.reset = latest` (not `earliest` — the v1.0 DRAFT and
  the KAFKA.md entry should be cross-checked for any earlier-propagated
  `earliest` claim).
- Add: Empty anonymous `CommonErrorHandler` — deserialization exceptions
  silently swallowed, no DLT, no log, no counter. Add this as a row-level
  note alongside the existing pre-ACK annotation.
- Concurrency = 1 (default — `setConcurrency()` never called). Max poll
  records, max poll interval, session timeout, heartbeat interval are all
  env-injected with no YAML defaults — values not determinable from source.

**WDP-KAFKA.md · Section 6 Components Confirmed Kafka-Free**
- No change. COMP-14 is a consumer, not producer-free. However, add an
  explicit note to the CaseCreationConsumer row in Section 3/4 stating
  "no Kafka producer in this component — no `KafkaTemplate`, no
  `ProducerFactory`, no reference to `business-rules` topic. This closes
  Observability-doc OQ-02 (COMP-14 does not publish to business-rules)."

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: COMP-14 has zero `@Transactional` annotations — all DB writes
  auto-commit. Parent SUCCESS and EVIDENCE_ATTACH child writes are not
  atomic. Implication is that DEC-001 deviation in COMP-14 is not just
  "outbox-not-transactional" but "no transactions at all on the DB path".
- Add: COMP-14 `auto.offset.reset = latest` — cold starts with no committed
  offset SKIP messages rather than replay.
- Add: COMP-14 registers an empty `CommonErrorHandler`; combined with
  `ErrorHandlingDeserializer` and pre-ACK, deserialization exceptions are a
  distinct silent-loss class from the pre-ACK window.
- Add: COMP-14 has no liveness probe and no startup probe — only readiness.
  A stuck listener thread will not restart the pod.
- Add: COMP-14 Layer-1 prior-chargeback dedup is in-memory stream filter
  over a 2-column query — safe only by concurrency=1 operational constraint.
- Resolved open questions:
    - OQ-02 (observability doc): COMP-14 does not publish to `business-rules`.
      Confirmed — no producer in source.
    - DEC-003 consumer-side verification: compound key
      `keyNetworkCaseCardNetworkId` logged on receipt; not used for routing.
      Deviation confirmed both sides.
    - Kafka `auto.offset.reset` value: `latest` (not `earliest`).
    - ACK line: KafkaConsumer.java:38 (not :36).
    - Cert group: `new-case-events-group-cert` (not `-dev`).
    - Workflow hardcode trigger: `QueueName.COMP / PRECOMP` (not
      `schemeRef.category`).
    - NAP guard: confirmed NAP is not blocked — flows through lookups.
    - OQ-03 (observability doc): `wdp.chbk_outbox_row.status = SUCCESS` writer
      is COMP-14 via `chbkOutboxRepository.save()` at ChbkOutboxServiceImpl.
      EVIDENCE_ATTACH children set to PENDING on the NEW path, also by
      COMP-14, via a separate auto-commit `saveAll`.
- New open questions:
    - OQ-14.1: Mastercard `RE2` / `reversal=Y` handling in COMP-14.
      **Follow-up Claude Code question** against the COMP-14 repo:
      *"Search `gcp-case-creation-consumer` for any branch, switch, or
      conditional that evaluates `cardNetwork == MASTERCARD` together with
      either `disputeStage == RE2` or `reversal == Y`. Cite file:line. If no
      such branch exists, search for any reference to `MASTERCARD` that
      invokes a distinct action filter. Report whether the logic is inside
      `firstActionRuleLookup` generically or is absent entirely."*
    - OQ-14.2: `preActionStatusRule` lookup `@Recover` behaviour.
      **Follow-up Claude Code question:** *"In `EventProcessingServiceImpl`
      (gcp-case-creation-consumer), locate the `@Recover` method associated
      with the pre-action-status-rule lookup. Report its return value
      (null / FAILED write / other) and whether its null return is null-
      checked by the caller."*
    - OQ-14.3: Production values for `dsCaseLookup`, `skipStageList`,
      `creditMerchantActionCode`, `complianceRecordType`,
      `precomplianceRecordType`, `apcCaseLookupDisputeStages`,
      `acfCaseLookupDisputeStages` — **environment config / team
      confirmation.** None have YAML defaults; all env-var only.
    - OQ-14.4: Actual production replica count — **environment config /
      team confirmation.** Helm/Jinja placeholder in repo.
    - OQ-14.5: Env-injected Kafka client tunings (`max_poll_records`,
      `max_poll_interval`, `session_timeout_ms`, `heartbeat_interval_ms`)
      — **environment config / team confirmation.**

**WDP-DECISIONS.md · Candidate new ADRs**
- **Candidate ADR — "CaseCreationConsumer Operates Without @Transactional"
  🔴 HIGH.** Currently covered partially under the DEC-001 deviation map, but
  this component is a more extreme case than "no outbox atomicity" — it has
  no transactions at all on the Kafka processing path. Consider whether this
  warrants a dedicated risk-accepted ADR (analogous to DEC-019 and DEC-020)
  to make the condition explicit and to record operational compensations
  (e.g. monitoring for EVIDENCE_ATTACH drift).
- **Candidate addition to DEC-005 deviation map:** COMP-14's pre-ACK combined
  with the empty `CommonErrorHandler` and absence of a liveness probe
  compounds the at-most-once behaviour — deserialization errors on cold
  start with `auto.offset.reset = latest` produce silent skip; thread stalls
  produce silent block with a Ready pod. Add this as a "compound risk
  pattern" note.

**WDP-ARCHITECTURE.md**
- Clarify the CaseCreationConsumer section: enrichment paths for PIN share
  the same `gcp-merchant-transaction-service` URL as CORE (confirmed). The
  open question "does PIN follow MerchantTransactionService path" is
  resolved — yes, with additional per-platform validation fields
  (`termSequence`, `fromAcro`, `toAcro`).
- Clarify NAP guard: no code-level guard exists. Protection is operational
  (COMP-04 does not publish NAP events to this topic). If this assumption
  ever changes, COMP-14 will process NAP events and route them through
  non-NAP URL templates.

**WDP-NFRS.md · Section 6 Risk Register**
- Add RISK-COMP-14-A 🔴 HIGH: "CaseCreationConsumer has zero `@Transactional`
  annotations — all DB writes auto-commit. Parent SUCCESS and EVIDENCE_ATTACH
  child writes are two independent operations with no atomicity guarantee.
  Combined with DEC-005 pre-ACK, a crash between them produces silent
  inconsistency that is not self-healing."
- Add RISK-COMP-14-B 🔴 HIGH: "CaseCreationConsumer `RestTemplate` is bare-
  constructed — no connect timeout, no read timeout, no pool. Concurrency = 1
  means any hung downstream stalls the entire consumer thread. No liveness
  probe exists — pod remains Ready indefinitely. Compound with the 16+
  downstream REST dependencies."
- Add RISK-COMP-14-C 🔴 HIGH: "CaseCreationConsumer registers an empty
  `CommonErrorHandler`. Combined with `ErrorHandlingDeserializer` and
  MANUAL_IMMEDIATE pre-ACK, deserialization exceptions and any unhandled
  application exception are silently swallowed. No log line, no DLT, no
  counter. This is a silent-loss class distinct from the pre-ACK offset
  window."
- Add RISK-COMP-14-D 🟡 MEDIUM: "Four `@Recover` methods on the enrichment
  chain (`transactionLookup`, `productEntitleMentLookup`, `fraudSwitch`,
  `displayCodeDescription`) return `null` silently with a
  `// Error handling -- todo` comment. Processing continues with null fields
  and may reach SUCCESS on an under-enriched case."

**WDP-INTEGRATIONS.md**
- No change. COMP-14's external integration (Fraud Switch, AID user-detail)
  already listed. Internal dependencies are intra-platform.

#### Deviation flags for COMP-14

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⛔ DEVIATES | 🔴 HIGH |
| DEC-003 Kafka Partition Key = merchantId | ⛔ DEVIATES | 🟡 MEDIUM |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES | — |
| DEC-005 Manual Kafka Offset Commit | ⛔ DEVIATES | 🔴 HIGH |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⛔ DEVIATES | 🔴 HIGH |

**DEC-001 DEVIATES detail:** Zero `@Transactional` annotations anywhere in
`gcp-case-creation-consumer` source. Parent `chbkOutboxRepository.save()`
(SUCCESS write) and EVIDENCE_ATTACH `chbkOutboxRepository.saveAll()` (children
→ PENDING) are independent auto-commit operations. The upstream case-creation
REST call (`mdvs-gcp-case-management-service`) is a third independent
operation with no atomicity relative to either DB write. The `ChkbOutbox`
block on the case-management request body partially delegates outbox-closure
responsibility to COMP-23, making data-integrity ownership ambiguous. Retry
count promotion (`retry_count > 2 → ERROR`) is Java-side logic — any direct-DB
manipulation of the row bypasses it.

**DEC-003 DEVIATES detail:** Producer-side (COMP-12) already records the
deviation — compound key `networkCaseId + cardNetwork + platform`. Consumer-
side confirmed 2026-04-18: key received as `keyNetworkCaseCardNetworkId` and
logged only; not used for routing inside the consumer. Per-partition ordering
is scoped to the compound key, not to `merchantId`.

**DEC-005 DEVIATES detail:** `acknowledgment.acknowledge()` is called at
KafkaConsumer.java:38, immediately before `processKafkaNotificationEvent(...)`
at line 43. `AckMode.MANUAL_IMMEDIATE` with `syncCommits = true`. At-most-once
from the moment the listener enters. Combined with the empty
`CommonErrorHandler` and `auto.offset.reset = latest`, this produces three
distinct silent-loss classes: (1) deserialization errors swallowed by
`CommonErrorHandler`; (2) crashes between ACK and any outbox write; (3) cold
starts skipping any uncommitted offset.

**DEC-014 DEVIATES detail:** No Resilience4j dependency on classpath. No
`@CircuitBreaker`, `@Retry` (in the Resilience4j sense), `@RateLimiter`. The
only resilience is Spring-Retry `@Retryable` + `@Recover` at the individual
REST-method level — 3 of 7 `@Recover` methods write FAILED/ERROR; 4 absorb
failures by returning null with no outbox write. No timeouts on
`RestTemplate` — a hung downstream blocks the consumer thread indefinitely.
Platform-wide VOID already recorded.

**DEC-020 DEVIATES detail:** Multiple compounding violations of at-least-once
idempotency — pre-ACK (W2 crash-window loss), non-atomic parent-child writes
(W4/W5 inconsistency), Layer-1 dedup done in application memory over a
2-column query with no DB lock, silent-null `@Recover` stubs on enrichment,
empty `CommonErrorHandler` on deserialization, and reliance on downstream
services' un-verified idempotency via `idempotency-key` HTTP header. Local
dedup for concurrent-partition-assignment safety depends entirely on
operational concurrency = 1.

#### Doc status after this change
- `WDP-COMP-14-CASE-CREATION-CONSUMER.md` → `v2.0 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending

#### Remaining gaps

- **OQ-14.1 Mastercard RE2 / reversal=Y handling** — follow-up Claude Code
  question against the COMP-14 repo (exact wording above under New open
  questions).
- **OQ-14.2 `preActionStatusRule` @Recover wiring** — follow-up Claude Code
  question (exact wording above).
- **OQ-14.3 Production feature-flag values** — environment config / team
  confirmation. Seven env-only flags with no YAML defaults.
- **OQ-14.4 Production replica count** — environment config / team
  confirmation. Helm/Jinja placeholder only.
- **OQ-14.5 Env-injected Kafka client tunings** — environment config / team
  confirmation. Four tunings injected, no YAML defaults.
---

### 2026-04-18 — COMP-13 FileAcknowledgementProcessor · v1.0 DRAFT → v2.0 DRAFT

**Source:** `wdp-evidence-ack-scheduler` — source-verified by Claude Code 2026-04-18.
Architect confirmation pending.

**Nature of change:** Correction pass against source. No functional change in
production; the v1.0 DRAFT (GitHub Copilot CLI extraction) contained multiple
transcription errors in the Identity block and S3 key patterns, a referenced
profile file that does not exist, and missed three latent runtime bugs and one
third `headObject` failure branch. Three findings rise to HIGH severity.

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.file_job` row**
- No change to the writer list. Clarify under COMP-13's column scope: this
  component writes exactly four columns (`ack_status`, `ack_generated_at`,
  `updated_at`, `updated_by`), enforced by `@DynamicUpdate` on the `FileJob`
  entity. The broader "never-touches" column list previously implied by the
  DRAFT is NOT verifiable from this repo — the entity class declares only a
  narrow column set (`id`, `status`, `source`, `ack_required`, `ack_status`,
  `ack_generated_at`, `total_rows`, `total_evidences`, `error_code`,
  `updated_at`, `updated_by`). Columns that may exist in the live schema but
  are not bound on this entity (e.g. `file_name`, `s3_key`, `successful_rows`,
  `completed_at`, `error_message`, `created_by`) cannot be confirmed from
  this repo.
- Add cross-component note (already present for the COMP-11 attribution but
  reinforced here): COMP-11 never writes `status = COMPLETED`; COMP-12
  Scheduler2 owns that transition per WDP-DB.md. COMP-13 filters on
  `status IN (COMPLETED, ERROR)` so the ACK pipeline depends entirely on
  COMP-12 actually flipping the status. Contract is undocumented in all
  three repos.

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- No change to writer list. Reinforce read-only consumer attribution for
  COMP-13 (already present).
- Add note: schema ownership remains outside this repo (no DDL / Flyway /
  Liquibase / schema.sql). `ddl-auto = false`. `@Id` on `id` is the only
  entity-level constraint. Unique constraints and indexes remain not
  determinable from source — same open question as COMP-07, COMP-08, COMP-11.

**WDP-KAFKA.md · Sections 3 and 4**
- No change. COMP-13 remains in Section 4's Kafka-free components list. Audit
  re-confirms zero `spring-kafka`, `kafka-clients`, `aws-msk-iam-auth` in
  `pom.xml` and zero `@KafkaListener` / `KafkaTemplate` in source.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Corrections to DRAFT Identity block (all five values wrong):
- Artifact version: `1.0.3` → **`1.0.5`**
- Spring Boot: `3.5.11` → **`4.0.3`** (parent)
- Spring Cloud AWS: `3.1.0` → **`4.0.0`** (only `spring-cloud-aws-starter-s3`
  declared)
- JAXB: `4.0.6` → not pinned; `jaxb2-maven-plugin` = `4.0.0`, `jaxb-runtime`
  resolved via Spring Boot BOM
- logstash-logback-encoder: `9.0` → **`8.1`**

Corrections to S3 key patterns (digit-zero rendered as letter-O, digit-one as
letter-L, and CMRTR/BJWC outer prefixes previously shown as identical when
they differ by one character in source):
- Walmart filename: `AUF2_DWSG_ROBCDWL1_WMSIG_CONF_` →
  **`AUE2_DWSG_R0BCDWL1_WMSIG_CONF_`**
- CMRTR outer zip: `DCPO_RODMRDOA.WP01.ACK.` → **`DCPO_R0DMRDDA.WP01.ACK.`**
- CMRTR inner XML: `WP.CL.CMRTRTROUBLE.ACK.PROD.` →
  **`WP.C1.COMMTRANTROUBLE.ACK.PROD.`**
- BJWC outer zip: was shown identical to CMRTR →
  **`DCPO_R0DMRBDA.WP01.ACK.`** (distinct — differs from CMRTR by one char:
  `DD` vs `DB`)
- BJWC inner XML: `WP.BJW.CL.TRANTROUBLE.ACK.PROD.` →
  **`WP_BJWC.C1.TRANTROUBLE.ACK.PROD.`**

Corrections to scheduler / profile facts:
- `application-local.yml` **does not exist** in repo. Only `application.yml`,
  `application-cert.yml`, `application-prod.yml`. The hardcoded local-dev
  cron `0 */1 * * * *` previously attributed to `application-local.yml` is
  not present anywhere in source. The prod/cert value is env-injected via
  `${cron_scheduler}`; which of two `envFrom` secrets
  (`wdp-evidence-ack-scheduler-secrets` or `wdp-common-secrets`) supplies
  it is not determinable from source.

Corrections to flow / skip semantics:
- **DBLK has NO explicit skip branch.** v1.0 DRAFT implied one. Source shows
  `BulkResponseAckServiceImpl.generateAck` calls `uploadAckFileToS3`
  unconditionally with whatever `buildFile(...)` returned; on `IOException`
  the formatter returns `null` and the service does not guard, so `null`
  reaches the S3 upload path and is swallowed there.
- **`headObject` has THREE outcomes, not two.** v1.0 DRAFT flowchart
  showed "exists / not-found" binary. Source shows: (a) 200 exists → no
  upload; (b) `NoSuchKeyException` → proceed to `putObject`; (c) any other
  Exception → generic catch at `CommonAckServiceImpl`, returns `false`,
  no upload, `ack_status` stays `PENDING`. Branch (c) is a distinct
  stuck-job-adjacent path.
- `performTask` has an **outer catch** that aborts the entire cron fire on
  any uncaught exception from the poll or loop wrapper. A single bad job or
  a poll failure starves the whole batch until the next cron.

Add: three new latent bugs not surfaced in v1.0:
- **DBLK `yyyyMMddHHmmSS` pattern bug.** `BulkResponseConstants` timestamp
  pattern uses capital `SS` (fraction-of-second in `DateTimeFormatter`), not
  seconds. Formatting a `LocalDateTime.now()` throws
  `UnsupportedTemporalTypeException` at runtime. Every DBLK ACK attempt may
  currently be failing at the formatter stage, with the exception swallowed
  by the service-level generic catch. Severity 🔴 HIGH. Requires prod log
  evidence to confirm production-active impact — no test coverage because
  `src/test/` does not exist.
- **`uploadAckFileToS3` guard bug — `||` should be `&&`.** Guard
  `if (!key.isEmpty() || content != null)` lets null content or empty key
  through; downstream S3 SDK throws, caught by generic catch. Severity
  🟡 MEDIUM. Obscures root cause of any stuck-by-null-content case.
- **CapitalOne minute-resolution stuck-job.** CapitalOne S3 key timestamp
  truncates to `yyyyMMddHHmm`. Two CapitalOne ACK jobs processed within the
  same calendar minute in the same JVM produce the **same S3 key**. The
  second job's `headObject` returns 200, skips upload, never reaches the
  DB write — stuck on first encounter, not just after a failure.
  Deterministic failure mode under throughput > 1 CapitalOne ACK / minute.
  Severity 🔴 HIGH.

Add: `UniqueTimestampGenerator` behaviour.
- JVM-local `AtomicReference<LocalDateTime>` initialised to `LocalDateTime.MIN`.
  `generate()` is `synchronized`; returns `max(now(), last.plusSeconds(1))`.
- Minimum spacing 1 second inside one JVM. Not pinned to any input.
- Pod restart resets the counter.
- Service callers immediately re-format with a per-source
  `DateTimeFormatter` — this is what collapses the minimum-1-second
  guarantee down to seconds / minute resolution per source.

Add: `@DynamicUpdate` on `FileJob` entity — emits only dirty columns in
UPDATE. The write set is constrained by the method, but shared-column
collision protection relies on this annotation.

Add: TaskScheduler pool size = 1 (Spring Boot default, not explicitly set).
Intra-JVM overlapping cron fires are **queued**, not concurrent.

Add: dead config / hygiene findings.
- `spring.cloud.aws.region.static: "us-east-2"` is present in cert and prod
  profiles but not read; `S3ClientConfig` hardcodes `Region.US_EAST_2`.
- Redundant `jackson-core` re-declaration in `pom.xml` (already resolved via
  Spring Boot BOM).
- `spring-cloud-aws-starter-s3` autoconfig is shadowed by a manual
  `S3Client` bean in `config/S3ClientConfig.java`. Candidate for replacing
  with just the raw SDK module.
- Stale hardcoded Logstash destination (`10.43.145.125:5044`) commented out
  in `logback-spring.xml:17-18`.
- `@Configuration` annotation on both `FileJob` and `ChbkOutboxRow`
  entities — unintended, no observed downstream impact.
- No `src/test/` — zero tests of any kind.

Resolved open questions (from v1.0 DRAFT):
- "CapitalOne 2000-element cap behaviour" — RESOLVED. 2000 is XSD-only
  (`maxOccurs="2000"`). Java has no slicing, no truncation, no guard.
  Marshaller not configured with `setSchema`, so no runtime validation.
  Full list marshalled regardless; downstream receives XSD-invalid file
  if count > 2000.
- "CapitalOne MessageType four-way branch" — RESOLVED. `claim-request` and
  `outcome-chargeback` handled correctly; `claim-response` and
  `outcome-no-chargeback` fall through catch-all `else` to
  `status=REJECTED, validReject=false`. No tests cover either.
- "Does `headObject` throw anything other than `NoSuchKeyException` get
  handled?" — RESOLVED. Third branch exists (generic catch), returns
  `false`, no upload, ack_status stays PENDING.
- "Is @Transactional meaningful for a single save?" — RESOLVED. Propagation
  REQUIRED, default isolation (effectively PG `READ COMMITTED`). Rolls back
  the single save on throw, which is semantically inert because there is
  nothing else to roll back.
- "TaskScheduler behaviour on overlapping fires" — RESOLVED. Default pool
  size 1; overlapping fires are queued.

New open questions to add to the platform list:
- OQ: Is the DBLK `yyyyMMddHHmmSS` pattern actually failing in production,
  or has a pre-prod path been patched that never propagated to source?
  Requires prod log evidence from Logstash / ELK.
- OQ: Has the CapitalOne minute-resolution stuck-job been observed in
  production? Requires confirmation of current CapitalOne ACK volume vs
  1-per-minute threshold from Integration Team.
- OQ: Do `claim-response` and `outcome-no-chargeback` MessageType values
  ever appear in production CapitalOne data? Requires confirmation from
  Integration Team.
- OQ: Production replica count (XL Deploy placeholder). If > 1, HIGH
  concurrency-race risk is live.
- OQ: Production cron expression value.
- OQ: Which of `wdp-evidence-ack-scheduler-secrets` or `wdp-common-secrets`
  supplies `cron_scheduler`.
- OQ: Whether upstream (COMP-11 Walmart path) can write unmasked PAN into
  `chbk_outbox_row.record_detail.walmartSigCap.recordData`. DEC-004 /
  DEC-019 verdict at component egress depends on this. Requires upstream
  data analysis.
- OQ: DDL-level unique constraints on `wdp.file_job` and
  `wdp.chbk_outbox_row` (same open question as COMP-07 / COMP-08 / COMP-11).
- OQ: Cross-component ownership of `file_job.status = COMPLETED`
  transition. WDP-DB.md attributes to COMP-12 Scheduler2. Contract is
  undocumented in any of COMP-11, COMP-12, COMP-13 repos.

**WDP-DECISIONS.md · Candidate new ADRs**

- **Candidate new RISK (🔴 HIGH):** "Silent runtime bug via unreviewed
  `DateTimeFormatter` pattern in DBLK ACK generation — zero test coverage
  across the ACK component allowed a capital-`SS` typo to potentially reach
  production." Applies specifically to COMP-13 but surfaces a platform-wide
  concern: no test directory exists in this repo at all. Raise alongside
  any test-strategy ADR.
- **Candidate new RISK (🔴 HIGH):** "Minute-resolution S3 key timestamp
  collapse — CapitalOne ACK generation for two jobs in the same calendar
  minute produces the same S3 key, silently stranding the second job."
  Specific to COMP-13 CapitalOne path.
- **DEC-010 (Immutable Versioned ACK Snapshots) — candidate review.** The
  timestamp-in-S3-key pattern does not reliably produce unique versions at
  the precision the formatters actually use (seconds / fraction-buggy /
  minute). Needs restated, clarified, or voided. Raise as a dedicated ADR
  update.
- **DEC-023 scope extension:** extend the covered-component list to
  include COMP-13. Pattern matches COMP-07, COMP-08, COMP-09 — no code-level
  guard, replica=1 is operational policy only.
- **DEC-001 deviation map:** add COMP-13 — S3 write + DB write pair with
  no outbox, no saga, no compensating path. Severity 🔴 HIGH due to the
  stuck-job scenario.
- **DEC-020 deviation map:** add COMP-13 — three distinct at-least-once
  gaps (stuck-job, cross-replica race, CapitalOne minute collapse).
  Severity 🔴 HIGH.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- No new rows required from the RISK-013 / RISK-014 platform-wide coverage
  perspective.
- Minor clarification on RISK-013 (replica constraint has no automated
  enforcement): extend covered-components list to include COMP-13
  explicitly — pattern matches COMP-07, COMP-08, COMP-09.
- Candidate new RISK row pending architect decision on the CapitalOne
  minute-resolution failure mode: "Throughput-dependent silent ACK stuck-job
  on CapitalOne path — activates when > 1 CapitalOne ACK per calendar
  minute." HIGH.
- Candidate new RISK row pending architect decision on the DBLK
  `SS` pattern: "Unreviewed `DateTimeFormatter` pattern in DBLK ACK
  generation throws `UnsupportedTemporalTypeException` at runtime — no test
  coverage catches it." HIGH.

**WDP-INTEGRATIONS.md**
- No change. COMP-13 has no external integration contract (S3 is internal
  WDP infrastructure; ControlM / Sterling Mailbox / DM Mainframe handoff is
  out of this component's scope and already documented under COMP-10
  placeholder).

#### Deviation flags for COMP-13

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⛔ DEVIATES — not using outbox | 🔴 HIGH |
| DEC-003 Kafka Partition Key = merchantId | ✅ N/A | — |
| DEC-004 PAN Encryption Before Persistence | ⚠️ UNVERIFIABLE (Walmart pass-through) | 🟡 MEDIUM |
| DEC-005 Manual Kafka Offset Commit | ✅ N/A | — |
| DEC-014 Resilience4j Circuit Breaker | ⛔ ABSENT (void platform-wide) | 🟡 MEDIUM |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES at component scope | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-022 `removeItemFromQueueDisabled` Safety Switch | ✅ N/A | — |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY | 🟡 MEDIUM |

**DEC-001 detail:** S3 `putObject` and DB `updateAckStatus` are two
independent operations separated by a `@Transactional` boundary. No outbox
table, no mark-before-send, no saga, no compensating path. `headObject` gate
partially prevents duplicates but actively blocks recovery in the stuck-job
scenario. No source-level comment acknowledges this as deliberate.

**DEC-004 UNVERIFIABLE detail:** Walmart `recordData` is an opaque
pass-through from `chbk_outbox_row.record_detail`. Component does not
inspect, mask, or encrypt. Three non-Walmart formats carry no card-number
fields. PAN presence at ACK egress is an upstream data question and cannot
be resolved from this repo alone.

**DEC-020 PARTIAL detail — escalated to HIGH for COMP-13.** Three
independent gaps combine:
(1) stuck-job when `putObject` succeeds and DB write fails — no automatic
    recovery, `headObject=200` on next cron blocks retry;
(2) cross-replica race — if replicas > 1, two pods can both pass the
    `headObject` 404 window before either completes `putObject`;
(3) CapitalOne minute-resolution collapse — two CapitalOne ACKs in the same
    calendar minute produce the same S3 key; second is stuck on first
    encounter, not just after a failure.

**DEC-023 OPERATIONAL ONLY detail:** Same pattern as COMP-07, COMP-08,
COMP-09. No `@SchedulerLock`, no ShedLock dependency, no advisory lock, no
synchronized guard, no `@Version`. Replicas value is an XL Deploy
placeholder with no static `replicas: 1` assertion in the manifest.
Replica=1 is policy, not code.

#### Remaining gaps

- **OQ-Prod-Replicas:** Environment config — XL Deploy / deployit value for
  `{{ replicas-wdp-evidence-ack-scheduler }}` in each environment. Not in
  repo.
- **OQ-Prod-Cron:** Environment config — actual cron expression bound to
  `${cron_scheduler}` in production and cert. Not in repo.
- **OQ-Secret-Source:** Environment config — which of
  `wdp-evidence-ack-scheduler-secrets` or `wdp-common-secrets` supplies
  `cron_scheduler`. Not in repo.
- **OQ-DBLK-SS-Pattern:** Follow-up Claude Code or team confirmation —
  *"Does production Logstash / ELK show any
  `UnsupportedTemporalTypeException` or formatter-related failure coming
  from `wdp-evidence-ack-scheduler` BulkResponse path over the last 30
  days? If zero, is there a runtime-reachable branch that bypasses the
  `yyyyMMddHHmmSS` pattern? Cite file:line."*
- **OQ-CapitalOne-MinuteRace:** Team confirmation — is CapitalOne ACK
  volume ever > 1 per calendar minute in production? If yes, the
  minute-resolution stuck-job is active.
- **OQ-CapitalOne-MessageTypes:** Integration Team — do
  `claim-response` or `outcome-no-chargeback` MessageType values ever appear
  in production CapitalOne RTSI data? If yes, the catch-all REJECTED branch
  is a confirmed bug, not theoretical.
- **OQ-Walmart-RecordData-PAN:** Upstream data analysis — can
  `chbk_outbox_row.record_detail.walmartSigCap.recordData` contain
  unmasked PAN in production? Requires COMP-11 source + upstream data
  inspection. Not resolvable from this repo.
- **OQ-DDL-Constraints:** Schema-owner / DBA confirmation — unique
  constraints and indexes on `wdp.file_job` and `wdp.chbk_outbox_row`. Not
  in any component repo. Same open question as COMP-07 / COMP-08 / COMP-11.
- **OQ-FileJob-COMPLETED-Owner:** Architect decision — cross-component
  contract for `file_job.status = COMPLETED`. WDP-DB.md attributes to
  COMP-12 Scheduler2. Follow-up Claude Code question on COMP-12 source:
  *"Does `InboundDisputeEventScheduler` (COMP-12) update
  `wdp.file_job.status` to COMPLETED after successfully publishing all
  `chbk_outbox_row` rows for a given `file_job_id`, and does that
  transition trigger under both COMP-11 source profiles (DCPO/DNWK/DWSG/DBLK)
  uniformly? Cite file:line. If not, which component owns this transition
  and does every ACK-required source reach it?"*
- **OQ-DEC-010-Review:** Architect decision — is DEC-010 Immutable
  Versioned ACK Snapshots still in force given per-source timestamp
  precision (seconds / fraction-buggy / minute)?
- **OQ-Test-Strategy:** Architect decision — zero tests of any kind
  in `wdp-evidence-ack-scheduler`. Is this acceptable for a
  production component?

#### Doc status after this change
- `WDP-COMP-13-FILE-ACK-PROCESSOR.md` → `v2.0 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending. Supersedes v1.0 DRAFT.
---

### 2026-04-18 — COMP-12 InboundDisputeEventScheduler · v1.0 DRAFT → v1.1 DRAFT

**Source:** `wdp-chargeback-evidence-event-scheduler` — source-verified by
Claude Code 2026-04-18. Architect confirmation pending.

**Nature of change:** Correction pass — audit re-characterised the
transaction semantics of the Kafka relay (at-least-once, not at-most-once),
corrected one channel-map value, and confirmed several previously-open
items (K8s probes absent, bare `RestTemplate` everywhere, asymmetric
Kafka-metadata write-back across the five topics).

#### Platform-level impacts

**WDP-DB.md**
- `wdp.chbk_outbox_row` row: no change to COMP-12 ownership scope. Add
  note: `idempotencyId` is typed as `UUID` on this table's entity.
- `wdp.outgoing_event_outbox` row: add note — `idempotencyId` is typed
  as `String` on this table's entity. **Cross-table type inconsistency
  with `chbk_outbox_row` (UUID) — flag for downstream consumers that do
  shared dedup logic.**
- `wdp.chbk_outbox_row_archive` row: add note — archive INSERT SQL
  references column `c_ntwrk_phase_id` that the `ChbkOutboxArchiveEntity`
  does not map. Either dead column reference or incomplete entity.
  **DDL verification required** (schema lives outside this repo).
- Shared Table Risk Register row for `wdp.chbk_outbox_row`: no change —
  writers and mitigations remain as documented.

**WDP-KAFKA.md**
- Section 3 (Topic Registry) row for `case-action-events` /
  `core-request-events` / `external-request-events`: correct the
  `channelTypeTopicMap` non-prod default mapping from `SEN_EVENTS` to
  **`BEN_EVENTS`**. Prior v1.0 DRAFT listed `SEN_EVENTS` — this was a
  transcription error. Source: `application-local.yml:14`.
- Section 3 rows for the five topics COMP-12 publishes to: add a note
  on Kafka-metadata write-back asymmetry — `new-case-events` and
  `case-evidence-events` have `kafka_offset` / `kafka_partition` /
  `kafka_topic` persisted back to the outbox row; the other three paths
  (Scheduler3 and Scheduler4) pass a NULL entity and **do not persist
  Kafka metadata** to `outgoing_event_outbox` or
  `bre_orchestration_outbox`.
- Section 4 (Consumer Groups): no change — COMP-12 has no consumer.
- Section 5 (Outbox tables that feed Kafka): existing rows for
  `chbk_outbox_row`, `outgoing_event_outbox`, `bre_orchestration_outbox`
  remain accurate.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add / correct:
    - COMP-12 uses mark-and-send **within a single `@Transactional`** —
      broker ack precedes TX commit. This is **at-least-once with
      duplicate-possible crash window**, NOT at-most-once with
      silent-loss window as v1.0 DRAFT stated. Consumer-side
      `idempotency-key` dedup is the intended mitigation.
    - COMP-12 has **no Kubernetes probes** (liveness, readiness,
      startup all absent in `resources.yml`). Previously listed as
      an open question — now concrete.
    - COMP-12 uses a **bare `RestTemplate`** in `config/CommonConfig.java`
      for the Scheduler5 email POST — no connect timeout, no read
      timeout, no connection pool. Stalled peer blocks a scheduler
      thread indefinitely.
    - COMP-12 has a **shared `ThreadPoolTaskScheduler` with poolSize=5**,
      across five schedulers. Sibling-starvation risk under load or
      slow peer.
    - COMP-12 `channelTypeTopicMap` non-prod default keys are
      `EXPIRY_EVENTS`, `CORE_EVENTS`, `GP_EVENTS`, `BEN_EVENTS`
      (NOT `SEN_EVENTS`).
    - COMP-12 has **asymmetric Kafka-metadata write-back**: Chargeback
      and Evidence paths persist offset/partition/topic back to the
      outbox row within the same TX; Outgoing and BRE paths pass a
      NULL entity — operators cannot trace outgoing/BRE Kafka messages
      back to their outbox rows via offset.
    - COMP-12 retry policy is **fixed 1-hour backoff, 3 attempts** —
      `ApplicationConstants.NEXT_RETRY = 3600000L`. Not exponential.
    - COMP-12 has **no Spring Batch** (no `@EnableBatchProcessing`,
      no `BATCH_*` metadata tables, no `Job` / `Step` / `ItemReader` /
      `ItemWriter`), and no `fixedDelay` / `fixedRate` — all five
      triggers are cron expressions.
    - COMP-12 `ChbkOutboxEntity.idempotencyId` is `UUID` while
      `OutgoingEventOutboxEntity.idempotencyId` is `String` —
      cross-table type inconsistency.
    - COMP-12 has **no application-level observability** — OTel agent
      is injected by K8s annotation but no code-level MDC, no
      Micrometer registry, no custom meters.
    - COMP-12 S3 credentials use `StaticCredentialsProvider` from K8s
      secrets (migration away from `InstanceProfileCredentialsProvider`
      was completed — old IAM code retained commented).
    - COMP-12 has `Region.US_EAST_2` hardcoded in
      `S3PresignerConfiguration` — multi-region blocker.
    - COMP-12 has two `@Value` / yml path mismatches:
      (1) `S3PresignerConfiguration` reads `app.aws.accesskey` /
          `app.aws.secretkey`; yml defines `app.s3.aws.*`.
      (2) `EvidenceErrorEmailServiceImpl` reads
          `app.presignedUrlExpiryTime`; yml defines
          `app.s3.presignedUrlExpiryTime`.
      Both would fail Spring startup unless env-overrides resolve them.
- Resolved open questions (remove from HANDOVER):
    - COMP-12 transaction semantics (mark-before-send vs in-TX-with-send)
      — now documented; in-TX-with-send confirmed.
    - COMP-12 K8s probes presence — now documented; confirmed absent.
    - COMP-12 `RestTemplate` timeout config — now documented;
      confirmed bare.
    - COMP-12 retry backoff shape — now documented; fixed 1h confirmed.
    - COMP-12 Spring Batch presence — now documented; confirmed absent.
    - COMP-12 thread model — now documented; `ThreadPoolTaskScheduler`
      poolSize=5 confirmed.
- New open questions (add to HANDOVER):
    - Production replica count for COMP-12 — XLD placeholder
      `{{ replicas-wdp-chargeback-evidence-event-scheduler }}` not
      visible in source. **Any value > 1 is an unmitigated 🔴 HIGH
      concurrency race** — no ShedLock, no `SELECT FOR UPDATE`, no
      advisory lock, no synchronized guard.
    - Production cron values for all five schedulers — all externalised
      to K8s secrets (`chargeback_evidence_scheduler_cron`,
      `file_completion_scheduler_cron`, `outgoing_scheduler_cron`,
      `bre_outbox_event_scheduler_cron`, `evidence_email_scheduler_cron`).
    - Production `channelTypeTopicMap` complete mapping — K8s secret
      `${channel_topic_map}`; non-prod shows four channels.
    - `PUBLISHED → SUCCESS` transition owner for `chbk_outbox_row`
      CHARGEBACK_PROCESS rows — likely COMP-14 CaseCreationConsumer,
      but not verified from either repo. The Scheduler1 Phase 2
      unblock query depends on this transition.
    - Initial `PENDING` writer(s) for `wdp.outgoing_event_outbox` and
      `wdp.bre_orchestration_outbox` — COMP-17 / COMP-18 / COMP-43
      candidates identified, not verified against source of those
      components.
    - Archive-table column `c_ntwrk_phase_id` — present in
      `ChbkOutboxArchiveRepository` native INSERT, absent from the
      entity. DBA / DDL verification required.
    - Two `@Value` / yml path mismatches — are env-var overrides or
      profile-specific properties resolving them in deployed
      environments? Dev team confirmation required.
    - Actuator endpoint exposure on port 8082 —
      `management.endpoints.web.exposure.include` value not captured
      this pass; needed for probe wiring plan.

**WDP-DECISIONS.md · Candidate new ADRs**
- **MEDIUM candidate (cross-component):** "Outbox relays that mark-and-
  send within a single `@Transactional` accept a duplicate-possible
  crash window between broker-ack and TX-commit; consumer-side
  `idempotency-key` dedup is the contracted mitigation." Explicitly
  recognising this as an intentional design pattern (vs DEC-001's
  strict relay model) across COMP-12 and any future outbox relays
  would be useful. Recommend raising at next DECISIONS rebuild.
- **MEDIUM candidate:** "No Kubernetes probes on continuously-running
  Deployment-hosted schedulers" — confirmed on COMP-12. Pattern likely
  matches other scheduler components (COMP-13 at minimum). Recommend
  cross-component re-audit before formalising an ADR.
- **MEDIUM candidate:** "`idempotencyId` column typing is inconsistent
  across outbox tables (`UUID` on `chbk_outbox_row`, `String` on
  `outgoing_event_outbox`)." Either normalise to one type or document
  the divergence as intentional per table domain.
- DEC-001 deviation map: add COMP-12 as a **compliant variant** (not
  a deviation) — mark-and-send-within-TX is a valid outbox-relay
  implementation. Remove any prior "PARTIAL deviation / silent-loss"
  framing if present.
- DEC-003 deviation map: add COMP-12 entry — five topics published,
  all with non-`merchantId` keys (networkCaseId+cardNetwork+platform,
  caseNumber, caseNumber/networkCaseId).
- DEC-020 deviation map: add COMP-12 partial — no replica lock;
  terminal ERROR rows have no DLQ; `PUBLISHED` rows with failed
  metadata UPDATE are stranded (no scheduler re-queries `PUBLISHED`).

**WDP-ARCHITECTURE.md**
- No change to platform topology. COMP-12 remains the sole Kafka
  producer for the inbound relay chain and a retry relay for the
  outgoing / BRE outbox tables.
- Section 6 (Kafka Event Bus) — minor: when describing the
  `channelTypeTopicMap`, use `BEN_EVENTS` not `SEN_EVENTS` in any
  example mapping.

**WDP-NFRS.md · Section 6 Risk Register**
- **RISK-015** (`bre_orchestration_outbox` PUBLISHED orphan rows —
  no auto-redrive) — already present; covers COMP-12 / COMP-18. Extend
  the note to mention the asymmetric Kafka-metadata write-back (NULL
  entity for Scheduler4 means PUBLISHED-status orphans carry no offset
  breadcrumb — makes orphan identification harder).
- **New RISK candidate:** "No Kubernetes health probes on COMP-12 —
  continuously-running Deployment with five cron schedulers; a
  deadlocked JVM or broken DB connection is invisible to Kubernetes."
  🔴 HIGH. Likely applies to other scheduler components — cross-audit
  before formalising.
- **New RISK candidate:** "Bare `RestTemplate` (no timeouts, no pool)
  on COMP-12's Scheduler5 email POST — sibling-starvation risk on
  the shared 5-thread pool if the email relay is slow." 🔴 HIGH in
  interaction with the shared pool.
- **New RISK candidate:** "COMP-12 has no replica guard (no ShedLock,
  no `SELECT FOR UPDATE`, no advisory lock). Replica=1 is operational
  policy, not code-enforced." Extend RISK-013 to explicitly cover
  COMP-12 (current coverage: COMP-07, COMP-08) — or create a new
  parallel RISK if DEC-023 is considered scoped to Spring-Batch
  polling components only.
- **New RISK candidate (MEDIUM):** "COMP-12 asymmetric Kafka-metadata
  write-back — Scheduler3 and Scheduler4 paths pass NULL entity, so
  `outgoing_event_outbox` and `bre_orchestration_outbox` rows never
  record Kafka offset/partition/topic. Incident correlation degraded."
- **New RISK candidate (MEDIUM):** "COMP-12 `idempotencyId` type
  inconsistency across outbox tables — UUID vs String. Shared dedup
  logic on consumer side must handle both."
- **New RISK candidate (MEDIUM):** "COMP-12 archive-table SQL
  references unmapped column `c_ntwrk_phase_id` — either DDL column
  exists and entity is incomplete, or column is missing and archive
  fails at runtime. DBA verification required."

**WDP-INTEGRATIONS.md**
- No change. COMP-12 has no external integrations — all Kafka
  targets are WDP-owned MSK, the email relay is an internal WDP
  endpoint, and S3 is internal. No Visa / MasterCard / CORE / NAP
  integration contract changes.

#### Deviation flags for COMP-12

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ COMPLIES (compliant variant — mark-and-send within single `@Transactional`) | — |
| DEC-003 Kafka Partition Key = merchantId | ⛔ DEVIATES (all five topics) | 🟡 MEDIUM |
| DEC-004 PAN Encryption Before Persistence | ✅ NOT APPLICABLE (no PAN) | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE (producer only, no consumer) | — |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ NOT APPLICABLE (no PAN) | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY (no code-level guard) | 🔴 HIGH |

**DEC-001 compliant-variant detail:** Upstream writers (COMP-07, 08, 09,
11 for `chbk_outbox_row`; COMP-17, 18, 43 for `outgoing_event_outbox`;
COMP-18 for `bre_orchestration_outbox`) perform the strict outbox write
atomically with their business write. COMP-12 is the relay layer. The
mark-and-send within a single `@Transactional` method (broker ack before
TX commit) is a valid outbox-relay implementation — at-least-once with a
duplicate-possible crash window between broker-ack and TX-commit, handled
by consumer-side `idempotency-key` deduplication. v1.0 DRAFT's "PARTIAL /
silent-loss" characterisation was incorrect and has been reversed.

**DEC-003 DEVIATES detail:** All five topics use non-`merchantId` keys.
`new-case-events` uses `networkCaseId + cardNetwork + platform`.
`case-evidence-events` uses `caseNumber` with fallback to `networkCaseId`.
`case-action-events` / `core-request-events` / `external-request-events`
(dynamic via `channelTypeTopicMap`) use `caseNumber`. `business-rules`
uses `caseNumber`. `outgoing-events` uses `caseNumber`. All consumers
downstream receive case-scoped ordering, not merchant-scoped.

**DEC-014 DEVIATES detail:** No Resilience4j on classpath. Bare
`RestTemplate` with no connect/read timeouts, no pool. No circuit
breaker anywhere. Only resilience is Kafka-client `retries=3` on send.

**DEC-020 PARTIAL detail:** Four concurrent gaps: (a) no replica lock,
so replicas > 1 produce cross-replica duplicates (producer idempotence
dedupes within a session only); (b) Kafka metadata UPDATE failure at
Scheduler1 Step 4 leaves the row at PUBLISHED with null metadata and
no scheduler re-selects PUBLISHED rows; (c) terminal ERROR rows on
any of the four outbox tables have no DLQ or re-drive path; (d)
Scheduler3 and Scheduler4 do not persist Kafka metadata (NULL entity
passed), degrading incident correlation on the outgoing/BRE paths.

**DEC-023 OPERATIONAL ONLY detail:** No `@SchedulerLock`, no ShedLock
dependency, no advisory lock, no synchronized guard, no `SELECT FOR
UPDATE`. The shared `ThreadPoolTaskScheduler` with poolSize=5 does not
provide cross-pod mutual exclusion. Horizontal scaling is unsafe as
built — replica=1 is operational policy, not enforced by code. Suggest
extending DEC-023's covered-components list to include COMP-12, or
create a parallel ADR if DEC-023 is considered scoped to Spring Batch
polling components only.

#### Remaining gaps

- **OQ: Production replica count** — XLD placeholder not auditable
  from repo. Team / XLD config confirmation. **Any value > 1 is 🔴
  HIGH and active.**
- **OQ: Production cron values** (five) — K8s secrets, not auditable
  from repo. Team confirmation.
- **OQ: Production `channelTypeTopicMap`** — K8s secret, non-prod
  shows four channels. Team confirmation.
- **OQ: `PUBLISHED → SUCCESS` transition owner** on `chbk_outbox_row`
  CHARGEBACK_PROCESS rows. Follow-up Claude Code question against
  the COMP-14 repo: *"Search `wdp-case-creation-consumer` (or the
  COMP-14 repo) for any write of `chbk_outbox_row.status='SUCCESS'`.
  Report the exact condition, transaction boundary, and timing
  relative to the downstream case creation."*
- **OQ: Initial PENDING writer** for `outgoing_event_outbox` and
  `bre_orchestration_outbox`. Follow-up Claude Code question for
  each candidate: *"In `<repo>`, search for any INSERT / `save()`
  that creates a fresh `OutgoingEventOutboxEntity` or
  `BreOrchestrationOutboxEntity` with `status=PENDING`. Report the
  exact condition, transaction boundary, and discriminator value
  (channel_type / component)."* Candidates: COMP-17, COMP-18,
  COMP-43 for outgoing; COMP-18 for BRE.
- **OQ: Archive column `c_ntwrk_phase_id`** — DBA / schema-owner
  confirmation whether the column exists in DDL. If yes, add mapping
  to `ChbkOutboxArchiveEntity`; if no, remove from archive SQL.
- **OQ: `@Value` / yml path mismatches in production** — do env-var
  overrides or profile-specific properties resolve
  `app.aws.accesskey` / `app.aws.secretkey` and
  `app.presignedUrlExpiryTime` in deployed environments? Dev team
  confirmation.
- **OQ: Actuator endpoints exposed on port 8082** — not captured this
  pass. Follow-up Claude Code question: *"Read
  `application*.yml` and any `management` config class; report
  `management.endpoints.web.exposure.include` value and any
  endpoint-specific enable / disable settings."*

#### Doc status after this change
- `WDP-COMP-12-INBOUND-EVENT-SCHEDULER.md` → `v1.1 DRAFT` —
  source-verified 2026-04-18 · architect confirmation pending
---

### 2026-04-18 — COMP-11 FileProcessor · v1.0 DRAFT → v1.1 DRAFT

**Source:** `wdp-file-processor` — source-verified by Claude Code 2026-04-18.
Architect confirmation pending.

**Nature of change:** Correction pass — audit corrected several facts in the
prior DRAFT (created_by attribution, MC_REVREJ routing status, CHARGEBACK_PROCESS
status-flow model, DCPO empty-catch orphan risk, DNWK silent-row-loss pattern,
DEC-004 non-numeric acctNum branch) and closed several previously-open items.

#### Platform-level impacts

**WDP-DB.md**
- `wdp.file_job` row: correct `created_by` from `"WPFLEPR"` to `"FILE_PROCESSOR"`.
  Correct `updated_by` likewise. Add note: this component owns only the columns
  `s3_key, file_name, s3_bucket, file_size_bytes, source, status, total_rows,
  total_evidences, ack_required, ack_status, error_code, error_message,
  created_by, updated_by`. It does not write `completed_at, successful_rows,
  failed_rows, error_rows, attached_evidences, failed_evidences` — those
  belong to downstream components sharing the table.
- `wdp.file_evidence` row: correct `created_by` from `"WPFLEPR"` to
  `"FILE_PROCESSOR"`.
- `wdp.chbk_outbox_row` row: correct `created_by` from `"WPFLEPR"` to
  `"FILE_PROCESSOR"` in the COMP-11 attribution. No change to writer list.
  Clarify: entity in this repo does not declare `kafka_topic / kafka_partition /
  kafka_offset / idempotency_id / retry_count / next_retry_at / published_at` —
  consistent with COMP-12 being sole setter. Initial CHARGEBACK_PROCESS status
  is PENDING (PrePersist); LOADING is transient during evidence phase; BLOCKED
  applies only to DCPO evidence.
- Shared-write risk table row for `wdp.chbk_outbox_row`: no change to severity.
  Note the new per-record-loss gap in DNWK handlers under "Mitigations".
- Shared-write risk table row for `wdp.file_job`: no change to severity.
  Note COMP-11 never writes COMPLETED → cross-component break with COMP-13
  already captured; unchanged.

**WDP-KAFKA.md**
- No change. COMP-11 remains in Section 4 Kafka-free components list with the
  note "SQS-triggered — uses Spring Cloud AWS SQS listener, not Kafka. Zero
  Kafka infrastructure. Confirmed from source." Audit re-confirms zero
  `spring-kafka`, `kafka-clients`, `spring-cloud-stream` in pom.xml and zero
  `@KafkaListener` / `KafkaTemplate` in source.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: COMP-11 `created_by` constant is `"FILE_PROCESSOR"` on all three
  owned/shared tables (not `"WPFLEPR"` as v1.0 DRAFT stated).
- Add: DNWK silent-file-loss paths are **two** — `DISCHYB` (no bean
  registered for `DISCHYB_NETWORK` qualifier) and `AMEXOPTB` (no `FileAcroEnum`
  prefix entry). `MC_REVREJ` **is** registered (`McRevRejNetworkFileServiceImpl`)
  — v1.0 DRAFT incorrectly listed it as silent-loss.
- Add: COMP-11 initial CHARGEBACK_PROCESS status is PENDING via PrePersist.
  LOADING appears only transiently during the evidence phase (parent flipped
  by `insertEvidenceOutBox` @Transactional). DCPO evidence rows are explicitly
  BLOCKED; DCPO parent stays PENDING. DNWK writes CHARGEBACK_PROCESS at PENDING
  directly with no evidence phase.
- Add: Only two `@Transactional` sites exist in the whole repo —
  `insertEvidenceOutBox` and `updateStatusToPendingByFileJobIdAndStatusLoading`.
  Everything else is auto-commit Hibernate.
- Add: DCPO CHARGEBACK_PROCESS save is auto-commit AND the per-image evidence
  loop is wrapped in an empty catch block — orphan-parent risk on any image
  failure, no observability signal.
- Add: DNWK per-record failure path is a one-retry-then-silent-row-loss
  pattern — on second save exception, `rowCount` has already advanced and
  resume logic skips the row.
- Add: `RestTemplate` is constructed with no customization — no connect
  timeout, no read timeout, no pooling. `IdpRestInvoker` has no `@Retryable`.
- Add: DEC-004 has a non-numeric-acctNum edge case — `NetworkFileSupport`
  encrypts only when `acctNum` matches `\d+`; otherwise passes raw into
  payload.account_number.
- Add: Container binds port 8082 (Tomcat starts because `spring-boot-starter-web`
  is on classpath even though no REST routes are registered). AWS region
  `us-east-2` is hardcoded in `S3ClientConfiguration`.
- Add: `S3ServiceImpl` silently swallows get/move/put failures and returns null
  — not a throw path.
- Resolved open questions: none from the platform-level list — the resolved
  items were COMP-11-internal (DCPO mapping confirmed, DNWK handler registry
  confirmed, @Transactional site count confirmed).
- New open questions to add to the platform list:
    - OQ: DB-level UNIQUE constraints on `wdp.file_job (file_name, s3_key)`
      and `wdp.chbk_outbox_row (file_job_id, event_type, row_number)` — not
      in JPA entities, no DDL in this repo. Schema owned outside COMP-11.
    - OQ: DCPO `file_job.source` value and CMRTR vs BJWC sub-type detection
      (follow-up Claude Code question).
    - OQ: DCPO evidence BLOCKED → PENDING promotion — which component owns
      the gate? Not COMP-11, not COMP-12 (which publishes PENDING rows).
    - OQ: DEC-004 non-numeric `acctNum` branch — can production DNWK ever
      produce a non-numeric PAN-bearing value?
- Keep open: SQS visibility timeout numeric value (OQ-1); SQS DLQ config
  (OQ-2); production replica count (OQ-6); `file_job.status = COMPLETED`
  transition ownership (WDP-DB.md attributes it to COMP-12; COMP-11 source
  neither writes nor observes it — confirm via COMP-12 source).

**WDP-DECISIONS.md · Candidate new ADRs**
- No new ADRs. DEC-004 deviation map should add a note that COMP-11 DNWK
  handler encrypts only when `acctNum` matches `\d+` — the non-numeric branch
  is a latent risk but not a confirmed production-active violation. Treat as
  a clarification to the existing DEC-004 entry, not a new ADR.
- DEC-001 deviation map already records COMP-11 as a deviation; audit
  reinforces the detail (three separate transaction boundaries, plus DCPO
  empty-catch and DNWK silent-row-loss) — consider enriching the existing
  entry at next DEC reconciliation.
- DEC-020 deviation map already records COMP-11 idempotency gap; audit
  reinforces the cross-replica race and the DCPO/DNWK row-loss paths —
  consider enriching at next DEC reconciliation.

**WDP-ARCHITECTURE.md**
- No change to platform topology. COMP-11 remains the single inbound
  file-ingestion component between the Sterling → ControlM → S3 pipeline
  and the `chbk_outbox_row` outbox consumed by COMP-12.

**WDP-NFRS.md · Section 6 Risk Register**
- Add (or confirm if already present) RISK rows:
    - COMP-11 DCPO empty-catch evidence loop — orphaned CHARGEBACK_PROCESS
      rows published without evidence. 🟡 MEDIUM.
    - COMP-11 DNWK per-record one-retry-then-silent-loss — row counter
      advances even if both save attempts fail. 🟡 MEDIUM.
    - COMP-11 DEC-004 non-numeric acctNum branch — raw PAN-shaped value
      written to outbox payload. 🟡 MEDIUM (contingent on OQ confirming
      whether production can ever produce non-numeric acctNum).
    - COMP-11 `S3ServiceImpl` silently returns null on all S3 failures —
      transient S3 errors produce no alert. 🟡 MEDIUM.
    - COMP-11 no K8s liveness / readiness / startup probes; Actuator
      absent from classpath. 🔴 HIGH.
    - COMP-11 no HPA — throughput scales only by manual replica increase.
      🔴 HIGH.
    - COMP-11 DISCHYB + AMEXOPTB silent-file-loss paths. 🔴 HIGH.
      Correction: v1.0 DRAFT additionally listed MC_REVREJ — that entry
      should be removed; MC_REVREJ handler is registered.

**WDP-INTEGRATIONS.md**
- No change. COMP-11 inbound integration contract (Sterling → ControlM →
  S3 → SQS `wdp-file-arrivals`) unchanged. DNWK sub-source list unchanged
  at the integration level (registration status is a component-level
  concern).

#### Deviation flags for COMP-11

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🟡 MEDIUM |

**DEC-001 PARTIAL detail:** `file_job`, `chbk_outbox_row`, and `file_evidence`
are written in separate transactions. Only `insertEvidenceOutBox`
(parent-flip-to-LOADING + EVIDENCE_ATTACH insert + file_evidence insert) and
the bulk LOADING→PENDING UPDATE carry `@Transactional`. DCPO CHARGEBACK_PROCESS
is auto-commit with an empty-catch evidence loop — orphan-parent risk on any
image failure. DNWK per-record saves are auto-commit with a
one-retry-then-silent-row-loss pattern.

**DEC-004 PARTIAL detail:** DNWK encryption runs only when `acctNum` matches
`\d+`; non-numeric values pass raw into `payload.account_number`. Non-DNWK
sources carry no PAN fields. Open question OQ-5: can production ever produce
a non-numeric PAN-bearing `acctNum`?

**DEC-014 DEVIATES detail:** Platform-wide void already recorded. COMP-11
confirms: no Resilience4j on classpath, no circuit breakers, no `RestTemplate`
connect/read timeouts, no `IdpRestInvoker` retry, no connection pooling. Only
resilience is `@Retryable` 3×/2s on PAN encryption call.

**DEC-019 PARTIAL detail:** Contingent on the same DEC-004 non-numeric branch.
Raw PAN-shaped value could reach `chbk_outbox_row.payload` persistent store
via this branch.

**DEC-020 PARTIAL detail:** File-level idempotency is application-side only;
no DB UNIQUE visible on `(file_name, s3_key)`. Cross-replica race window.
`file_job.status ≠ PENDING` short-circuit means redelivery after a mid-archive
crash is treated as duplicate (SQS message deleted, no reprocessing). DCPO
empty-catch orphan and DNWK silent-row-loss paths are additional at-least-once
violations.

#### Remaining gaps

- **OQ-1 SQS visibility timeout numeric value** — environment config or team
  confirmation. Not in repo.
- **OQ-2 SQS DLQ (RedrivePolicy / maxReceiveCount)** — team confirmation via
  AWS console / Terraform / CDK. Not in repo.
- **OQ-3 DB UNIQUE constraints** on `file_job (file_name, s3_key)` and
  `chbk_outbox_row (file_job_id, event_type, row_number)` — team confirmation
  or schema-owner repo inspection. Schema owned outside COMP-11.
- **OQ-4 `file_job.status = COMPLETED` transition** — architect decision.
  WDP-DB.md attributes it to COMP-12; COMP-11 source neither writes nor
  observes it. Follow-up Claude Code question on COMP-12 source: *"Does
  InboundDisputeEventScheduler (COMP-12) update `wdp.file_job.status` to
  COMPLETED after successfully publishing all chbk_outbox_row rows for a
  given `file_job_id`? Cite file:line. If not, which component does?"*
- **OQ-5 DEC-004 non-numeric acctNum branch** — team confirmation from
  Integration team: can production DNWK flat files ever produce a
  non-numeric `acctNum` that should still be treated as PAN-sensitive?
- **OQ-6 production replica count** — environment config. Not in repo.
- **OQ-7 DCPO `file_job.source` value and sub-type detection** —
  follow-up Claude Code question: *"In `CaponeDisputesIncomingServiceImpl`,
  what exact string is written to `file_job.source`? Is CMRTR vs BJWC sub-type
  determined (filename pattern, XML content, constant)? What is `ack_required`
  set to on the final `updateFileJobTable` call, and under which condition?"*
- **OQ-8 DCPO evidence BLOCKED → PENDING promotion** — architect decision.
  Not in this repo, not in COMP-12 (which publishes PENDING). Follow-up
  needed with the evidence-consumer team.
- **OQ-9 DWSG / DBLK / DISR / MFAD column-level outbox mapping** — follow-up
  Claude Code pass if needed. Audit did not re-read `EventChargebackSupport`,
  `CoreBulkSupport`, `DiscoverMapSupport` line-by-line.
- **Spring Boot 4.0.3 `EntityManagerFactoryBuilder` import path** — developer
  confirmation, not an architecture matter. Flagged in audit, noted here for
  completeness.

#### Doc status after this change
- `WDP-COMP-11-FILE-PROCESSOR.md` → `v1.1 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending
---

### 2026-04-18 — COMP-09 CaseFillingBatch · v1.0 DRAFT → v2.0 DRAFT

**Source:** `wdp-mcm-receiver-case-filing-queue-batch` — source-verified by
Claude Code 2026-04-18. Architect confirmation still pending.

**Nature of change:** Correction pass against source. No functional change in
production; 14 corrections and clarifications to the documented behaviour of
an existing component. Four findings rise to HIGH risk severity.

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- Add `created_by` / `updated_by` constant for COMP-09: `"WMPAPB"`. This is
  distinct from COMP-07 (`"WVDPB"`), COMP-08 (`"WMFDPB"`), and COMP-11
  (`"WPFLEPR"`). Four batch writers, four different user constants.
- Add `c_case_stage` concatenation convention: COMP-09 writes the base stage
  (`PAB`, `ARB`, `PRA`, `AII`, `AIM`) **except** when `issuerCaseStatus =
  withdraw`, in which case the column is written as `{stage}_withdraw`
  (e.g. `PAB_withdraw`, `PRA_withdraw`). Downstream consumers that key on
  `c_case_stage` must tolerate this concatenation.
- Add `network_notes` to COMP-09's written columns — conditional, only when
  `schemeRef.notes` is non-empty. No other batch writer populates this
  column.
- Clarify `c_ntwk_case_id` semantics for COMP-09: value is `standardClaimId`,
  which equals `claimDetail.standardClaims` when `claimType = "CaseFiling"`,
  else `claimId`. The DRAFT labelled this simply `networkCaseId`.
- Add to the Shared Table Risk Register notes: **COMP-09 update path always
  INSERTs a new row**. Existing rows are never UPDATEd by COMP-09. A claim
  that progresses through its phase lifecycle produces multiple rows, one
  per phase observation, distinguished by status (PENDING / SKIPPED). Table
  growth is proportional to phase count × rerun frequency.
- Add to risk notes: **COMP-09 writer swallows all save exceptions**. A DB
  save failure produces no error row, no retry, and no Spring Batch skip
  counter increment — the chunk reports success. Same pattern observed on
  COMP-07's `BatchItemWriter`. This is a cross-batch platform pattern.
- Add to risk notes: **COMP-09 skip paths write no row at all**. Claims that
  fail null-field validation, stage-determination, IDP token, or encryption
  leave zero database trace. `error_code` / `error_message` columns exist
  on the entity but are never populated by COMP-09.
- Confirm "no DB unique constraint" for COMP-09: no DDL, Flyway, Liquibase,
  or `schema.sql` in the COMP-09 repo; `ddl-auto=false`;
  `initialize-schema=never`; no entity-level `uniqueConstraints`.

**WDP-KAFKA.md**
- No change. COMP-09 has no active Kafka producer or consumer. Kafka
  artifacts in `pom.xml` (`spring-kafka`, `kafka-clients`,
  `aws-msk-iam-auth`) are declared but not wired. An accidental import of
  `org.apache.kafka.shaded.com.google.protobuf.ServiceException` is the
  only active Kafka-package reference.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Add / correct:
- COMP-09 writes `wdp.chbk_outbox_row` with writer user `"WMPAPB"`. Shared
  table; four writer components, four distinct user constants.
- COMP-09 update path always INSERTs a new outbox row; existing rows are
  never UPDATEd by this batch.
- COMP-09 skip paths write no database row — null-validation, unmatched
  stage, IDP failure, and Encryption failure all silently drop the claim.
- COMP-09 writer catches and swallows all save exceptions — DB save
  failures are invisible to Spring Batch.
- COMP-09 MCM acknowledgement is not implemented. `DataPowerRestInvoker.put()`
  is fully implemented with `@Retryable` but never called. `McmService`
  exposes 3 methods, no acknowledge method.
- COMP-09 has no liveness, readiness, or startup probes in its Kubernetes
  manifest.
- COMP-09 has no MDC-per-claim correlation scope. Per-call
  `v-correlation-id` headers are added by DataPowerRestInvoker and
  RestInvoker; **IdpRestInvoker does not propagate any correlation id**.
- COMP-09 `enrichmentFailure=true` on payload is set unconditionally on
  every row, for both new and update paths. It does not indicate a failure
  and downstream consumers should not infer meaning from it.
- COMP-09 Spring Batch metadata tables run on the `@Primary wdpDataSource`;
  no dedicated batch datasource; `spring.batch.job.enabled=false`;
  `isolation-level-for-create=REPEATABLE_READ`.
- COMP-09 `JobLauncher` uses Spring Boot auto-configured
  `TaskExecutorJobLauncher` + `SyncTaskExecutor` — synchronous, single
  thread. No `@SchedulerLock`, no ShedLock, no advisory lock, no
  synchronized block. Replica=1 is operational policy, not code-enforced.

Resolved open questions:
- COMP-09 writer failure semantics — now documented.
- COMP-09 update-path behaviour — now documented (always INSERT).
- COMP-09 skip-path row semantics — now documented (no row written).
- COMP-09 outbox column-by-column field mapping — now documented.
- COMP-09 concurrent execution guard — now documented (none; SyncTaskExecutor).
- COMP-09 DB unique constraints — now documented (none in this repo).
- COMP-09 RestTemplate pooling — now documented (bare, no pool).
- COMP-09 health probes — now documented (none).
- COMP-09 observability and correlation — now documented.
- COMP-09 feature flag defaults — now documented (all K8s secrets, no fallback).
- COMP-09 caseType meanings — now documented (1=PreArb, 2=Arb, 3/4 dead).

New open questions (add to HANDOVER):
- Production replica count for COMP-09 — XL Deploy placeholder
  `{{ replicas-wdp-mcm-receiver-case-filing-queue-batch }}`; any value > 1
  is a DEC-023 violation and creates duplicate-INSERT risk.
- Production cron value for COMP-09 — K8s secret `scheduler_cron`, not
  auditable from repo. Application fails to start if absent.
- COMP-09 Spring Batch `table_prefix` K8s secret value — schema name not
  derivable from source. DRAFT conjecture of `WDP.BATCH_` is unverified.
- Downstream MCM queue lifecycle for un-ACKed processed claims — requires
  MasterCard MCM team input. WDP never sends `PUT` acknowledgement; MCM's
  re-queue / TTL / back-pressure behaviour is unknown from our side.
- COMP-12 Scheduler1 filter behaviour on COMP-09 output rows — whether
  Scheduler1 applies any `event_type`, `c_case_ntwk`, or `c_acq_platform`
  filter that would exclude COMP-09's rows. Filter lives in COMP-12 repo.
- `c_case_stage` concatenation (`PAB_withdraw`, `ARB_withdraw`,
  `PRA_withdraw`) — do downstream consumers (COMP-14 CaseCreationConsumer)
  parse this concatenation, or do they match on the base stage only?
  Architect decision required.
- `enrichmentFailure=true` unconditional semantics — protocol-level
  permanent flag or a misconfiguration? No downstream consumer currently
  treats it as a signal, but the name implies it should.

**WDP-DECISIONS.md · Candidate new ADRs**

- **HIGH candidate (cross-component):** "Silent exception swallow in batch
  writers" — confirmed at COMP-07 and now COMP-09. Likely applies to
  COMP-08 by pattern. Recommend raising at next DECISIONS rebuild window.
  Proposed wording: "WDP polling batches catch and swallow all writer
  exceptions, causing DB save failures to appear as successful chunks with
  no audit trail." Severity: HIGH — produces invisible data loss.
- **HIGH candidate:** "MCM acknowledgement not implemented on COMP-09."
  Fully built PUT method and `@Retryable` exist; never called; no interface
  method declared. Open architectural question around MCM queue lifecycle.
  Recommend raising as formal ADR — either implement, formally remove, or
  document the accepted risk with MCM team confirmation on re-queue
  behaviour.
- **HIGH candidate:** "Skip paths in polling batches write no audit row."
  Confirmed at COMP-09 — null-validation, unmatched stage, IDP failure,
  encryption failure all result in no database row. Combined with the
  writer exception swallow, this means there is no operational view of
  which claims were dropped or why, beyond unstructured INFO logs. Likely
  applies to COMP-07 null-return paths too. Severity: HIGH — no
  reconciliation path, no DLQ.
- **MEDIUM candidate:** "Outbox rows are INSERT-only — never UPDATED by
  polling batches." Confirmed design pattern at COMP-09 update path (and
  consistent with COMP-07, COMP-08). Worth recording as an explicit
  architectural principle so future components do not inadvertently mutate
  outbox rows. Severity: MEDIUM — design-clarity, not a defect.
- **MEDIUM candidate (cross-component):** "No health probes on polling
  batches." Confirmed at COMP-07 and COMP-09. Batches have no liveness,
  readiness, or startup probe, so a deadlocked or slow-starting JVM is
  invisible to Kubernetes. Recommend raising at next DECISIONS rebuild.
- **MEDIUM candidate (cross-component):** "No code-level concurrency guard
  on polling batches — DEC-023 operational-only enforcement." Reconfirmed
  at COMP-09. Replica = 1 is policy, not enforced by code anywhere.
- **LOW candidate:** "Inconsistent correlation-id propagation across REST
  invokers in polling batches." COMP-09 confirmed: DataPowerRestInvoker
  and RestInvoker both add `v-correlation-id`; IdpRestInvoker omits it.
  Partial trace continuity per claim.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- Existing RISK-013 (DEC-023 operational-only) — confirmed applies to
  COMP-09. Same clarification as COMP-07 — no code-level guard, only
  operational enforcement.
- **New RISK candidate:** "Batch writer exception swallow — invisible DB
  save failures across polling batches." COMP-07 and COMP-09 confirmed;
  likely COMP-08. Severity HIGH. Raise alongside the corresponding ADR.
- **New RISK candidate:** "Skip paths write no audit row in polling
  batches." Severity HIGH. Applies to all four null-return / silent-skip
  exits on COMP-09; likely same pattern on COMP-07 and COMP-08. Raise
  alongside the corresponding ADR.
- **New RISK candidate (MEDIUM):** "No health probes on polling batches —
  JVM deadlock invisible to Kubernetes." Confirmed COMP-07 and COMP-09.

**WDP-INTEGRATIONS.md**
- No change. COMP-09 integration to MasterCard MCM via DataPower was
  already documented under Section 3.2 / 3.3. No new integration contracts;
  corrections are internal to COMP-09.

#### Deviation flags for COMP-09

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ COMPLIES | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES (with logging finding) | 🟡 MEDIUM (HPAN in logs) |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-014 Resilience4j Circuit Breaker | ⛔ DEVIATES (void platform-wide) | 🟡 MEDIUM |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-022 `removeItemFromQueueDisabled` Safety Switch | ✅ NOT APPLICABLE | — |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY | 🟡 MEDIUM |

**DEC-004 logging finding detail:** HPAN is logged at INFO level post-encryption
in `DisputeServiceImpl`, and the full `CommonEvent` (including HPAN via
`originalTransIdentifier.accountNumber`) is logged via Lombok `@Data` toString
in `ProcessorUtil`. Logs ship to Logstash/ELK via `LogstashTcpSocketAppender`.
`CheckmarxUtil.sanitizeString()` (corrected spelling — DRAFT had `CheckmarkUtil`)
is defined but never called. No `@ToString.Exclude` on HPAN-bearing fields.
This constitutes effective persistence of encrypted cardholder data in logs.
PCI scoping review required.

**DEC-020 PARTIAL detail — escalated to HIGH vs COMP-07 MEDIUM.** Four
independent gaps combine:
(1) no DB unique constraint;
(2) **skip paths write no audit row** — null-validation, unmatched stage,
    IDP failure, and Encryption failure all silently drop the claim;
(3) **writer exception swallow** — DB save failures report success, leaving
    no trace;
(4) crash window between MCM poll and DB write. Unlike COMP-07 — which at
    least writes an ERROR row on missing key data and a SKIPPED row on
    duplicate detection — COMP-09 has **no mechanism whatsoever** for
    observing dropped claims beyond unstructured INFO logs.

**DEC-023 OPERATIONAL ONLY detail:** No `@SchedulerLock`, no ShedLock
dependency, no advisory lock, no synchronized guard. `SyncTaskExecutor`
JobLauncher by Spring Boot auto-config. Replica=1 is policy, not enforced
by code.

#### Remaining gaps

- **Needs follow-up Claude Code question (in a different repo):** "In the
  COMP-12 InboundDisputeEventScheduler Scheduler1 poll query, does any
  WHERE clause filter on `event_type`, `c_case_ntwk`, or `c_acq_platform`
  that would exclude rows written by COMP-09 (event_type=CHARGEBACK_PROCESS,
  c_case_ntwk=MASTERCARD, c_acq_platform=CORE)? Report the exact predicate."
- **Needs architect decision:** Are `PAB_withdraw`, `ARB_withdraw`,
  `PRA_withdraw` concatenated `c_case_stage` values intentional, and is
  any downstream consumer coded to parse them? If not, COMP-14
  CaseCreationConsumer may be silently treating them as unrecognised
  stages.
- **Needs architect decision:** Is the unconditional `enrichmentFailure=true`
  on the payload intentional or legacy? If intentional, consider removing
  the field name that implies meaning it does not carry.
- **Needs architect decision:** Formal ADR for the missing MCM
  acknowledgement — implement, formally remove, or accept with a team
  confirmation on MCM queue behaviour for un-ACKed claims.
- **Needs environment / team confirmation:** Production XL Deploy replica
  count value for COMP-09. If > 1, duplicate-INSERT risk is live.
- **Needs environment / team confirmation:** K8s secret `table_prefix`
  value — Spring Batch metadata schema. Prior DRAFT conjecture `WDP.BATCH_`
  is unverified.
- **Needs environment / team confirmation:** Production cron value for
  COMP-09 (`scheduler_cron` secret).
- **Needs MasterCard team confirmation:** What does MCM do with a claim
  that WDP has processed but never acknowledged? Re-queue? TTL? This is
  the blocker for resolving the MCM ACK ADR.

#### Doc status after this change
- `WDP-COMP-09-CASE-FILLING-BATCH.md` → `v2.0 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending

---
### 2026-04-18 — COMP-08 FirstChargebackBatch · v1.0 DRAFT → v2.0 DRAFT

**Source:** `wdp-mcm-first-chargeback-queue-batch` — source-verified by Claude
Code 2026-04-18. Architect confirmation still pending.

**Nature of change:** Correction pass against source, with three new
architectural findings surfaced that were not visible in the v1.0 DRAFT.
No functional change in production; corrections plus new findings for the
documented behaviour of an existing component.

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- Add the `created_by` distinction: `"WMFDPB"` for COMP-08 (distinct from
  COMP-07 `"WVDPB"` and COMP-11 `"WPFLEPR"`).
- Add "COMP-08 duplicate-check path writes SKIPPED marker rows rather than
  suppressing the insert — `skipCase` flag on `ProcessedItem` is computed
  but never consumed by its Writer. One SKIPPED row accumulates per
  re-polled known chargeback per COMP-08 scheduler run. Growth boundedness
  depends on whether COMP-12 Scheduler2 archives SKIPPED rows — not
  currently confirmed." to the notes.
- Add "COMP-08 update-path writes status=PENDING with accountNumber=null on
  newly-arrived chargebacks on an existing claim — `processUpdatedClaims`
  never sets `isAccountNumberRequired=true`. Suspected defect." to the
  notes.
- Add "COMP-08 writer-ACK hazard — mid-chunk JPA save failure does not
  prevent the subsequent ACK PUT to MCM; ACK PUT exceptions are swallowed
  silently with no log, no metric, no outbox marker. Items acknowledged
  off MCM with no corresponding outbox row are unrecoverable." to the
  notes.
- Reinforce "no DB unique constraint confirmed — no DDL / Flyway /
  Liquibase / entity-level `uniqueConstraints` in COMP-08 repo;
  `ddl-auto: false`; `initialize-schema: never`." (same as COMP-07 —
  pattern repeats.)

**WDP-DB.md · Section 2 · Spring Batch Metadata Tables section**
- Add "COMP-08 sets `spring.batch.jdbc.initialize-schema = never` and
  `spring.jpa.hibernate.ddl-auto = false` — Spring Batch metadata tables
  must pre-exist. DDL is not managed by COMP-08." (same pattern as
  COMP-07 — applies to both.)

**WDP-DB.md · Section 4 · Shared Table Risk Register**
- Append clarifying note to the `wdp.chbk_outbox_row` risk entry: "COMP-08
  source verification 2026-04-18 also confirms no DB unique constraint
  visible in repo. Whether a unique index exists in the live schema must
  be confirmed with the DBA team — same open question as COMP-07."
- Append new risk note: "SKIPPED-row accumulation is a shared-table growth
  concern introduced by COMP-08's insert-a-marker idempotency pattern.
  Whether COMP-07 and COMP-09 share the same pattern needs re-audit."

**WDP-KAFKA.md · Sections 3 and 4**
- No change. COMP-08 has no active Kafka involvement (staged POM deps
  only). Explicit confirmation for next reconciliation session: grep
  against source for `@EnableKafka`, `KafkaTemplate`, `ProducerFactory`,
  `@KafkaListener` returns zero hits.

**WDP-KAFKA.md · Section 6 · Components Confirmed Kafka-Free**
- Minor enrichment to COMP-08 row: note that the entity declares `kafka_*`
  columns but they are never written by this component (owned by COMP-12
  downstream), and that `DisputeServiceImpl` carries a commented-out
  Kafka-shaded Protobuf import — only classpath-visible Kafka reference
  in the codebase.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Add or correct:
- COMP-08 duplicate-check path writes SKIPPED marker rows rather than
  suppressing the insert. The `skipCase` flag on `ProcessedItem` is
  computed but never consumed by the Writer. First confirmed
  "insert-a-marker" idempotency pattern in WDP (distinct from
  "suppress-insert").
- COMP-08 update-path `processUpdatedClaims` never sets
  `isAccountNumberRequired=true`. Newly-arrived chargebacks on an
  existing claim write `status=PENDING` with `accountNumber=null` — no
  HPAN in the payload. Suspected defect.
- COMP-08 writer-ACK hazard: mid-chunk JPA save failure aborts the inner
  save loop but control still reaches the ACK PUT with whatever pairs
  already reached `acknowledgeList`. ACK PUT exception swallowed silently
  in the MCM client — no log of failed pairs, no outbox marker, no
  metric. Potential silent data loss.
- COMP-08 uses a single shared `RestTemplate` bean for all six outbound
  integrations with no connect timeout, no read timeout, no connection
  pool, no custom `ClientHttpRequestFactory`.
- COMP-08 IDP Token GET is unauthenticated (no Authorization header, no
  API key, no mTLS on the application side).
- COMP-08 DataPower auth uses static Vantiv licence as the raw
  `Authorization` header value (no `Bearer` / `Basic` prefix) — Vantiv
  scheme, not a defect.
- COMP-08 `correlationId` is a per-event UUID, not per-job or per-item.
  No MDC correlation across a cron fire.
- COMP-08 `enrichmentFailure=true` inside `OriginalTransIdentifier` is
  set unconditionally on every outbox payload — flag name misleading.
- COMP-08 currency exponents are hardcoded (`USD_EXPONENT=2`,
  `CAD_EXPONENT=2`); actual currency field is never consulted.
- COMP-08 masked-PAN last-4 extraction takes the last 4 characters of
  the masked string — correct only if upstream mask preserves last-4
  positions.
- COMP-08 `created_by` is `"WMFDPB"` (distinct from COMP-07 `"WVDPB"`
  and COMP-11 `"WPFLEPR"`).
- COMP-08 Spring Batch job uniqueness is per-millisecond via a single
  timestamp `JobParameter` (`yyyyMMdd_HHmmss.SSS`). Two launches within
  the same ms collide; different-ms parallel launches proceed.
- COMP-08 has no liveness, readiness, or startup probes wired. Actuator
  is on classpath but Kubernetes is not configured to use it.
- COMP-08 `minReadySeconds: 30` is placed at `spec.template.spec` level
  in `resources.yml` — effectively ignored by Kubernetes for a
  Deployment.
- COMP-08 JobLauncher is default `SyncTaskExecutor` — no custom
  `TaskExecutor`.
- COMP-08 has no `@SchedulerLock`, no advisory lock, no `synchronized`
  guard, no `SELECT ... FOR UPDATE`. Replica=1 is operational policy,
  not code-enforced.
- COMP-08 has no `@Recover` method anywhere. Retry-exhaustion is either
  caught in the processor outer catch (claim skipped, no ACK, re-queued)
  or swallowed in the MCM client (for ACK PUT) — never propagated.
- COMP-08 `ddl-auto=false`, `initialize-schema=never` — both application
  and Spring Batch metadata tables must pre-exist.

Resolved open questions (remove from HANDOVER):
- Writer failure semantics for COMP-08 — now documented.
- ACK failure semantics for COMP-08 — now documented.
- COMP-08 masked-PAN branch semantics — confirmed acceptable under
  DEC-019.
- COMP-08 Spring Batch metadata schema provisioning — now documented
  (same pattern as COMP-07: no default, must pre-exist).

New open questions (add to HANDOVER):
- Production replica count for COMP-08 — XL Deploy placeholder value
  needs confirmation; any value > 1 is a DEC-023 violation.
- Production cron value for COMP-08 — K8s Secret, not auditable from
  repo.
- DB unique constraint on `wdp.chbk_outbox_row (c_ntwk_case_id,
  c_ntwk_phase_id)` — DBA team confirmation needed (same open question
  as COMP-07).
- Does COMP-12 Scheduler2 archive SKIPPED rows, or only SUCCESS rows?
  Determines whether COMP-08 SKIPPED-row accumulation is bounded or
  unbounded.
- Does MCM deliver chargebacks in non-two-decimal-exponent currencies
  (e.g. JPY, KWD) on the `AcquirerFirstCBUnworked` queue? Relevant to
  the hardcoded `USD_EXPONENT=2` / `CAD_EXPONENT=2` risk.
- Under what upstream conditions does MCM return `primaryAccountNum`
  containing `*`? Is the mask guaranteed to preserve the last 4
  positions?
- COMP-08 update-path PENDING-without-HPAN — defect or deliberate
  design? Architect decision required.
- COMP-08 writer-ACK hazard — accepted risk or bug? Architect decision
  required on whether to gate the ACK on per-item save success.
- Do COMP-07 VisaDisputeBatch and COMP-09 CaseFillingBatch exhibit the
  same writer-ACK hazard pattern? Needs targeted re-audit.
- Do COMP-07 and COMP-09 use the same insert-a-marker idempotency
  pattern that COMP-08 does? Needs targeted re-audit.

**WDP-DECISIONS.md · Candidate new ADRs**
- **HIGH candidate (new):** "Idempotency pattern classification —
  suppress-insert vs insert-a-marker on shared outbox tables." COMP-08
  is the first confirmed instance of the insert-a-marker approach on
  `wdp.chbk_outbox_row`. Platform-level ADR required to document which
  pattern is acceptable, because insert-a-marker has table-growth
  implications for a shared outbox.
- **HIGH candidate (new):** "Writer-ACK hazard on MCM-style outbox
  ingest — DB save failure does not prevent upstream ACK." If the
  pattern is confirmed in COMP-07 and COMP-09 on re-audit, this needs a
  platform-level ADR covering all three ingestion batches.
- **HIGH candidate (extends COMP-07 raise):** "Silent exception swallow
  in batch writer and ACK paths" — COMP-08 now also confirmed.
  Strengthens the case for a platform-level ADR covering COMP-07,
  COMP-08, and likely COMP-09.
- **MEDIUM candidate (extends COMP-07 raise):** "No health probes on
  polling batches" — COMP-08 also confirmed absent.
- **MEDIUM candidate (extends COMP-07 raise):** "No code-level
  concurrency guard on polling batches — DEC-023 operational-only
  enforcement" — COMP-08 also confirmed.
- **MEDIUM candidate (new):** "`enrichmentFailure=true` misleading flag
  name — unconditionally true on every COMP-08 outbox payload." Needs
  developer intent confirmation before becoming an ADR or being removed
  from the payload.
- **MEDIUM candidate (new):** "Currency exponent hardcoding
  (`USD_EXPONENT=2`, `CAD_EXPONENT=2`) in COMP-08 — actual currency
  never consulted." Needs confirmation of whether MCM delivers
  non-two-decimal currencies on this queue before disposition.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- No new rows required. RISK-013 (replica constraint has no automated
  enforcement) and RISK-014 (`removeItemFromQueueDisabled` has no
  automated state check) already cover the HIGH risks for COMP-08 as
  well — same as COMP-07.
- Minor clarification opportunity on RISK-013: extend the covered-
  components list to include COMP-08 explicitly (pattern matches
  COMP-07).
- Candidate new RISK row pending architect decision on writer-ACK
  hazard: "Silent ACK-after-DB-failure data loss on MCM-style ingest —
  COMP-08 confirmed; COMP-07 and COMP-09 re-audit pending."

**WDP-INTEGRATIONS.md**
- No change. COMP-08 integration to MCM via IBM DataPower is already
  documented and unchanged.

#### Deviation flags for COMP-08

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ COMPLIES | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES (conditional) | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-022 `removeItemFromQueueDisabled` Safety Switch | ✅ PRESENT | — |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY | 🟡 MEDIUM |

**DEC-004 CONDITIONAL detail:** Compliant on the new-claim first-chargeback
path — clear PAN encrypted by COMP-35 before the outbox save. Masked PANs
arriving masked from MCM are written verbatim without encryption —
acceptable under DEC-019 (no clear PAN reaches storage) but noted as
conditional compliance rather than unconditional. Non-first chargebacks on
a new claim and all chargebacks on the existing-claim path write
`accountNumber=null` — no PAN is persisted on these paths, so DEC-004 does
not apply to them.

**DEC-020 PARTIAL detail (severity elevated to HIGH vs COMP-07's MEDIUM):**
Four concurrent gaps: (a) no DB unique constraint visible in repo,
(b) Spring Batch JobInstance uniqueness is per-millisecond only (no
distributed lock), (c) writer-ACK hazard can ACK items off MCM without a
corresponding outbox row (silent data loss path), and (d) update-path
writes PENDING rows with `accountNumber=null` on newly-arrived chargebacks
on existing claims — idempotency semantically completes but delivers
incomplete data. The severity elevation vs COMP-07 is driven by (c) and
(d), which are specific to COMP-08.

**DEC-023 OPERATIONAL ONLY detail:** Identical pattern to COMP-07. No
`@SchedulerLock`, no advisory lock, no synchronized guard. Default
`SyncTaskExecutor` JobLauncher. Rollout with `maxSurge=1
maxUnavailable=0` can temporarily run replicas+1 pods. Replica=1 is
policy, not code.

#### Doc status after this change
- `WDP-COMP-08-FIRST-CHARGEBACK-BATCH.md` → `v2.0 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending

---
---

### 2026-04-18 — COMP-07 VisaDisputeBatch · v1.0 DRAFT → v1.1 DRAFT

**Source:** `gcp-visa-disputes-processor-batch` — source-verified by Claude
Code 2026-04-18. Architect confirmation still pending.

**Nature of change:** Correction pass against source. No functional change in
production; corrections to the documented behaviour of an existing component.

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- Add "COMP-07 also writes SKIPPED on duplicate and ERROR on missing key data;
  `skipCase` items write no row" to the writer list.
- Add the `created_by` distinction: `"WVDPB"` for COMP-07 vs `"WPFLEPR"` for
  COMP-11.
- Add "no DB unique constraint confirmed — no DDL / Flyway / Liquibase /
  entity-level `uniqueConstraints` in COMP-07 repo; `ddl-auto: false`;
  `initialize-schema: never`" to the notes.

**WDP-DB.md · Section 4 · Shared Table Risk Register**
- Append clarifying note to the `wdp.chbk_outbox_row` risk entry: "COMP-07
  source verification 2026-04-18 confirms no DB unique constraint visible in
  repo. Whether a unique index exists in the live schema must be confirmed
  with the DBA team."

**WDP-KAFKA.md · Sections 3 and 4**
- No change. COMP-07 has no active Kafka involvement (staged POM deps only).
  Explicit confirmation for next reconciliation session: grep against source
  for `@EnableKafka`, `KafkaTemplate`, `ProducerFactory`, `@KafkaListener`
  returns zero hits.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Add or correct:
- COMP-07 `skipCase` path writes no outbox row at all (previously implied
  SKIPPED was written).
- COMP-07 writer save failures and MarkAsRead retry-exhaustion are
  caught-and-logged at `BatchItemWriter.java:139-142`, never propagated.
  Spring Batch sees chunk completion as successful.
- COMP-07 has no `@Transactional` anywhere. Chunk transaction is the only
  transactional boundary.
- COMP-07 has no liveness, readiness, or startup probes wired.
- COMP-07 feature flags (`coreDataNeedToMigrate`,
  `removeItemFromQueueDisabled`, `readSpecificItemFromQueue`) have no
  defaults — startup fails if K8s Secret absent.
- COMP-07 JobLauncher is default `SyncTaskExecutor` — no custom TaskExecutor.
- COMP-07 correlation ID is a fresh UUID `v-correlation-id` per outbound call,
  not per-item or per-run, and not added by `IdpRestInvoker`. No MDC.

Resolved open questions (remove from HANDOVER):
- Writer failure semantics — now documented.
- MarkAsRead retry-exhaustion behaviour — now documented.
- `skipCase` path end-to-end — now documented.
- Spring Batch metadata schema behaviour — now documented (no default,
  startup-fails-if-absent).
- Feature flag defaults — now documented.

New open questions (add to HANDOVER):
- Production replica count for COMP-07 — XL Deploy placeholder value needs
  confirmation; any value > 1 is a DEC-023 violation.
- Production cron value for COMP-07 — K8s Secret, not auditable from repo.
- DB unique constraint on `wdp.chbk_outbox_row (networkCaseId,
  networkPhaseId, disputeStage)` — DBA team confirmation needed.
- `migrationStatus = "Y"` hardcoded — permanent design or migration-era
  workaround? Architect decision required.

**WDP-DECISIONS.md · Candidate new ADRs**
- **HIGH candidate:** "Silent exception swallow in batch writer and
  MarkAsRead paths" — affects COMP-07 confirmed, likely affects COMP-08 and
  COMP-09 by pattern. Recommend raising at next DECISIONS rebuild window.
- **MEDIUM candidate:** "No health probes on polling batches" — affects
  COMP-07 confirmed, likely COMP-08 and COMP-09.
- **MEDIUM candidate:** "No code-level concurrency guard on polling
  batches — DEC-023 operational-only enforcement" — explicit recognition
  that replica=1 is policy, not code.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- No new rows required. RISK-013 (replica constraint has no automated
  enforcement) and RISK-014 (`removeItemFromQueueDisabled` has no
  automated state check) already cover the HIGH risks for this component.
- Minor clarification opportunity on RISK-013: add note that there is also
  no code-level guard, only operational enforcement.

**WDP-INTEGRATIONS.md**
- No change. COMP-07 integration to Visa RTSI via DataPower is already
  documented and unchanged.

#### Deviation flags for COMP-07

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ COMPLIES | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-022 `removeItemFromQueueDisabled` Safety Switch | ✅ PRESENT | — |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY | 🟡 MEDIUM |

**DEC-020 PARTIAL detail:** RECALL bypasses duplicate check; no DB unique
constraint; null-return paths write no audit; writer and MarkAsRead
exceptions caught-and-logged; crash window exists between Visa poll and
DB save.

**DEC-023 OPERATIONAL ONLY detail:** No `@SchedulerLock`, no advisory lock,
no synchronized guard. `SyncTaskExecutor` JobLauncher. Replica=1 is policy,
not enforced by code.

#### Doc status after this change
- `WDP-COMP-07-VISA-DISPUTE-BATCH.md` → `v1.1 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending

---

## Pending Entries

*New entries are appended here by per-component chats. Reconciliation sessions
consume entries from this section and move them to the most recent **Reconciled**
section above.*



---

### 2026-04-29 — COMP-42 BENConsumer · v1.0 DRAFT → v2.0 DRAFT

**Source:** `wdp-ben-consumer` — source-verified by GitHub Copilot CLI 2026-04-29.
Architect confirmation pending.

**Nature of change:** Correction pass — first source-verified audit of COMP-42
(file was previously a v1.0 Copilot-extracted DRAFT that had not been reconciled
in the 2026-04-18/23/25 cycle). Five material corrections identified, two new
DEC ratings added, DEC-014 reframed as VOID (now Risk), three probe absences
confirmed, container port surfaced, and the `wdp.outgoing_event_outbox` writers
list corrected platform-wide.

#### Platform-level impacts

**WDP-DB.md**
- 🔴 **CRITICAL** — `wdp.outgoing_event_outbox` writers list correction.
  Current Section 2 row (line 99) and Section 4 shared-table row (line 241)
  list writers as **COMP-17, COMP-41, COMP-43**. **Add COMP-42** with:
    - `channel_type = BEN_EVENTS`
    - `created_by` / `updated_by` constant = `WBENC`
    - Status set used: PUBLISHED / SUCCESS / FAILED / ERROR / PENDING_DEFERRED
      (SKIPPED defined in enum but never written)
- Update Section 4 shared-table risk note: now FOUR consumers writing
  `wdp.outgoing_event_outbox` discriminated by `channel_type`. Each writer
  uses its own constant. **No DB-level UNIQUE constraint visible in any of
  the four repos** — DBA confirmation required.
- Update Section 4 cross-action interference / predecessor-scope notes:
  COMP-42 predecessor scope is `caseNumber + channel_type=BEN_EVENTS + id 
  currentId` applied as in-memory filter after `findByCaseNumber` returns
  ALL channel_types. Same scope-pattern class as COMP-17 (predecessor scope
  is per-channel, not per-action_seq).

**WDP-KAFKA.md**
- Section 5 (Database-Backed Outbox Pattern) row for `wdp.outgoing_event_outbox`
  — add COMP-42 to writers list with `channel_type=BEN_EVENTS` and
  `created_by=WBENC`. Reflects the same correction as WDP-DB.md.
- Section 3 (Pre-ACK consumers row, line 85) for COMP-42 — already lists
  pre-ACK position but should be tagged as "✅ Confirmed from COMP-42 source"
  (was unmarked).
- Section 4 (Partition-key inbound observation, line 123) — confirmation
  reaffirms variable name `caseNumber` of type `String` via
  `@Header(KafkaHeaders.RECEIVED_KEY)`. Producer-side determination remains
  a COMP-18 OQ.
- No new topic, no new consumer group, no new producer cluster registration.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: COMP-42 is the FOURTH writer of `wdp.outgoing_event_outbox`
  (`channel_type=BEN_EVENTS`, writer constant `WBENC`). Previous handover
  fact list and WDP-DB.md v2.1 omitted COMP-42 from the writers list — this
  is a known gap from the v1.0 doc not having been audited in the 2026-04-25
  reconciliation pass.
- Add: COMP-42 has zero `@Transactional` in `src/main` — all outbox writes
  are bare auto-commit JPA `save()` calls. Same pattern class as COMP-14
  (RISK-A) and COMP-43.
- Add: COMP-42 has all three K8s probes ABSENT — no liveness, no readiness,
  no startup. Same operational pattern as COMP-14, COMP-12, COMP-07,
  COMP-21.
- Add: COMP-42 BEN product cache is `@Cacheable` lazy with `@Scheduled`
  `@CacheEvict` on `${app.cache-delay-hours}` fixed delay — NOT pre-warmed
  at startup as v1.0 doc implied.
- Add: COMP-42 IDP token is dual-mode — lazy ThreadLocal for per-event
  CMS/CAS calls, eager (`fetchNewToken=true`) for the Display Code Service
  startup call only.
- Add: COMP-42 predecessor lookup loads ALL channel_types for the
  caseNumber from DB then filters to BEN_EVENTS in-memory — unbounded
  query per caseNumber.

**Resolved open questions**
- "COMP-42 outbox involvement" — RESOLVED. COMP-42 IS a writer of
  `wdp.outgoing_event_outbox` with `channel_type=BEN_EVENTS`, `created_by=WBENC`.
  WDP-DB.md v2.1 row needs correction.
- "COMP-42 BEN delivery transport" — RESOLVED (was already in handover line
  280). Confirmed Kafka-only via BEN-owned MSK cluster — no REST webhook.
- "COMP-42 partition key on outbound BEN producer" — RESOLVED. `merchantId`
  from `CaseSearchResponse`. DEC-003 compliant.
- "COMP-42 PAN handling" — RESOLVED. DEC-019 / DEC-004 compliant.
  `cardNumberLast4` is the only card field forwarded; full PAN never
  persisted, never published.
- "COMP-42 spring-retry direct-dependency status" — RESOLVED. Transitive
  via spring-kafka — not declared in pom.xml. Same pattern as COMP-41
  RISK-080.
- "COMP-42 OAuth2 starters usage" — RESOLVED. Both
  `spring-boot-starter-oauth2-resource-server` and
  `spring-boot-starter-oauth2-client` are dead — zero usage in `src/main`.

**New open questions**
- **OQ-COMP42-1: Scheduler3 channel_type filter behaviour for BEN_EVENTS.**
  Same filter-class question as COMP-41 (OQ-COMP41-1). Does Scheduler3 read
  `channel_type=BEN_EVENTS` rows at FAILED / PENDING_DEFERRED? Does it ever
  read PUBLISHED for any channel? Cross-component scan against COMP-12
  source needed. **Action: Claude Code follow-up on COMP-12.**
- **OQ-COMP42-2: BEN platform idempotency.** A retry-driven re-publish (or
  a manual recovery of a PUBLISHED-orphan) will deliver the same
  `BENNotificationEvent` to BEN twice. Does BEN deduplicate at its end?
  Same question class as COMP-41 (Signifyd). **Action: BEN integration
  team confirmation.**
- **OQ-COMP42-3: DB-level UNIQUE constraint on
  `(idempotency_id, channel_type, event_timestamp)`.** Now four repos
  (COMP-17, COMP-41, COMP-42, COMP-43) all confirm "no UNIQUE visible in
  source." **Action: DBA confirmation — does the constraint exist at the
  DB level, or is per-`channel_type` race protection only operational
  (`concurrency=1`)?**
- **OQ-COMP42-4: Production runtime values** — `${ben_topic}`,
  `${ben_bootstrap_servers}`, `${ben_product_url}`, `${ben_product_license}`,
  `${max_poll_records}`, `${max_poll_interval}`, `${session_timeout_ms}`,
  `${heartbeat_interval_ms}`, `${app.cache-delay-hours}`, `replicas-wdp-ben-consumer`.
  All XL Deploy / K8s secret resolved — not in source. **Action: Environment
  config / team confirmation.**

**WDP-DECISIONS.md · Candidate new ADRs**
- **No new ADR candidates from COMP-42 alone.** All findings reinforce
  existing ADR/RISK patterns:
  - DEC-001 partial-deviation pattern (PUBLISHED-as-initial-status,
    inline-publish-no-relay) — fourth instance after COMP-18, COMP-41, COMP-43
  - DEC-005 pre-ACK pattern — fourth instance after COMP-14, COMP-41, COMP-43
  - DEC-014 VOID — sixth-component evidence base
  - DEC-020 deviation pattern — fourth instance after COMP-14, COMP-41, COMP-43
- **Reaffirms existing candidate ADR backlog (handover line 128):**
  - "PUBLISHED-orphan / Scheduler3 channel_type-filter contract" —
    COMP-42 adds `channel_type=BEN_EVENTS` to the scope of this candidate
  - "DB-level UNIQUE on `wdp.outgoing_event_outbox`" — COMP-42 is the
    fourth writer requiring this contract clarification

**WDP-ARCHITECTURE.md**
- No topology change. COMP-42 is already shown in the BEN integration path.
- Minor wording correction candidate for the next reconciliation pass:
  if any version of WDP-ARCHITECTURE.md still describes BEN delivery as
  "webhook," correct to "Kafka publish to BEN-owned MSK cluster (separate
  SASL/JAAS)." Confirmed already corrected in WDP-INTEGRATIONS.md v2.0
  per handover.

**WDP-NFRS.md · Section 6 Risk Register**

New / updated RISK candidate rows for the next reconciliation pass:

| Risk ID candidate | Severity | Detail |
|---|---|---|
| RISK-COMP42-A | 🔴 HIGH | PUBLISHED-orphan crash window Step 4 → Step 13. Same class as RISK-040 (COMP-41) and the COMP-43 crash-window. Scheduler3 invisibility — no automatic recovery. |
| RISK-COMP42-B | 🟡 MEDIUM | All three K8s probes ABSENT — stuck consumer thread will not restart pod. Combined with `concurrency=1` and infinite REST timeouts → operational stall risk. |
| RISK-COMP42-C | 🟡 MEDIUM | Bad payload silently dropped — `ErrorHandlingDeserializer` + empty `CommonErrorHandler`. Same class as RISK-077 (COMP-41). |
| RISK-COMP42-D | 🟡 MEDIUM | All six outbound dependencies have no read/connect timeout configured. Hung downstream blocks consumer indefinitely. |
| RISK-COMP42-E | 🟢 LOW | Predecessor lookup loads all channel_types for caseNumber, filters to BEN_EVENTS in memory. Unbounded per-case query, no pagination, no LIMIT. |
| RISK-COMP42-F | 🟢 LOW | spring-retry transitive (not declared in pom.xml). Same class as RISK-080 (COMP-41). Future Spring Boot upgrade could break BEN Product retry silently. |
| RISK-COMP42-G | 🟢 LOW | Outer-catch ERROR write may itself fail silently — same class as COMP-08 writer-ACK hazard. |
| RISK-COMP42-H | 🟢 LOW | Two dead OAuth2 starter dependencies in pom.xml (resource-server + client) — security surface and confusion. |

**WDP-INTEGRATIONS.md**
- BEN integration entry — already corrected in v2.0 per handover
  (Kafka-only via BEN-owned MSK cluster; no REST webhook). No further
  change.
- BEN Product API — confirm in next reconciliation pass: GET with
  static license-key auth (`Bearer ${ben_product_license}`), Spring
  Retry 3 × 1000ms, no timeout, no circuit breaker. Same dependency
  resilience class as the other five COMP-42 dependencies.

**WDP-COMP-INDEX.md**
- Status update: line 642 row from `📝 DRAFT` to
  `📝 DRAFT 🔍 v2.0 (2026-04-29)` — source-verified, architect
  confirmation pending. Aligns with COMP-43's status format.
- Description block (lines 663–669) — no rewrite needed; already
  corrected in 2026-04-25 to confirm Kafka-only delivery via BEN-owned
  MSK cluster. Optional minor addition: add `channel_type=BEN_EVENTS`
  for parallelism with COMP-41 (`GP_EVENTS`) and COMP-43 (`CORE_EVENTS`)
  description blocks.

#### Deviation flags for COMP-42

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-003 | ✅ COMPLIES (outbound producer; inbound consumer-side observation only — producer-side determination is COMP-18 OQ) | — |
| DEC-004 | ✅ NOT APPLICABLE / COMPLIES | — |
| DEC-005 | ⛔ DEVIATES | 🔴 HIGH |
| DEC-014 | ⛔ VOID — recorded as Risk only | 🟢 LOW (per-component evidence row) |
| DEC-019 | ✅ COMPLIES | — |
| DEC-020 | ⛔ DEVIATES | 🔴 HIGH |

**DEC-001 PARTIAL narrative.** `wdp.outgoing_event_outbox` is used as an
outbox table, but: (1) initial status is `PUBLISHED` (set before BEN Kafka
publish), (2) outbox INSERT and offset commit are NOT atomic — zero
`@Transactional` in `src/main`, (3) no relay/poller in this codebase —
publish is inline in the consumer thread, (4) recovery depends on COMP-12
Scheduler3 which reads only FAILED / PENDING_DEFERRED. Same pattern
class as COMP-18, COMP-41, COMP-43 — fourth confirmed instance.

**DEC-005 DEVIATES narrative.** `MANUAL_IMMEDIATE` + `syncCommits=true`
acknowledge at Step 4 — after the idempotency DB INSERT, but BEFORE the
CMS REST call (Step 6), BEN Product call (Step 7), CAS REST call
(Step 10), BEN Kafka publish (Step 12), and the final outbox status
UPDATE (Step 13). Crash between Step 4 and Step 12 → committed offset
with no BEN notification, stale PUBLISHED row, no automatic recovery.

**DEC-014 VOID narrative.** No Resilience4j artifact in `pom.xml`. Zero
`@CircuitBreaker` / `@Bulkhead` / `@RateLimiter` / `@TimeLimiter` matches
in `src/main`. Six outbound dependencies (IDP, CMS, CAS, BEN Product,
Display Code, BEN Kafka) all unprotected. Per the 2026-04-25 strengthened
evidence note in DEC-014, this is recorded as a Risk row, not a deviation.

**DEC-020 DEVIATES narrative.** Pre-ACK at Step 4 means the broker will
not redeliver if processing fails post-ACK. End-to-end at-least-once
relies on Scheduler3 driving FAILED / PENDING_DEFERRED rows. PUBLISHED-
orphans (Step 4 → Step 13 crash) have no recovery path. The idempotency
SELECT-then-INSERT runs as separate short JPA transactions with no
DB-level UNIQUE constraint visible — operational `concurrency=1` is the
sole protection.

#### Remaining gaps after this entry

- **OQ-COMP42-1 (Scheduler3 BEN_EVENTS filter behaviour)** — needs Claude
  Code follow-up on the COMP-12 source. Exact question to ask:
  > In `wdp-inbound-dispute-event-scheduler` (COMP-12), specifically
  > Scheduler3, does the FAILED / PENDING_DEFERRED query against
  > `wdp.outgoing_event_outbox` filter by `channel_type` or by some other
  > criterion? Does it read `channel_type='BEN_EVENTS'` rows? Does it
  > ever read `status='PUBLISHED'` for any channel? Cite the JPQL/SQL
  > and the schedule cron for Scheduler3.
- **OQ-COMP42-2 (BEN-side idempotency)** — BEN integration team
  confirmation. Cannot be answered from any WDP source.
- **OQ-COMP42-3 (DB-level UNIQUE constraint)** — DBA confirmation. Now
  spans four repos (COMP-17, COMP-41, COMP-42, COMP-43). Cannot be
  answered from any component repo.
- **OQ-COMP42-4 (production runtime values)** — environment config /
  team confirmation. Replica count, BEN topic name, BEN bootstrap
  servers, BEN product URL, BEN product license, all max-poll / session
  / heartbeat / cache values. Not in source by design.
- **Runtime observation gaps** (Copilot CLI cannot answer on a re-run):
  - Whether the actual production `wdp.outgoing_event_outbox` table has
    a UNIQUE constraint at the DB level.
  - Whether BEN-side deduplicates duplicate `BENNotificationEvent`s.
  - Actual production volumes of BEN_EVENTS rows accumulating in
    `wdp.outgoing_event_outbox` (capacity-planning input).
  - Frequency at which `@Scheduled` `@CacheEvict` clears the BEN product
    cache — value held in `${app.cache-delay-hours}` K8s config.

#### Doc status after this change
- `WDP-COMP-42-BEN-CONSUMER.md` → `v2.0 DRAFT` — source-verified
  2026-04-29 · architect confirmation pending.
---
### 2026-04-29 — COMP-40 VisaResponseQuestionnaire · v1.0 DRAFT → v1.1 DRAFT

**Source:** `gcp-visa-respond-questionnaire-consumer` — source-verified by
GitHub Copilot CLI on 2026-04-29. Architect confirmation pending.

**Nature of change:** Correction pass — line-number re-verification, gap closures
on probes / observability / idempotency posture, and platform-pattern findings
(misplaced `minReadySeconds`, IDP-token-per-message, Kafka-path MDC absence,
unused `spring-boot-starter-cache`).

#### Platform-level impacts

**WDP-DB.md**
- No change. COMP-40 owns no database state and reads no tables directly.
  No JPA / JDBC / Hibernate / DynamoDB SDK on classpath. Confirms v1.0 posture.

**WDP-KAFKA.md**

*Section 3 — Topic Registry — `internal-integration-events`:* no row change.
COMP-40 already listed as consumer with group `internal-integration-events-ques-group`
(reconciled 2026-04-25). v1.1 audit re-confirms.

*Section 4 — Consumer Map row for COMP-40:* no row change. Re-confirmed:

| Component | Consumes from | Produces to | Notes |
|---|---|---|---|
| COMP-40 VisaResponseQuestionnaire | `internal-integration-events` | None | AckMode `MANUAL_IMMEDIATE`. Offset committed pre-ACK (at-most-once). Concurrency=1. Container factory `notificationListener`. No DLQ. No local error table. Stateless. Single `@KafkaListener` in codebase. |

*Section "Confirmed DEC-005 Deviations":* COMP-40 row already present
(reconciled 2026-04-25). v1.1 audit re-confirms `acknowledge()` at
`KafkaConsumer.java:L30` precedes `processContestEvent()` at `:L32`.

**WDP-DECISIONS.md** *(candidate inputs for next reconciliation)*

- **DEC-019 attestation pattern — COMP-40 confirms.** Same verifiable
  read-only-via-architecture pattern as COMP-28 / COMP-32 — DEC-019 compliance
  can be asserted at architecture level by codebase scan for
  `pan` / `cardNumber` / `accountNumber` / `acctNum` plus absence of any
  persistent write surface. COMP-40 returns DTO declarations only, zero writes.
- **DEC-020 platform-pattern reinforcement.** COMP-40 confirms that pre-ACK
  consumers structurally cannot meet at-least-once idempotency. Add to the
  candidate "DEC-020 formal void or scope-narrow" ADR — COMP-40 is now the
  10th confirmed pre-ACK consumer with no compensating idempotency mechanism.
- **DEC-023 scope clarification — COMP-40 stateless-Kafka-consumer pattern.**
  COMP-40 documents the explicit "DEC-023 NOT APPLICABLE" reasoning for
  multi-replica stateless Kafka consumers under at-most-once: concurrency=1
  per pod, partition assignment distributes work, no duplication risk, no
  scheduler-lock needed. Worth recording as a verifiable architecture-level
  attestation pattern alongside the DEC-019 one.
- **Candidate new ADR — "IDP token caching contract".** Same anti-pattern
  class as COMP-21 (`CachedTokenServiceImpl` with no actual cache).
  COMP-40 has `spring-boot-starter-cache` on classpath but no
  `@EnableCaching` and no `@Cacheable` — fetches IDP token per message.
  Recommend a platform-wide IDP token caching contract: required cache TTL,
  required cache backend (Caffeine/Redis), and a positive test that a
  second call within TTL does not hit IDP.
- **Candidate new ADR — "minReadySeconds placement contract".** Fifth
  confirmed component (COMP-25 / COMP-28 / COMP-34 / COMP-08 / COMP-40)
  with `minReadySeconds: 30` declared inside `spec.template.spec` where
  Kubernetes silently ignores it. This is a recurring K8s manifest-template
  defect. Recommend a platform manifest-lint rule and a sweep across the
  remaining 45 components.
- **Candidate new ADR — "Kafka-path MDC enrichment contract".** Same gap
  class as COMP-43 / COMP-17 / COMP-18 / COMP-51. COMP-40 has
  `HttpInterceptor` for HTTP MDC but no equivalent on the Kafka listener
  path. Recommend a platform-wide Kafka-listener MDC enrichment contract.

**WDP-INTEGRATIONS.md** *(no contract changes)*

- No external integration contract changes. The Visa RTSI integration
  (NAP via `mdvs-gcp-visa-adapter` + Bearer IDP; non-NAP via DataPower with
  `vantiveLicense` raw header) is already documented in Section 3.1.
- Worth recording at next reconciliation: COMP-40 confirms "single
  `${dispute_rtsi_url}` for all non-NAP platforms (CORE / VAP / LATAM / PIN)"
  — no per-platform DataPower variant. Closes a latent ambiguity in
  the Section 3.1 description.

**WDP-NFRS.md · Section 6 Risk Register** *(candidate new RISK rows)*

- **🔴 HIGH — RISK-COMP-40-A:** *"COMP-40 — `minReadySeconds: 30` misplaced
  under `spec.template.spec.minReadySeconds`; silently ignored by Kubernetes
  at runtime. The intended 30-second rollout stability gate is not actually
  applied — pods become Ready as soon as readiness probe passes. Same defect
  class as COMP-25 / COMP-28 / COMP-34 / COMP-08."* Pattern sweep candidate.
- **🔴 HIGH — RISK-COMP-40-B:** *"COMP-40 — `allocarb` `responseType`
  iterates all `visaResponseIds` with a separate RTSI call and document
  upload per ID, with no transactional bracketing. Failure on iteration N
  leaves iterations 1..N-1 successfully uploaded in DocumentManagementService
  while iterations N+1..end are never attempted. Recovery requires manual
  reasoning from SNOTE plus DocumentManagementService state."* Distinct
  failure mode — does not fit existing partial-success risk classes.
- **🟡 MEDIUM — RISK-COMP-40-C:** *"COMP-40 — IDP token fetched per Kafka
  message with no caching. `spring-boot-starter-cache` on classpath but no
  `@EnableCaching` and no `@Cacheable`. Compounds with the
  IDP-call-before-filter path (every discarded message still incurs an IDP
  RTT). Performance concern at high message volume."*
- **🟡 MEDIUM — RISK-COMP-40-D:** *"COMP-40 — Kafka listener path performs
  no MDC enrichment. `HttpInterceptor` is HTTP-only. Per-message log
  correlation depends entirely on OTel agent context — no application-level
  MDC fields. Same gap as COMP-43 / COMP-17 / COMP-18 / COMP-51."*
- **🟡 MEDIUM — RISK-COMP-40-E:** *"COMP-40 — IDP token call positioned
  BEFORE the entry filter (Decision A). Every COMP-19 `AcceptEvent` and every
  event with null `visaResponseIds` still incurs a wasted IDP token RTT
  before being silently discarded. Compounds with no-cache risk above."*
- **🟡 MEDIUM — RISK-COMP-40-F:** *"COMP-40 — `additionalImagesList`
  extracted from RTSI response but never uploaded to DocumentManagementService.
  Confirmed incomplete work — no TODO, no issue reference. Production cases
  may have evidence images that exist in Visa but never reach WDP storage."*
  Pattern: re-evaluate after architect decision (intentional gap vs backlog item).
- **🟡 MEDIUM — RISK-COMP-40-G:** *"COMP-40 — IDP token failure (Step 5)
  is logged INFO and swallowed by the outer Kafka listener catch with offset
  already committed. No SNOTE written, no error trail. Distinct from the
  RTSI/upload failure path which does write an SNOTE."*
- **🟢 LOW — RISK-COMP-40-H:** *"COMP-40 — `${app.name}` referenced in
  `application.yml` for `management.metrics.tags.application` but never
  defined. Only `spring.application.name` is set. Resolves to empty string
  / produces startup warning. Prometheus tag is effectively unset."*
- **🟢 LOW — RISK-COMP-40-I:** *"COMP-40 — `spring-boot-starter-cache`
  on classpath but no `@EnableCaching` and no cache configuration. Dead
  dependency. Best-practice deviation."*
- **🟢 LOW — RISK-COMP-40-J:** *"COMP-40 — `KafkaConsumer.java` log label
  reads `key-MerchantId` but binds the `caseNumber` variable. Stale label
  following producer-side key-convention shift from `merchantId` to
  `caseNumber`. No behavioural impact; observability noise."*
- **🟢 LOW — RISK-COMP-40-K:** *"COMP-40 — happy-path
  `testUploadDocument_valid` unit test commented out due to mock setup
  incompatibility. The primary upload success path has no unit test coverage."*

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged. COMP-40 remains in
  Section 8 Outbound Processing — Card Network Response. The findings are
  component-level (defect class) rather than topology-level.

**WDP-COMP-INDEX.md**
- COMP-40 row: status indicator update from `📝 DRAFT` to
  `📝 DRAFT 🔍 v1.1 (2026-04-29)` — same convention as COMP-41 (v1.1) and
  COMP-43 (v2.0).
- Description text: no change required. v1.0 description still accurate.

**WDP-FLOW-INDEX.md**
- No change. The "Visa contest end-to-end" flow involving COMP-20 →
  Kafka → COMP-40 → COMP-37 will be authored when the flow document is
  scheduled — partial-success behaviour on `allocarb` loops should be
  captured at that point.

#### Deviation flags for COMP-40

| DEC | Status | Severity |
|---|---|---|
| DEC-001 Transactional Outbox | ⛔ DEVIATES | 🔴 HIGH |
| DEC-003 Kafka partition key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN encryption before persistence | ✅ NOT APPLICABLE | — |
| DEC-005 Manual offset commit | ⛔ DEVIATES | 🔴 HIGH |
| DEC-014 Resilience4j | ⛔ DEVIATES (platform-VOID) | 🟢 LOW (accepted) |
| DEC-019 No clear PAN in persistent store | ✅ COMPLIES | — |
| DEC-020 Full at-least-once idempotency | ⛔ DEVIATES | 🔴 HIGH |
| DEC-023 Replica = 1 hard constraint | ✅ NOT APPLICABLE | — |

**DEC-001 detail:** No outbox table possible (no DB). Pure pass-through.
Partial-success state on `allocarb` loops cannot be reconciled — uploaded
documents persist without rollback; SNOTE records the failure but does not
undo prior uploads.

**DEC-005 detail:** `acknowledgement.acknowledge()` at `KafkaConsumer.java:L30`
precedes `processContestEvent()` at `:L32`. Joins the platform-wide pre-ACK
cluster (COMP-05/14/15/16/17/18/39/41/42/43 — COMP-40 is the 10th confirmed
case after COMP-43 in 2026-04-25 reconciliation).

**DEC-014 detail:** `io.github.resilience4j` absent from `pom.xml`. Spring
Retry on `VisaRTSIService` and `DisputeService` interfaces only — IDP token
and SNOTE calls have neither retry nor circuit breaker. No connect or read
timeout on any of the five outbound REST calls (single shared
`new RestTemplate()` in `CommonConfig`).

**DEC-020 detail:** Pre-ACK at-most-once defeats the at-least-once
idempotency contract. No `idempotency-key` header read, no dedup table, no
DB UNIQUE (no DB). Severity tracks DEC-005.

**DEC-023 detail:** Standard scaled stateless Kafka consumer. Concurrency=1
per pod; multiple replicas distribute across consumer-group partitions with
no duplication risk under at-most-once. No `@SchedulerLock`, ShedLock, or
advisory lock — confirmed absent. Documents the verifiable
architecture-level attestation pattern for stateless Kafka consumers.

#### Remaining gaps

| Gap | Type | Action required |
|-----|------|-----------------|
| Production replica count | Environment config | Confirm value for `{{ replicas-gcp-visa-respond-questionnaire-consumer }}` per environment via XL Deploy / Helm. |
| Production `app.retrycount` and `app.retrydelay` values | Environment config | Confirm K8s secret values for `api_retrycount` / `api_retrydelay` per environment — no defaults in YAML. |
| Production `dispute_rtsi_url` value | Environment config | Confirm DataPower URL per environment via K8s secret. |
| Production `vantive_license` value | Environment config | Confirm DataPower licence secret per environment. |
| Production `logstash_server_host_port` | Environment config | Confirm Logstash endpoint per environment. |
| `additionalImagesList` — intentional gap or backlog item? | Architect decision | Decide whether (a) backlog item with user story exists, (b) record as approved gap in WDP-DECISIONS.md, or (c) remediate. Architect call. |
| DEC-005 remediation posture for COMP-40 | Architect decision | Same as platform-wide pre-ACK posture. Either record formal acceptance with documented recovery procedure (manual log scan + RTSI re-pull + COMP-37 re-upload) or remediate by moving `acknowledge()` after `processContestEvent()` succeeds. |
| `allocarb` partial-success recovery procedure | Architect decision | Mid-loop failure leaves uploaded documents from earlier iterations unreconciled. Either record formal acceptance with manual recovery runbook or remediate via per-iteration outbox / per-iteration SNOTE / loop-level transactional bracket. |
| IDP-token-before-filter — accept or remediate? | Architect decision | Move IDP token fetch to AFTER Decision A short-circuit, or accept the wasted RTT and add IDP token caching to compensate. |
| IDP token caching | Architect decision | Decide caching strategy (Caffeine TTL, Redis, OAuth2 client manager). Cross-component decision — same pattern needed at COMP-21. |
| `${app.name}` undefined Prometheus tag | Team / runtime confirmation | Confirm whether the empty tag value causes any downstream Prometheus query / Grafana dashboard issue, or define `app.name` in `application.yml`. |
| `spring-boot-starter-cache` removal | Team confirmation | Remove from `pom.xml` if no caching is planned, or wire `@EnableCaching` if IDP token caching is approved. |
| `httpclient` 4.5.14 transitive vs explicit | Follow-up Copilot CLI question | Re-ask: *"Is `httpclient` 4.5.14 used by any class on classpath at runtime, or is it only present transitively from another dependency? If transitive, identify the parent dependency. If unused entirely, recommend removal."* |
| Per-replica Kafka partition assignment in production | Runtime observation | At runtime, observe broker-side consumer-group state to confirm partition distribution under multi-replica deployment. Cannot be answered from source. |
| Whether `micrometer-registry-prometheus` resolves transitively at runtime | Runtime observation / Maven dependency-tree | Confirm via `mvn dependency:tree` or runtime `/actuator/prometheus` smoke test that the registry is actually present. Source classpath scan was inconclusive. |
| Cross-component impact of DEC-019 attestation pattern | Architect decision | Decide whether the COMP-28 / COMP-32 / COMP-40 verifiable architecture-level DEC-019 attestation pattern should be formalised in WDP-DECISIONS.md as a recognised compliance class. |
---
### 2026-04-29 — COMP-39 NAPOutcomeProcessor · v1.0 DRAFT → v2.0 DRAFT

**Source:** `gcp-nap-dispute-update-consumer` — source-verified via GitHub
Copilot CLI 2026-04-29. Architect confirmation pending.

**Nature of change:** Source-verified correction pass. Surfaces a previously
undocumented REST entry path (Block A added), corrects multiple field-name
and constant transcription errors, sources rolling-update strategy and
JWT auth posture, classifies dependency usage, surfaces a cross-component
shared-table consumption hazard, and confirms `napcacrt.jks` is an
unreferenced orphan in the repo (NAP-DPS auth model is therefore handled
outside this component — Ingress / mesh / network).

#### Platform-level impacts

**WDP-DB.md**

- **Section 2 — `nap` schema · `NAP.DISPUTE_EVENT_CONSUMER_ERROR` row:**
  Add **COMP-39 NAPOutcomeProcessor** to the writers list. New writers
  list reads:
  *"COMP-05 NAPDisputeEventProcessor (primary; owns DDL); COMP-23
  CaseManagementService (NAP create path blind-merge); COMP-24
  CaseActionService (insert-path conditional, outbox-style UPDATE);
  COMP-39 NAPOutcomeProcessor (writes outbound-delivery error rows
  with C_ACQ_PLATFORM=`NAP`, C_EVENT_TYPE in {OUT_SRV118, OUT_SRV117};
  also reads back the table on every Kafka message via prior-error
  scan)."*

- **Section 2 — add new row for `NAP.NAP_UPDATE_RESPONSE_RULES`:**
  Read-only by COMP-39. Owner not determinable from this repo. JPA
  entity declared in COMP-39 but no DDL/migration is present. Flag
  for owner confirmation.

- **Shared Table Risk Register — update existing row for
  `NAP.DISPUTE_EVENT_CONSUMER_ERROR`:** Add a third risk note:

  *"🔴 NEW (2026-04-29): COMP-39 prior-error scan filters on
  `caseNumber` + `sourceSystemName="NAP"` + status IN (FAILED1, FAILED2)
  with **no `C_EVENT_TYPE` filter**. Error records written by COMP-05,
  COMP-23, and COMP-24 against the same case may be scooped up and
  reprocessed through COMP-39's SRV118/SRV117 outbound pipeline
  regardless of their original origin. Cross-component reprocessing
  semantics undefined. Architect decision required — intentional
  shared-recovery behaviour or defect."*

  Severity remains 🔴 HIGH on this row; the new note expands the
  failure mode rather than introducing a new row.

**WDP-KAFKA.md**

- **Section 3 / 4 — `internal-integration-events` consumer row for
  COMP-39:** Refine — the topic config key is `${spring.kafka.consumer.topic}`
  which resolves via `application.yml` to env var `${kafka_consumer_topic}`.
  Runtime value `internal-integration-events` was already confirmed
  2026-04-23. Confirm that v1.0's claim of pre-ACK (`acknowledgment.acknowledge()`
  as first call in `onMessage()`) and concurrency=1 (Spring Kafka default,
  `setConcurrency()` not invoked) are still accurate.

- **Section "auto-offset-reset":** add COMP-39 to the `latest` group.

- **No new producer entries.** COMP-39 is consumer-only — no
  `KafkaTemplate`, no `ProducerFactory`, no outbox table.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

- Add: *"COMP-39 hosts the manual NAP error reprocessing REST endpoint
  (`POST /event`, JWT-authenticated via `JwtIssuerAuthenticationManagerResolver`
  resolved from `${trusted_issuers}`). This resolves the long-standing
  question of which component hosts the manual reprocessing surface
  referenced in COMP-05's documentation — it is COMP-39, not COMP-05."*

- Add: *"COMP-39 is a fourth confirmed writer of
  `NAP.DISPUTE_EVENT_CONSUMER_ERROR` (alongside COMP-05 primary,
  COMP-23, COMP-24). Discriminator: `C_ACQ_PLATFORM` (mapped from
  `sourceSystemName`); COMP-39 sets `"NAP"` constant on insert.
  `C_EVENT_TYPE` values for COMP-39 rows: `OUT_SRV118`, `OUT_SRV117`."*

- Add: *"COMP-39 has `@EnableRetry` wired on the application class —
  Spring Retry is live (distinct from COMP-41 where the imports were
  dead)."*

- Add: *"COMP-39 confirms at-most-once pre-ACK as a second component
  with this pattern (alongside COMP-05). Both write to the same
  shared error table."*

- Add: *"COMP-39 audit user constant is `\"NCSEUPDTC\"` (corrected from
  v1.0 transcription `\"NCSEUDPTC\"`)."*

- Add: *"COMP-39 has no `application-{env}.yml` profiles in repo —
  every `${...}` placeholder resolves at runtime from environment
  variables only."*

- Resolved open questions: remove
  *"COMP-24 ActionEvent topic name (`${kafka.topic}`) and consumer
  identity"* row's "likely COMP-39 NAPOutcomeProcessor" hypothesis.
  COMP-39 source confirms a **single** `@KafkaListener` on
  `internal-integration-events` and no second listener — therefore
  COMP-39 is **not** the consumer of COMP-24's `${kafka.topic}`
  ActionEvent. The consumer of that topic remains unidentified.
  Reword the open question accordingly.

- New open questions to add to the Currently Open table:

  | Question | Source | Action needed |
  |----------|--------|---------------|
  | NAP-DPS authentication mechanism — `napcacrt.jks` is an unreferenced orphan in COMP-39 repo. Auth must be handled at Ingress / service mesh / network layer outside this component. Where? | COMP-39 OQ | Team confirmation — infrastructure team |
  | `NAP.NAP_UPDATE_RESPONSE_RULES` table owner | COMP-39 OQ | Team confirmation — no DDL in COMP-39 repo |
  | COMP-39 prior-error scan has no `C_EVENT_TYPE` filter — does it scoop up COMP-05/23/24-originated rows for the same case and reprocess them through SRV118 pipeline? Intentional or defect? | COMP-39 risk | Architect decision |
  | COMP-39 probe path mismatch — `resources.yaml` probes hit `/merchant/gcp/update-consumer/nap/livez` but actuator `additional-path: server:/livez` exposes endpoint at server root. Are probes resolving correctly via Ingress? | COMP-39 risk | Runtime observation (kubectl describe pod / probe failure logs) |
  | COMP-39 `POST /event` — is the controller-level exception handling enumerated for response codes? Body schema for success and error responses? | COMP-39 contract | Follow-up Copilot question (see Remaining Gaps) |
  | COMP-39 concurrency=1 — intentional or default-by-omission? No comment / commit message / property override documents intent | COMP-39 OQ | Architect decision |

**WDP-DECISIONS.md · Candidate new ADRs**

- **Candidate ADR — Cross-component shared-table consumption pattern
  on `NAP.DISPUTE_EVENT_CONSUMER_ERROR`:** 🔴 HIGH severity. Whether
  the shared error table is intended as a single platform-wide DLQ
  (in which case every co-writer's prior-error scan must coordinate
  on `C_EVENT_TYPE`), or whether COMP-39's lack of `C_EVENT_TYPE`
  filter is a defect, requires a formal decision. Same architectural
  class as DEC-019/DEC-020 (risk-accepted shared-state ADRs).

- **Candidate ADR — Manual operator reprocessing endpoint pattern:**
  🟡 MEDIUM. COMP-39 exposes `POST /event` for one-record reprocess,
  driving through the same business pipeline as the Kafka path with
  no per-record locking. Two simultaneous reprocesses for the same
  record (Kafka prior-error scan + operator POST) can race. Pattern
  used elsewhere on the platform? Should this be a uniform pattern
  with a coordination guarantee?

- Note: COMP-39 reuses the existing DEC-005 pre-ACK at-most-once
  pattern — no new ADR. It also reuses the platform-wide Spring Retry
  / no-Resilience4j pattern (DEC-014 voided) — no new ADR.

**WDP-ARCHITECTURE.md**

- No topology change. COMP-39 was already shown as the NAP outcome
  delivery consumer. Add a clarifying note that COMP-39 also exposes
  a manual operator REST endpoint (`POST /event`) on the WDP Ops
  Portal access path — this aligns the architecture diagram with
  the discovered REST surface.

**WDP-NFRS.md · Section 6 Risk Register**

- New RISK rows (candidate, to be reconciled in next NFR rebuild):

  - **RISK-COMP39-A:** Cross-component shared-table consumption
    hazard on `NAP.DISPUTE_EVENT_CONSUMER_ERROR` (no `C_EVENT_TYPE`
    filter on prior-error scan) — 🔴 HIGH
  - **RISK-COMP39-B:** NAP-DPS authentication relies on
    out-of-repo network/mesh trust; `napcacrt.jks` is an unreferenced
    orphan in this repo — 🔴 HIGH (cannot be remediated within
    component)
  - **RISK-COMP39-C:** RestTemplate without timeouts on a
    single-thread consumer + 8 outbound REST hops; one hung dep
    blocks the consumer — 🟡 MEDIUM
  - **RISK-COMP39-D:** Silent deserialization drop via no-op
    `CommonErrorHandler` — 🟡 MEDIUM
  - **RISK-COMP39-E:** Probe path mismatch (resources.yaml vs
    actuator additional-path) — 🟡 MEDIUM
  - **RISK-COMP39-F:** No CPU limits/requests — 🟡 MEDIUM
  - **RISK-COMP39-G:** No HPA, no PDB, single replica — 🟡 MEDIUM
  - **RISK-COMP39-H:** Two unused dependencies in pom.xml
    (`spring-boot-starter-cache`, `spring-boot-starter-oauth2-client`) — 🟢 LOW
  - **RISK-COMP39-I:** `dataRecord` always null in SRV118 (notesLookUp
    commented out); `checkCRMRAction()` commented out — 🟡 MEDIUM
    (functional gap rather than runtime risk)
  - **RISK-COMP39-J:** Manual REST reprocess + Kafka prior-error
    scan can race on the same record (no per-record lock) — 🟡 MEDIUM

**WDP-INTEGRATIONS.md**

- **NAP-DPS contract row:** Update to clarify that COMP-39's outbound
  HTTP calls set no Authorization header and no client cert is loaded
  in source. Authentication is handled outside this component — to be
  documented once team confirms the boundary (Ingress / mesh / network).

- **JustAI / Signifyd:** No change. COMP-39 has no third-party
  notification path.

**WDP-COMP-INDEX.md**

- COMP-39 entry — update Type to `Kafka Consumer + REST API` (was
  `Kafka Consumer`).
- COMP-39 entry — update Doc Status to `📝 DRAFT 🔍 v2.0 (2026-04-29)`.
- COMP-39 description — replace with:
  *"Consumes internal-integration-events and delivers dispute outcomes
  to NAP-DPS for NAP platform money movement. Filters to platform=NAP
  only — all other platform events silently discarded. Drives SRV118
  (chargeback outcome / representment) and/or SRV117 (department
  notice letter) outbound calls. Built-in prior-error reprocessing on
  every Kafka message. Also exposes a JWT-authenticated `POST /event`
  REST endpoint for manual operator reprocessing of a single error
  record. ⚠️ Pre-ACK (DEC-005 deviation, same as COMP-05). ⚠️
  notesLookup, checkCRMRAction commented out. ⚠️ napcacrt.jks
  unreferenced — NAP-DPS auth handled outside the component. Planned
  migration to EDIA route — no source scaffolding visible."*

#### Deviation flags for COMP-39

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ NOT APPLICABLE | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES | — |
| DEC-005 Manual Kafka Offset Commit (after processing) | ⛔ DEVIATES | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⛔ DEVIATES | 🔴 HIGH |
| DEC-014 Resilience4j (platform-VOID) | ✅ ABSENT (factual record only) | — |

**DEC-005 DEVIATES detail:** `acknowledgment.acknowledge()` is the first
call inside `KafkaConsumer.onMessage()`, before any downstream processing.
Identical pattern to COMP-05. Recovery is application-level only via
the shared error table — broker-level redelivery does not protect against
JVM crash post-commit.

**DEC-020 DEVIATES detail:** Pre-ACK pattern is structurally at-most-once.
There is no inbound idempotency-key check; if duplicate delivery did
occur (e.g. via a future at-least-once migration), NAP-DPS would be
re-called. The `idempotencyId` constructed for the SRV118 payload
(`caseNumber + actionSequences`) is for outbound consumption by NAP-DPS
only — not used for inbound dedup.

**DEC-004 / DEC-019 COMPLIES detail:** The `cardNumber` field in the
SRV117 outbound payload is constructed as `BIN6 + "******" + last4` —
masked PAN only. Inbound `NotificationEvent` schema has no PAN field.
Raw `NotificationEvent` JSON persisted to `C_KAFKA_EVENT` on error
contains no clear PAN. No encryption library imported.

#### Remaining gaps

| Gap | Classification | Detail |
|-----|----------------|--------|
| NAP-DPS authentication mechanism | **Team confirmation** (infrastructure / network team) | `napcacrt.jks` exists at repo root but is referenced by zero code paths — no `SSLContext`, `KeyStore`, `KeyManagerFactory`, `TrustManagerFactory`, `HttpClientBuilder`, `ClientHttpRequestFactory`, or `server.ssl.*` property loads it. Auth must therefore be handled outside this component. Confirm: Ingress mTLS? Service-mesh mutual TLS? Pure network-level trust? Cannot be answered by re-running Copilot on this repo. |
| `NAP.NAP_UPDATE_RESPONSE_RULES` table owner | **Team confirmation** | No DDL/migration in this repo. Confirm which component or team owns the schema and the data lifecycle (static reference data vs dynamically maintained). |
| Cross-component shared-table consumption hazard intent | **Architect decision** | Is COMP-39's lack of `C_EVENT_TYPE` filter on the prior-error scan intentional (single-pipeline recovery) or a defect (cross-pollination of error origin)? Decision should produce a formal ADR — same severity class as DEC-019/DEC-020. |
| Probe path mismatch (resources.yaml vs actuator) | **Runtime observation** | kubectl describe pod / readiness/liveness probe success counts in production. Cannot be answered by Copilot — requires live cluster inspection. |
| `POST /event` controller-level response contract (success body, error body, HTTP status mapping) | **Follow-up Copilot question** | *"In `controller/NapNotificationController.java`, enumerate the full request-body schema for `ConsumerErrorRequest`, the success response body, and any explicit `@ExceptionHandler` mappings. List each HTTP status code returned by the controller and the trigger condition for each."* |
| NAP-DPS SRV118/SRV117 response body schema | **Follow-up Copilot question** | *"In `NotificationServiceImpl.processSrv118Notification()` and `processSrv117Notification()`, what response DTO is returned by NAP-DPS for SRV118 and SRV117 calls? What fields are checked beyond the HTTP status code? Is the response body parsed at all, or is only 2xx vs non-2xx evaluated?"* |
| Concurrency = 1 intent | **Architect decision** | No comment / commit message / property documents the choice. Decide: explicitly raise to N>1, or codify =1 with operational rationale (and add per-record locking on the REST reprocess endpoint to prevent races with the Kafka prior-error scan). |
| EDIA migration scaffolding absence | **Architect decision** | No feature flag, no commented-out alternative route, no TODO references EDIA in source. Confirm timeline for adding migration scaffolding, or accept that migration will be a hard cut-over rather than a flag-gated rollout. |
| `migrationStatus` field intent | **Architect decision** | Field is read from action summaries and persisted on error rows but drives no branching logic. Is this passive observation only, or is conditional behaviour intended that has not been wired? |
| Production replica count | **Environment config / team confirmation** | XL Deploy variable `{{ replicas-gcp-nap-dispute-update-consumer }}` — runtime value not in repo. |
| Runtime values for all `${...}` env-injected properties | **Environment config / team confirmation** | No `application-{env}.yml` profiles in repo. Confirm topic, group ID, retry count, retry delay, max-poll values, all 8 URLs from XL Deploy / K8s secrets. |

#### Doc status after this change

- `WDP-COMP-39-NAP-OUTCOME-PROCESSOR.md` → `v2.0 DRAFT 🔍 source-verified
  2026-04-29` · architect confirmation pending
- `WDP-COMP-INDEX.md` → COMP-39 type, status, and description updates
  pending (next reconciliation session)
- `WDP-DB.md` → `NAP.DISPUTE_EVENT_CONSUMER_ERROR` writers list +
  shared-table risk register update pending; new
  `NAP.NAP_UPDATE_RESPONSE_RULES` row pending (next reconciliation
  session)
- `WDP-KAFKA.md` → topic-key-resolution clarification + auto-offset-reset
  group update pending (next reconciliation session)
- `WDP-NFRS.md` → 10 candidate RISK rows pending (next reconciliation
  session)
- `WDP-DECISIONS.md` → 2 candidate ADRs pending (cross-component
  shared-table pattern; manual reprocess endpoint pattern)
- `WDP-ARCHITECTURE.md` → minor REST surface annotation pending
- `WDP-INTEGRATIONS.md` → NAP-DPS auth boundary clarification pending
---

### 2026-04-29 — COMP-36 TokenService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `wdp-idp-token-service` — source-verified by GitHub Copilot CLI
2026-04-29. Architect confirmation pending.

**Nature of change:** Correction pass — confirms most v1.0 claims, corrects
four (D-10 Redis deserialisation mechanism, D-44 `minReadySeconds`
mis-indentation defect, D-53 ingress placeholder casing, R-3
`commons-lang3` usage), and surfaces three new findings (JVM/UTC timezone
mismatch on IDP-fetch path; probe initialDelay values now exact;
`{{ ingressTLSsecretName }}` mounted via `envFrom.secretRef` flagged as
likely manifest defect).

#### Platform-level impacts

**WDP-DB.md**

- **Section 2 — ElastiCache row enrichment.** The current row
  (`AWS ElastiCache | Redis | WDP Team | JWT token cache for TokenService
  | COMP-36 TokenService (owns)`) overstates ownership. Source-verification
  confirms COMP-36 performs **HGET only** — zero write operations exist in
  the repository. Rewrite the row as:

  > `AWS ElastiCache | Redis | WDP Team (writer unknown) | JWT token cache.
  > Redis hash key: wdpinternalidptoken, field: token. Value:
  > Jackson-serialised IdpToken (tokenValue, expiresIn, tokenType,
  > expirationDate). COMP-36 TokenService reads this hash (HGET only — no
  > writes anywhere in repo). Hash is populated by an UNKNOWN EXTERNAL
  > COMPONENT not present in the wdp-idp-token-service repository. ⚠ External
  > writer identity is an open question — see COMP-36 Planned Changes.
  > | COMP-36 TokenService (reads only — does NOT own writes) |`

- **No new shared-table risk** — ElastiCache remains used by exactly one
  WDP component on the read side. The risk is the inverse of the usual
  shared-write pattern: a single reader with no identifiable writer in
  any documented WDP component.

**WDP-KAFKA.md**

- **No change.** COMP-36 has no Kafka involvement — confirmed by absence
  of `spring-kafka` from `pom.xml` and full source scan. No additions to
  Sections 3 or 4.

#### Deviation flags for COMP-36

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ NOT APPLICABLE — no Kafka publish | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE — no Kafka | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES — no PAN data handled (full-codebase scan for `pan` / `cardNumber` / `accountNumber` / `acctNum` returned zero matches) | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE — no Kafka consumer | — |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform-wide VOID) — no Resilience4j on Redis or IDP. Pattern matches the platform-wide void posture already recorded for COMP-21, COMP-28, COMP-31, COMP-37, COMP-43. | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES — no persistent writes at all (no relational DB; Redis read-only) | — |
| DEC-020 Full At-Least-Once Idempotency | ✅ NOT APPLICABLE — read-only retrieval service; intrinsically idempotent | — |
| DEC-023 Replica = 1 Hard Constraint | ✅ NOT APPLICABLE — standard scaled stateless service | — |

**DEC-014 detail:** `io.github.resilience4j:*` absent from `pom.xml`. No
`@CircuitBreaker`, `@Bulkhead`, `@TimeLimiter`, `@Retry` anywhere in
source. Both Redis (Jedis) and IDP (`DefaultClientCredentialsTokenResponseClient`)
calls have zero resilience patterns. Spring Retry is also absent. This is
a clean addition to the DEC-014 evidence corpus.

#### Candidate ADRs surfaced (for next WDP-DECISIONS.md reconciliation)

1. **External Redis writer pattern** — formal decision needed on whether the
   "shared cache populated by an external job, read by a service" pattern
   is acceptable, and where the writer ownership lives. Either name the
   writer or remove the pattern.
2. **Inbound JWT validation on internal token-issuance endpoint** —
   architectural intent for `jwt.trustedIssuers` /
   `spring-security-oauth2-resource-server` (currently inert). Either
   wire it up or remove the dependency and the property.
3. **JVM timezone contract for services using `LocalDateTime` for time
   comparisons** — pattern: rely on container-image `TZ=UTC`? Or mandate
   `Instant`/`ZonedDateTime`? Cross-component review needed (COMP-36
   first instance flagged).
4. **`minReadySeconds` mis-indentation** — now confirmed on COMP-25,
   COMP-28, COMP-34, COMP-36. Promote to platform-wide manifest-lint
   recommendation (preflight CI check).

#### Remaining gaps

| Gap | Type | Action / exact follow-up question |
|-----|------|-----------------------------------|
| External Redis writer identity for `wdpinternalidptoken:token` | Team confirmation | Ask: *"Which component or process writes the Redis hash `wdpinternalidptoken:token` to ElastiCache? Is it a WDP component, an infrastructure bootstrap job, a Lambda, or something else? Is it running in production today, and what is its failure mode?"* |
| Caller inventory of consumers/batches that call `GET /merchant/gcp/idp-token/token` | Copilot CLI follow-up across consumer repos | For each WDP service repo (COMP-07, 08, 09, 11, 14, 15, 16, 17, 18, 19, 20, 21, 23, 24, 25, 26, 27, 31, 34, 37, 41, 42, 43, 51), ask Copilot: *"Does this component call `GET /merchant/gcp/idp-token/token` or use the host pattern `/merchant/gcp/idp-token`? If yes, at which step in the flow and under what conditions?"* Aggregate the answers into a caller list. |
| JVM timezone in production (kpack `jammy-base` builder) | Runtime observation / infra confirmation | Cannot be answered by Copilot CLI re-runs. Inspect a running pod's `date` / `/etc/timezone` / JVM `TimeZone.getDefault()` log line. Or confirm the kpack base image sets `TZ=UTC` via DevOps. **Critical for correctness of expiry comparison.** |
| Production replica count for `wdp-idp-token-service` | Environment config | Confirm via XL Deploy / Helm values for each environment — placeholder `{{ replicas-wdp-idp-token-service }}` has no default in repo. |
| Production IDP URL, Redis endpoint, SSL flag | Environment config | Values come from K8s secrets `wdp-token-service-secrets` and `wdp-common-secrets`. Confirm with DevOps per environment. |
| `${app.name}` resolution mechanism | Environment config | Used in `management.metrics.tags.application` but not defined in `application.yml`. Confirm whether `APP_NAME` env var is set externally and where. |
| `{{ ingressTLSsecretName }}` mounted via `envFrom.secretRef` | Architect / DevOps decision | Manifest defect (TLS material as env vars is unusual) or intentional? If defect, remediate at next manifest revision. |
| `minReadySeconds: 30` mis-indentation defect | Architect decision | Lift to `spec.minReadySeconds` (apply intended 30-s gate) or remove the field and accept current behaviour? Pattern recurs across COMP-25, COMP-28, COMP-34, COMP-36 — candidate for platform-wide remediation sprint. |
| Inbound JWT validation intent (`jwt.trustedIssuers` orphan + `spring-security-oauth2-resource-server` unused) | Architect decision | Either wire up resource-server JWT validation on this endpoint (the same trusted-issuer set every other WDP service uses) or remove the dependency and the property. Currently the token-issuance service is the *only* unauthenticated endpoint in the platform — security inversion. |
| Latent NPE on null `authorizedClient` in `TokenServiceImpl.getAuthorizationTokenFromAuthServer()` | Architect decision (defect or design?) | Add the guard return, or rely on the NPE-as-HTTP-500 pathway and document explicitly? |

#### Doc status after this change

- `WDP-COMP-36-TOKEN-SERVICE.md` → `v1.1 DRAFT` — source-verified
  2026-04-29 · architect confirmation pending
- `WDP-COMP-INDEX.md` → no row addition; status text update at next
  reconciliation: COMP-36 row to read *"📝 DRAFT 🔍 v1.1 (2026-04-29)"*
  matching the COMP-37 pattern
- `WDP-DB.md` → ElastiCache row rewrite pending (next reconciliation
  session)
- `WDP-KAFKA.md` → no change
- `WDP-NFRS.md` → 4 new candidate RISK rows pending (next reconciliation
  session): timezone-mismatch on IDP-fetch path; `minReadySeconds`
  silently-ignored defect (extends the COMP-25/28/34 family);
  `{{ ingressTLSsecretName }}` envFrom anomaly; orphan
  `jwt.trustedIssuers` + inert resource-server dependency on the
  unauthenticated token-issuance endpoint
- `WDP-DECISIONS.md` → 4 candidate ADRs surfaced (see above) pending
  architect promotion at next reconciliation
- `WDP-INTEGRATIONS.md` → no change (no new external integration
  contract; IDP and ElastiCache already documented platform-wide)
- `WDP-ARCHITECTURE.md` → no change (topology unchanged; no new
  component, no new flow)
---

### 2026-04-29 — COMP-35 EncryptionService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `wdp-encryption-service` — source-verified by Copilot CLI 2026-04-29.
Architect confirmation pending.

**Nature of change:** Correction pass. v1.0 DRAFT confirmed in substance;
v1.1 absorbs five material corrections and adds two new findings.

#### Platform-level impacts

**WDP-DB.md**
- **NEW Section-2 row** — `wdp.hash_key_store` · primary writer COMP-35
  EncryptionService (INSERT on encrypt-miss; UPDATE `last_seen_at` on
  encrypt-hit and on every decrypt). Key cols: `id` (PK seq), `hpan`
  (64-char hex), `pan_ct` (Base64 IV‖ciphertext‖GCM-tag), `dek_id` (FK),
  `inserted_at`, `last_seen_at`. ⚠️ No JPA `unique=true` on `hpan`.
  DB-level UNIQUE existence not determinable from repo (no migration
  scripts present). No external readers — ciphertext never leaves this
  service. Transaction manager: `wdpTransactionManager`.
- **NEW Section-2 row** — `wdp.data_enc_key` · primary writer COMP-35
  EncryptionService (INSERT on DEK rotation; SELECT-by-id on decrypt;
  SELECT-top-by-inserted_at-desc on rotation). Key cols: `dek_id` (PK
  seq `wdp.hash_key_store_dek_id_seq`), `dek_enc` (Base64 KMS-wrapped
  DEK), `inserted_at`. ⚠️ `insertEncryptedDEK` runs in a Spring-Data
  default short transaction — separate from the surrounding
  `rotateDEK()` orchestration. Multi-pod DEK row proliferation is
  uncontrolled (no distributed lock). No external readers.
- **No shared-table risk introduced** — both tables are owned and
  read exclusively by COMP-35.

**WDP-KAFKA.md**
- No change. COMP-35 has no Kafka involvement (no listener, no
  template, no client dependency). It remains correctly absent from
  Sections 3 and 4. Confirms Kafka-Free classification.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: COMP-35 owns `wdp.hash_key_store` and `wdp.data_enc_key` —
  sole writer and sole reader of both. No shared-table writes.
- Add: COMP-35 DEK rotation interval is **days**, configured via
  `${dek_rotation_interval_days}`. Prior platform docs stating
  "6-hour DEK cache" are incorrect.
- Add: COMP-35 decrypt `@Transactional` boundary brackets the KMS
  network call. A Hikari connection is held during the KMS round-trip
  on every decrypt request.
- Add: COMP-35 `/actuator/prometheus` and `/actuator/info` are NOT
  in the security permit-all list; they require JWT.
- Add: COMP-35 `rotateDEK()` reuse-existing-DEK branch carries a
  Base64 bug — `dek_enc` is passed to KMS as raw UTF-8 bytes of the
  Base64 string instead of decoded ciphertext. Symptom only manifests
  on pod restart that finds a recent-enough DEK row to reuse.
  Failure is silently swallowed; pod starts with `initialized=false`.
- Add: COMP-35 has no method-level authorization on the decrypt
  endpoint. Decrypt-caller policy (COMP-14 only) is convention,
  not technical control.
- Add: COMP-35 `PANHashingServiceImpl` logs last-4 PAN at DEBUG.
  Production log level governance is operationally critical.
- Add: COMP-35 Spring Boot version is 3.5.6 (precision).
- Resolved open questions: none currently in the COMP-35 row of the
  open-question table — all gaps recorded by v1.0 are now addressed
  or restated as DBA / ops confirmations below.
- New open questions:
  - DB-level UNIQUE on `wdp.hash_key_store.hpan` — DBA confirmation
    (no migration scripts in repo).
  - Additional indexes / constraints on `wdp.data_enc_key` — DBA
    confirmation.
  - Production replica count for COMP-35 — XL Deploy / ops.
  - Production value of `${dek_rotation_interval_days}` — ops.
  - Production injected value of `${jwt_trusted_issuer_urls}` — ops.
  - Prometheus scrape configuration for COMP-35 — does the scrape
    job carry a JWT? If not, scrape is currently failing or the
    whitelist needs updating.
  - Log-pipeline PAN-scrubbing posture (Logstash target side) —
    not verifiable from this repo.

**WDP-DECISIONS.md · Candidate new ADRs**
- **ADR-CAND-035-A · Decrypt @Transactional spanning external KMS call**
  Severity: 🟡 MEDIUM. The current method-level transaction holds a
  Hikari connection for the duration of the KMS round-trip on every
  decrypt request. Candidate decision: narrow the transaction to the
  `last_seen_at` UPDATE only, or accept the pool-exhaustion risk
  formally with capacity guidance.
- **ADR-CAND-035-B · Multi-pod DEK rotation race accepted**
  Severity: 🟡 MEDIUM. Source code carries an explicit comment
  acknowledging that multiple pods can reach the DEK-generate path
  concurrently. Candidate decision: formally accept the multi-DEK
  posture (correct on decrypt, uncontrolled row proliferation) or
  add a distributed lock (advisory lock / ShedLock).
- **DEC-008 narrative correction** (not a new ADR — content fix)
  DEC-008 still describes a 6-hour DEK cache. Source confirms the
  unit is days. DEC-008 narrative, COMP-INDEX entry for COMP-35, and
  any WDP-ARCHITECTURE.md reference to "6-hour DEK cache" must be
  corrected at the next platform-level reconciliation.

**WDP-ARCHITECTURE.md**
- DEC-008 / COMP-35 narrative: replace "6-hour DEK cache" with
  "DEK rotation interval configured in days
  (`${dek_rotation_interval_days}`)". No topology change.

**WDP-NFRS.md · Section 6 Risk Register**
- **NEW** RISK-COMP35-A — Single global dependency for all PAN
  ingestion paths (COMP-07/08/09/11). 🔴 HIGH availability impact.
- **NEW** RISK-COMP35-B — `rotateDEK()` reuse-path Base64 bug.
  🔴 HIGH for pod-restart hygiene during rotation window.
- **NEW** RISK-COMP35-C — Decrypt `@Transactional` holds DB
  connection during KMS call. 🟡 MEDIUM pool-exhaustion risk under
  KMS slowdown plus sustained decrypt load.
- **NEW** RISK-COMP35-D — Multi-pod DEK rotation race
  (no distributed lock). 🟡 MEDIUM.
- **NEW** RISK-COMP35-E — No DB-level UNIQUE on
  `wdp.hash_key_store.hpan` verifiable from repo. 🟡 MEDIUM
  duplicate-INSERT race across pods.
- **NEW** RISK-COMP35-F — Crypto failures mapped to HTTP 404.
  🟡 MEDIUM caller-error-handling correctness.
- **NEW** RISK-COMP35-G — DEK startup failure silently swallowed;
  probes do not check `initialized` flag. 🟡 MEDIUM.
- **NEW** RISK-COMP35-H — DEK rotation failure silently swallowed;
  service runs with expired DEK without alert. 🟡 MEDIUM.
- **NEW** RISK-COMP35-I — `PANHashingServiceImpl` logs last-4 PAN
  at DEBUG. 🟢 LOW (operational governance).
- **NEW** RISK-COMP35-J — `/actuator/prometheus` requires JWT;
  scrape posture must be confirmed. 🟢 LOW (observability gap if
  scrape is unauthenticated).

**WDP-INTEGRATIONS.md**
- §6.2 AWS KMS — no contract change. v1.1 source-verifies the
  existing entry. **Caller list update** for the "Key callers of
  COMP-35" sub-bullet: confirm COMP-09 CaseFillingBatch and
  COMP-27 CaseSearchService (v2 search path) are present alongside
  COMP-07 / COMP-08 / COMP-11 / COMP-43.
- **NEW §6 row candidate** — AWS Secrets Manager. Currently not
  documented as a distinct §6 entry. WDP integration point: COMP-35
  EncryptionService (sole caller, startup-only). Protocol: AWS SDK v2.
  Auth: IAM role. Status: ✅ Production. Recommend adding alongside
  AWS KMS §6.2.

#### Deviation flags for COMP-35

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ NOT APPLICABLE | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES (caveat) | 🟢 LOW |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-008 AWS KMS — design narrative | ⚠️ DOC CORRECTION | 🟡 MEDIUM |
| REST convention — crypto failure → 404 | ⛔ DEVIATES | 🟡 MEDIUM |
| DEC-014 Resilience4j (platform-VOID) | ✅ ABSENT (factual record only) | — |

**DEC-004 caveat detail:** Service IS the DEC-004 enforcement boundary
— no clear PAN persisted, `@ToString(exclude='pan')` on both request
and response, log statements reference only HPAN. ⚠️ Caveat:
`PANHashingServiceImpl` writes the last 4 PAN digits to log at DEBUG.
Production log level must remain INFO or higher; DEBUG would expose
partial PAN. Operational governance, not architectural defect.

**DEC-020 PARTIAL detail:** Encrypt path does an in-memory
`findByHpan` idempotency check inside a single `@Transactional` on
`encryptPAN()`, which protects against intra-pod duplicate INSERTs.
However the JPA entity declares no `unique=true` on `hpan`, and no DB
migration scripts are in this repository — DB-level UNIQUE existence
is not verifiable. Two pods racing on the same PAN can both pass the
SELECT and both INSERT. Subsequent `findByHpan` returns one of the
duplicates non-deterministically. Decrypt is correct in this scenario
because each row carries its own `dek_id`, so duplicates decrypt to
the same plaintext PAN — but row proliferation is uncontrolled.

**DEC-008 DOC CORRECTION detail:** DEC-008 narrative and the COMP-35
description in WDP-COMP-INDEX.md state a "6-hour DEK cache". Source
confirms the rotation interval unit is **days**, configured via
`${dek_rotation_interval_days}`. No hardcoded default exists in any
committed configuration profile. This is a content fix, not a
runtime risk.

**REST convention DEVIATES detail:** `GlobalExceptionHandler` maps
both `PANEncryptionException` and `PANDecryptionException` to
HTTP 404. On decrypt, four distinct conditions collapse to the same
status code: HPAN not found in DB, DEK not found in DB, KMS unwrap
failure, AES decrypt failure. Callers cannot distinguish via HTTP
status alone. Operationally, alerting on "decrypt 404 rate" cannot
separate "expected HPAN-miss from upstream typo" from "KMS dependency
slipped".

#### Doc status after this change

- `WDP-COMP-35-ENCRYPTION-SERVICE.md` → `v1.1 DRAFT` —
  source-verified 2026-04-29 · architect confirmation pending.
- `WDP-COMP-INDEX.md` → COMP-35 description needs "6-hour DEK cache"
  removed (next reconciliation session).
- `WDP-DB.md` → two new Section-2 rows pending (next reconciliation
  session).
- `WDP-KAFKA.md` → no change.
- `WDP-DECISIONS.md` → DEC-008 narrative correction, two
  ADR-CAND-035-* candidates pending (next reconciliation session).
- `WDP-NFRS.md` → ten new RISK-COMP35-* rows pending
  (next reconciliation session).
- `WDP-INTEGRATIONS.md` → §6.2 caller list refresh + new §6 row for
  AWS Secrets Manager pending (next reconciliation session).
- `WDP-ARCHITECTURE.md` → DEC-008 / COMP-35 narrative correction
  pending (next reconciliation session).
---

### 2026-04-28 — COMP-32 RulesService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `gcp-rules-service` — source-verified by GitHub Copilot CLI 2026-04-28. Architect confirmation pending.

**Nature of change:** Correction pass against source. The v1.0 DRAFT was largely accurate on the architectural core — endpoint inventory, backing tables, no-Kafka / no-PAN / no-writes posture, cache mechanics, and DEC deviation map. Source audit corrected a cluster of metadata errors (artifact version, security whitelist paths, K8s probe paths, liveness initialDelay, json-path version, `/actionrules` HTTP status set), surfaced one MEDIUM defect in JWT-claim-absence handling, and flagged three new MEDIUM-class concerns (cache-key bloat from null-vs-blank inputs, the empty-`AuthorizationList`-claim silent-no-permission path, and the coincidental rather than designed pattern of three endpoints returning HTTP 200 instead of 404 on no-rule-found). DEC-014 deviation re-confirmed; DEC-023 added as NOT APPLICABLE for completeness.

#### Platform-level impacts

**WDP-DB.md**

- Section 2 — 16 reader rows for COMP-32 (15 in `wdp` schema + 1 in `nap` schema): **No ownership change.** Reinforce: COMP-32 is a runtime READ-ONLY consumer of every row; writes are by DBA scripts / rule-administration tooling — write owner remains TBC for all 16 tables. Existing reader attribution is unchanged. List of reader rows (already present per existing reader-row addition in v1.0):
  - `wdp.dispute_action_rules`, `wdp.business_event_rules`, `wdp.dispute_accept_item_rules`, `wdp.dispute_new_action_rules`, `wdp.dispute_new_case_action_respond_rules`, `wdp.dispute_first_action_rules`, `wdp.dispute_case_exp_rules`, `wdp.pre_action_status_rules`, `wdp.disputes_visa_queue_status_rules`, `wdp.notification_rules`, `wdp.nap_chbk_response_rules`, `wdp.workflow_rules`, `wdp.dispute_doc_details_type_rules`, `wdp.document_type_rules` (disabled endpoint, table still queryable), `wdp.dispute_update_event_log_rules` (disabled endpoint, table still queryable), `nap.dispute_nap_new_case_action_rules`.
- Datasource bean-name confirmation: primary `wdpdataSource` (single word, lowercase d) with `@Primary`; secondary `napdataSource`. Add a Notes clarification on the primary-table cluster row that the JDBC URLs are externalised via K8s secrets — whether `wdp` and `nap` resolve to the same Aurora cluster is **not determinable from source**.
- No new shared-table risk introduced. No cross-component co-writer concerns.

**WDP-KAFKA.md**

- No change. COMP-32 remains in Section 4's Kafka-free components list. Audit re-confirms zero `spring-kafka` / `kafka-clients` / `aws-msk-iam-auth` in `pom.xml`, zero `@KafkaListener` / `KafkaTemplate` / `ProducerFactory` references in source.

**WDP-COMP-INDEX.md**

- Doc status row update: COMP-32 → `📝 DRAFT v1.1 — source-verified 2026-04-28, architect confirmation pending`.
- Description line is accurate; no correction required.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

- Add: COMP-32 RulesService is a pure read-only REST API. Zero writes, zero Kafka, zero PAN, zero outbound REST/HTTP, zero `@Transactional` mutations. Sole outbound dependency is two PostgreSQL datasources (`wdpdataSource` `@Primary` + `napdataSource`), both via `DataSourceBuilder.create().build()` with no explicit pool tuning.
- Add: COMP-32 hosts 14 active REST endpoints + 2 controller-disabled endpoints with full backing code intact. The 14 active endpoints map 1:1 to 14 distinct `@Cacheable` cache names backed by Spring's default `ConcurrentMapCacheManager`.
- Add: COMP-32 `/actionrules` is the sole endpoint that performs caller-identity-aware response shaping, via JWT `AuthorizationList` claim ∩ `app.action.roles` config list. All other 13 active endpoints return identical content for identical inputs regardless of caller.
- Add: COMP-32 `/actionrules` `migrationStatus="N"` is an active production migration kill-switch — DB bypassed, hardcoded all-N response. Confirmed via `ApplicationConstants.NO = "N"`. Constant `MIGRATION_STATUS = "Y"` is misleadingly named (holds the non-bypass sentinel).
- Add: COMP-32 K8s probes are Spring Boot Actuator health groups exposed via `additional-path` — `/livez` (liveness, initialDelay 40s) and `/readyz` (readiness, initialDelay 30s) under the servlet context `/merchant/gcp/rules`. No custom controller method backs these paths.
- Resolved open questions: none (all open questions on COMP-32 in v1.0 DRAFT remain open — they are cross-component sweep questions, not in-repo gaps).
- New open questions:
  - **OQ-COMP-32-NEW-1:** `/actionrules` `AuthorizationList` claim absence (claims map non-null, key missing) returns HTTP 500 via uncaught NPE in `split`, not HTTP 400 with "Token is blank". Defect or accepted?
  - **OQ-COMP-32-NEW-2:** `/actionrules` empty-string `AuthorizationList` claim splits to `[""]`, intersection yields empty set, DB query runs with empty IN clause, user sees no permissions silently. Defect or accepted?
  - **OQ-COMP-32-NEW-3:** Three endpoints (`/notification`, `/business-event`, `/cbk-response`) return three different HTTP 200 shapes on no-rule-found while the other 11 throw 404. Source contains zero comment evidence of intent. Architect decision: align all to 404 (defect remediation) or document each as intentional API contract?
  - **OQ-COMP-32-NEW-4:** `@Cacheable` key is computed from raw method args before in-method blank→"NA" normalisation, producing duplicate cache entries for null vs `""` inputs. Memory bloat / hit-rate halving. Architect decision: explicit `key` SpEL with normalisation, or accept?
  - **OQ-COMP-32-NEW-5:** `/documentType` and `/eventRule` controller mappings commented out with full backing code intact and no comment / JIRA / feature flag explaining intent. Permanently abandon (delete code + tables) or re-enable?

**WDP-DECISIONS.md · Candidate new ADRs**

- **DEC-014 deviation map for COMP-32:** Re-confirm — `io.github.resilience4j` absent from `pom.xml`; no circuit breaker / retry / bulkhead / rate limiter on either datasource; no socket or connection timeout configured. Consistent with platform-wide DEC-014 VOID posture (now confirmed on COMP-22, COMP-25, COMP-28, COMP-31, COMP-37, COMP-32 — read-only REST services consistently exhibit the deviation).
- **Candidate clarification — DEC-004 / DEC-019 attestation pattern:** COMP-32 is a clean attestation that DEC-004 / DEC-019 compliance can be confirmed at architecture level by codebase scan for entity classes containing `pan` / `cardNumber` / `accountNumber` / `acctNum` fields plus absence of any persistent write. Same pattern as COMP-28. Worth documenting as a verifiable read-only-service compliance pattern when WDP-DECISIONS.md is rebuilt.
- **Candidate new ADR — "Cache-key normalisation contract":** COMP-32 `@Cacheable` methods normalise blank → `"NA"` *inside* the method, but Spring computes the key *before* method entry. Result: two requests differing only in null vs `""` cache separately despite producing identical responses. Same anti-pattern likely exists in other read-only WDP services that follow this normalisation convention. Recommend a platform-wide cache-key contract: normalisation must occur in the controller or via explicit `key` SpEL, never inside the cached method body.
- **Candidate new ADR — "Endpoint-pair contract divergence on no-rule-found":** COMP-32 has 14 active endpoints; 11 throw `RuleNotFoundException` → HTTP 404; 3 (`/notification`, `/business-event`, `/cbk-response`) return three different HTTP 200 shapes. No source-side comment establishes intent. Pair-class with the COMP-28 permission-shape divergence flagged 2026-04-28 — both are evidence that the platform lacks a documented "what does no-result-found return on a read-only configuration endpoint" convention. Recommend a formal platform read-only-API contract.
- **Candidate new ADR — "JWT claim-absence error contract":** The COMP-32 `/actionrules` `AuthorizationList`-claim-absent → HTTP 500 path (vs the documented HTTP 400 "Token is blank") is a defect in classification. Worth documenting platform-wide that any auth-claim absence must produce a 4xx response, not a 5xx, for security-monitoring-rule integrity.

**WDP-ARCHITECTURE.md**

- No change. Topology and principles unchanged. COMP-32 remains a WDP Core Service in Section 4. The pattern observations (cache-key normalisation, no-rule-found contract divergence) are component-level rather than topology-level — they belong in WDP-DECISIONS.md.

**WDP-NFRS.md · Section 6 Risk Register** *(candidate new RISK rows)*

- **🟠 HIGH (re-confirmed):** *"COMP-32 — DB outage takes down all 14 endpoints simultaneously. No circuit breaker, no socket timeout, no retry, no fallback. All callers fail at the same instant. Same DEC-014-VOID pattern as COMP-22 / COMP-25 / COMP-28 / COMP-31 / COMP-37."*
- **🟠 HIGH (re-confirmed):** *"COMP-32 — `migrationStatus="N"` is an undocumented production migration kill-switch on `/actionrules`. When triggered, every UI action is blocked for the affected case. Not in any runbook."*
- **🟡 MEDIUM candidate (NEW):** *"COMP-32 — `/actionrules` `AuthorizationList` claim absence (claims map non-null, key missing) returns HTTP 500 via uncaught `NullPointerException` in `String.split`, not HTTP 400 'Token is blank' as documented. Defeats security-monitoring rules keyed on auth-related response codes. Trivial fix: add an explicit null check before split."*
- **🟡 MEDIUM candidate (NEW):** *"COMP-32 — `/actionrules` empty-string `AuthorizationList` claim silently produces an empty role filter, which is passed as an empty IN clause to the DB. User receives no permissions despite a valid token; no error logged."*
- **🟡 MEDIUM candidate (NEW):** *"COMP-32 — three endpoints (`/notification`, `/business-event`, `/cbk-response`) fall through to HTTP 200 on no-rule-found with three different response shapes (null body, empty fields, empty list) while the other 11 throw HTTP 404. Source contains no intent evidence. Callers must handle four distinct success-shapes per service. Pattern appears coincidental, not deliberate."*
- **🟡 MEDIUM candidate (NEW):** *"COMP-32 — `@Cacheable` key is computed from raw method args before in-method blank→`NA` normalisation, producing duplicate cache entries for null vs `\"\"` inputs that resolve to identical DB queries. Inflates memory; halves the effective hit rate for fields routinely sent both ways by different callers."*
- **🟡 MEDIUM candidate (extension of existing pattern):** *"COMP-32 — no CPU limit, no CPU request set in `resources.yaml`. Same class as COMP-28 / others. Recommend platform-wide manifest sweep."* Consistent with prior pattern findings on the platform.
- **🟡 MEDIUM candidate:** *"COMP-32 — `/documentType` and `/eventRule` controller mappings commented out with full backing code intact (service, DAO, entity, repository) and zero source-side comment, JIRA, or feature flag explaining intent. Risk: rules tables `wdp.document_type_rules` and `wdp.dispute_update_event_log_rules` continue to be reachable via DB access; backing code rots; intent lost on team rotation."*
- **🟢 LOW candidate (extension of existing topology-spread risk):** *"COMP-32 — topology spread `whenUnsatisfiable: ScheduleAnyway` plus `BRANCH_NAME_PLACEHOLDER` label-mismatch risk. No hard cross-AZ guarantee."* Extension of the existing platform-wide topology-spread RISK row.
- **🟢 LOW candidate:** *"COMP-32 — no PodDisruptionBudget. Rolling updates and node maintenance can terminate all replicas simultaneously."* Same class as several other components.
- **🟢 LOW candidate (hygiene cluster):** dead code in `GlobalExceptionHandler` (unused `errorCode` variables, two unused private helper methods); unused `<json-path.version>2.9.0</json-path.version>` property; unused `ApplicationProps.version` map; commented-out `partialIndicator` block in `NewActionRulesServiceImpl` alongside `releaseVersionMap`; commented-out alternative Logstash destination; stale Eclipse-generated TODO comments. No behavioural impact; trivial PRs.

**WDP-INTEGRATIONS.md**

- No change. COMP-32 has no external integration contract — all outbound calls are PostgreSQL JDBC to the WDP-internal Aurora cluster(s). The WDP-shared external IdP for JWKS validation is already documented platform-wide. Logstash is internal observability infrastructure.

#### Deviation flags for COMP-32

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ NOT APPLICABLE (no Kafka publish, no writes) | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE (no Kafka) | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES (no PAN handled — full-codebase scan for `pan` / `cardNumber` / `accountNumber` / `acctNum` returns zero matches; no persistent writes) | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE (no Kafka consumer) | — |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform-wide VOID posture — confirmed) | 🟡 MEDIUM (accepted) |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES (no persistent writes at all) | — |
| DEC-020 Full At-Least-Once Idempotency | ✅ NOT APPLICABLE (read-only service, idempotency intrinsic) | — |
| DEC-023 Replica = 1 Hard Constraint | ✅ NOT APPLICABLE (standard scaled stateless read service) | — |

**DEC-014 detail:** Confirmed pattern matches the platform-wide DEC-014 VOID posture — no Resilience4j (`io.github.resilience4j` absent from `pom.xml`), no socket or connection timeout on either datasource, no retry. DB unavailability propagates as HTTP 500 to every caller simultaneously. Severity remains MEDIUM per platform-wide acceptance.

**DEC-004 / DEC-019 detail:** Source-verified by full-codebase scan for `pan` / `cardNumber` / `accountNumber` / `acctNum` fields on entity classes — none found. No persistent writes occur at runtime. Same clean-attestation pattern as COMP-28.

#### Remaining gaps

| Gap | Type | Action |
|-----|------|--------|
| Production replica count (`{{ replicas-gcp-rules-service }}`) | Environment config | Confirm via XL Deploy / Kubernetes for each environment. |
| Whether `wdp` and `nap` JDBC URLs resolve to the same Aurora cluster | Environment config / DBA confirmation | Externalised via K8s secrets; not in repo. |
| Actual `${jwt.trustedIssuers}` allowlist values per environment | Environment config | Confirm IdP host(s) per environment from K8s secrets. |
| Actual `${LOGSTASH_SERVER_HOST_PORT}` values | Environment config | Confirm per environment from K8s secrets. |
| HikariCP pool sizing on both datasources (running with Spring Boot defaults) | **Architect decision** | No explicit pool config in repo. With 14 endpoints all hitting one of two datasources, the Spring Boot default pool (10) may be a production bottleneck. Confirm or remediate. |
| Caller identity per endpoint | **Follow-up Copilot CLI question on consumer repos** — exact question to ask each candidate caller: *"Does this repository call any endpoint matching `/merchant/gcp/rules/*`? If so, cite file:line of every call site, the HTTP client used, the request payload shape, and any caching of the response."* — to be run against COMP-04, COMP-05, COMP-06, COMP-12, COMP-14, COMP-15, COMP-16, COMP-17, COMP-18, COMP-19, COMP-20, COMP-21, COMP-23, COMP-24, COMP-49 (Merchant Portal), COMP-50 (Ops Portal). |
| DB-bypass pattern — do other components read these 16 rule tables directly from the DB? | **Follow-up Copilot CLI question on the same caller list** — exact question: *"Does this component read any of these tables directly from the database: `wdp.dispute_action_rules`, `wdp.business_event_rules`, `wdp.dispute_accept_item_rules`, `wdp.dispute_new_action_rules`, `wdp.dispute_new_case_action_respond_rules`, `wdp.dispute_first_action_rules`, `wdp.dispute_case_exp_rules`, `wdp.pre_action_status_rules`, `wdp.disputes_visa_queue_status_rules`, `wdp.notification_rules`, `wdp.nap_chbk_response_rules`, `wdp.workflow_rules`, `wdp.dispute_doc_details_type_rules`, `wdp.document_type_rules`, `wdp.dispute_update_event_log_rules`, `nap.dispute_nap_new_case_action_rules`?"* The COMP-16 → BusinessRulesService precedent makes this a real risk. |
| Write owner of all 16 rule tables | **Team confirmation** | Whether rows are managed by a rule-administration UI (which service?), DBA scripts, Liquibase / Flyway migrations, or another component. Not answerable from this repo. |
| `/notification`, `/business-event`, `/cbk-response` non-404 fall-through | **Architect decision** | Align all to 404 (defect remediation) or document each as intentional API contract. |
| `/actionrules` `AuthorizationList`-claim-absent NPE → HTTP 500 | **Architect decision** | Trivial fix; affects security observability. |
| `/actionrules` empty-`AuthorizationList`-claim silent-no-permission | **Architect decision** | Defect or accepted? |
| Cache-key normalisation contract (null vs blank) | **Architect decision** | Component-level fix or platform-wide convention. |
| `/documentType` and `/eventRule` re-enable / decommission decision | **Architect decision** | Permanently abandon (remove code + tables) or re-enable? |
| `migrationStatus="N"` runbook documentation | **Team confirmation** | Add to ops runbook; rename `MIGRATION_STATUS` constant to remove the "Y"-as-default-value confusion. |
| Whether IdP JWKS unreachability has caused production failures | **Runtime observation** | Cannot be answered from source. Splunk / Kibana log analysis. |
| `WDP-COMP-INDEX.md` description correction | Documentation | Existing description is accurate per source — no change required. |

#### Doc status after this change

- `WDP-COMP-32-RULES-SERVICE.md` → `v1.1 DRAFT` — source-verified 2026-04-28 · architect confirmation pending
- `WDP-COMP-INDEX.md` → COMP-32 status row update pending (next reconciliation session)
- `WDP-DB.md` → existing 16 reader rows for COMP-32 confirmed accurate; minor Notes clarification on cluster-resolution pending (next reconciliation session)
- `WDP-KAFKA.md` → No change
- `WDP-NFRS.md` → 4 new MEDIUM-class candidate RISK rows + 1 MEDIUM candidate (CPU limits, extension) + 2 LOW candidates pending (next reconciliation session)
- `WDP-DECISIONS.md` → 4 candidate new ADRs / clarifications pending architect decision (cache-key normalisation contract, no-rule-found platform contract, JWT claim-absence error contract, read-only-service DEC-004/019 attestation pattern)
- `WDP-INTEGRATIONS.md` → No change
- `WDP-ARCHITECTURE.md` → No change
- `WDP-HANDOVER.md` → 5 new open questions to add (OQ-COMP-32-NEW-1 through -5); 5 new confirmed facts to add
---
### 2026-04-28 — COMP-30 UserQueueSkillService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `gcp-user-queue-skill-service` — source-verified by GitHub Copilot CLI 2026-04-28. Architect confirmation pending.

**Nature of change:** Correction pass — confirms v1.0 structure and surfaces ten corrections, including two new 🔴 HIGH atomicity findings and one 🟡 MEDIUM observability gap that materially change the component's risk posture.

#### Platform-level impacts

**WDP-DB.md**
- No new tables — full owned-table list was already added in v1.0 reconciliation. Refinements:
  - `nap.users` row: append note "POST /user has no @Transactional — auto-enrol side effect runs in a separate implicit transaction; failure leaves orphaned user row".
  - `nap.queues`, `nap.queue_criterion`, `nap.user_queue` rows: append note "Service-level @Transactional on createQueue/updateQueue binds to @Primary usTransactionManager — UK writes NOT covered by the outer TX. Per-repository implicit TX only".
  - `wdp.queues`, `wdp.queue_criterion`, `wdp.user_queue` rows: append note "Service-level @Transactional binds correctly here (US datasource is @Primary)".
  - Tables-Read rows: append confirmation `nap.nap_parent_entity` / `wdp.case` / `wdp.action` are **READ-ONLY in COMP-30 source** — owners still TBC.

**WDP-KAFKA.md**
- No change. COMP-30 remains in the Kafka-Free list. Confirmed unchanged: zero `spring-kafka`, `KafkaTemplate`, `@KafkaListener`, `@EnableKafka` occurrences in repo.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: COMP-30 v1.1 DRAFT — source-verified 2026-04-28; architect confirmation pending.
- Add: COMP-30 confirmed dual-datasource design with `ukTransactionManager` (UK / nap) and `usTransactionManager` (US / wdp, `@Primary`). No XA.
- Add: COMP-30 confirmed Kafka-free at runtime — zero spring-kafka surface area.
- Add: Maven artifactId is `user-queue-skill-service` (the `gcp-` prefix appears only in K8s deployment naming).
- Add: Auto-enrol on POST /user targets queues of type `SKILLS_INTERNAL` **OR** `DEFAULT_QUEUE` (v1.0 said SKILLS_INTERNAL only).
- Resolved open questions: none — the previously-resolved "COMP-30 is data provider only, not routing decision-maker" remains correct.
- New open questions:
  - **Queue `@Transactional` / `@Primary` mismatch** — UK queue writes are not covered by the service-level transaction. Architect decision needed: remediate or accept?
  - **POST /user atomicity** — user upsert and auto-enrol run in separate implicit transactions. Architect decision needed: add `@Transactional` to wrap both?
  - **CHAS infra → HTTP 403 conflation** — should infrastructure failure surface differently from authorization denial?
  - **logback-spring.xml runtime source** — file is absent from repo; Logstash integration is non-functional unless mounted via ConfigMap/sidecar at runtime. Confirm with deployment team.
  - **Owners of `nap.nap_parent_entity`, `wdp.case`, `wdp.action`** — read-only consumers in this repo, owners still TBC.
  - **Component that transitions `wdp.lft_report.status`** beyond PENDING — COMP-30 only writes PENDING and reads INPROGRESS; downstream owner unidentified.
  - **Known callers per endpoint** — no naming convention identifies callers in source.

**WDP-DECISIONS.md · Candidate new ADRs**
- **ADR candidate (HIGH severity):** Multi-datasource service-level `@Transactional` binding pattern. The `jakarta.transaction.Transactional` default-bean-selection behaviour silently picks the `@Primary` transaction manager. In dual-datasource components this fails to wrap operations against the non-primary datasource. Platform-level guidance needed: explicit `transactionManager` attribute, or `@Qualifier`-bound annotations, or a service-per-datasource decomposition.
- **ADR candidate (HIGH severity):** Side-effect-with-no-transaction pattern. POST /user user-upsert + auto-enrol runs as two implicit transactions. Platform guidance needed on whether multi-step write paths require `@Transactional` even when the steps are on the same datasource.
- **ADR candidate (MEDIUM severity):** Infrastructure-failure-vs-authorization-denial conflation. CHAS `RestClientException` → HTTP 403 in COMP-30 mirrors a similar pattern across the platform — should infra faults surface as 5xx (e.g. 503) to distinguish them from genuine auth denials?
- **DEC-014 deviation map enrichment:** Add COMP-30 — five outbound REST integrations share a single `new RestTemplate()` bean with no timeouts and no Resilience4j. Same pattern as COMP-25 / COMP-31 / others. Reinforces the platform-wide DEC-014 VOID posture.
- **DEC-020 deviation map enrichment:** Add COMP-30 — no `@UniqueConstraint` on any owned table. POST /{region}/queue has confirmed app-level race window between duplicate-name check and insert. POST /lft-report, POST /user, POST /user-view rely on load-then-save last-write-wins.
- **Candidate observability ADR:** Logback configuration ownership. COMP-30 declares the `logstash-logback-encoder` dependency and the `logstash.server.host.port` property but ships no `logback-spring.xml`. Same pattern observed elsewhere — platform guidance needed on whether logback config is service-owned (in repo) or platform-owned (mounted at runtime).

**WDP-ARCHITECTURE.md**
- No topology change. COMP-30 remains a Queue & Workflow Service (Section 4.4.2). The dual-datasource pattern is component-internal and does not change the platform topology.

**WDP-NFRS.md · Section 6 Risk Register**
- Add 🔴 HIGH RISK-NEW: COMP-30-A — Queue `@Transactional` binds to @Primary US TX manager; UK writes not wrapped. Mid-operation UK failure leaves partially committed queue + criterion + user_queue rows.
- Add 🔴 HIGH RISK-NEW: COMP-30-B — POST /user user-upsert and auto-enrol are separate implicit transactions. Auto-enrol failure leaves orphaned user with no queue assignments.
- Add 🔴 HIGH RISK-NEW: COMP-30-C — CHAS infra failure mapped to HTTP 403; indistinguishable from auth denial. Silent service degradation pattern.
- Add 🔴 HIGH RISK-NEW: COMP-30-D — `USCaseSearchDaoImpl` swallows DB exceptions during PIN authorization lookup; cascades to misleading 403.
- Add 🔴 HIGH RISK-NEW: COMP-30-E — No XA between nap and wdp datasources; current paths don't write both schemas atomically, but the pattern is one architectural change away from real data inconsistency.
- Add 🔴 HIGH RISK-NEW: COMP-30-F — No Resilience4j on IDP calls; portal-access failure cascade for users not yet in local DB during IDP outage.
- Add 🟡 MEDIUM RISK-NEW: COMP-30-G — All five REST integrations share one `new RestTemplate()` with no timeouts. Sustained slow-response from any dependency exhausts Tomcat threads.
- Add 🟡 MEDIUM RISK-NEW: COMP-30-H — POST /lft-report insert is non-transactional; partial-write windows possible.
- Add 🟡 MEDIUM RISK-NEW: COMP-30-I — No `@UniqueConstraint` on any owned table. POST /{region}/queue has confirmed race window.
- Add 🟡 MEDIUM RISK-NEW: COMP-30-J — Logstash integration non-functional without externally-mounted logback-spring.xml. Same pattern as other components.
- Add 🟢 LOW RISK-NEW: COMP-30-K — `AsyncConfiguration` is dead config (`@EnableAsync`, executor bean created, but no `@Async` anywhere).
- Add 🟢 LOW RISK-NEW: COMP-30-L — `spring-boot-devtools` in pom.xml without `<scope>runtime</scope>`. Likely benign due to Spring Boot Maven plugin repackage exclusion, but best-practice deviation.

**WDP-INTEGRATIONS.md**
- No external system contract changes. All five integrations (IDP internal, IDP external standard, IDP external MFD, Display Code Service, CHAS, AWS S3) are internal WDP services or platform-internal infrastructure. No external partner contract affected.

#### Deviation flags for COMP-30

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 | ⛔ DEVIATES | 🟡 MEDIUM |
| DEC-003 | ✅ NOT APPLICABLE | — |
| DEC-004 | ✅ NOT APPLICABLE | — |
| DEC-005 | ✅ NOT APPLICABLE | — |
| DEC-014 | ⛔ DEVIATES | 🔴 HIGH |
| DEC-019 | ✅ NOT APPLICABLE | — |
| DEC-020 | ⛔ DEVIATES | 🟡 MEDIUM |

**DEC-001 detail:** No transactional outbox. All writes are direct table mutations. No event publication accompanies any write. Severity is MEDIUM rather than HIGH because COMP-30 publishes no Kafka events at all — there is no event-loss-on-rollback hazard, only the absence of downstream notifications when state changes.

**DEC-014 detail:** Confirmed by absence of `resilience4j` dependency in pom.xml. All five outbound REST integrations (IDP internal, IDP external standard, IDP external MFD, Display Code Service, CHAS) plus AWS S3 SDK have no circuit breaker, no bulkhead, no rate-limiter. All share a single `new RestTemplate()` bean with no connect timeout and no read timeout. HIGH severity driven by IDP and CHAS being session-establishing dependencies — outage of either produces user-visible portal failure.

**DEC-020 detail:** No `idempotency-key` header read or written. No `@UniqueConstraint` on any owned table. POST /{region}/queue has confirmed race window between application-level duplicate-name check and insert — concurrent requests can both pass the check and both insert. POST /user, POST /lft-report, POST /user-view rely on load-then-save patterns that produce last-write-wins under concurrent traffic.

#### Remaining gaps

| Gap | Type | Action required |
|-----|------|-----------------|
| Production replica count | Environment config | Check XL Deploy / Helm variable `{{ replicas-gcp-user-queue-skill-service }}` value per environment. |
| Owner of `nap.nap_parent_entity` | Team confirmation | Identify owning component — read-only consumer in COMP-30. |
| Owner of `wdp.case` and `wdp.action` | Team confirmation | Confirm CaseManagementService (COMP-23) vs DisputeService (COMP-22) vs CaseActionService (COMP-24). |
| Component that transitions `wdp.lft_report.status` from PENDING → INPROGRESS → terminal | Architect decision / cross-component review | COMP-30 only writes PENDING and reads INPROGRESS. Cross-repo sweep needed to identify the report-generation worker. |
| Known callers per endpoint | Follow-up Copilot CLI question | Ask in caller repos: *"Is there any client implementation, OpenAPI tag, or comment in [Merchant Portal / Ops Portal / report worker repos] that identifies the gcp-user-queue-skill-service endpoints they call?"* |
| Live case-to-queue eligibility matcher | Architect decision / cross-component review | Identify the component that reads `nap.queue_criterion` and `wdp.queue_criterion` rows and applies them against live cases. |
| `logback-spring.xml` runtime source | Runtime observation / team confirmation | Inspect deployed pod for mounted ConfigMap or sidecar-provided logback config. Cannot answer from source — Copilot CLI re-run will not help. |
| AWS credentials mechanism for S3 presigner | Runtime observation / team confirmation | Confirm whether IRSA, env vars, or instance profile provides credentials in production. Cannot answer from source. |
| `@Transactional` / `@Primary` mismatch on queue ops — remediate? | Architect decision | Three remediation paths: (a) explicit `transactionManager` attribute, (b) split into UK and US sub-services, (c) accept current state and document. |
| POST /user atomicity — add `@Transactional`? | Architect decision | Same datasource (nap), so `@Transactional` would correctly wrap both writes. Decide whether the auto-enrol failure mode is tolerable. |
| CHAS infra → HTTP 403 — remediate? | Architect decision | Pattern repeats across platform; this might warrant a platform-level ADR rather than a component fix. |
| Whitelist endpoint specificity (two paths, not wildcard) — intentional? | Team confirmation | Confirm whether `/user-queue-skill-service-api-docs/swagger-config` is the only ancillary swagger path needed, or whether a wildcard was intended. |
| `validateUserRole` removal ADR | Team confirmation | Confirm whether the region-specific role validation was removed deliberately and whether an ADR or ticket exists. |
| `app.name` env var | Environment config | Confirm whether `APP_NAME` is set in deployment manifests; referenced in `management.metrics.tags.application` but never defined in YAML. |

#### Doc status after this change

- `WDP-COMP-30-USER-QUEUE-SKILL-SERVICE.md` → `v1.1 DRAFT` — source-verified 2026-04-28 · architect confirmation pending
- `WDP-DB.md` → annotation refinements pending (next reconciliation session)
- `WDP-KAFKA.md` → no change
- `WDP-HANDOVER.md` → Confirmed Architectural Facts and Open Questions updates pending (next reconciliation session)
- `WDP-DECISIONS.md` → 3 new candidate ADRs + 3 deviation-map enrichments pending (next reconciliation session)
- `WDP-ARCHITECTURE.md` → no change
- `WDP-NFRS.md` → 12 new RISK rows pending (next reconciliation session)
- `WDP-INTEGRATIONS.md` → no change
---


### 2026-04-28 — COMP-28 DisplayCodeService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `gcp-display-code-service` (Worldpay-mdvs-gcp-display-code-service.git) — source-verified by GitHub Copilot CLI 2026-04-28. Architect confirmation pending.

**Nature of change:** Correction pass against source. No functional change in production. The v1.0 DRAFT (original GitHub Copilot CLI extraction) contained two version-string transcription errors, one config-key error in the Spring profile activation pattern, an incomplete K8s probes section, and missed one new architectural finding worth recording at risk level — the permission-shape divergence between POST /search and GET /privileges. One MEDIUM-HIGH risk and three new MEDIUM/LOW risks added.

#### Platform-level impacts

**WDP-DB.md**

- Section 2 — `wdp.display_codes` row: **No ownership change.** Reinforce: COMP-28 is a runtime READ-ONLY consumer; writes are by DBA scripts / migrations only. Existing reader attribution (COMP-28 DisplayCodeService; COMP-31 BusinessRulesService for criteria enrichment) is unchanged and remains accurate.

- Section 2 — `wdp.dispute_static_tabs_rules` row: **No ownership change.** Reinforce: COMP-28 is a runtime READ-ONLY consumer; writes by DBA scripts / migrations only. Add to Notes: *"Two repository projections — `findByRoles` (used by POST /search) returns 11 Y/N flag columns; `getPermissions` (used by GET /privileges) returns 17 Y/N flag columns. The 6-column delta (`fax_match`, `fax_report`, `trans_detail`, `auth_detail`, `settle_detail`, `dispute_history`) is the source of the COMP-28 permission-shape divergence between endpoints."*

- No new shared-table risk introduced. No new cross-component co-writer concerns.

**WDP-KAFKA.md**

- No change. COMP-28 remains in Section 4's Kafka-free components list. Audit re-confirms zero `spring-kafka`, `kafka-clients`, `aws-msk-iam-auth` in `pom.xml` and zero `@KafkaListener` / `KafkaTemplate` reference in source.

**WDP-COMP-INDEX.md**

- COMP-28 description correction (already flagged in v1.0 DRAFT, re-confirmed 2026-04-28): the existing entry implies this service "determines TIER1 sub-product eligibility from fraud and INR reason code lists." Source confirms this is incorrect — the service returns raw code lists; eligibility logic is the caller's responsibility (e.g. COMP-04 NAPDisputeEventService). COMP-INDEX entry should be updated at next reconciliation.

- Doc status row update: COMP-28 → `📝 DRAFT v1.1 — source-verified 2026-04-28, architect confirmation pending`.

**WDP-NFRS.md · Section 6 Risk Register** *(candidate new RISK rows)*

- **🟠 MEDIUM-HIGH candidate:** *"Permission-shape divergence between COMP-28 POST /search and GET /privileges — same nominal `UserPermission` envelope returns 11 flags on /search vs 17 flags on /privileges. UI portals gated on the 6 extended flags (faxMatch, faxReport, transDetail, authDetail, settleDetail, disputeHistory) appear unauthorised when the page is rendered from /search response data even when caller's roles authorise them. Architect decision required: intentional minimisation or accidental drift?"* New finding.

- **🟡 MEDIUM candidate:** *"COMP-28 `UnauthorizedException` (blank JWT `iss` claim) returns HTTP 500 instead of HTTP 401. No dedicated `@ExceptionHandler(UnauthorizedException.class)` exists; falls through catch-all `RuntimeException` handler. Defeats security-monitoring rules that key on auth-related response codes."* New finding.

- **🟡 MEDIUM candidate (extension of existing pattern RISK):** *"COMP-28 `minReadySeconds: 30` misplaced inside Pod template spec — silently ignored at runtime."* Same copy-paste-class defect previously confirmed on COMP-25, COMP-34, COMP-08. Strengthens the case for a platform-wide manifest sweep. Recommend extending the existing pattern-audit RISK to include COMP-28 explicitly.

- **🟡 MEDIUM candidate:** *"COMP-28 lazy JWKS resolution — `JwtIssuerAuthenticationManagerResolver` resolves issuer endpoints on first request per issuer, not at startup. Cold-start latency anomaly per impacted issuer; first impacted request returns 401 if IdP unreachable."* New finding.

- Existing 🔴 HIGH RISK on `DriverManagerDataSource` (no connection pool, no timeouts) — re-confirmed 2026-04-28 with no remediation evidence in repo.

- Existing 🟡 MEDIUM RISK on `userId` field declared but never read — re-confirmed.

- Existing 🟢 LOW RISK on `hibernate.show-sql = true` in production — re-confirmed.

- New 🟢 LOW candidates (code-quality / hygiene):
  - `ValidationMessages.properties` contains messages from a different service (rules-service references — `stage`, `owner`, `network`, `actionCode`, etc.) not used by any validator in this service.
  - `EnumName` / `EnumNameValidator` declared but applied to no field.
  - Duplicate exception-handler helper methods in `GlobalExceptionHandler` (`createDuplicateResponseEntity`, `createErrorResponseEntity` patterns) appear unused.
  - `spring-boot-devtools` present in `pom.xml` (auto-disabled when running from packaged jar but should be removed).

**WDP-DECISIONS.md** *(candidate ADR / deviation map enrichment)*

- **DEC-014 deviation map for COMP-28:** Re-confirm — no `io.github.resilience4j` dependency, no circuit breaker / retry / rate limiter / bulkhead annotations anywhere. Sole DB dependency runs with no retry, no circuit breaker, no socket timeout. Consistent with platform-wide DEC-014 VOID posture.

- **Candidate new ADR — "Endpoint-pair contract divergence on shared response envelope":** The COMP-28 `UserPermission` envelope is shared between `POST /search` and `GET /privileges` but populated from two different repository projections (11 vs 17 columns) and two different platform-derivation strategies (substring match vs hardcoded role sets). Recommend a formal decision: align the two paths (defect remediation) or document the divergence as intentional API minimisation. Pattern is unique to COMP-28 in the audit corpus to date.

- **Candidate clarification — DEC-019 / DEC-004 attestation for read-only services:** COMP-28 is a clean attestation that DEC-004 / DEC-019 compliance can be confirmed at architecture level by codebase scan for entity classes containing `pan` / `cardNumber` / `accountNumber` / `acctNum` fields plus absence of any persistent write. Worth documenting as a verifiable pattern when WDP-DECISIONS.md is rebuilt.

**WDP-INTEGRATIONS.md**

- No change. COMP-28 has no external integration contract — its outbound dependencies are PostgreSQL `wdp` schema (internal), the WDP-shared external IdP for JWKS (already documented platform-wide), and Logstash (internal observability infrastructure). No card-network, no acquiring-platform, no third-party integration.

**WDP-ARCHITECTURE.md**

- No change. Topology and principles unchanged. COMP-28 remains a WDP Core Service in Section 4. The permission-shape divergence is component-level, not platform-level.

#### Deviation flags for COMP-28

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ NOT APPLICABLE (no Kafka publish) | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE (no Kafka) | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES (no PAN handled; `fullPan` is a permission flag, not PAN data) | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE (no Kafka consumer) | — |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform-wide VOID) | 🟡 MEDIUM (accepted) |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES (no persistent writes at all) | — |
| DEC-020 Full At-Least-Once Idempotency | ✅ NOT APPLICABLE (read-only service) | — |
| DEC-023 Replica = 1 Hard Constraint | ✅ NOT APPLICABLE (standard scaled stateless service) | — |

**DEC-014 detail:** Confirmed pattern matches the platform-wide DEC-014 VOID posture — no Resilience4j, no `RestTemplate` timeouts (no `RestTemplate` bean exists at all), no retry on the JDBC dependency. Consistent with COMP-25, COMP-31, COMP-37. Severity remains MEDIUM per platform-wide acceptance.

**DEC-004 / DEC-019 detail:** Source-verified by full-codebase scan for `pan` / `cardNumber` / `accountNumber` / `acctNum` fields on entity classes — none found. `fullPan` field on `UserPermission` model is `Y` / `N` permission indicator only. No persistent writes occur at runtime.

#### Remaining gaps

| Gap | Type | Action |
|-----|------|--------|
| Production replica count | Environment config | Confirm via XL Deploy / Kubernetes for each environment — value is the template variable `{{ replicas-mdvs-gcp-display-code-service }}`. |
| Actual JDBC URL host pattern (Aurora `globaldisputedatabase` vs separate GCP-hosted instance) | Environment config / team confirmation | `${wdp_datasource_jdbc_url}` injected from K8s secret bundle (`gcp-display-code-service-secrets` and/or `wdp-common-secrets`); host not in repo. |
| Actual `jwt_trusted_issuer_urls` values | Environment config | Confirm IdP host(s) per environment from K8s secrets. |
| Actual `logstash_server_host_port` values | Environment config | Confirm per environment from K8s secrets. |
| Actual `display_code_types` SpEL map content | Environment config | Confirm per environment — runtime startup will fail if malformed (no try/catch around the `@Value("#{${displaycodes.map}}")` injection). |
| Permission-shape divergence (11 vs 17 flags) — intentional or accidental? | **Architect decision** | If intentional, document the rationale; if accidental, raise as a defect and align both paths to use the 17-flag superset. |
| `UnauthorizedException` → 500 instead of 401 — defect or accepted? | **Architect decision** | Affects security observability. Trivial fix (add a dedicated `@ExceptionHandler`). |
| HikariCP migration plan for `DriverManagerDataSource` | **Architect decision** | No migration evidence in repo. Confirmed production throughput risk. Plan or accept the risk with documented operational ceiling. |
| Full caller inventory beyond COMP-04 | **Follow-up Copilot CLI question on consumer repos** — exact question to ask: *"Does this repository call any endpoint matching `/merchant/gcp/display-code/search` or `/merchant/gcp/display-code/privileges`? If so, cite file:line of every call site, the HTTP client used, the request payload shape, and any caching of the response."* — to be run against COMP-49 (Merchant Portal) and COMP-50 (Ops Portal) repositories when those audits are performed. |
| `minReadySeconds` misplacement — fix as part of platform-wide manifest sweep | **Architect decision** | Pattern confirmed on COMP-25, COMP-34, COMP-08, COMP-28. Recommend single coordinated PR. |
| WDP-COMP-INDEX.md description correction (TIER1 eligibility) | Documentation | Update at next WDP-COMP-INDEX.md reconciliation session. |
| Whether IdP JWKS unreachability has caused production failures | **Runtime observation** | Cannot be answered from source — Splunk / Kibana log analysis on 401 spikes correlated with IdP outages. Operational follow-up. |
| `EnumName` / `EnumNameValidator` framework — keep for future use or remove? | **Architect decision** | Dead code currently; cheap to leave but cleaner to remove. |
| `ValidationMessages.properties` rules-service residue | **Defect / hygiene** | Trivial PR — remove unused messages. |

#### Doc status after this change

- `WDP-COMP-28-DISPLAY-CODE-SERVICE.md` → `v1.1 DRAFT` — source-verified 2026-04-28 · architect confirmation pending
- `WDP-COMP-INDEX.md` → COMP-28 description and status updates pending (next reconciliation session)
- `WDP-DB.md` → `wdp.dispute_static_tabs_rules` row Notes update pending (next reconciliation session)
- `WDP-KAFKA.md` → No change
- `WDP-NFRS.md` → 4 candidate new RISK rows pending (next reconciliation session)
- `WDP-DECISIONS.md` → 1 candidate new ADR (endpoint-pair contract divergence) pending architect decision
---

### 2026-04-28 — COMP-26 QuestionnaireService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `mdvs-gcp-questionnaire-service` — source-verified by GitHub
Copilot CLI 2026-04-28. Architect confirmation pending.

**Nature of change:** Correction pass. The v1.0 DRAFT was largely accurate
on auth, controllers, validation rules, and surface contracts. Source
audit confirmed the bulk of those claims and surfaced four material new
findings that change the platform-level risk picture: (1) the codebase
has NO `@Transactional` boundary anywhere, (2) the JPA datasource is
`DriverManagerDataSource` rather than HikariCP, (3) malformed JSON bodies
return HTTP 500 rather than 400, and (4) `minReadySeconds: 30` is
mis-indented under Pod spec and is likely ignored by Kubernetes. Endpoint
count corrected from 16 to 20 (15 Visa + 5 Dispute). Error body field
name corrected from `message` to `errorMessage`. HTTP 405 added to the
documented status set.

#### Platform-level impacts

**WDP-DB.md**

- Section 2 row for `wdp.disputes_questionnaire` — UPDATE: append to the
  COMP-26 portion of the writer cell:
  *"COMP-26 has NO `@Transactional` boundary on any path; every JPA call
  runs in Spring Data JPA's per-call default transaction. PUT (find →
  save) and B1 UPSERT (find → decide → save) span multiple short
  transactions and are vulnerable to TOCTOU races. App tier uses
  `DriverManagerDataSource` (no HikariCP); effective pooling depends on
  PgBouncer / Aurora-side."*
  Severity on the shared-table risk row stays 🔴 HIGH; reasoning extends
  beyond the previously-documented duplicate-POST gap to also cover
  concurrent B1 UPSERT inserts.
- Section 2 row for `wdp.disputes_questionnaire_doc_status_rules` — ADD
  if not present, with COMP-26 as read-only consumer and writer marked
  ⚠️ TBC (no INSERT/UPDATE/DELETE in COMP-26's repo; no migration files;
  writer must be located by cross-repo grep).

**WDP-KAFKA.md**

- No change. COMP-26 has no Kafka producer or consumer. Confirm by
  reaffirming COMP-26 in the Kafka-Free component list at next
  reconciliation.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

- Add: COMP-26 has NO `@Transactional` annotation anywhere in service or
  controller code. Multi-step flows (PUT, B1 UPSERT) span multiple
  per-call default transactions.
- Add: COMP-26 uses `DriverManagerDataSource` (no HikariCP, no
  application-tier connection pool). Every JPA call opens a new JDBC
  connection.
- Add: COMP-26 endpoint count is 20 (15 Visa controller + 5 Dispute
  controller).
- Add: COMP-26 error body field name is `errorMessage` (not `message`).
- Add: COMP-26 produces HTTP 405 via `HttpRequestMethodNotSupportedException`
  handler. Status set is {200, 201, 400, 401, 404, 405, 500}; no 403,
  409, 415, 422, 503 paths.
- Add: COMP-26 maps `HttpMessageNotReadableException` to HTTP 500 (not
  the conventional 400). Invalid platform string also returns HTTP 500
  via `IllegalArgumentException` → `RuntimeException` handler.
- Add: COMP-26 has no Liquibase, Flyway, or SQL migration files in repo.
  DB-level constraints (UNIQUE, NOT NULL) on `wdp.disputes_questionnaire`
  are not determinable from source.
- Add: COMP-26 deploys 3 K8s resources (Deployment + Service + Ingress)
  with 6 ingress host rules via templated host names. No HPA, no PDB,
  no startup probe, no CPU limit/request configured.
- Add: COMP-26 `minReadySeconds: 30` is mis-indented under
  `spec.template.spec` (Pod spec) rather than `spec` (Deployment spec)
  and is likely ignored by Kubernetes — actual rolling-update behaviour
  may differ from declared intent.
- Add: COMP-26 has no `application-{profile}.yaml` files; profile
  overrides are inline in `application.yaml` and only override the
  `case-actions-service` URL. `local` profile uses HTTPS; all other
  profiles use HTTP intra-cluster.
- Add: COMP-26 `StandardEntityError` and `createDuplicateResponseEntity()`
  helpers in `GlobalExceptionHandler` are dead code — not reachable from
  any active `@ExceptionHandler`.
- Add: COMP-26 A2 Pre-Compliance cross-field validation is symmetric —
  DECL forbids `partialAcceptanceReason`; PART forbids
  `continuePreFilingReason` (prior draft only documented the required-side
  half).

- Resolved open questions:
  - "COMP-26 POST idempotency gap — duplicate POSTs insert new rows" —
    confirmed: POST has no application-level pre-existing-row check and
    no DB-level UNIQUE constraint visible. Open question now narrows to
    the architect decision on remediation (see New open questions below).

- New open questions:
  - COMP-26 no-`@Transactional` posture — intentional design or
    remediation needed? PUT and B1 UPSERT both span multiple short
    transactions and have TOCTOU race exposure.
  - COMP-26 `DriverManagerDataSource` choice — intentional (e.g. behind
    PgBouncer) or remediate to HikariCP?
  - COMP-26 `HttpMessageNotReadableException` → 500 mapping — intentional
    or fix to 400?
  - COMP-26 invalid platform string returns 500 — intentional or fix to
    400 with explicit pre-validation?
  - COMP-26 `checkWriteOffReasonWriteOffNote` commented out on POST but
    active on PUT — restore symmetry or document the asymmetry?
  - COMP-26 `checkCTMTemplate` body fully commented out — restore or
    remove?
  - COMP-26 `minReadySeconds: 30` indentation defect — DevOps fix?
  - COMP-26 POST idempotency — add DB UNIQUE on (I_CASE, I_ACTION_SEQ)
    and caller `Idempotency-Key`, or accept current behaviour?
  - Cross-repo: writer of `wdp.disputes_questionnaire_doc_status_rules`?
  - Cross-repo: ContestService (COMP-20) relationship to QuestionnaireService?

**WDP-DECISIONS.md · Candidate new ADRs**

- Candidate ADR — COMP-26 `DriverManagerDataSource` over HikariCP.
  Severity 🟡 MEDIUM. Decision needed: is this an accepted local
  deviation (e.g. PgBouncer fronts the service) or a defect to remediate?
- Candidate ADR — COMP-26 No `@Transactional` boundaries declared.
  Severity 🔴 HIGH. Decision needed: accept the per-call default-tx
  posture and the resulting TOCTOU exposure on PUT/B1, or introduce
  service-method-level transactions and pessimistic locking on
  read-before-write paths?
- Candidate ADR — COMP-26 POST duplicate-row behaviour and DEC-020
  posture. Severity 🔴 HIGH. Decision needed: DB UNIQUE + caller
  `Idempotency-Key`, or formal acceptance of duplicate-row insertion?
- Candidate ADR — COMP-26 outbound `case-actions-service` REST has no
  timeout, retry, or circuit breaker. Severity 🔴 HIGH (thread-blocking
  exposure). Same shape as the DEC-014-VOID factual record observed for
  other components; needs a new local decision since DEC-014 is voided.

**WDP-ARCHITECTURE.md**

- No topology change. The Aurora connection-pool topology assumption
  may need a footnote: COMP-26 (and possibly other services pending
  audit) does not use HikariCP at the application tier — production
  pooling depends on PgBouncer / Aurora-side, not visible from the
  service repo.

**WDP-NFRS.md · Section 6 Risk Register**

Candidate new RISK rows:

- RISK — COMP-26 thread-blocking exposure: outbound REST to
  `case-actions-service` has no connect/read timeout, no retry, no
  circuit breaker. Severity 🔴 HIGH. Mitigation: configure timeouts
  and Resilience4j circuit breaker; or accept and document.
- RISK — COMP-26 connection-storm exposure: `DriverManagerDataSource`
  opens a new JDBC connection per JPA call; throughput is bounded by
  Postgres / PgBouncer accept rate. Severity 🟡 MEDIUM. Mitigation:
  HikariCP with tuned pool, or confirm PgBouncer fronts the service
  with adequate capacity.
- RISK — COMP-26 concurrent-write race on B1 UPSERT and on PUT (no
  `@Transactional`, no `@Version`, no DB UNIQUE confirmed). Severity
  🔴 HIGH for B1 (duplicate-row insertion possible). Mitigation:
  service-method `@Transactional`, DB UNIQUE on (I_CASE, I_ACTION_SEQ),
  optimistic locking via `@Version`, or pessimistic lock on read.
- RISK — COMP-26 POST duplicate-row insertion (no app-level check, no
  confirmed DB UNIQUE). Severity 🔴 HIGH. Mitigation: DB UNIQUE +
  caller `Idempotency-Key`.
- RISK — COMP-26 declared `minReadySeconds: 30` is ineffective due to
  manifest indentation. Severity 🟡 MEDIUM (rolling updates may proceed
  without the intended cool-off). Mitigation: DevOps fix.
- RISK — COMP-26 `HttpMessageNotReadableException` → 500 misleads
  observability and breaks client error-handling conventions.
  Severity 🟡 MEDIUM. Mitigation: map to 400 in
  `GlobalExceptionHandler`.

**WDP-INTEGRATIONS.md**

- COMP-26 → `mdvs-gcp-case-actions-service` REST contract. ADD entry if
  not present:
  - Endpoint: `GET /merchant/gcp/case-actions/{platform}/case/{caseNumber}/actions/{actionSequence}`
  - Auth: client-credentials OAuth2, registration `wdp-internal-auth`,
    scope `openid`
  - Failure mode: hard fail, no retry, no circuit breaker, no timeout.
    `RestClientException` rethrown → HTTP 500 to caller.
  - `local` profile uses HTTPS; all other profiles use HTTP intra-cluster.
- COMP-26 → enterprise IdP token endpoint contract. ADD entry if not
  present.

#### Deviation flags for COMP-26

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⛔ DEVIATES (by design — REST API, no Kafka producer) | 🟢 LOW |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE (no Kafka) | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES (no PAN field exists) | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE (no Kafka consumer) | — |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⛔ DEVIATES — POST inserts unconditionally; no `Idempotency-Key`; no app-level pre-check; no DB UNIQUE confirmed | 🔴 HIGH |

**DEC-001 deviation detail (LOW)**: Component is a pure synchronous CRUD
REST API. Outbox pattern does not apply because there is no Kafka
producer. Recorded as a deviation only if DEC-001 is interpreted as
applying to all WDP services regardless of integration shape.

**DEC-020 deviation detail (HIGH)**: POST handlers (5 Visa + B2 Non-Visa)
build a new entity and call `save()` without any pre-existing-row check,
without `Idempotency-Key` header handling, and without a dedupe table.
The JPA entity has only a non-unique `@Index` on `(I_CASE, I_ACTION_SEQ)`;
no DB-level UNIQUE constraint is visible in the repo (no Liquibase /
Flyway / SQL migration files exist). Effective duplicate-row protection
depends entirely on whether a DB UNIQUE was added externally. The B1
UPSERT path additionally exposes a TOCTOU race because find-then-decide
runs across separate per-call transactions with no pessimistic lock.

#### Doc status after this change

- `WDP-COMP-26-QUESTIONNAIRE-SERVICE.md` → `v1.1 DRAFT` —
  source-verified 2026-04-28 · architect confirmation pending
- `WDP-COMP-INDEX.md` → no count change (still 50 components); status
  remains 📝 DRAFT
- `WDP-DB.md` → `wdp.disputes_questionnaire` writer-cell amendment +
  doc_status_rules row addition pending (next reconciliation session)
- `WDP-KAFKA.md` → no change
- `WDP-DECISIONS.md` → 4 candidate ADRs surfaced (next reconciliation)
- `WDP-INTEGRATIONS.md` → 2 outbound contracts pending registration
  (next reconciliation)
- `WDP-NFRS.md` → 6 candidate RISK rows surfaced (next reconciliation)
---


### 2026-04-28 — COMP-25 NotesService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `mdvs-gcp-notes-service` — source-verified by GitHub Copilot CLI 2026-04-28.
Architect confirmation pending.

**Nature of change:** Correction pass — 11 v1.0 claims corrected against source,
9 new findings added, 6 new risk rows surfaced. No new component, no new flows,
no new external integration boundaries.

#### Platform-level impacts

**WDP-DB.md**
- `wdp.NOTES` row — no change to writers, key columns, or shared-write
  severity. Source verification confirms COMP-25 has no awareness of COMP-23
  / COMP-24 co-writers (no `@UniqueConstraint`, no `@Version`, no advisory
  lock, no co-ordination annotation). Existing 🟠 MEDIUM-HIGH severity rating
  on COMP-23 duplicate-insert defect remains accurate.
- `nap.NOTES` row — confirm COMP-25 as primary writer via `napTransactionManager`.
  Note that COMP-23 also writes on the NAP create path (per COMP-23 transaction
  boundary table) — this dual-writer fact is already documented in
  WDP-DB.md against `wdp.NOTES`; consider adding equivalent shared-table
  treatment for `nap.NOTES` at next reconciliation.

**WDP-KAFKA.md**
- Section 3 Topic Registry — COMP-25 row for `business-rules` (publisher):
  no change to topic, key, or consumer side. Update **payload schema** to
  reference the 11-field `AddNotesBREvent` structure now confirmed
  (eventType=`"NOTE_ADDED"`, startRuleGroup=`"NOTE_ADDED_TO_CASE"`,
  `previousActionSequence` and `documentNameList` always null).
- Section 4 DEC-003 deviation map — COMP-25 row "Confirmed from source"
  remains accurate. **Augment with mid-batch atomicity note**: per-event loop
  inside `@Transactional` produces deterministic Kafka-orphan events on
  partial-batch failure (not just JVM-crash window).
- Add observation under "How to read this table" or component notes:
  `kafka_business_event_topic` env var has no default in `application.yaml` —
  service fails to start if unset. Same operational footgun as
  `kafka_retry_count`.

**WDP-NFRS.md** *(Section 6 Risk Register — candidate rows for next reconciliation)*
- New 🔴 RISK — COMP-25 mid-batch Kafka orphan window: deterministic split-brain
  on every multi-note POST that fails partway through publishing. K-1 events
  on topic, 0 rows in DB after rollback. Reproducible, not crash-only. (Upgrade
  from latent-edge-case framing in v1.0.)
- New 🟡 RISK — COMP-25 `/actuator/prometheus` requires authentication: endpoint
  exposed but not in Spring Security whitelist. Scraper auth required or silent
  metrics blackout. Pattern audit owed against other WDP services.
- New 🟡 RISK — COMP-25 `kafka_retry_count` has no default value: service fails
  to start if env var unset. CrashLoopBackOff on misconfigured deployment
  manifest.
- New 🟡 RISK — COMP-25 `minReadySeconds: 30` misplaced inside PodSpec instead
  of Deployment spec: Kubernetes silently ignores it. The 30-second rollout
  stability gate is not actually applied at runtime. Pattern audit owed —
  this is a copy-paste-class defect that may be replicated across other WDP
  manifests.
- New 🟡 RISK — COMP-25 `actionSequence` regex-injection latent bug: request
  value used as a regex pattern in `String.matches()` against
  `ActionSummary.actionSequence`. Currently mitigated by `@Pattern("^[0-9]*$")`
  validation; defense-in-depth absent.
- New 🟢 RISK — COMP-25 Swagger/code drift on NoteType: enum allows 16 values,
  Swagger `@Schema` declares 14. `NOTE` and `SNOTE` accepted but undocumented.
- Updated 🟡 RISK (existing) — COMP-25 DEC-014 deviation framing: stronger
  finding than v1.0 implied. **No `RestTemplate` bean exists** — `new
  RestTemplate()` per call. Even if a future fix adds a bean with timeouts,
  the inline instantiations must be removed first.

**WDP-DECISIONS.md** *(candidate ADR / deviation map enrichment)*
- DEC-001 deviation map for COMP-25: existing entry "Synchronous publish
  inside `@Transactional` boundary — not atomically coupled (broker ACK
  pre-commit, commit-time exception emits ghost event)" is **incomplete**.
  Update to include the mid-batch case: per-event loop inside the transaction
  produces deterministic orphan events on partial-batch failure, not only
  JVM-crash windows.
- DEC-014 deviation map for COMP-25: confirmed pattern matches the
  platform-wide DEC-014 VOID posture — no Resilience4j, no timeouts, no
  `RestTemplate` bean. Stronger finding than v1.0.
- DEC-020 deviation map for COMP-25: confirmed — `idempotency-key` forwarded
  on Kafka header but never checked. Replays produce N additional rows AND
  N additional Kafka events.

**WDP-COMP-INDEX.md**
- COMP-25 status row: `📋 PENDING → 📝 DRAFT` (per HANDOVER backlog).
- Doc status: `📝 DRAFT v1.1 — source-verified 2026-04-28, architect
  confirmation pending`.

**WDP-INTEGRATIONS.md**
- No external system contract changes. NotesService is internal-only — its
  outbound dependencies (Action Service, Display-Code Service, IDP, MSK)
  are all internal WDP services or platform-internal infrastructure. No
  WDP-INTEGRATIONS row to add.

**WDP-ARCHITECTURE.md**
- No topology-level changes. NotesService remains a WDP Core Service in
  Section 4. The mid-batch orphan finding is component-level, not
  platform-level — it tightens the DEC-001 deviation language but does not
  change the platform topology or principles.

#### Deviation flags

| DEC | Severity | Detail |
|-----|----------|--------|
| **DEC-001** | 🔴 HIGH | No transactional outbox. Two failure modes: (a) JVM crash between DB commit and Kafka publish, (b) **mid-batch publish failure produces deterministic orphan events on the topic — DB rolls back, Kafka does not**. Confirmed deterministic and reproducible. |
| **DEC-003** | 🟡 MEDIUM | Kafka partition key is `caseNumber`, not `merchantId`. Consistent with all six confirmed publishers of `business-rules` topic. Candidate ADR pending platform-wide. |
| **DEC-004** | 🟢 N/A | COMPLIANT — no PAN data path. |
| **DEC-005** | 🟢 N/A | NOT APPLICABLE — no Kafka consumer. |
| **DEC-014** | 🔴 HIGH | No Resilience4j. **No `RestTemplate` bean** — `new RestTemplate()` per call, no timeouts, no retries, no circuit breakers. Stronger finding than v1.0. |
| **DEC-019** | 🟢 N/A | COMPLIANT — no PAN handling. |
| **DEC-020** | 🟡 MEDIUM | `idempotency-key` forwarded but never checked. No DB unique constraint. Replays produce duplicate rows + duplicate Kafka events. |

#### Remaining gaps

| Gap | Type | Action |
|-----|------|--------|
| Literal name of `${kafka_business_event_topic}` | Environment config | Confirm with deployment / DevOps team. WDP-KAFKA.md cross-evidence indicates `business-rules` but not source-confirmed for this service. |
| Downstream consumers of `AddNotesBREvent` from this service | Cross-repo | Cross-repo Copilot CLI sweep across `gcp-business-rules-processor` (COMP-16) and any other suspected consumer of `business-rules` topic. **Suggested follow-up Copilot question (run against COMP-16 repo):** "List every distinct event type / payload class that the `business-rules` Kafka consumer in this repo deserialises and processes. For each, cite file:lines and identify the source service if visible from `eventType` / `startRuleGroup` / `source` field discrimination." |
| Case-status eligibility gates (notes on closed cases) | Cross-repo | Cross-repo investigation against COMP-24 CaseActionService — does it block notes on closed cases, or is no such check applied platform-wide? |
| `minReadySeconds` misplacement — replicated in other WDP manifests? | Pattern audit | Run a one-line grep across all known component repo `resources.yaml` files for `minReadySeconds` indentation level. **Suggested follow-up Copilot question:** "In `resources.yaml` (or equivalent K8s manifest), is `minReadySeconds` defined at `spec.minReadySeconds` (Deployment-level, correct) or at `spec.template.spec.minReadySeconds` (PodSpec-level, ignored by Kubernetes)? Cite file:lines." |
| `/actuator/prometheus` authentication — intentional or oversight? | Architect decision + pattern audit | Compare with COMP-23 / COMP-24 / COMP-27 Spring Security whitelist patterns. If platform standard is unauthenticated scraping, this is a deviation. If platform standard is authenticated scraping, scraper config must be confirmed. |
| Production replica count for COMP-25 | Environment config | XL Deploy placeholder — confirm from environment config or team. Same posture as every other continuously-running WDP component. |
| HikariCP pool sizes and connection timeouts (NAP + WDP datasources) | Runtime observation | Not configured in source — uses Spring Boot defaults or runtime env overrides. Confirm via runtime inspection. |
| Whether `nap.NOTES` deserves its own shared-table row in WDP-DB.md | Architect decision | COMP-23 also writes `nap.NOTES` on NAP create path. Currently only `wdp.NOTES` carries an explicit shared-table risk row. Symmetry pass owed. |
---

### 2026-04-28 — COMP-22 DisputeService · v1.0 DRAFT → v2.0 DRAFT

**Source:** `mdvs-gcp-disputes-service` — source-verified by GitHub
Copilot CLI 2026-04-28. Architect confirmation pending.

**Nature of change:** Correction pass against source. No functional
change in production. Six factual corrections to v1.0, five new
findings, one cross-component shared-table speculation disproven.

#### Platform-level impacts

**WDP-DB.md**
- **Section 4 — Shared Table Risk Register · `wdp.CASE` row:** remove
  the speculation that COMP-22 might be the INSERT/DELETE owner.
  Source-verified zero writes from COMP-22 — no `INSERT`, `UPDATE`,
  `DELETE`, no Spring Data repositories, no `@Modifying`, no
  `JdbcTemplate`. The DocumentManagementService (COMP-37)
  column-level UPDATE on `wdp.CASE` is now confirmed orphan to
  COMP-22 — **the cross-component review item recorded in HANDOVER
  for COMP-22/COMP-37 is closed on the COMP-22 side**. The COMP-37
  write owner question stands.
- **Section 2 · `wdp.CASE` row · Downstream readers:** no change —
  COMP-22 already listed as read-only; this entry now stands on
  source verification.
- **Section 2 · `wdp.action` row · Downstream readers:** no change —
  COMP-22 already listed as read-only.
- **Section 3 · IBM DB2 cross-platform table — `BC.TBC_DM_CASE,
  BC.TBC_DM_OCCUR · COMP-22` row:** confirm the existing entry —
  no change required, source-verified read-only on
  `coreTransactionManager` when `coreMigrationStatus=false`.

**WDP-KAFKA.md**
- **Section 3 — Topic registry · `business-rules` row · note [1]:**
  enrich the existing "COMP-22 wired but commented out" note with
  the disablement attribution: commit `c29018cd` by Shringi Nitin
  (WP) on 2025-08-08, message "code changes (#93)". No in-source
  rationale recorded.
- **Section 4 — Kafka-Free components register · COMP-22 row:**
  no change — entry remains accurate ("Kafka producer wired to
  `business-rules` but call site commented out — effectively
  Kafka-free at runtime").

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Add or correct:
- COMP-22 runtime is **Spring Boot 3.5.12** (not 3.5).
- COMP-22 readiness probe is **port 8082 path `/readyz`** (not port
  8052; 8052 does not exist anywhere in the repo — v1.0 was wrong).
- COMP-22 SFG SFTP fallback is **fully `@Async`** end-to-end. The
  v1.0 doc claim that error paths run synchronously is wrong — the
  `@Async` annotation applies to all invocations of `writeFile`.
  HTTP 200 is returned before the SFTP write completes. Exceptions
  inside the executor may be lost.
- COMP-22 SFG SFTP filename is `file.getOriginalFilename()` raw —
  no case-number prefix, no action-sequence, no timestamp.
  Filename collision is possible.
- COMP-22 internal-firm enforcement is application-layer (not Spring
  Security), `contains` check on the JWT `iss` claim against literal
  `us_worldpay_fis_int`. `AuthorizationServiceImpl` constructs
  `ForbiddenException` with `HttpStatus.UNAUTHORIZED` but the global
  handler returns 403 — constructor's status code is dead.
- COMP-22 does **not** propagate `v-correlation-id` on any outbound
  REST call. The interceptor places it in MDC for local logs only;
  `RestInvoker` and `IdpRestInvoker` do not forward it.
- COMP-22 HikariCP pools are unconfigured on both datasources —
  Spring Boot defaults apply (`maximumPoolSize=10`,
  `connectionTimeout=30s`).
- COMP-22 Tomcat threads are unconfigured — Spring Boot defaults
  apply (`threads.max=200`, `accept-count=100`,
  `connection-timeout=20s`).
- COMP-22 `@Async` SFTP executor is sized via env vars
  `gcp_async_corepoolsize`, `gcp_async_maxpoolsize`,
  `gcp_async_queuecapacity` — **no defaults**, startup-fails-if-
  absent. Same startup-fails-if-absent applies to
  `core_migration_status`.
- COMP-22 ships exactly two application endpoints. Sweep confirmed
  zero `@KafkaListener`, zero `@Scheduled`, zero `@JmsListener`,
  zero `@RabbitListener`, zero WebSocket / SSE handlers anywhere
  in the repo.
- COMP-22 platform set on the `/documents` path is `NAP`, `VAP`,
  `CORE`, `LATAM`. NAP comparisons use `equalsIgnoreCase`.
- COMP-22 Swagger / OpenAPI is exposed in non-prod environments
  only — `/disputes-service-api-docs/**` and `/swagger-ui/**`
  whitelisted in `SecurityConfig`.
- COMP-22 Kafka publish disablement attribution: commit
  `c29018cd`, Shringi Nitin (WP), 2025-08-08, message "code
  changes (#93)". No in-source rationale.
- COMP-22 response field names are `dsptSummary`,
  `creditDsptSummary`, `debitDsptSummary` (not `dept*`), and
  `outstandingDsptAmount` / `outstandingDsptCount` (not
  `outstandingAmount` / `outstandingItems`).

Resolved open questions (remove from HANDOVER):
- "COMP-22/COMP-37 cross-component ownership of
  `USCaseEntity`/`UKCaseEntity` — INSERT/DELETE owner vs
  column-level UPDATE co-writer" — **resolved on COMP-22 side**:
  COMP-22 owns zero writes. The COMP-37 ownership question stands
  but is now a single-component question, not cross-component.

New open questions (add to HANDOVER):
- COMP-22 known callers of `POST /summary` and `POST /documents`
  — no contract tests, postman collections, or docs in repo;
  team confirmation needed.
- COMP-22 reason for Kafka publish disablement on 2025-08-08
  (PR #93 review or Shringi Nitin confirmation).
- COMP-22 production replica count — XL Deploy placeholder; matters
  for the `@Async` SFTP executor capacity sizing review.
- COMP-22 JWT trusted-issuer URLs per environment —
  `jwt_trusted_issuer_urls` env var, not in repo.
- COMP-22 SFG SFTP filename collision — is namespacing required
  (case-number / action-seq / timestamp prefix)? Architect decision.
- COMP-22 unconfigured HikariCP and Tomcat pools — intentional
  defaults or oversight given the no-timeouts / no-CB posture?

**WDP-DECISIONS.md · Candidate new ADRs**
- **HIGH candidate:** "Mandatory client timeouts and circuit breakers
  on all inter-service REST calls" — DEC-014 deviation in COMP-22 is
  total: zero timeouts on six dependencies, zero circuit breakers,
  HikariCP and Tomcat at defaults. Matches the same posture observed
  on COMP-27 CaseSearchService (also `new RestTemplate()`, no
  Resilience4j). Recommend a platform-level ADR mandating connect /
  read timeouts and a circuit-breaker library on every active
  outbound REST dependency.
- **MEDIUM candidate:** "Mandatory `v-correlation-id` propagation on
  outbound REST calls" — COMP-22 does not propagate. COMP-27 also
  observed without propagation. Distributed tracing breaks at every
  service boundary out of these components. Platform-level decision
  needed.
- **MEDIUM candidate:** "Required env vars must declare a default in
  YAML (`${var:default}`) where startup is acceptable in the unset
  case, otherwise startup-failure must be deliberate and documented"
  — COMP-22 has four env vars with no default
  (`core_migration_status`, three `gcp_async_*`), all
  startup-blockers. Pattern likely applies platform-wide.
- **MEDIUM candidate:** "SFG SFTP filename namespacing for the NAP
  fallback path" — current behaviour writes
  `getOriginalFilename()` raw, allowing collisions on the SFTP
  target.

**WDP-ARCHITECTURE.md**
- No change. COMP-22 topology unchanged. The two-endpoint, no-Kafka,
  no-DB-write reader-and-orchestrator pattern is consistent with
  the existing platform topology description.

**WDP-NFRS.md · Section 6 Risk Register**
- **New RISK candidate:** "COMP-22 fully `@Async` SFG SFTP fallback —
  HTTP 200 is returned before SFTP write completes; executor
  exceptions may be lost; caller has no signal that file landed"
  — 🟡 MEDIUM. NAP-platform-only. Fix candidate: add audit-row
  write inside the executor to record SFTP outcome.
- **New RISK candidate:** "COMP-22 SFG SFTP filename collision — raw
  `getOriginalFilename()` written to `/Outgoing/`" — 🟡 MEDIUM.
- **New RISK candidate:** "COMP-22 no idempotency on `POST /documents`
  — DEC-020 deviation; network-retried uploads re-upload, re-transfer
  ownership, re-publish metadata" — 🟡 MEDIUM.
- **New RISK candidate:** "COMP-22 `v-correlation-id` not propagated
  to any downstream REST call — distributed tracing breaks at every
  service boundary" — 🟡 MEDIUM. Likely covers other components on
  the same shared `RestTemplate` pattern; flag for platform sweep.
- **New RISK candidate:** "COMP-22 four required env vars have no
  defaults — `core_migration_status`, `gcp_async_corepoolsize`,
  `gcp_async_maxpoolsize`, `gcp_async_queuecapacity`. Application
  fails to start on placeholder resolution if any one is unset" —
  🟡 MEDIUM. Pattern likely covers other components.
- **New RISK candidate:** "COMP-22 HikariCP and Tomcat pools at
  defaults compounded with no client timeouts and no circuit
  breakers — single slow dependency saturates entire pod with no
  bulkhead between `/summary` and `/documents`" — 🔴 HIGH (already
  partly covered by DEC-014 deviation note; add the
  pool-saturation-blast-radius detail).
- **New RISK candidate:** "COMP-22 401/403 status-code inconsistency
  — `ForbiddenException` constructed with `HttpStatus.UNAUTHORIZED`
  but global handler returns 403; constructor's status is dead
  code" — 🟢 LOW.

**WDP-INTEGRATIONS.md**
- No change to existing integration contracts. The CaseSearchService,
  DocumentManagementService, CaseActionService, IDP Token Service,
  and SFG SFTP integration mappings are already documented.
- Worth recording at next reconciliation: COMP-22 makes no use of
  the `business-rules` Kafka topic at runtime despite being
  registered as a producer.

#### Deviation flags for COMP-22

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ NOT APPLICABLE | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ COMPLIES (when active; producer disabled) | — |
| DEC-004 PAN Encryption Before Persistence | ✅ NOT APPLICABLE | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-014 Resilience4j Circuit Breakers and Timeouts | ⛔ DEVIATES | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⛔ DEVIATES | 🟡 MEDIUM |

**DEC-014 DEVIATES detail:** Plain `new RestTemplate()` with no
connect or read timeout on every outbound REST call. No
Resilience4j dependency in `pom.xml`. No circuit breaker on any of
the six active outbound dependencies (CaseSearch, DocMgmt POST,
DocMgmt PUT, CaseAction, IDP token, SFG SFTP). Compounded by
HikariCP and Tomcat pools at defaults — no bulkhead between
`/summary` and `/documents` flows on the same Tomcat thread pool.
A single slow dependency saturates the entire pod.

**DEC-020 DEVIATES detail:** No idempotency mechanism on
`POST /documents`. No `Idempotency-Key` header check, no
application-level dedup table, no replay-protection filter, no
DB unique constraint (no DB writes at all). Duplicate calls
re-execute the entire chain — re-upload to DocumentManagementService,
re-transfer ownership, re-publish metadata with `notifyBRQueue=true`,
re-trigger downstream BR-queue work.

#### Remaining gaps

| Gap | Resolution path |
|-----|-----------------|
| Reason for Kafka publish disablement on 2025-08-08 | Team confirmation — PR #93 review or ask Shringi Nitin (WP) |
| Known callers of `POST /summary` and `POST /documents` | Team confirmation — no docs, contract tests, or postman in repo |
| Production replica count | Environment config / ops team — XL Deploy placeholder `{{ replicas-mdvs-gcp-disputes-service }}` |
| JWT trusted-issuer URLs per env | Environment config — `jwt_trusted_issuer_urls` env var, not in repo |
| `app.name` resolution for Prometheus metric tag | Runtime observation — inspect a live `/prometheus` scrape to see whether the `application` tag has a value |
| `max_req_header_size` runtime value | Environment config — not in repo |
| `gcp_env` active-profile mapping | Environment config — not in repo |
| Whether HikariCP and Tomcat defaults are intentional | Architect decision |
| Whether SFG SFTP filename should be namespaced | Architect decision |
| Whether DEC-014 / DEC-020 deviations are accept-and-document or remediate | Architect decision (same severity class as DEC-019/020 risk-accepted ADRs) |
| Follow-up Copilot CLI question — none required for COMP-22 | All in-repo questions now answered. Future Copilot questions belong on COMP-37 to settle the remaining `wdp.CASE` write-ownership ambiguity for the column-level UPDATE on the desk-blanking branch. |

#### Doc status after this change

- `WDP-COMP-22-DISPUTE-SERVICE.md` → `v2.0 DRAFT` — source-verified
  2026-04-28 by GitHub Copilot CLI · architect confirmation pending
---

### 2026-04-25 — COMP-51 CaseExpiryProcessor (NEW COMPONENT) · v1.0 DRAFT

**Source:** `gcp-case-expiry-processor-batch` — source-verified by Claude Code
2026-04-25. Architect confirmation pending.

**Nature of change:** New component registration. CaseExpiryProcessor is the
deadline-enforcement reader half of the case-expiry subsystem; COMP-17
CaseExpiryUpdateConsumer is the writer half. The two share `wdp.case_expiry`
as their coordination surface and write to `wdp.outgoing_event_outbox` with
distinct `channel_type` discriminators. No prior architecture documentation
existed for COMP-51.

#### Platform-level impacts

**WDP-COMP-INDEX.md**

- Add new row in the registry table: `| 51 | CaseExpiryProcessor |
  Batch/Scheduler | ✅ Production | 📝 DRAFT |
  WDP-COMP-51-CASE-EXPIRY-PROCESSOR.md |`. Insert in numerical order
  immediately after COMP-50 WDP Ops Portal.
- Add a one-line responsibility entry in the responsibility-summary
  section: "COMP-51 — CaseExpiryProcessor: Scheduled batch that scans
  `wdp.case_expiry` for past-due deadlines and acts on each row by calling
  Case Action / Case Management / Expiry Rules APIs and one of three
  terminal APIs (Update Action / Accept / Add Action). Reader half of the
  case-expiry subsystem; COMP-17 is the writer half. Captures retry-
  exhausted failures into `wdp.outgoing_event_outbox` with
  `channel_type = EXPIRY_BATCH`."
- Add a "Case Expiry Subsystem" cross-reference note linking COMP-17 and
  COMP-51 — both share `wdp.case_expiry` coordination.

**WDP-DB.md**

- **Section 2 · `wdp.case_expiry` row:** change Owning Component(s) from
  "COMP-17 CaseExpiryUpdateConsumer" to **"COMP-17 CaseExpiryUpdateConsumer
  (INSERT/UPSERT/DELETE on action close); COMP-51 CaseExpiryProcessor (SELECT
  page-fetch, UPDATE i_retry_count, DELETE on success/CLOSED-action/retry-
  exhaustion)"**.
- **Section 2 · `wdp.case_expiry` row · Notes:** append "⚠️ Shared write
  boundary between COMP-17 and COMP-51 with no coordination — no row-level
  lock, no version column, no SELECT FOR UPDATE. Race between COMP-17
  upsert and COMP-51 retry-counter UPDATE possible. Documented and pending
  architect decision."
- **Section 2 · `wdp.case_expiry` row · Open question:** resolved partially
  — COMP-51 confirmed as the primary downstream reader. Other downstream
  readers (if any) still need cross-component sweep on remaining WDP repos.
- **Section 2 · `wdp.outgoing_event_outbox` row:** add COMP-51 to the
  Owning Component(s) list with discriminator: **"COMP-51 CaseExpiryProcessor
  (channel_type=EXPIRY_BATCH, created_by=WCSEEXPB, status=ERROR direct
  on retry exhaustion)"**.
- **Section 2 · `wdp.outgoing_event_outbox` row · Notes:** append "COMP-51
  writes status=ERROR directly (no transition through PUBLISHED→FAILED)
  and channel_type=EXPIRY_BATCH. ⚠️ No platform component currently
  identified as consumer of EXPIRY_BATCH rows — COMP-12 Scheduler3 reads
  FAILED and PENDING_DEFERRED only, so COMP-51 outbox rows may be terminal-
  write-only. Architect decision required."
- **Section 4 · Shared Table Risk Register · `wdp.case_expiry` row:** add
  new entry — "🔴 HIGH — Shared write between COMP-17 and COMP-51 with no
  coordination mechanism. COMP-17 upserts on Kafka events; COMP-51 reads,
  UPDATEs retry counter, and DELETEs. Race conditions possible between the
  two components on the same row (i_retry_count overwrite, mid-flight
  upsert during chunk processing). No row-level lock, no version column."
- **Section 4 · Shared Table Risk Register · `wdp.outgoing_event_outbox`
  row:** update existing 🟢 LOW entry to add: "COMP-51 added 2026-04-25
  with channel_type=EXPIRY_BATCH (distinct from COMP-17's EXPIRY_EVENTS).
  Discriminator isolation maintained — no cross-component reads of each
  other's channel_type rows. Logical isolation remains intact at LOW
  severity."

**WDP-KAFKA.md**

- **Section 5 — Components without Kafka involvement:** add COMP-51
  CaseExpiryProcessor to the list. Note: declares `spring-kafka`,
  `kafka-clients`, and `aws-msk-iam-auth` as POM dependencies but no
  producer or consumer code exists; staged-but-unwired.
- **Sections 3 and 4:** no change. COMP-51 has no active topic or
  consumer-group registration.

**WDP-FLOW-INDEX.md**

- Candidate new flow document: **WDP-FLOW-CASE-EXPIRY-LIFECYCLE.md** —
  end-to-end deadline-tracking workflow covering COMP-18 (publishes to
  `case-action-events` when an action's expiry data changes), COMP-17
  (consumes and maintains `wdp.case_expiry`), COMP-51 (scans and acts on
  past-due deadlines), and the COMP-51 → outbox `EXPIRY_BATCH` failure
  path. Recommend creation at next flow-document rebuild window.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Add or correct:
- COMP-51 CaseExpiryProcessor exists as a standalone Spring Batch
  Deployment in `gcp-case-expiry-processor-batch` repository. Deadline-
  enforcement reader half of the case-expiry subsystem; COMP-17 is the
  writer half.
- COMP-51 trigger is a Spring `@Scheduled` cron — NOT a Kubernetes CronJob.
  Cron expression and page size are env-templated and not in source.
- COMP-51 chunk size = 1; reader paginates with PageRequest.of(0, pageSize,
  Sort.by("z_insrt")) plus monotonic-id cursor.
- COMP-51 calls 7 distinct upstream services: IDP token, Case Action API,
  Case Management API, Expiry Rules API, Update Action API, Accept API,
  Add Action API.
- COMP-51 retry mechanism is hand-rolled — increment `i_retry_count` on
  `wdp.case_expiry` if < 3, write outbox + delete row on 3rd consecutive
  failure. `spring-retry` is on classpath but unused.
- COMP-51 writes outbox rows with `channel_type=EXPIRY_BATCH` (distinct
  from COMP-17's `EXPIRY_EVENTS`), `created_by=WCSEEXPB` (distinct from
  COMP-17's `WCSEEXPC`), and `status=ERROR` direct (no PUBLISHED→FAILED
  transition).
- COMP-51 has no `@SchedulerLock`, no ShedLock, no advisory lock. DEC-023
  operational-only — replicas must be 1.
- COMP-51 has no liveness/readiness/startup probes wired.
- COMP-51 has no test coverage. `src/test/` is empty.
- COMP-51 generates a fresh random `v-correlation-id` UUID per outbound
  REST call — same anti-pattern as COMP-17. End-to-end audit trail across
  the 4–5 calls per record cannot be reconstructed from headers alone.
- COMP-51 IDP token call does NOT carry `v-correlation-id` — same gap as
  COMP-17.
- COMP-51 uses `DriverManagerDataSource` (not pooled) — acceptable for
  low-frequency batch.
- COMP-51 outbox INSERT and `wdp.case_expiry` DELETE share the Spring
  Batch chunk transaction (atomic by chunk boundary).
- COMP-17 and COMP-51 share `wdp.case_expiry` as their coordination
  surface — COMP-17 is the inserter / upserter / closure-deleter, COMP-51
  is the deadline-enforcement-deleter and retry-counter updater. No lock
  or version column between them.

Resolved open questions (remove from HANDOVER):
- "Downstream consumers of `wdp.case_expiry` not yet confirmed" —
  partially resolved: COMP-51 confirmed as primary reader. Cross-component
  sweep on remaining WDP repos still pending for any other potential
  readers.

New open questions (add to HANDOVER):
- COMP-51 writes outbox rows with `channel_type=EXPIRY_BATCH` and
  `status=ERROR`. **No platform component currently identified as
  consumer of these rows.** COMP-12 Scheduler3 reads only FAILED and
  PENDING_DEFERRED, so EXPIRY_BATCH ERROR rows may be terminal-write-only
  with no recovery path. Architect decision required: define consumer or
  accept as audit-only sink.
- COMP-51 ↔ COMP-17 race conditions on `wdp.case_expiry`: COMP-17 upsert
  may collide with COMP-51 retry-counter UPDATE or with COMP-51 success-
  path DELETE. Architect decision required: accept as risk or remediate
  via row-version / SELECT FOR UPDATE.
- COMP-51 cron value, page size, table prefix, and replica count — all
  env-templated; XLD config inspection needed.
- COMP-51 `v-correlation-id` regenerated per call — bug or intentional?
  Same question as COMP-17. Likely a platform-wide remediation candidate.
- COMP-51 Discover hybrid-merchant RE2/REPR special case — permanent
  regulatory rule or migration-era workaround? Hardcoded constants
  (`RE2`, `REPR`, `WPAYOPS`, `EXPIRY_DAYS=32`).
- COMP-51 HTTP 504 retry on POST endpoints (Accept, Add Action, Rules) —
  is repeating a 504-timed-out POST safe across all 7 upstream services?
  Architect / team confirmation needed.
- COMP-51 hardcoded user IDs `WCSEEXPB` (this component) and `WCSEEXPC`
  (COMP-17) — confirm distinction is intentional and recorded in the
  platform user-ID registry.

**WDP-DECISIONS.md · Candidate new ADRs**

- **HIGH candidate:** "EXPIRY_BATCH outbox channel — terminal write-only
  vs. recoverable" — COMP-51 writes `status=ERROR` rows that no platform
  consumer currently picks up. Architecture decision needed: define a
  consumer (manual ops portal? automated retry scheduler?) or formally
  accept as audit-only failure sink. Affects every future component that
  uses outbox-as-failure-sink pattern.
- **HIGH candidate:** "Shared-table coordination on `wdp.case_expiry`" —
  COMP-17 and COMP-51 are co-writers with no coordination. Pattern is
  novel for WDP (unlike `chbk_outbox_row` which has clear LOADING/PENDING/
  PUBLISHED hand-off semantics). Architectural decision needed on
  coordination strategy: row-version, advisory lock, SELECT FOR UPDATE,
  or accept-as-low-probability.
- **MEDIUM candidate:** "v-correlation-id propagation — per-call UUID
  regeneration anti-pattern" — confirmed in COMP-17 and COMP-51. Likely
  platform-wide. Could be one ADR mandating per-record (or per-message)
  correlation IDs across all outbound REST calls within a single
  processing unit.
- **MEDIUM candidate:** "Hand-rolled retry counter persistence vs.
  spring-retry" — COMP-51 (and COMP-07/08/09 via similar pattern) uses
  hand-rolled retry counters persisted to the source table. Worth
  formalising as a platform pattern or replacing with a uniform retry
  abstraction.
- **LOW candidate:** "No probes on polling batches" — already raised by
  COMP-07 entry. COMP-51 confirms the pattern. Strengthens the case for
  remediation across all batch components.

**WDP-ARCHITECTURE.md**

- Add COMP-51 to the platform-summary topology / component list. Group
  with COMP-17 under a "Case Expiry Subsystem" sub-heading if applicable
  to the topology diagram structure.
- No principle change; topology evolves only by addition of one new
  Deployment in the `wdp-micro` namespace.

**WDP-NFRS.md · Section 6 Risk Register**

- Add new RISK row: "RISK-NN — `EXPIRY_BATCH` outbox rows have no
  identified consumer. COMP-51 writes failure-capture rows with
  channel_type=EXPIRY_BATCH and status=ERROR. No platform component is
  currently confirmed to read them. Severity 🔴 HIGH until a consumer is
  defined or the rows are formally accepted as audit-only."
- Add new RISK row: "RISK-NN — Shared write to `wdp.case_expiry` between
  COMP-17 and COMP-51 has no coordination mechanism. Race conditions on
  retry counter and deletion possible. Severity 🟡 MEDIUM until coordination
  strategy decided."
- The existing RISK on at-most-once delivery and the existing RISK on
  no-DLQ posture cover COMP-51's per-record failure modes generically; no
  new entries needed there.

**WDP-INTEGRATIONS.md**

- No change. COMP-51's REST dependencies are all internal `wdp-micro`
  namespace services. No external integrations.

#### Deviation flags for COMP-51

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY | 🟡 MEDIUM |
| DEC-014 Resilience4j (platform-VOID) | ✅ ABSENT (factual record only) | — |

**DEC-001 PARTIAL detail:** Outbox is used as a failure-capture sink, not
as a transactional producer outbox. Outbox INSERT and `wdp.case_expiry`
DELETE share the Spring Batch chunk transaction (atomic), but the business
state changes already executed by the failed REST call (Update Action /
Accept / Add Action) are NOT reconstructable from the outbox row — the
entry contains the source `CaseExpiryEntity` only, not the in-flight
downstream payload. Recovery requires manual reasoning.

**DEC-020 PARTIAL detail:** Idempotency is only via `wdp.case_expiry` row
deletion after success or retry exhaustion. No idempotency key stored or
checked against downstream APIs. If the job re-runs after a crash that
occurred after a successful Update Action / Accept / Add Action call but
before the row was deleted, the call would be re-issued — downstream-side
dedup must protect, not COMP-51-side. Two replicas have no guard — see
DEC-023.

**DEC-023 OPERATIONAL ONLY detail:** No `@SchedulerLock`, no ShedLock,
no advisory lock. Concurrency safety relies entirely on K8s `replicas: 1`.
Same posture as COMP-07 / COMP-08 / COMP-09 polling batches.

#### Cross-component coordination notes

**Impact on COMP-17 (CaseExpiryUpdateConsumer):**

The COMP-51 audit confirms COMP-17's previously-open question on downstream
readers of `wdp.case_expiry` — COMP-51 is the primary reader (cross-repo
sweep still pending for any other readers). This does NOT trigger a
parallel COMP-17 file revision; the next COMP-17 reconciliation will
absorb this fact via the HANDOVER update above.

The COMP-17 v1.1 DRAFT note "Downstream consumers of wdp.case_expiry not
yet confirmed" should be updated at next COMP-17 reconciliation to read:
"COMP-51 CaseExpiryProcessor confirmed as primary reader; cross-repo
sweep for any other readers still pending."

#### Doc status after this change

- `WDP-COMP-51-CASE-EXPIRY-PROCESSOR.md` → `v1.0 DRAFT` — source-verified
  2026-04-25 · architect confirmation pending
- `WDP-COMP-INDEX.md` → registry needs new row at position 51 (next
  reconciliation session)
- `WDP-DB.md` → `wdp.case_expiry` and `wdp.outgoing_event_outbox`
  ownership changes pending (next reconciliation session)
- `WDP-KAFKA.md` → Section 5 addition pending (next reconciliation session)
- `WDP-FLOW-INDEX.md` → new flow document candidate pending

---

## Template for new entries

*Per-component chats copy this template and fill it out. Paste the filled
block immediately above the most recent Pending Entry so Pending Entries
remain reverse-chronological.*

```
### YYYY-MM-DD — COMP-NN ComponentName · vX.Y DRAFT → vX.Y+1 DRAFT

**Source:** `<repo-name>` — source-verified by Claude Code YYYY-MM-DD.
Architect confirmation [pending / confirmed].

**Nature of change:** [Correction pass / New component / Feature addition /
Decommission / Other — one sentence].

#### Platform-level impacts

**WDP-DB.md**
- [Row / section affected + what changes, or "No change."]

**WDP-KAFKA.md**
- [Topic / consumer group / section affected + what changes, or "No change."]

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add / correct: [facts to add or correct]
- Resolved open questions: [list items to remove]
- New open questions: [list items to add]

**WDP-DECISIONS.md · Candidate new ADRs**
- [New ADR candidates with severity, or "No candidates."]

**WDP-ARCHITECTURE.md**
- [Topology / principle change, or "No change."]

**WDP-NFRS.md · Section 6 Risk Register**
- [New RISK rows or clarifications, or "No change."]

**WDP-INTEGRATIONS.md**
- [Integration contract change, or "No change."]

#### Deviation flags for COMP-NN

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 | [✅ COMPLIES / ⚠️ PARTIAL / ⛔ DEVIATES / ✅ NOT APPLICABLE] | [— / 🟢 / 🟡 / 🔴] |
| DEC-003 | ... | ... |
| DEC-004 | ... | ... |
| DEC-005 | ... | ... |
| DEC-019 | ... | ... |
| DEC-020 | ... | ... |

[Add narrative paragraphs for any ⚠️ PARTIAL or ⛔ DEVIATES entries.]

#### Doc status after this change
- `WDP-COMP-NN-NAME.md` → `vX.Y DRAFT` — source-verified YYYY-MM-DD ·
  architect confirmation [pending / confirmed]
```

---

*End of WDP-CHANGE-LOG.md*
