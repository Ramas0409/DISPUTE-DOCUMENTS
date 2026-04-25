# WDP-HANDOVER.md
**Worldpay Dispute Platform — Architecture Session Handover**
*Version: 3.1 | April 2026*
*Reconciled: 2026-04-25 — absorbed Pending Entries from WDP-CHANGE-LOG.md for COMP-07, COMP-08, COMP-09, COMP-11, COMP-12, COMP-13, COMP-14, COMP-15, COMP-16, COMP-17, COMP-18, COMP-19, COMP-20, COMP-21, COMP-23, COMP-24, COMP-27, COMP-37, COMP-41, COMP-43.*
*Component file source-verification status: 📝 DRAFT — architect confirmation pending on all 20 components.*

---

## Your Role

You are the senior architecture partner for Ram, who is a Solution
Architect for the Worldpay Dispute Platform (WDP). You work at
system and component level only. You never suggest code, classes,
configs, or implementation details. When Ram asks implementation
questions, remind him to use Claude Code.

---

## Ground Rules

- Think and respond at system and component level only
- Never suggest code, classes, configs, or implementation details
- Always check NFRs before validating any architecture decision —
  confirm with Ram that NFRs are still current before applying them
- When proposing changes, lead with Mermaid diagrams first,
  written description second
- Flag any architecture decision that conflicts with existing
  decisions in WDP-DECISIONS.md
- When architecture evolves, produce a WDP-CHANGE-LOG.md entry capturing
  the platform-level impact — do not rewrite derivative documents
  inline. Derivatives are reconciled in dedicated reconciliation sessions.
- For implementation questions, redirect to Claude Code

---

## Knowledge Base Structure

WDP architecture knowledge is organised in four tiers. Always
search the most specific tier first.

```
TIER 1 — Platform Level (stable, rarely changes)
  WDP-ARCHITECTURE.md      Platform topology, principles, deployment context
  WDP-DECISIONS.md         Architecture Decision Records (ADRs)
  WDP-INTEGRATIONS.md      External system contracts and integration patterns
  WDP-NFRS.md              Performance, resilience, security constraints
  WDP-HANDOVER.md          This file — session continuity
  WDP-CHANGE-LOG.md        Platform-level change log — Pending Entries +
                           Reconciled history

TIER 2 — Reference Indexes (navigation hubs)
  WDP-COMP-INDEX.md        Master component registry — 50 components, 01–50
  WDP-KAFKA.md             Master Kafka topic registry and MSK configuration
  WDP-DB.md                Database schema ownership map
  WDP-FLOW-INDEX.md        Index of all workflow documents

TIER 3 — Individual Component Files (detail layer)
  WDP-COMP-[NN]-[NAME].md  One file per component — REST contracts,
                            Kafka contracts, DB ownership, flow diagrams
  WDP-COMP-TEMPLATE.md     Master template for creating component files

TIER 4 — Workflow Documents (cross-component choreography)
  WDP-FLOW-[NAME].md       One file per business workflow — sequence
                            diagrams, step narratives, failure paths
```

**Search priority for a component question:**
WDP-COMP-INDEX.md → find the component number → retrieve
WDP-COMP-[NN]-*.md → answer from that file.

**Search priority for a workflow question:**
WDP-FLOW-INDEX.md → find the flow → retrieve WDP-FLOW-*.md.

**Search priority for a platform-level question:**
WDP-ARCHITECTURE.md first, then WDP-DECISIONS.md.

---

## Document Status

### Tier 1 — Platform Level

| Document | Status | Notes |
|----------|--------|-------|
| WDP-ARCHITECTURE.md | ✅ Current — v2.0 | Fully rebuilt April 2026. 15 Mermaid diagrams confirmed. ⚠️ Minor updates pending from 2026-04-18/23/25 reconciliation — see "Doc reconciliation backlog" below. |
| WDP-DECISIONS.md | ⚠️ Reconciliation pending — v2.0 | Rebuilt April 2026. DEC-011 and DEC-014 voided. DEC-019/020 recorded. **8 candidate ADRs surfaced from 2026-04-18/23/25 component audits** — see "Doc reconciliation backlog" below. |
| WDP-INTEGRATIONS.md | ⚠️ Reconciliation pending — v2.0 | Rebuilt April 2026. BEN Kafka correction applied. **JustAI scope correction needed** — active in COMP-21, planned in COMP-41. **COMP-21 enterprise Cert eAPI Entitlement Lookup to add.** **COMP-20 MCM stage detail to add.** |
| WDP-NFRS.md | ⚠️ Reconciliation pending — v2.0 | Rebuilt April 2026. **~25 candidate RISK rows surfaced from 2026-04-18/23/25 component audits** — see "Doc reconciliation backlog" below. |
| WDP-HANDOVER.md | ✅ Current — v3.1 | This file. Reconciled 2026-04-25. |
| WDP-CHANGE-LOG.md | ✅ Current | Reconciliation Marker dated 2026-04-25 — 20 entries moved from Pending to Reconciled. |

### Tier 2 — Reference Indexes

| Document | Status | Notes |
|----------|--------|-------|
| WDP-COMP-INDEX.md | ⚠️ Reconciliation pending — v2.0 | All 50 components registered. **Status updates needed: COMP-22 description correction, COMP-25 PENDING→DRAFT, COMP-24 repo name correction.** |
| WDP-KAFKA.md | ✅ Current — v2.1 | Reconciled 2026-04-25. COMP-37 removed from "Kafka-Free", added as 6th publisher of `business-rules`. COMP-18 removed as writer of `outgoing_event_outbox`. |
| WDP-DB.md | ✅ Current — v2.1 | Reconciled 2026-04-25. `wdp.dispute_event_change_log` row removed. COMP-43 PAN column corrected to `I_ACCT_CDH`. New row for `NAP.DISPUTE_EVENT_CONSUMER_ERROR` shared-table risk. |
| WDP-FLOW-INDEX.md | ✅ Created — v1.0 Skeleton | 11 core flows identified. None documented yet. |
| WDP-COMP-TEMPLATE.md | ✅ Created — v1.1 | Master template with 4 type blocks. |

### Tier 3 — Individual Component Files

| Status | Count | Details |
|--------|-------|---------|
| ✅ COMPLETE (individual file created and confirmed) | 1 | COMP-13 FileAcknowledgementProcessor |
| 📝 DRAFT (individual file created, architect confirmation pending) | 40 | COMP-01 through COMP-43 (excluding COMP-10, COMP-13, COMP-33, COMP-44) |
| 📋 PENDING (enterprise-owned, lower priority) | 1 | COMP-10 DM Mainframe |
| ⬜ NOT STARTED | 6 | COMP-33 (OrgManagementService — no repo found), COMP-44 (EDIAConsumer — planned), COMP-45, COMP-46, COMP-47 (File Generation — planned next sprint), COMP-48 (NYCEFileGenerationProcessor — planned) |
| 🔲 UI — separate action item | 2 | COMP-49 WDP Merchant Portal, COMP-50 WDP Ops Portal |

**Component migration complete.** 20 of 40 DRAFT files have undergone source-verified correction passes between 2026-04-18 and 2026-04-25. All carry "📝 DRAFT — architect confirmation pending" status.

### Tier 4 — Workflow Documents

No workflow files exist yet. 11 flows identified in WDP-FLOW-INDEX.md.
Workflow documentation to be done as a parallel work stream.

---

## Doc Reconciliation Backlog

The 2026-04-25 reconciliation session rebuilt **WDP-DB.md v2.1** and **WDP-KAFKA.md v2.1** from the Pending Entries. The following Tier 1 derivatives still need their own reconciliation passes against the same set of 20 entries:

| Document | Pending updates from 2026-04-18/23/25 entries |
|----------|------------------------------------------------|
| WDP-NFRS.md | ~25 candidate RISK rows: COMP-21 A–H (8 rows), COMP-23 PAN-clear / orphan-Kafka / NOTES-duplicate / RequestCorrelation leak / case-number NPE / spring-boot-devtools, COMP-24 open-action race / EP-9 three-tx no-compensation / NAP-EP5 asymmetry / no-timeouts / Kafka client defaults / IDP UAT URL / devtools, COMP-37 5-step non-atomic write chain / RestTemplate timeouts / DDB duplicate-check race, COMP-41 PUBLISHED-orphan paths / silent @Cacheable / no probes / Spring Retry imports dead, COMP-43 R3/R4/R6/R10 / R5/R8 / R7 / R2 / R13 / R14, COMP-19 NAP split-brain (RISK-NEW-1) / compensation-no-inner-trycatch (RISK-NEW-2), COMP-12 replicas>1 unmitigated, COMP-15 V3 silent-ATTACHED / V3 PATCH ghost-upload, COMP-14 RISK-A/B/C/D, COMP-13 three latent runtime bugs, COMP-08 writer-ACK hazard / SKIPPED accumulation, COMP-09 swallowed save exceptions / skip-no-row, COMP-11 DCPO empty-catch / DNWK silent-loss / DEC-004 non-numeric branch / S3ServiceImpl null-on-failure / no probes / no HPA / DISCHYB+AMEXOPTB silent-file-loss |
| WDP-DECISIONS.md | 8+ candidate ADRs: AcceptService split-brain (HIGH), idempotency pattern (suppress-insert vs insert-a-marker on shared outbox), writer-ACK hazard on MCM-style outbox ingest, silent exception swallow in batch writer/ACK paths, no health probes on polling batches, no concurrency guard on polling batches (DEC-023 operational-only), `kafka-before-commit` pattern naming, `@Transactional(rollbackOn=Exception.class)` stopgap atomicity pattern, DEC-020 idempotency-key formal void, IDP token caching pattern (COMP-21), async executor sizing (COMP-21), shared `RestTemplate` mutation, business-rules partition-key formalisation, mark-and-send within `@Transactional` cross-component pattern (COMP-12), feature-flag framework adoption, file_job COMPLETED transition ownership |
| WDP-INTEGRATIONS.md | JustAI scope correction (active in COMP-21 `JUSTTAI`, planned in COMP-41); COMP-21 enterprise Cert eAPI Entitlement Lookup added; COMP-20 MCM stage-dependent path detail (CH1/RE2 vs PAB); COMP-19 AMEX/DISCOVER NAP split-brain note in Section 3.1; Mastercard MCM CHI silent-no-op note clarified to apply to both NAP and PIN |
| WDP-ARCHITECTURE.md | Section 7.3 file_generation_event correction (was `wdp.file_notifications`); NAP migration route is live (CORE migration `coreMigration=true` activates routing); Section 8.1 Card Network Direct API Calls — note that AMEX/DISCOVER show as targets but fall through to no-op for AcceptService; Data Storage section — clarify S3 + DynamoDB + dual PostgreSQL datasources are unique to COMP-37 |
| WDP-COMP-INDEX.md | COMP-22 description: clarify Kafka producer is wired-but-commented-out (Kafka-free at runtime); COMP-25 status: PENDING → DRAFT; COMP-24 repo name: confirmed as `gcp-cas-actions-service` (artifact `case-actions-service`); COMP-37 description: add 6th publisher of `business-rules`; status updates for all 20 reconciled components to indicate "v2.0 / v1.1 DRAFT — source-verified [date], architect confirmation pending" |

---

## Current Work Position

**Phase:** Source-verification reconciliation phase. Cross-cutting derivatives being rebuilt from accumulated Pending Entries.

**Reconciliation history:**

| Date | Derivatives rebuilt | Entries consumed | Notes |
|------|---------------------|------------------|-------|
| 2026-04-25 | WDP-DB.md v2.1, WDP-KAFKA.md v2.1, WDP-HANDOVER.md v3.1 | 20 entries (COMP-07, 08, 09, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 23, 24, 27, 37, 41, 43) | First reconciliation session. NFRS / DECISIONS / INTEGRATIONS / ARCHITECTURE / COMP-INDEX still pending. |

**Next work — Reconcile remaining derivatives in priority order:**

1. **WDP-NFRS.md** — Section 6 Risk Register expansion (~25 candidate RISK rows from accumulated audits)
2. **WDP-DECISIONS.md** — 8+ candidate ADRs surfaced from audits, plus deviation map enrichment
3. **WDP-INTEGRATIONS.md** — JustAI scope correction, COMP-21 Cert eAPI, COMP-20 MCM stage detail
4. **WDP-ARCHITECTURE.md** — minimal updates (file_generation_event naming correction, AMEX/DISCOVER no-op note)
5. **WDP-COMP-INDEX.md** — status updates and description corrections

**After reconciliation backlog cleared:**

6. **Architect confirmation rounds** — promote 📝 DRAFT files to ✅ COMPLETE as architect signs off
7. **WDP-FLOW-[NAME].md files** — document the 11 core flows identified in WDP-FLOW-INDEX.md
8. **COMP-33 OrgManagementService** — document when GitHub repo is found
9. **COMP-45, 46, 47 File Generation** — document when sprint work is complete
10. **UI Portal documentation** — COMP-49 and COMP-50 as a separate action item

---

## Component Numbering

Components are numbered 01–50. Numbers are permanent.
See WDP-COMP-INDEX.md for the full registry.

**Current last number: 50**
**Next available: 51**

When adding a new component each quarter:
1. Assign number 51 (or next available)
2. Add to WDP-COMP-INDEX.md registry
3. Create WDP-COMP-[NN]-[NAME].md using the template
4. Append a Pending Entry to WDP-CHANGE-LOG.md (don't rewrite derivatives inline)

---

## How to Start a New Session

**Option A — Continuing the reconciliation backlog:**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first, then WDP-CHANGE-LOG.md.
Today I want to reconcile WDP-NFRS.md against the
already-Reconciled entries from 2026-04-25."
```

**Option B — Starting a workflow documentation session:**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first, then WDP-FLOW-INDEX.md.
Today I want to document the [FLOW-NAME] workflow.
Start with the Mermaid sequence diagram."
```

**Option C — Starting an architecture decision session:**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first.
I have a new architecture decision to discuss: [topic]."
```

**Option D — Answering a component question:**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first, then WDP-COMP-INDEX.md.
Question: [specific component or cross-component question]"
```

**For all sessions:** Claude always searches project knowledge before answering. The tier order above is the search priority.

---

## Confirmed Architectural Facts

Do not contradict these without explicit confirmation from Ram. Source-verification dates are noted in parentheses where relevant.

### Access and Authorization

- Case-level authorization is FAIL-CLOSED for NAP and PIN. Any exception including timeouts returns 403. The fail-open branch in WDP-COMPONENTS.md is incorrect — confirmed from COMP-01 source.
- No CORE platform value exists in API Gateway source code. How CORE, VAP, and LATAM requests receive case-level authorization is an open question.
- `validateOrgId()` is commented out on GET /orgentity in CHAS (COMP-03). Any authenticated PIN-platform caller can query any org hierarchy without scope validation. Confirmed security gap.
- COMP-24 CaseActionService has no RBAC enforcement — `RestInvoker.authorizeUser()` exists but is never called from any controller. Recorded as DEC-018 (Accepted Risk).
- **(2026-04-24)** COMP-27 CaseSearchService `POST /lft` derives internal-vs-external authorization from a request-body field (`LftSearchParams.isInternal`), not from the JWT. Flagged for architect review.
- **(2026-04-24)** External VAP and LATAM callers are not routed through any entity-scope authorization service in COMP-27. Only NAP (→ UAMS) and PIN/CORE (→ CHAS) are. Confirm intent.
- **(2026-04-24)** Internal PIN regular-user role exists (`WDP_PIN_REGULAR` in `AuthorizationList`) alongside the already-known `WDP_NAP_REGULAR`. COMP-27 confirmed.
- **(2026-04-25)** COMP-21 ChargebackService `/actuator/prometheus` requires JWT — NOT in security whitelist (corrected from prior draft).
- **(2026-04-23)** COMP-19 / COMP-20 liveness/readiness probe paths end in `livez` / `readyz` (with `z`), not `liver` / `lives` / `ready` as v1.0 stated.

### Kafka — Platform-Wide Patterns

- Every confirmed Kafka consumer in WDP uses pre-ACK or mid-flow ACK. DEC-005 (commit after full processing) is aspirational — it is not the current implementation.
- No Kafka DLQ topics exist in WDP. Error handling is via database error tables per consumer (e.g. `NAP.DISPUTE_EVENT_CONSUMER_ERROR` for COMP-05; `wdp.outgoing_event_outbox` for COMP-17/41/43; `wdp.bre_orchestration_outbox` for COMP-18; `wdp.chbk_outbox_row` for COMP-14/15).
- **(2026-04-18+)** Empty anonymous `CommonErrorHandler{}` is registered platform-wide on multiple consumers (COMP-14, COMP-15, COMP-16, COMP-17, COMP-18, COMP-39, COMP-41, COMP-42, COMP-43). Combined with `ErrorHandlingDeserializer`, deserialisation exceptions and unhandled application exceptions are silently swallowed — no DLT, no log, no counter. Distinct silent-loss class from the pre-ACK offset window.
- No circuit breakers (Resilience4j) anywhere in WDP. Confirmed absent across all 40 component files. DEC-014 formally voided.
- **business-rules topic has SIX confirmed publishers:** COMP-12 Scheduler4 (BUSINESS_RULES rows), COMP-15 EvidenceConsumer (WDP path, isMultiDocPending=false — `kafkaTemplate.send(...).get()` blocking inside `@Transactional`), COMP-23 CaseManagementService (kafka-before-commit pattern), COMP-24 CaseActionService (BRE inside `@Transactional`), COMP-25 NotesService (inside `@Transactional`), **COMP-37 DocumentManagementService (added 2026-04-23)** — Endpoints 1, 9, 10 (upload paths) and Endpoint 11 (questionnaire path with `@Transactional(rollbackOn=Exception.class)` — stronger atomicity). All use `caseNumber` as partition key — DEC-003 uniform deviation.
- **(2026-04-18)** **COMP-14 CaseCreationConsumer is NOT a publisher of `business-rules`** — confirmed by absence audit (no `KafkaTemplate`, no `ProducerFactory`, no reference to topic in source). Closes Observability-doc OQ-02.
- `wdp.bre_orchestration_outbox` is shared between COMP-18 (`component=NOTIFICATION_ORCHESTRATOR` rows) and COMP-12 Scheduler4 (`component=BUSINESS_RULES` rows). PUBLISHED-status orphan rows have no automatic re-drive — manual intervention required.
- **(2026-04-18)** **COMP-18 does NOT write to `wdp.outgoing_event_outbox`** — WDP-DB.md v2.0 was incorrect on this point. Source grep confirms zero references. Corrected in WDP-DB.md v2.1 and WDP-KAFKA.md v2.1.
- **(2026-04-18)** COMP-12 InboundDisputeEventScheduler uses **mark-and-send within a single `@Transactional`** — broker ACK precedes TX commit. **At-least-once with duplicate-possible crash window** (NOT at-most-once as v1.0 implied). Consumer-side `idempotency-key` dedup is the contracted mitigation.
- **(2026-04-18)** COMP-12 channelTypeTopicMap non-prod default keys: `EXPIRY_EVENTS`, `CORE_EVENTS`, `GP_EVENTS`, `BEN_EVENTS` (NOT `SEN_EVENTS` — corrected from v1.0).
- **(2026-04-18)** COMP-12 Kafka-metadata write-back asymmetry: `new-case-events` and `case-evidence-events` paths persist `kafka_offset`/`kafka_partition`/`kafka_topic` back to the source outbox row; the other three paths (Scheduler3 → `outgoing_event_outbox`; Scheduler4 → `bre_orchestration_outbox`) pass NULL entity and **do not persist Kafka metadata**. Incident correlation requires log-side join on `idempotencyId`.
- **(2026-04-25)** COMP-41 ThirdPartyNotificationConsumer writes `channel_type=GP_EVENTS` to `wdp.outgoing_event_outbox` (NOT `GF_EVENTS`). Three distinct PUBLISHED-orphan paths confirmed.
- case-action-events publisher confirmed as COMP-18 NotificationOrchestrator (Filter 1 EXPIRY_EVENT routing). cert environment uses distinct topic `case-action-events-cert` and group `case-action-events-group-cert`.
- **(2026-04-25)** COMP-43 CoreNotificationConsumer is the sole WDP component that writes to IBM DB2 Core Platform. **WDP-DB.md v2.0 incorrectly listed BC.TBC_DM_OCCUR and BC.TBC_DM_NOTES as DB2 reads** — corrected in v2.1: COMP-43 reads only `BC.TBC_DM_CASE` (enrichment is fully REST-driven). Clear-PAN column is `I_ACCT_CDH` (NOT `I_ACCT_CDR` as v1.0 stated). PAN-last-4 column is `I_ACCT_CDH_LST`.
- `wdp.file_notifications` does not exist. The actual table is `wdp.file_generation_event`. COMP-18 is the sole writer (`created_by/updated_by=WDPOUCU`, INSERT-only, status hardcoded `STAGED`).

### Confirmed DEC Violations and Voids

- DEC-011 ⛔ VOID — BRE step checkpointing confirmed never implemented in COMP-16. Formally voided in WDP-DECISIONS.md v2.0.
- DEC-014 ⛔ VOID — Resilience4j confirmed absent across all 40 component files. Formally voided in WDP-DECISIONS.md v2.0.
- DEC-004 violation in COMP-23 — clear PAN written to persistent storage on standard case creation (`wdp.CASE.I_ACCT_CDH` and `nap.case.I_ACCT_CDH`). Formally recorded as DEC-019 (Accepted Risk).
- **(2026-04-25)** DEC-004 / DEC-019 violation in COMP-43 — clear PAN written to `BC.TBC_DM_CASE.I_ACCT_CDH` (DB2) on CREATE + actionSeq=01 path after Encryption Service decrypt at Step 6 PAN-gate. Architect decision required (OQ-COMP-43-6) — document approved exception or remediate.
- DEC-001 (transactional outbox) — confirmed deviations:
  - COMP-04 (direct publish on HTTP thread)
  - COMP-15 (synchronous publish inside `@Transactional` via `kafkaTemplate.send(...).get()` — ghost-event window)
  - COMP-16 (direct synchronous publish, no outbox)
  - COMP-18 ⚠️ PARTIAL (uses `wdp.bre_orchestration_outbox` but no `@Transactional`; four distinct write points are independent auto-commits)
  - COMP-19 (direct synchronous publish; HTTP 500 on Kafka failure does not undo case action)
  - COMP-20 (direct synchronous publish)
  - **(2026-04-23 corrected)** COMP-23 — `kafka-before-commit` pattern: publish inside `@Transactional` BEFORE commit. Send failure rolls back the DB; broker ACK followed by a later commit-time exception emits a published-but-unpersisted event (narrow but non-zero window). v1.0's "publish after DB commits" claim was withdrawn.
  - **(2026-04-23 corrected)** COMP-24 ⚠️ PARTIAL — BRE publish IS inside `@Transactional` (corrected from v1.0). ActionEvent publish is OUTSIDE `@Transactional` on EP 2/8/9 — genuine post-commit split-brain when `napUpdateEvent=true`.
  - COMP-25 (synchronous publish inside `@Transactional`)
  - **(2026-04-23 added)** COMP-37 (direct synchronous publish post-DDB on primary upload; questionnaire path mitigates with `@Transactional(rollbackOn=Exception.class)`)
  - **(2026-04-25 added)** COMP-43 (separate transaction managers — outbox INSERT on `wdpTransactionManager` commits before DB2 write on `coreTransactionManager`; crash window wide open)
- DEC-020 (Accepted Risk) — no idempotency on case creation (COMP-23). Formally recorded.

### Key Confirmed Platform Facts

- BusinessRulesProcessor (COMP-16) makes direct DB calls to `nap.rules` / `wdp.rules` — does NOT call BusinessRulesService (COMP-31).
- NotificationOrchestrator (COMP-18) does NOT publish to `internal-integration-events` — that topic is published by AcceptService (COMP-19), ContestService (COMP-20), and conditionally COMP-16 (UK/NAP path).
- COMP-22 DisputeService is read-only — owns no database state, performs no writes. Kafka producer to business-rules is wired but commented out — effectively Kafka-free at runtime.
- COMP-28 DisplayCodeService does NOT determine TIER1 eligibility — returns raw code lists. Eligibility logic belongs to calling services.
- COMP-38 APILogService has no AOP inside it. Callers hold catch-blocks and invoke this service directly via REST.
- TokenService (COMP-36) is JWT management only — no relation to PAN tokenisation. Redis hash `wdpinternalidptoken:token` written by an external component not yet identified.
- COMP-33 OrgManagementService — GitHub repository not found. Documentation deferred.
- `removeItemFromQueueDisabled` flag is a second active operational safety switch in COMP-07 and COMP-08. When true, suppresses ALL MarkAsRead/ACK PUT calls globally.
- `saveChildWithMerchant` in UAMS (COMP-02) uses `@Primary wdpTransactionManager` instead of `napTransactionManager`. All three NAP tables it writes are in the NAP schema. Rollback on failure is not guaranteed. Confirmed bug.
- BEN notification is delivered via Kafka publish to a BEN-owned MSK cluster using separate SASL/JAAS credentials — NOT via REST or webhook. Confirmed from COMP-42 source.
- **(2026-04-25 — corrected)** **JustAI integration scope:** JustAI is **active in COMP-21 ChargebackService** (consumer name `JUSTTAI` triggers partner branch in `isPartner` — note double-T spelling). It **remains absent from COMP-41**. The platform-wide statement "JustAI is planned only" was correct *for outbound third-party notification (COMP-41)* but not platform-wide. Signifyd is the sole live third-party vendor for outbound notification.

### Component-Specific Source-Verified Findings (2026-04-18 / 23 / 24 / 25)

#### COMP-07 VisaDisputeBatch
- `skipCase` path writes no outbox row at all (previously implied SKIPPED was written).
- Writer save failures and MarkAsRead retry-exhaustion are caught-and-logged at `BatchItemWriter.java:139-142`, never propagated. Spring Batch sees chunk completion as successful.
- No `@Transactional` anywhere. Chunk transaction is the only transactional boundary.
- No liveness, readiness, or startup probes wired.
- Feature flags (`coreDataNeedToMigrate`, `removeItemFromQueueDisabled`, `readSpecificItemFromQueue`) have no defaults — startup fails if K8s Secret absent.
- JobLauncher is default `SyncTaskExecutor` — no custom TaskExecutor.
- `created_by` / `updated_by` constant: `"WVDPB"` (distinct from COMP-08 `"WMFDPB"`, COMP-09 `"WMPAPB"`, COMP-11 `"FILE_PROCESSOR"`).

#### COMP-08 FirstChargebackBatch
- `created_by` / `updated_by` constant: `"WMFDPB"`.
- Duplicate-check path writes SKIPPED marker rows (insert-a-marker pattern, distinct from COMP-07's suppress-insert). One SKIPPED row accumulates per re-polled known chargeback per scheduler run.
- Update-path writes `status=PENDING` with `accountNumber=null` on newly-arrived chargebacks on existing claims — `processUpdatedClaims` never sets `isAccountNumberRequired=true`. Suspected defect.
- **Writer-ACK hazard** — mid-chunk JPA save failure does not prevent the subsequent ACK PUT to MCM; ACK PUT exceptions are swallowed silently. Items acknowledged off MCM with no corresponding outbox row are unrecoverable.
- Currency exponent hardcoded (`USD_EXPONENT=2`, `CAD_EXPONENT=2`) — actual currency never consulted.

#### COMP-09 CaseFillingBatch
- `created_by` / `updated_by` constant: `"WMPAPB"`.
- **Update path always INSERTs new row** — existing rows are never UPDATEd. Multiple rows per claim through phase lifecycle (PENDING / SKIPPED).
- Writer swallows all save exceptions — chunk reports success even on DB save failure.
- Skip paths write no row at all — null-field validation, stage-determination, IDP token, encryption failures leave zero database trace.
- `c_case_stage` concatenation: base stage (`PAB`/`ARB`/`PRA`/`AII`/`AIM`) EXCEPT when `issuerCaseStatus = withdraw` → `{stage}_withdraw`. Downstream consumers must tolerate this.
- `network_notes` column: written conditionally only when `schemeRef.notes` is non-empty.
- `c_ntwk_case_id` semantics: `standardClaimId` = `claimDetail.standardClaims` if `claimType == "CaseFiling"`, else `claimId`.

#### COMP-11 FileProcessor
- `created_by` constant: `"FILE_PROCESSOR"` *(corrected 2026-04-18 from previously documented `"WPFLEPR"`)* on all three owned tables (`wdp.file_job`, `wdp.file_evidence`, `wdp.chbk_outbox_row`).
- DNWK silent-file-loss paths are **two**: `DISCHYB` (no bean registered for `DISCHYB_NETWORK` qualifier) and `AMEXOPTB` (no `FileAcroEnum` prefix entry). **`MC_REVREJ` IS registered** — v1.0 incorrectly listed it as silent-loss.
- Initial CHARGEBACK_PROCESS status is PENDING via `@PrePersist`. LOADING is transient during the evidence phase only. DCPO evidence rows are explicitly BLOCKED; DCPO parent stays PENDING. DNWK writes CHARGEBACK_PROCESS at PENDING directly with no evidence phase.
- Only **two `@Transactional` sites** in the whole repo: `insertEvidenceOutBox` and `updateStatusToPendingByFileJobIdAndStatusLoading`. Everything else is auto-commit.
- DCPO CHARGEBACK_PROCESS save is auto-commit AND the per-image evidence loop has empty catch — orphan-parent risk on any image failure.
- DNWK per-record failure path is one-retry-then-silent-row-loss — second save exception advances `rowCount` and resume logic skips the row.
- `RestTemplate` constructed with no customization (no timeouts, no pool); `IdpRestInvoker` has no `@Retryable`.
- DEC-004 non-numeric-acctNum edge case — `NetworkFileSupport` encrypts only when `acctNum` matches `\d+`; otherwise passes raw into `payload.account_number`.
- `S3ServiceImpl` silently swallows get/move/put failures and returns null.

#### COMP-12 InboundDisputeEventScheduler
- Mark-and-send within single `@Transactional` — at-least-once duplicate-possible (NOT at-most-once silent-loss as v1.0 stated).
- No K8s probes (liveness, readiness, startup all absent).
- Bare `RestTemplate` in `config/CommonConfig.java` for Scheduler5 email POST — no timeouts, no pool.
- Shared `ThreadPoolTaskScheduler` with `poolSize=5` across five schedulers — sibling-starvation risk.
- channelTypeTopicMap non-prod default keys: `EXPIRY_EVENTS`, `CORE_EVENTS`, `GP_EVENTS`, `BEN_EVENTS`.
- `idempotencyId` typed as `UUID` on `chbk_outbox_row` entity but `String` on `outgoing_event_outbox` entity — cross-table type inconsistency.
- Archive INSERT SQL references column `c_ntwrk_phase_id` not mapped on `ChbkOutboxArchiveEntity` — DDL verification required.

#### COMP-13 FileAcknowledgementProcessor
- Writes EXACTLY four columns to `wdp.file_job` enforced by `@DynamicUpdate`: `ack_status`, `ack_generated_at`, `updated_at`, `updated_by`.
- Cross-component COMPLETED-transition contract: COMP-11 never writes COMPLETED; COMP-12 Scheduler2 owns it per WDP-DB.md; COMP-13 polls for `status IN (COMPLETED, ERROR)`. Contract is undocumented in all three repos.
- Three latent runtime bugs and one third `headObject` failure branch surfaced during 2026-04-18 audit (HIGH severity).
- Zero tests of any kind in `wdp-evidence-ack-scheduler`.

#### COMP-14 CaseCreationConsumer
- Zero `@Transactional` annotations anywhere — all DB writes auto-commit. Parent SUCCESS save and EVIDENCE_ATTACH child `saveAll` are independent auto-commits.
- `auto.offset.reset = latest` — cold starts with no committed offset SKIP messages rather than replay.
- ACK at `KafkaConsumer.java:38` precedes `processKafkaNotificationEvent()` at `:43` (was `:36` in v1.0).
- Empty `CommonErrorHandler` registered — combined with `ErrorHandlingDeserializer` and pre-ACK, deserialization exceptions are a distinct silent-loss class.
- No liveness probe, no startup probe — only readiness. Stuck listener thread will not restart pod.
- Layer-1 prior-chargeback dedup is in-memory stream filter over a 2-column query (`networkCaseId + cardNetwork`). Safe only by `concurrency=1` operational constraint.
- Cert profile naming: `new-case-events-group-cert` (v1.0 incorrectly used `-dev`).
- Confirmed NOT a publisher of `business-rules` (closes Observability OQ-02).

#### COMP-15 EvidenceConsumer
- V3 path defect: MISCDOC / DRFTDOC / RESPQDOC / ISSRQDOC document types silently mark `file_evidence.attachment_status=ATTACHED` without any upload occurring — data-integrity defect, not "unfinished state" as v1.0 implied.
- V3 RESPDOC path: failed ownership-transfer PATCH leaves the document uploaded to V3 Core but the WDP DB unmarked — ghost-upload failure mode. No automatic reconciliation.
- WDP-path RESPDOC publishes **two** BR events per success (`ACTION_UPDATED` then `DOCUMENT_UPLOADED`, sorted alphabetically), sent synchronously inside `TXN_WDP` via `kafkaTemplate.send(...).get()`.
- DSNOTDOC WDP events publish with `source=""` — builder has no DSNOTDOC branch (suspected downstream-routing defect; COMP-16 tolerance OQ).
- V3 Update Action PATCH mutates the shared `RestTemplate` bean's request factory — after the first V3 PATCH call, every subsequent REST call on the pod inherits 30s/30s timeouts. Order-dependent global state.
- Inbound Kafka key (`caseNumber`) and `idempotency-key` header are both `@Nullable`. Nulls pass through to DMS upload and downstream `business-rules` message.
- WDP-path gating is narrower than prior documentation implied — IDP fetch, required-field validation, multi-doc sibling scan all skipped on V3 legacy path.
- `coreMigrationFlag` sourced from env var `${core_migration_flag}` via K8s secrets/configmap; fixed at container start.
- Platform string is uppercased on publish (`setPlatform(platform.toUpperCase())`).

#### COMP-16 BusinessRulesProcessor
- Pre-ACK at `KafkaConsumer.java:38`; processing at `:41` *(corrected from v1.0's `:37`/`:40`)*.
- FCMG guard correction: actual guard is `FCHG / RCAL` pairing (FirstChargeback paired with `RCAL`), NOT `FCMG`. On guard failure, UK code calls `updateCase(...)` at `NAPRulesProcessServiceImpl.java:207`. The "queue to WEXQUE" v1.0 behaviour is not supported by source — removed.
- US DB bean name: `wdpdataSource` (lowercase `d`, NOT `wdpDataSource`).
- UK transaction manager: `ukTransactionManager` (NOT `napTransactionManager`).
- Token cleanup asymmetric — UK `tokenService.clear()` INSIDE finally; US `tokenService.clear()` OUTSIDE finally.
- US-only `DOCUMENT_ATTACHED_TO_OPEN_CASE` recursive fallback: when issuer-doc check returns `issuerDocAddedToCase=true` and matched rule's `applyRuleGroup` is blank, US path recursively evaluates rules under `DOCUMENT_ATTACHED_TO_OPEN_CASE`.
- `ErrorLogService` class does not exist in source tree — autowire and call sites commented out.
- `isErrorOccured` flag from outgoing publish is ignored by caller — publish failures surface only as SNOTE via REST.
- Audit-log sequences: `nap.br_case_audit_log_id_seq`, `wdp.br_case_audit_log_id_seq`, both `allocationSize=1`. No `@Transactional` on `saveAuditLog`.

#### COMP-17 CaseExpiryUpdateConsumer
- Kafka headers are kebab-case on the wire: `idempotency-key`, `event-timestamp` (NOT camelCase).
- `max.poll.records=500` and `max.poll.interval.ms=600000` (10 min) *(corrected from v1.0's `280` / `300000`)*.
- Cert environment uses distinct topic `case-action-events-cert` and group `case-action-events-group-cert`.
- SASL: `AWS_MSK_IAM` over `SASL_SSL`.
- Predecessor query is **not scoped on `i_action_seq`** — a stuck row for action A blocks events for action B on the same case (cross-action interference).
- Predecessor blocking: any non-ERROR prior row (FAILED, PENDING, PENDING_DEFERRED, BLOCKED, PUBLISHED) maps to PENDING_DEFERRED. Predecessor candidate filter excludes SUCCESS and SKIPPED.
- Outbox writes have **no service-level `@Transactional`**; `case_expiry` writes are `@Transactional REQUIRED`. Both share `wdpTransactionManager` but no method brackets both — non-atomic split unrecoverable in either path.
- Poison-message handling: payload that fails JSON deserialisation causes NPE at path-selection branch BEFORE any ACK in either path. Same poison message redelivers indefinitely until `max.poll.interval.ms` (10 min) trips a rebalance.
- `v-correlation-id` not propagated on IDP token call — only on case-search call. Correlation chain breaks at IDP boundary.
- No liveness, readiness, or startup probes wired. No MDC context, no custom Micrometer metrics.
- Six fully unused + 1 partially-wired pom dependencies (`spring-boot-starter-cache` enabled but no `@Cacheable` consumers).
- Dedup detection key fields: `(idempotencyId, channelType, eventTimestamp)` — confirmed.
- No `@SchedulerLock`, advisory lock, or singleton guard. Replica > 1 relies entirely on Kafka consumer-group rebalance.

#### COMP-18 NotificationOrchestrator
- Step 6 previous-event guard is two-stage: SQL `WHERE caseId=?` returns all rows; `id<eventId`, `component`, and `status NOT IN (SUCCESS, SKIPPED)` filters applied in-memory in Java. Scales poorly for long-lived cases.
- Filter 4 Sub-condition B stage+action set is `{RE2+REPR, PAB+MDCL, REQ+RRSP, ARB+MDCL}` — REPR not REFR (corrected).
- `documentNameList` IS published on `PublishedNotificationEvent` when set (v1.0 incorrectly listed as absent).
- Has `readinessProbe` and `livenessProbe` configured at `/merchant/gcp/outgoing-event/actuator/health` port 8082, `initialDelaySeconds=120`. NO `startupProbe`. v1.0 wrongly implied all probes absent.
- Default Actuator exposure only — `management.endpoints.web.exposure.include` not set.
- Zero `@Transactional` annotations. Four distinct outbox write points (Step 3a ERROR, 3d PUBLISHED, 6 ERROR/PENDING_DEFERRED, 7e SUCCESS/FAILED) are independent auto-commits. Outbox INSERT at Step 3d not atomic with Kafka offset commit at Step 4 nor with Step 7 publishes.
- Sole writer of `wdp.file_generation_event` (`created_by/updated_by=WDPOUCU`, INSERT-only, status hardcoded `STAGED`, columns left NULL on INSERT: `batch_id`, `next_retry_at`, `error_code`, `error_message`).
- MDC correlation not wired — `HttpInterceptor` puts MDC but is never registered.
- Dead code to remove: `findByIdempotencyIdAndComponent`, `HttpInterceptor` class, `RequestCorrelation` ThreadLocal, `org.apache.httpcomponents:httpclient` dependency.
- FNS file-notification-service code is live awaiting downstream COMP-44 EDIA Consumer to be built.

#### COMP-19 AcceptService
- Stage code is **CHI** (letter I) throughout all three validators — NOT `CH1` as v1.0 stated.
- VISA PAB permits **WDNL** (in addition to IDCL/IPAB/CHGM). VISA APC uses **PCMP** (not POMP).
- `expiryFlow=true` EACP override applies to first action of outgoing `AddActionRequest` for ALL card networks (not Visa-only). It mutates AddActionRequest only — does NOT affect Kafka publish gate.
- MC CHI silent no-op applies to **both NAP and PIN** (not PIN only).
- `errorLogService.saveErrorLog` — 2 of 9 call sites are ACTIVE (`CaseServiceImpl` Step 6 case-action-add; `MasterCardServiceImpl` Step 5d MC PIN claim lookup). v1.0 incorrectly claimed all sites commented.
- Liveness probe: `GET /merchant/gcp/accept/livez`; readiness: `/readyz` (corrected from v1.0's `liver` / `ready`).
- **🔴 NEW HIGH split-brain finding:** MC CHI on NAP can publish `AcceptEvent` without MC network notification — `MasterCardServiceImpl.accept` silently returns for CHI; Step 8 Kafka gate still fires if inbound actionCode is `FCHG/IPAB/IARB/IDCL`. NAPOutcomeProcessor delivers acceptance to NAP-DPS while Mastercard was never asked.
- **🔴 NEW HIGH split-brain finding:** AMEX/DISCOVER on NAP can publish `AcceptEvent` without network notification. `cardNetwork` switch defaults to `log.warn` for AMEX/DISCOVER.
- **🟠 NEW HIGH:** Compensation has no inner try/catch — `updateAction(ERROR)` and `addErrorNotes` called without defensive handling. Secondary failure replaces original business exception in HTTP response, masking root cause. Likely shared with sibling COMP-20.
- Add-case-action runs AFTER the network call (not before). Compensation `updateAction(ERROR)` targets the original action sequence, not a newly-added row.
- Kafka gate actionCode is sourced from `CaseLookupResponse`, NOT from rules/AddActionResponse.

#### COMP-20 ContestService
- Max Kafka publishes per request is **2**, not 3 — CRMR branches are `if/else if`, mutually exclusive.
- Card-network failures return **HTTP 400** (wrapped as `BadRequestException`), not HTTP 500. Only Kafka publish exhaustion and internal `RuntimeException` paths return HTTP 500.
- Liveness probe path is `/livez`, not `/lives`.
- ErrorLogService commented-out call sites: 8 (not 6).
- CAD/USD commented-out currency blocks: 5 across all five Visa submission methods (not 1).
- DEC-014 reclassified: absence-of-library fact, not a deviation — consistent with platform-voided DEC-014.
- MCM submission contract documented: `createSecondPresentmentChargeback` (CH1/RE2 — POST) and `retrieveClaim + updateCase(action=REJECT)` (PAB — GET then PUT).
- DataPower non-NAP path uses same VANTIV license header as Visa DataPower (shared `licenseKey` secret).

#### COMP-21 ChargebackService
- 38 distinct downstream call sites across 12 target applications. 4 are dead code (3 unreachable, 1 dead `@Value` field).
- **No IDP token cache** — `CachedTokenServiceImpl` delegates straight through to a fresh GET on every call. WDP-path `contest` produces ~10 sequential round-trips on a no-pool `RestTemplate`.
- `asyncExecutor` is core=1, max=1, queue=5. The `CompletableFuture.allOf` parallel ACL+lookup pattern provides no real parallelism. 7th concurrent action request hits `RejectedExecutionException`.
- `IdpRestInvoker` mutates the shared `RestTemplate` `setErrorHandler` per token call — concurrency hazard on a global bean.
- Logstash appender effectively broken — production secret `logstash_server_host_port` is empty. Only stdout logs reach aggregation.
- Surfaces full `cardNumber` in two model classes (`SearchCaseList`, `Transaction`); `cardNumberLast4` in eight others. Populated from downstream `case-search-service` responses, no masking transformation in this service before serialisation.
- Dead config: `wdp.caseUpdateUrl` injected as field but never read.
- LATAM silent fall-through — `SourceSystem.LATAM` matches no controller branch; action requests return HTTP 200 empty body with no log, no metric, no error.
- `JustAI` IS active (consumer name `JUSTTAI` with double-T triggers partner branch).
- Rollout `maxSurge=4` (not `maxSurge=1`).
- `/actuator/prometheus` requires JWT — NOT in security whitelist.
- Confirmed Kafka-free (zero `@KafkaListener`, `KafkaTemplate`, `@EnableKafka`, `kafka.bootstrap` occurrences).
- Confirmed DB-free (no JPA, JdbcTemplate, DataSource references; classpath has neither `spring-boot-starter-data-jpa` nor `spring-boot-starter-jdbc`).
- Enterprise Cert eAPI Entitlement Lookup (`https://cert-eapi.../entitlement-lookup-api/v1/api-key-entitlements`) — Bearer (forwarded inbound JWT). Currently inactive in production (`entitlementFlag=false`) but wired and ready.

#### COMP-23 CaseManagementService
- `kafka-before-commit` pattern: publish inside `@Transactional` BEFORE commit (corrected from v1.0's "after commit").
- **Owns 6 tables, not 7** — `wdp.dispute_event_change_log` REMOVED; source grep confirms zero references in `mdws-gcp-case-management-service` repo.
- NAP `nap.case` PAN column is `I_ACCT_CDH` (NOT `I_ACCI_CDH` as v1.0 had — typo).
- US create path duplicate-`wdp.NOTES`-insert defect: two identical `USNotesEntity` save blocks execute when `notesRequest != null`, producing two rows per create. Defect not fixed.
- NAP create path blind-merge on `NAP.DISPUTE_EVENT_CONSUMER_ERROR`: calls `.save()` on freshly-constructed `ConsumerErrorEntity` without prior `findById` guard. Cross-component write into COMP-05-owned table with no owner check.
- `wdp.chbk_outbox_row` update path uses `findById(...).isPresent()` guard (no-op if missing). No terminal-status guard — can overwrite PROCESSED row. Same `@Transactional` as case save.
- No Flyway / Liquibase / `schema.sql` / `data.sql` in repo. DDL managed elsewhere.
- Owner of case-number sequences: `nap.case_i_case_sequence` (NAP) and `wdp.pin_case_i_case_sequence` (shared by PIN, CORE, VAP, LATAM).
- `RequestCorrelation` ThreadLocal leak — latent cross-request contamination on pooled Tomcat worker threads.
- `RestTemplate` mutation pattern — `setErrorHandler` per call on shared bean.
- `spring-boot-devtools` shipping in COMP-23 prod image — dev-time class-path scanning may execute in production.
- `getRandomDigits` returns null once sequence length + prefix + random alpha reaches 12 — NPE risk; deterministic once sequence grows.

#### COMP-24 CaseActionService
- BRE Kafka publish IS INSIDE `@Transactional` (NOT post-commit split-brain as v1.0 claimed).
- ActionEvent publish is OUTSIDE `@Transactional` on EP 2 / 8 / 9 — genuine post-commit split-brain when `napUpdateEvent=true`.
- FULL_CTM action code is **CHGM** (not `CHMR` as v1.0 stated).
- NAP conditional outbox writes to `nap.DISPUTE_EVENT_CONSUMER_ERROR` (NOT `wdp.chbk_outbox_row` as v1.0 implied).
- `wdp.dispute_event_change_log` does not exist in this repo — `EventChangeLogRepository` v1.0 claim withdrawn; dormant-table row removed from WDP-DB.md.
- Open-action constraint enforced in-memory only — race window on concurrent POSTs for same caseNumber.
- Last-write-wins on shared case/action tables — no `@Version`, no pessimistic lock, no advisory lock.
- NAP vs US functional asymmetry on EP 5: `UKCaseActionDaoImpl.updateAction` does not process `chbkOutbox` on any branch — NAP silently ignores `chbkOutbox` field that US handles on six branches.
- EP 9 three-transaction sequence with no compensation: insert RRSP action commits independently → Document Service POST → `merchantDocIndicator="Y"` update commits as separate transaction → optional ActionEvent post-commit. Failure at any step after Transaction 1 leaves RRSP action in DB without full completion.
- No connection or read timeouts on any outbound REST. Kafka producer has no client-level retry cap or timeout (Kafka client defaults).
- UAT IDP token URI hardcoded as default in `application.yml` line 42 — only client-id and client-secret externalised.
- `idempotency-key` captured by HttpInterceptor and forwarded on Kafka outbound but never validated server-side. No seen-key store.

#### COMP-27 CaseSearchService
- **15 REST endpoints** across 6 controllers (not 14 as v1.0 stated).
- **Two write operations** — V1 PATCH and V2 POST case-lock endpoints. Both target lock columns only on `nap.case` / `wdp.case` with no `@Transactional`, no `SELECT FOR UPDATE`, no optimistic version. Concurrent-locker race is unguarded.
- Platform-wide instance of DEC-014 VOID pattern — no Resilience4j, no circuit breaker, no timeout, no retry on any outbound REST call.
- `migrationStatus` mapping is LIVE (v1.0's "suppression" claim withdrawn). Real finding: SELECT list has TWO `as migrationStatus` aliases on different source columns (`A.c_migration_sta` and `A.o_migration_sta`) — second overrides first in most JDBC drivers.
- Endpoint count corrections: V1 default `pageSize` is 25 (not 200); V2 `startRecordNumber`/`pageSize` declared as String (not Integer); JWT claim is `iqorgid` (not `igorgid`).
- Ingress host count: 4 (not 6).
- Rollout `maxSurge=4` (not 1).
- `/progress` endpoint creates 4 futures but joins only 2 — `caseDetails` and `documentResponse` are created and orphaned (in-flight DB and downstream calls consumed but completion never awaited).
- NAP/WDP lock-timestamp column-name drift: NAP uses `x_locked_at`; WDP uses `z_locked_at`.
- `RECEIVED_DOCUMENT` queue criterion enum returns literal SQL identifier with space character — emits invalid SQL at runtime.
- `value` column content concatenated into SQL by `QueueSupport.buildQueueCriterion` — latent SQL-injection surface.

#### COMP-37 DocumentManagementService
- **IS a Kafka producer** (5th, now 6th publisher of `business-rules`) — was previously misclassified as Kafka-free. Source-verified 2026-04-23.
- Primary upload step order: `S3 → DDB → desk update → Kafka publish → action-indicator update`. Action-indicator update is LAST step — making action-indicator staleness the terminal failure mode if Kafka succeeds but final REST call fails.
- Endpoint 11 (POST questionnaire) is the only publish path in the platform that wraps Kafka send inside `@Transactional(rollbackOn=Exception.class)` — stronger atomicity than other publishers on `business-rules`.
- Endpoint 8 NAP base64 path NEVER publishes (controller forces `notifyBRQueue=false`).
- `USCaseEntity` and `UKCaseEntity` are column-level UPDATE co-writers only (desk blanking). INSERT/DELETE ownership likely belongs to COMP-22 — requires cross-component confirmation.
- Five DynamoDB GSIs declared (`C_STAGE_CODE`, `I_ACTION_SEQ`, `C_DOC_TYPE`, `N_DOC_NAME`, `Z_UPDT`) but queries none from Java — either external consumer dependency or over-provisioning.
- DEC-003 reframed: NOT "inconsistent per code path" but "uniformly non-compliant with legacy dead code". Legacy `sendBusinessRules(merchantId)` method has 0 callers; every reachable publish uses `caseNumber`.
- Endpoint 11 HTTP verb is POST (not PUT as v1.0 claimed).
- DynamoDB duplicate check is application-level query + in-memory compare, non-atomic — concurrent identical uploads both land.
- `idempotency-key` header accepted, logged, MDC-tagged, echoed, propagated as outbound Kafka record header but NOT used for deduplication at any write site.

#### COMP-41 ThirdPartyNotificationConsumer
- Writes `channel_type=GP_EVENTS` (NOT `GF_EVENTS` as v1.0 stated).
- Three distinct PUBLISHED-orphan paths — all unrecoverable in-component, all invisible to COMP-12 Scheduler3 if Scheduler3 reads only FAILED / PENDING_DEFERRED:
  - (a) post-ACK crash before Signifyd response
  - (b) Signifyd empty body (`"NO_DATA_FROM_SIGNIFYD"`) — no status transition
  - (c) final outbox UPDATE failure after ACK
- `@Cacheable("displaycodedetails")` and `@Cacheable("notificationRule")` are silent no-ops — `@EnableCaching` is absent, no `CacheManager` bean. Every event hits upstream Display Code POST and Notification Rule GET.
- Zero `@Transactional` annotations. Every save is independent auto-commit.
- No K8s liveness, readiness, or startup probes despite Actuator exposing `/livez` and `/readyz` paths on port 8082.
- Spring Retry imports (`@Retryable`, `@Backoff` in `SignifydService`, `TokenServiceRetry`) are dead — never applied. Class names containing "Retry" describe custom try/catch, not Spring Retry framework.
- `auto.offset.reset = latest` — cold start with no committed offset skips backlog.
- Predecessor lookup loads ALL outbox rows for the case, then filters in Java memory — unbounded memory and latency for long-lived cases.
- Two `RestTemplate` construction patterns: one shared `@Bean` plus two `RestInvoker` methods that bypass the bean and create their own bare `RestTemplate` locally.
- **Zero references to JustAI anywhere** — confirmed by absence audit across `src/`, all YAML, POM, properties, tests.

#### COMP-43 CoreNotificationConsumer
- DB2 PAN column name is `I_ACCT_CDH` (NOT `I_ACCT_CDR` as v1.0 stated). PAN-last-4 column: `I_ACCT_CDH_LST`.
- DB2 reads only `BC.TBC_DM_CASE` for enrichment (NOT `BC.TBC_DM_OCCUR` and `BC.TBC_DM_NOTES` as v1.0 listed). Enrichment is fully REST-driven.
- Step 6 4xx classification: only HTTP 400 OR 404 → ERROR; all other status codes (other 4xx, 5xx) → FAILED.
- STOP_DUP path: silent skip + ACK + no outbox write (v1.0 mermaid showed STOP_DUP terminating without ACK — corrected).
- `eventId` field type: `Long` (boxed, nullable) — not `String/null` as v1.0 stated.
- No K8s liveness / readiness / startup probes. Pod readiness governed only by `minReadySeconds: 30`.
- No `management:` block in any YAML profile — Spring Boot defaults expose only `health` and `info`. Prometheus / metrics not exposed.
- `created_by` / `updated_by` hardcoded to `"PCSECRTC"` on every outbox write.
- `next_retry_at` set on every non-SUCCESS update; `retry_count` increments only on incoming `status==FAILED`; ERROR escalation on third FAILED write.
- `original_event` JSON column carries fields not used by COMP-43 (`actionStatus`, `expirationDate`, `responseDueDate`, `dateReceivedByAcquirer`, `level1Entity`–`level5Entity`, `disputeStage`, `channelType`).
- No Kafka-metadata write-back columns — incident correlation requires log-side join on `idempotencyId`.
- `actionSequence == "01"` comparison is string equality `equalsIgnoreCase("01")`. No leading-zero normalisation. Upstream publishing `"1"` would skip PAN decryption AND treat case as subsequent occurrence.
- DataSource bean qualifiers `coredataSource` / `wdpdataSource` (lowercase, single-word) plus `@Primary` on WDP datasource.
- Idempotency check SELECT and INSERT run in separate JPA short transactions (`processNewCaseActionEvent` not `@Transactional`). No DB-level UNIQUE visible.
- UPDATE path runs `coreCaseRepository.save` outside any explicit `@Transactional` — Spring Data wraps in per-call short coreTx.

### Enterprise Shared Services (not WDP owned)
- Akamai — CDN and edge security (Merchant Portal only)
- APIGEE — B2B API gateway for external merchants
- IDP — enterprise OAuth 2.0 identity provider (SunGard for NAP)
- Sterling Mailbox — universal file aggregation hub
- ControlM — on-premise file transfer agent (bridges Sterling and S3)
- EDIA platform — enterprise Kafka streaming (not owned by WDP)
- MS SQL Server (legacy fax system) — shared with legacy systems outside WDP

### WDP Owned On-Premise
- DM Mainframe — mainframe-to-mainframe file transfers
- File Transfer Batch — DiscoverHybrid special pull via SFTP

### Planned Work (not yet built)
- COMP-33 OrgManagementService documentation (no repo found)
- COMP-44 EDIA Consumer (strategic outbound route)
- COMP-45, 46, 47 File Generation components (next sprint)
- COMP-48 NYCE File Generation Processor
- COMP-49, 50 UI Portal documentation (separate action item)
- Dashboard Section in both portals
- NAP inbound migration to common chbk_outbox_row path
- NAP outbound migration from direct NAP-DPS API to EDIA route
- LATAM full integration
- VAP full integration
- EDIA route migration for NAP Outcome Processor (COMP-39)
- JustAI third-party notification integration (COMP-41 planned extension — distinct from active JustAI handling in COMP-21)

---

## Open Questions Requiring Architect Confirmation

### Resolved 2026-04-18 / 23 / 24 / 25 (removed from open list)

The following previously-open questions are now resolved:
- COMP-14 publishes to `business-rules`? → **Resolved: NO** (no `KafkaTemplate`, no `ProducerFactory`, no topic reference in source)
- COMP-23 non-atomic cross-datasource write to `wdp.dispute_event_change_log` from NAP create path → **Resolved: WITHDRAWN** (table not referenced in source)
- COMP-23 Kafka publish ordering relative to commit → **Resolved: kafka-before-commit pattern (inside `@Transactional`)**
- COMP-24 BRE Kafka publish ordering → **Resolved: inside `@Transactional`** (post-commit only on ActionEvent EP 2/8/9)
- COMP-15 Step 6 `@Transactional` boundary → **Resolved: `wdpTransactionManager`, `rollbackFor=Exception.class`**
- COMP-15 `isMultiDocPending` scan semantics → **Resolved**
- COMP-16 ACK line numbers → **Resolved: line 38/41 (not 37/40)**
- COMP-12 transaction semantics (mark-before-send) → **Resolved: in-TX-with-send, at-least-once with duplicate-possible**
- COMP-17 publisher of `case-action-events` → **Resolved: COMP-18 NotificationOrchestrator (Filter 1 EXPIRY_EVENT routing)**
- COMP-17 dedup detection key fields → **Resolved: `(idempotencyId, channelType, eventTimestamp)`**
- COMP-37 Kafka involvement → **Resolved: IS a producer** (was misclassified)
- COMP-41 channel_type value → **Resolved: `GP_EVENTS` (not `GF_EVENTS`)**
- COMP-41 `@Cacheable` actually caching? → **Resolved: NO — silent no-op**
- COMP-41 Spring Retry governs outbound REST? → **Resolved: NO — imports dead**
- COMP-43 PAN column name → **Resolved: `I_ACCT_CDH` (not `I_ACCT_CDR`)**
- COMP-43 DB2 read scope → **Resolved: only `BC.TBC_DM_CASE` (enrichment is REST-driven)**

### Currently Open

| Question | Source | Action needed |
|----------|--------|---------------|
| COMP-24 ActionEvent topic name (`${kafka.topic}`) and consumer identity | COMP-24 Kafka gap | Confirm from deployment config; likely COMP-39 NAPOutcomeProcessor |
| `wdp.case_expiry` downstream consumers — which components read this table? | COMP-17 OQ | Cross-component sweep needed: Copilot question on every other WDP repo for `case_expiry` references |
| COMP-36 TokenService — which external component writes Redis hash `wdpinternalidptoken:token`? | COMP-36 OQ | Team confirmation — TokenService repo does not contain Redis writer |
| Scheduler3/4 query FAILED/PENDING_DEFERRED only — intentional or gap? | COMP-12 risk | Architect decision — does any process re-drive PUBLISHED orphans? |
| Scheduler3 channel_type filter behaviour (OQ-COMP41-1) | COMP-41 OQ | Architect / Claude Code follow-up on COMP-12 source — does Scheduler3 filter consistently? Does it ever read PUBLISHED? |
| `nap.queues` / `nap.queue_criterion` ownership | COMP-27 / COMP-31 OQ | Team confirmation. No DDL, entity, or migration in any audited repo |
| External VAP/LATAM entity-scope authorization absent in COMP-27 | COMP-27 OQ | Architect — intentional (internal-only platforms) or gap? |
| COMP-27 `/lft` `isInternal` derives from request body, not JWT | COMP-27 security finding | Architect review |
| COMP-23 PAN-clear remediation timeline (DEC-019) | COMP-23 risk | Team confirmation — target quarter to move encryption into standard create |
| COMP-43 PAN-clear in DB2 — intentional approved exception or remediate? (OQ-COMP-43-6) | COMP-43 risk | Architect decision — document approved exception in WDP-DECISIONS.md or remediate |
| COMP-19 split-brain on MC CHI / AMEX / DISCOVER on NAP — fail-close, accept-and-document, or remediate? (OQ-COMP-19-9) | COMP-19 finding | Architect decision — same severity class as DEC-019/020 risk-accepted ADRs |
| COMP-17 cross-action predecessor scope — intentional (case-level ordering) or bug (action-level)? | COMP-17 OQ | Architect decision |
| COMP-17 non-atomic outbox + `case_expiry` split — accept or remediate via single transaction bracketing? | COMP-17 OQ | Architect decision |
| COMP-26 POST idempotency gap — duplicate POSTs insert new rows | COMP-26 risk | Architect decision — DB unique constraint on (I_CASE, I_ACTION_SEQ) needed? |
| COMP-37 RESPDOC→source divergence: primary upload `BRRSUP`, questionnaire path `BRMRUP` | COMP-37 OQ | Architect decision — intended behaviour or bug? Requires COMP-16 verification |
| COMP-37 five DynamoDB GSIs declared but unused from Java — which external consumer depends on them? | COMP-37 OQ | Architect / data team confirmation; projection type per GSI |
| COMP-15 V3-path MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC silently mark `file_evidence=ATTACHED` — fix as upload path or explicit reject? | COMP-15 OQ | Architect decision |
| COMP-15 DSNOTDOC publishes `source=""` — does COMP-16 misroute? | COMP-15 / COMP-16 OQ | Claude Code follow-up on `wdp-business-rules-processor` |
| `wdp.chbk_outbox_row.status=PENDING_DEFERRED` re-driver ownership — which component re-drives? | COMP-15 OQ | Cross-component scan; if no re-driver, this is a latent stuck-record source |
| `wdp.outgoing_event_outbox` DB-level UNIQUE constraint on `(idempotency_id, channel_type, event_timestamp)` | COMP-17 / COMP-41 / COMP-43 OQ | DBA confirmation — no DDL in any component repo |
| `nap.rule_group`, `nap.rule_criteria`, `nap.rule_action_field` write ownership | COMP-31 gap | Confirm whether COMP-32 RulesService or DBA scripts own these |
| `wdp.api_route` table — which component owns writes? | COMP-01 gap | Confirm from team or infrastructure config |
| File job terminal status PROCESSING vs COMPLETED — COMP-11 never writes COMPLETED, COMP-13 polls for COMPLETED | COMP-11/12/13 contract | Architect decision — is ACK generation for successful file jobs broken? Cross-component contract undocumented in all three repos |
| `wdp.chbk_outbox_row` DB-level UNIQUE on `(file_job_id, event_type, row_number)` and `(c_ntwk_case_id, c_ntwk_phase_id)` | COMP-07/08/09/11 risk | DBA confirmation |
| COMP-08 SKIPPED-marker accumulation boundedness — does Scheduler2 archive SKIPPED rows? | COMP-08 risk | Confirm Scheduler2 archive query |
| COMP-08 update-path PENDING-without-HPAN — defect or design? | COMP-08 risk | Architect decision |
| COMP-08 writer-ACK hazard — accept or remediate? | COMP-08 risk | Architect decision |
| Production replica count for **every** continuously-running component (COMP-07/08/09/11/12/14/15/16/17/18/19/20/21/23/24/27/37/41/42/43) — XL Deploy placeholders | All components | Environment config / team confirmation |
| COMP-21 missing IDP token cache — defect in `CachedTokenServiceImpl`, or intentional? | COMP-21 OQ | Architect decision |
| COMP-21 `asyncExecutor` core=1/max=1/queue=5 sizing — intentional or leftover defaults? | COMP-21 OQ | Architect decision |
| COMP-21 empty `logstash_server_host_port` and stdout-only logging — intentional? | COMP-21 OQ | Team confirmation |
| COMP-29 FaxQueueService eViewer License proxy — runbook? | COMP-29 gap | Team confirmation |
| COMP-03 `validateOrgId()` commented out on GET /orgentity | COMP-03 security | Raise formal ADR |
| COMP-22/COMP-37 cross-component ownership of `USCaseEntity`/`UKCaseEntity` — INSERT/DELETE owner vs column-level UPDATE co-writer | COMP-37 OQ | Cross-component review |
| Production cron values (COMP-12 five schedulers, COMP-07/08 batch crons) | All schedulers | K8s secret confirmation |
| Archive column `c_ntwrk_phase_id` — exists in DDL? | COMP-12 OQ | DBA / schema-owner confirmation |
| OQ-FileJob-COMPLETED-Owner (cross-component contract) | COMP-11/12/13 | Architect decision; Claude Code question on COMP-12 source |

---

*End of WDP-HANDOVER.md v3.1.*
*Reconciled 2026-04-25 from 20 Pending Entries in WDP-CHANGE-LOG.md.*
