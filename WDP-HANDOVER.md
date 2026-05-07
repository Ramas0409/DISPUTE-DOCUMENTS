# WDP-HANDOVER.md
**Worldpay Dispute Platform — Architecture Session Handover**
*Version: 3.2*

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
- Always check NFRs before validating any architecture decision
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
  WDP-COMP-INDEX.md        Master component registry — 51 components, 01–51
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
| WDP-ARCHITECTURE.md | ✅ Current — v2.2 | Baseline. 27 Open Discussion Points captured for architect decision. |
| WDP-DECISIONS.md | ✅ Current — v2.2 | 23 active DECs. 37 candidate ADRs (ADR-CAND-023 through ADR-CAND-059) awaiting architect promotion. |
| WDP-INTEGRATIONS.md | ✅ Current — v2.2 | All external contracts reconciled. |
| WDP-NFRS.md | ✅ Current — v2.2 | RISK-001 through RISK-194. |
| WDP-HANDOVER.md | ✅ Current — v3.2 | This file. |
| WDP-CHANGE-LOG.md | ✅ Current | Baseline established. Pending Entries currently empty. |

### Tier 2 — Reference Indexes

| Document | Status | Notes |
|----------|--------|-------|
| WDP-COMP-INDEX.md | ✅ Current — v2.2 | All 51 components registered. 38 source-verified. |
| WDP-KAFKA.md | ✅ Current — v2.2 | All Kafka topics, consumer groups, and outbox channels documented. |
| WDP-DB.md | ✅ Current — v2.2 | All schema ownership reconciled. |
| WDP-FLOW-INDEX.md | ✅ Current — v1.1 | 11 flows identified, 0 documented. |
| WDP-COMP-TEMPLATE.md | ✅ Created — v1.1 | Master template with 4 type blocks. |

### Tier 3 — Individual Component Files

| Status | Count | Details |
|--------|-------|---------|
| ✅ COMPLETE (architect-confirmed) | 0 | |
| 📝 DRAFT 🔍 (source-verified, architect confirmation pending) | 40 | COMP-01, 02, 03, 04, 05, 06, 07, 08, 09, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 30, 32, 35, 36, 37, 39, 40, 41, 42, 43, 49, 50, 51 |
| 📝 DRAFT (Copilot CLI analysis, source-verification pending) | 4 | COMP-29, 31, 34, 38 |
| 📋 PENDING (enterprise-owned, lower priority) | 1 | COMP-10 DM Mainframe |
| ⬜ NOT STARTED | 6 | COMP-33 (OrgManagementService — no repo found), COMP-44 (EDIAConsumer — planned), COMP-45, COMP-46, COMP-47 (File Generation — planned next sprint), COMP-48 (NYCEFileGenerationProcessor — planned) |
| 🔲 UI — separate action item | 0 | (none — COMP-49/50 promoted to DRAFT 🔍 v3.0 May 2026) |

**Component migration status.** All 51 components are registered. 40 of 51 have undergone source-verified correction passes against their production repositories. All carry "📝 DRAFT 🔍 — architect confirmation pending" status. Architect confirmation is the next gate for these 40 components.

### Tier 4 — Workflow Documents

No workflow files exist yet. 11 flows identified in WDP-FLOW-INDEX.md. Two "Per-Flow Authoring Notes" captured in the index: FLOW-07 Visa contest end-to-end variant; FLOW-11 Case Expiry Subsystem coordination. Workflow documentation is a parallel work stream.

---

## Current Work Position

**Phase:** Post-baseline. v2.2 documentation set is the stable platform baseline. The source-verification phase is complete for 38 of 51 components.

**Active workstreams:**

1. **Architect confirmation rounds** — promote 📝 DRAFT 🔍 component files to ✅ COMPLETE as architect signs off (38 components ready).
2. **Architect ADR decisions** — 37 candidate ADRs (ADR-CAND-023 through ADR-CAND-059) await promotion to active DECs. Highest-stakes 11 candidates flagged in WDP-DECISIONS.md.
3. **Source-verification of remaining 4 components** — COMP-29, 31, 34, 38.
4. **Workflow documentation** — author the 11 WDP-FLOW-*.md files identified in WDP-FLOW-INDEX.md.
5. **Open architectural questions** — 27 platform-level discussion points captured in WDP-ARCHITECTURE.md for architect resolution.

**Reconciliation rhythm going forward:** per-component chats append Pending Entries to WDP-CHANGE-LOG.md; reconciliation sessions consume Pending Entries and rebuild affected derivatives. See WDP-CHANGE-LOG.md for the workflow detail.

---

## Component Numbering

Components are numbered 01–51. Numbers are permanent.
See WDP-COMP-INDEX.md for the full registry.

**Current last number: 51**
**Next available: 52**

When adding a new component each quarter:
1. Assign number 52 (or next available)
2. Add to WDP-COMP-INDEX.md registry
3. Create WDP-COMP-[NN]-[NAME].md using the template
4. Append a Pending Entry to WDP-CHANGE-LOG.md (don't rewrite derivatives inline)


---

## How to Start a New Session

**Option A — Architect ADR decision session:**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first, then WDP-DECISIONS.md.
Today I want to make a decision on [ADR-CAND-NNN topic]."
```

**Option B — Starting a workflow documentation session:**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first, then WDP-FLOW-INDEX.md.
Today I want to document the [FLOW-NAME] workflow.
Start with the Mermaid sequence diagram."
```

**Option C — Starting a new architecture decision session:**
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

**Option E — Per-component source-verification chat (Phase 2):**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first, then WDP-COMP-NN-NAME.md.
Today we are completing Phase 2 source verification on
COMP-NN. At end of Phase 2, append Pending Entry to
WDP-CHANGE-LOG.md."
```

**For all sessions:** Claude always searches project knowledge before answering. The tier order in Knowledge Base Structure above is the search priority.

---

## Confirmed Architectural Facts

This section captures cross-cutting platform facts that any session should treat as authoritative. Per-component source-verified findings live in the individual `WDP-COMP-NN-*.md` files (per the Knowledge Base Structure search priority).

Do not contradict these without explicit confirmation from Ram.

### Access and Authorization

- Case-level authorization is FAIL-CLOSED for NAP and PIN. Any exception including timeouts returns 403. The fail-open branch in WDP-COMPONENTS.md is incorrect — confirmed from COMP-01 source.
- No CORE platform value exists in API Gateway source code. **CORE/VAP/LATAM authorization is a confirmed gap** — these platform types receive no role-level or case-level authorization at the gateway layer. Architect decision required.
- **** COMP-01 listen port is `8082` — confirmed in both `application.yml:L2` and K8s manifest. Prior uncertainty about port 9082 was a documentation error.
- **** COMP-01 `v-correlation-id` header on all UAMS and CHAS calls is always null — `RequestCorrelation.setId` is never called anywhere in the codebase, and `ThreadLocal` is incompatible with the reactive WebFlux threading model.
- **** COMP-01 null-platform auto-authorize path in `AuthorizationServiceImpl` is **unreachable from `CaseNumberFilter`** — the filter guards `platform != null && caseId != null` before calling the service. Latent dead code, not active bypass.
- **** COMP-01 `wdp-internal-auth` dead config confirmed across all 7 environment YAMLs (local, dev, test, uat, stg, cert, prod) — all use the wrong namespace `spring.security.oauth2.resourceserver.client.*`.
- **** COMP-01 Ingress — all 7 hostname entries use path-based routing at `/api/merchant/gcp` (`pathType: ImplementationSpecific`) to port 8082.
- `validateOrgId` is commented out on GET /orgentity in CHAS (COMP-03). Any authenticated PIN-platform caller can query any org hierarchy without scope validation. The method body is itself absent from `RequestValidator` — remediation requires reimplementation, not uncomment.
- **** COMP-03 CHAS exposes **nine** REST endpoints across two controllers (`AuthorizationController`, `MerchantController`, `ProductEntitlementController`, `MerchantControllerV2`). v1.0 documented six.
- **** COMP-03 internal firm bypass (`iss` contains `us_worldpay_fis_int`) applies to **BOTH** `/authorize` AND `/entity-authorize` — both share `AuthorizationServiceImpl.authorizeEntity`.
- **** COMP-03 internal firm **partial-bypass** on `POST /{platform}/merchant/entitytype` (V1 and V2): internal callers skip JWT `iqentities` extraction and use `orgId` from request body directly. DB queries still execute.
- **** COMP-03 `validatePlatform` accepts **both PIN and CORE** (not PIN-only as v1.0 stated). Affects all four V1 hierarchy data endpoints.
- COMP-24 CaseActionService has no RBAC enforcement — `RestInvoker.authorizeUser` exists but is never called from any controller. Recorded as DEC-018 (Accepted Risk).
- **** COMP-27 CaseSearchService `POST /lft` derives internal-vs-external authorization from a request-body field (`LftSearchParams.isInternal`), not from the JWT. Flagged for architect review.
- **** External VAP and LATAM callers are not routed through any entity-scope authorization service in COMP-27. Only NAP (→ UAMS) and PIN/CORE (→ CHAS) are. Confirm intent.
- **** Internal PIN regular-user role exists (`WDP_PIN_REGULAR` in `AuthorizationList`) alongside the already-known `WDP_NAP_REGULAR`. COMP-27 confirmed.
- **** COMP-21 ChargebackService `/actuator/prometheus` requires JWT — NOT in security whitelist (corrected from prior draft).
- **** COMP-19 / COMP-20 liveness/readiness probe paths end in `livez` / `readyz` (with `z`), not `liver` / `lives` / `ready` as v1.0 stated.
- **** COMP-29 / COMP-35 / COMP-30 / others — `/actuator/prometheus` JWT-protected pattern confirmed widespread; **scrape-side configuration must carry JWT or whitelisting needs platform-wide remediation.**
- **** **COMP-49 WDP Portal user types are three, not two:** `MERCHANT_USER`, `OPS_USER`, and `PB_USER`. Assigned at runtime by `WdpSharedService` based on DNS hostname plus IDP firm name. `PB_USER` is the Worldpay-internal `us_worldpay_fis_int` firm authenticated via merchant DNS — distinct from `MERCHANT_USER` and from `OPS_USER`.
- **** **COMP-49 OPS super-user UI bypass amplifies DEC-018:** `WdpUtilService.canUserPerformDisputeAction()` (`wdp.util.service.ts:44-46`) returns `{canPerform: true}` unconditionally for any `OPS_USER`, with no inspection of action / case state / queue / per-action permissions. Combined with DEC-018 (no server-side RBAC at COMP-24 CaseActionService), there is no enforcement gate at either UI or backend layer for OPS_USER. RISK-195 / ADR-CAND-060 — top-priority architect resolution required.

### Kafka — Platform-Wide Patterns

- Every confirmed Kafka consumer in WDP uses pre-ACK or mid-flow ACK. DEC-005 (commit after full processing) is aspirational — it is not the current implementation.
- No Kafka DLQ topics exist in WDP. Error handling is via database error tables per consumer (e.g. `NAP.DISPUTE_EVENT_CONSUMER_ERROR` for COMP-05; `wdp.outgoing_event_outbox` for COMP-17/41/42/43/51; `wdp.bre_orchestration_outbox` for COMP-18; `wdp.chbk_outbox_row` for COMP-14/15).
- Empty anonymous `CommonErrorHandler{}` is registered platform-wide on multiple consumers (COMP-05, COMP-14, COMP-15, COMP-16, COMP-18, COMP-39, COMP-40, COMP-41, COMP-42, COMP-43). Combined with `ErrorHandlingDeserializer`, deserialisation exceptions and unhandled application exceptions are silently swallowed — no DLT, no log, no counter.
- No circuit breakers (Resilience4j) anywhere in WDP. Confirmed absent across all 38 source-verified component files. DEC-014 formally voided.
- **business-rules topic has SIX confirmed publishers:** COMP-12 Scheduler4 (BUSINESS_RULES rows), COMP-15 EvidenceConsumer (WDP path, isMultiDocPending=false), COMP-23 CaseManagementService (kafka-before-commit pattern), COMP-24 CaseActionService (BRE inside `@Transactional`), COMP-25 NotesService (inside `@Transactional`), COMP-37 DocumentManagementService. All use `caseNumber` as partition key — DEC-003 uniform deviation.
- **** **COMP-14 CaseCreationConsumer is NOT a publisher of `business-rules`** — confirmed by absence audit (no `KafkaTemplate`, no `ProducerFactory`, no reference to topic in source).
- **** **COMP-22 DisputeService is NOT a Kafka producer at runtime** — wired but commented out (commit `c29018cd` by Shringi Nitin (WP), 2025-08-08, message "code changes (#93)"). No in-source rationale.
- **`wdp.outgoing_event_outbox` has FIVE writers**: COMP-17 (EXPIRY_EVENTS, `WCSEEXPC`), COMP-41 (GP_EVENTS, `WNEC`), COMP-42 (BEN_EVENTS, `WBENC`), COMP-43 (CORE_EVENTS, `PCSECRTC`), COMP-51 (EXPIRY_BATCH, `WCSEEXPB`). Each writer uses a distinct `channel_type` discriminator. **No DB-level UNIQUE constraint visible in any of the five repos** — DBA confirmation required.
- `wdp.bre_orchestration_outbox` is shared between COMP-18 (`component=NOTIFICATION_ORCHESTRATOR` rows) and COMP-12 Scheduler4 (`component=BUSINESS_RULES` rows). PUBLISHED-status orphan rows have no automatic re-drive — manual intervention required.
- **** **COMP-18 does NOT write to `wdp.outgoing_event_outbox`** — WDP-DB.md was incorrect on this point. Source grep confirms zero references.
- **** COMP-12 InboundDisputeEventScheduler uses **mark-and-send within a single `@Transactional`** — broker ACK precedes TX commit. **At-least-once with duplicate-possible crash window** (NOT at-most-once). Consumer-side `idempotency-key` dedup is the contracted mitigation.
- **** COMP-12 channelTypeTopicMap non-prod default keys: `EXPIRY_EVENTS`, `CORE_EVENTS`, `GP_EVENTS`, `BEN_EVENTS` (NOT `SEN_EVENTS`).
- **** COMP-12 Kafka-metadata write-back asymmetry: `new-case-events` and `case-evidence-events` paths persist `kafka_offset`/`kafka_partition`/`kafka_topic` back to the source outbox row; the other three paths do not. Incident correlation requires log-side join on `idempotencyId`.
- **** COMP-41 ThirdPartyNotificationConsumer writes `channel_type=GP_EVENTS` (NOT `GF_EVENTS`). Three distinct PUBLISHED-orphan paths confirmed.
- **** COMP-04 NAPDisputeEventService partition-key per-endpoint mapping: `merchantId` on case-update and outcome paths; `cardAcceptorCodeId` on POST `/event` (new-dispute path) only.
- **** COMP-04 `${kafka_retry_count}` and `${kafka_retry_delay}` env vars **also govern** CaseManagement (new-event path), FraudSwitch, and DisplayCodeService outbound REST retries. Hidden coupling — tuning Kafka retries silently retunes those three REST dependencies.
- case-action-events publisher confirmed as COMP-18 NotificationOrchestrator (Filter 1 EXPIRY_EVENT routing). cert environment uses distinct topic `case-action-events-cert` and group `case-action-events-group-cert`.
- **** COMP-43 CoreNotificationConsumer is the sole WDP component that writes to IBM DB2 Core Platform. COMP-43 reads only `BC.TBC_DM_CASE` (enrichment fully REST-driven). Clear-PAN column is `I_ACCT_CDH` (NOT `I_ACCT_CDR`). PAN-last-4 column is `I_ACCT_CDH_LST`.
- `wdp.file_notifications` does not exist. The actual table is `wdp.file_generation_event`. COMP-18 is the sole writer (`created_by/updated_by=WDPOUCU`, INSERT-only, status hardcoded `STAGED`).
- **** **COMP-39 is the consumer of `internal-integration-events` for the NAP outbound path** — single `@KafkaListener` confirmed. COMP-39 is therefore **NOT** the consumer of COMP-24's `${kafka.topic}` ActionEvent topic; that consumer remains unidentified.
- **** Operational footgun pattern: env vars without YAML defaults — if unset, service fails to start. Confirmed: `${kafka_business_event_topic}` (COMP-25), `${kafka_retry_count}` and others (COMP-04), `gcp_async_corepoolsize` / `core_migration_status` (COMP-22), feature flags on COMP-07.

### Confirmed DEC Violations and Voids

- DEC-011 ⛔ VOID — BRE step checkpointing confirmed never implemented in COMP-16. Formally voided in WDP-DECISIONS.md.
- DEC-014 ⛔ VOID — Resilience4j confirmed absent across all 38 source-verified component files. Formally voided in WDP-DECISIONS.md.
- DEC-004 violation in COMP-23 — clear PAN written to persistent storage on standard case creation (`wdp.CASE.I_ACCT_CDH` and `nap.case.I_ACCT_CDH`). Formally recorded as DEC-019 (Accepted Risk).
- **** DEC-004 / DEC-019 violation in COMP-43 — clear PAN written to `BC.TBC_DM_CASE.I_ACCT_CDH` (DB2) on CREATE + actionSeq=01 path. Architect decision required (OQ-COMP-43-6).
- **🔴 NEW PCI-DSS finding — COMP-05 CVV at rest:** `NAP.DISPUTE_EVENT_CONSUMER_ERROR` persists CVV in two places per error row — `C_CVV` column directly mapped on the entity, AND raw `NotificationEvent` JSON serialised into `C_KAFKA_EVENT` which also contains the `cvv` field. Cross-link: COMP-04 logs the CVV via Lombok `toString`; COMP-05 persists it. Architect decision required.
- DEC-001 (transactional outbox) — confirmed deviations:
  - COMP-04 (direct publish on HTTP thread) — confirmed source-verified
  - COMP-15 (synchronous publish inside `@Transactional` — ghost-event window)
  - COMP-16 (direct synchronous publish, no outbox)
  - COMP-18 ⚠️ PARTIAL (uses `wdp.bre_orchestration_outbox` but no `@Transactional`; four distinct write points are independent auto-commits)
  - COMP-19 (direct synchronous publish; HTTP 500 on Kafka failure does not undo case action)
  - COMP-20 (direct synchronous publish)
  - COMP-23 — `kafka-before-commit` pattern: publish inside `@Transactional` BEFORE commit
  - COMP-24 ⚠️ PARTIAL — BRE publish IS inside `@Transactional`. ActionEvent publish is OUTSIDE `@Transactional` on EP 2/8/9
  - COMP-25 (synchronous publish inside `@Transactional`; **mid-batch atomicity** — per-event loop produces deterministic Kafka-orphan events on partial-batch failure)
  - COMP-37 (direct synchronous publish post-DDB on primary upload; questionnaire path mitigates with `@Transactional(rollbackOn=Exception.class)`)
  - COMP-43 (separate transaction managers — outbox INSERT on `wdpTransactionManager` commits before DB2 write on `coreTransactionManager`; crash window wide open)
- **🔴 DEC-021 wrong-TM scope expansion — COMP-02 UAMS:** previously documented as a one-method offence (`saveChildWithMerchant`). Source verification confirms **7 methods across 4 NAP tables** use `wdpTransactionManager` instead of `napTransactionManager` — `saveChildWithMerchant`, `createEntity`, `updateEntity`, `updateMerchantRelationships`, `deleteMerchantByParent`, `deleteChildEntity`, `deleteParentEntity`. Affects `nap_parent_entity`, `nap_child_entity`, `nap_merchant`, `nap_entity_rel`. Same severity class as DEC-019.
- **🔴 DEC-021 second offender — COMP-30 UserQueueSkillService:** service-level `@Transactional` on `createQueue`/`updateQueue` binds to `@Primary usTransactionManager` — UK writes to `nap.queues`, `nap.queue_criterion`, `nap.user_queue` are NOT covered by the outer TX. Same root cause class as COMP-02.
- DEC-020 (Accepted Risk) — no idempotency on case creation (COMP-23). Formally recorded.
- **** **DEC-019 UI input-layer PARTIAL exception (COMP-49):** WDP Portal accepts full PAN at three input forms across both modes — Filter Disputes (both), Filter Fax Dispute (Ops), Update Transaction Search (Ops). Encrypted in transit; decrypted to AG Grid memory only — no client-side persistence. Input-layer surface in PCI scope. Materially distinct from DEC-019 (1) and (2) (PAN-clear at rest in DB) and from (3) (CVV at rest in error tables/logs). RISK-198 — architect decision required (accept as PARTIAL or remediate).

### Key Confirmed Platform Facts

- BusinessRulesProcessor (COMP-16) makes direct DB calls to `nap.rules` / `wdp.rules` — does NOT call BusinessRulesService (COMP-31).
- NotificationOrchestrator (COMP-18) does NOT publish to `internal-integration-events` — that topic is published by AcceptService (COMP-19), ContestService (COMP-20), and conditionally COMP-16 (UK/NAP path).
- **** **COMP-22 DisputeService is read-only — confirmed source-verified zero writes.** No `INSERT`, `UPDATE`, `DELETE`, no Spring Data repositories, no `@Modifying`, no `JdbcTemplate`. Kafka producer to business-rules is wired but commented out — effectively Kafka-free at runtime. The COMP-37 column-level UPDATE on `wdp.CASE` is now confirmed orphan to COMP-22; cross-component review item recorded for COMP-22/COMP-37 is **closed on the COMP-22 side**. The COMP-37 INSERT/DELETE owner question stands.
- COMP-28 DisplayCodeService does NOT determine TIER1 eligibility — returns raw code lists. Eligibility logic belongs to calling services (e.g. COMP-04 NAPDisputeEventService).
- COMP-38 APILogService has no AOP inside it. Callers hold catch-blocks and invoke this service directly via REST.
- **** **COMP-36 TokenService is read-only on Redis** — source verified zero write operations. The `wdpinternalidptoken:token` Redis hash is populated by an UNKNOWN EXTERNAL COMPONENT not present in any audited WDP repo. External writer identity remains an open question.
- COMP-33 OrgManagementService — GitHub repository not found. Documentation deferred.
- `removeItemFromQueueDisabled` flag is a second active operational safety switch in COMP-07 and COMP-08. When true, suppresses ALL MarkAsRead/ACK PUT calls globally.
- BEN notification is delivered via Kafka publish to a BEN-owned MSK cluster using separate SASL/JAAS credentials — NOT via REST or webhook. Confirmed from COMP-42 source.
- **JustAI integration scope:** JustAI is **active in COMP-21 ChargebackService** (consumer name `JUSTTAI` triggers partner branch in `isPartner` — note double-T spelling). It **remains absent from COMP-41**. The platform-wide statement "JustAI is planned only" was correct *for outbound third-party notification (COMP-41)* but not platform-wide. Signifyd is the sole live third-party vendor for outbound notification.
- **** **COMP-32 RulesService is a pure read-only REST API** — zero writes, zero Kafka, zero PAN, zero outbound REST/HTTP. Sole outbound dependency is two PostgreSQL datasources.
- **** **COMP-35 EncryptionService DEK rotation interval is days** (`${dek_rotation_interval_days}`) — prior platform docs stating "6-hour DEK cache" are incorrect.
- **** **COMP-39 hosts the manual NAP error reprocessing REST endpoint** (`POST /event`, JWT-authenticated). This resolves the long-standing question of which component hosts the manual reprocessing surface referenced in COMP-05's documentation — it is COMP-39, not COMP-05.
- **** **`minReadySeconds` misplacement** is a platform-wide pattern. Eight components confirmed with the defect: COMP-03, 04 (untouched), COMP-08, COMP-09, COMP-12, COMP-25, COMP-26, COMP-28, COMP-34, COMP-40. `minReadySeconds: 30` is placed under `spec.template.spec` instead of `spec` — silently ignored by Kubernetes. Candidate for platform-wide remediation pass.
- **`v-correlation-id` regenerated-per-call anti-pattern** — confirmed in COMP-17 (write half) and COMP-51 (read half) of the Case Expiry Subsystem. End-to-end audit trail across the 4–5 calls per record cannot be reconstructed from headers alone. Same anti-pattern likely exists in other components — platform-wide remediation candidate.
- **** **Case Expiry Subsystem confirmed:** COMP-17 (writer half — Kafka consumer maintaining `wdp.case_expiry`) + COMP-51 (reader half — scheduled batch acting on past-due rows). They share `wdp.case_expiry` as their coordination surface with **no row-level lock, no version column, no SELECT FOR UPDATE**. Race possible.
- **** **COMP-49 + COMP-50 are a single Angular 19 SPA, not two applications.** Single `wdp-portal` repository, single build artifact, two runtime modes — Merchant (via Akamai → API Gateway) and Ops (direct → API Gateway). Mode identity derived from DNS hostname plus IDP firm name; `EnvironmentService` selects `apiBaseUrl` per host. No separate build, no compile-time flag, no distinct entry point.
- **** **COMP-49 documentation consolidated:** `WDP-COMP-49-WDP-PORTAL.md` is the canonical file covering both modes. `WDP-COMP-50-OPS-PORTAL.md` is retained as a stub pointer per the permanent-numbering convention — no component number reused or removed.
- **** **COMP-49 Disputes Section export threshold is > 200 records** (or `isAlwaysLFTExport=true`), at which point export is asynchronous via Large File Transfer (LFT). Synchronous CSV is returned up to 200 records. Prior platform docs stating "5,001–25,000 threshold" are incorrect.
- **** **COMP-49 Administration Section is NAP-only** (Ops mode). v1.0 statement that Administration applied to "both portals on all platforms" was incorrect.
- **** **COMP-49 Fax Queue is two distinct sections, not one** — Fax Matching and Fax Analytics. Both are Ops-mode-only and CORE-only. Backed by COMP-29 FaxQueueService.
- **** **COMP-49 More Actions surface has eleven sub-types**, not the smaller v1.0 list: Accept, Contest, Reopen, Pre-Arbitration, Arbitration, Compliance Case, Pre-Compliance, Reverse, Issuer Reversal, Issuer Accept, Convert to Partial DTP, Update Transaction. **Five are previously undocumented** (Reverse, Issuer Reversal, Issuer Accept, Convert to Partial DTP, Update Transaction) — audit and governance pending. RISK-199 / ADR-CAND-063.
- **** **COMP-20 ContestService supports three contest modes** (surfaced in COMP-49 UI as "Defend"): `SELF_ASSISTANCE` (merchant submits directly), `WORLDPAY_ASSISTANCE` (Worldpay ops prepares response on merchant's behalf), `RETRIEVAL_RESPONSE` (retrieval-request response, distinct from chargeback contest). Mode determines which questionnaire is presented and which downstream submission path is taken.

### Enterprise Shared Services (not WDP owned)
- Akamai — CDN and edge security (WDP Portal Merchant mode only)
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
- COMP-29, 31, 34, 38 source-verification (4 components remaining)
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

This section carries **component-level** open questions. **Platform-level** open architectural questions live in WDP-ARCHITECTURE.md "Open Discussion Points" (27 entries). Refer there for the platform discussion register.

### Currently Open

| Question | Source | Action needed |
|----------|--------|---------------|
| COMP-24 ActionEvent topic name (`${kafka.topic}`) and consumer identity — **NOT COMP-39** | COMP-24 / COMP-39 | Confirm from deployment config |
| `wdp.case_expiry` other downstream consumers beyond COMP-51 | COMP-17 / COMP-51 OQ | Cross-repo sweep on remaining WDP components |
| Scheduler3/4 query FAILED/PENDING_DEFERRED only — does any process re-drive PUBLISHED orphans? | COMP-12 risk | Architect decision |
| Scheduler3 channel_type filter behaviour — does it filter consistently across EXPIRY_EVENTS, GP_EVENTS, BEN_EVENTS, CORE_EVENTS, EXPIRY_BATCH? Does it ever read PUBLISHED? | COMP-41 / COMP-42 OQ | Architect / Claude Code follow-up on COMP-12 source |
| `nap.queues` / `nap.queue_criterion` other-writer ownership beyond COMP-30 | COMP-27 / COMP-30 / COMP-31 OQ | Cross-repo sweep |
| External VAP/LATAM entity-scope authorization absent in COMP-27 — intentional or gap? | COMP-27 OQ | Architect decision |
| COMP-27 `/lft` `isInternal` derives from request body, not JWT | COMP-27 security finding | Architect review |
| COMP-23 PAN-clear remediation timeline (DEC-019) | COMP-23 risk | Team confirmation |
| COMP-17 cross-action predecessor scope — intentional (case-level ordering) or bug (action-level)? | COMP-17 OQ | Architect decision |
| COMP-17 non-atomic outbox + `case_expiry` split — accept or remediate? | COMP-17 OQ | Architect decision |
| COMP-26 POST idempotency — add DB UNIQUE on (I_CASE, I_ACTION_SEQ) and caller `Idempotency-Key`, or accept current behaviour? | COMP-26 risk | Architect decision |
| COMP-26 no-`@Transactional` posture (PUT, B1 UPSERT TOCTOU) — intentional or remediate? | COMP-26 OQ | Architect decision |
| COMP-26 `DriverManagerDataSource` choice — intentional (e.g. behind PgBouncer) or remediate to HikariCP? | COMP-26 OQ | Architect decision |
| COMP-26 `HttpMessageNotReadableException` → 500 mapping; invalid platform string → 500 — intentional or fix to 400? | COMP-26 OQ | Architect decision |
| COMP-37 RESPDOC→source divergence: primary upload `BRRSUP`, questionnaire path `BRMRUP` | COMP-37 OQ | Architect decision |
| COMP-37 five DynamoDB GSIs declared but unused from Java — which external consumer depends on them? | COMP-37 OQ | Architect / data team confirmation |
| COMP-15 V3-path MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC silently mark `file_evidence=ATTACHED` — fix as upload path or explicit reject? | COMP-15 OQ | Architect decision |
| COMP-15 DSNOTDOC publishes `source=""` — does COMP-16 misroute? | COMP-15 / COMP-16 OQ | Claude Code follow-up |
| `wdp.chbk_outbox_row.status=PENDING_DEFERRED` re-driver ownership | COMP-15 OQ | Cross-component scan |
| `wdp.outgoing_event_outbox` DB-level UNIQUE on `(idempotency_id, channel_type, event_timestamp)` — five repos confirm "no UNIQUE in source" | COMP-17 / COMP-41 / COMP-42 / COMP-43 / COMP-51 OQ | DBA confirmation |
| `nap.rule_group`, `nap.rule_criteria`, `nap.rule_action_field` write ownership | COMP-31 / COMP-32 gap | Confirm whether COMP-32 RulesService or DBA scripts own these |
| `wdp.api_route` table — which component owns writes? | COMP-01 gap | Confirm from team or infrastructure config |
| File job terminal status PROCESSING vs COMPLETED — COMP-11 never writes COMPLETED, COMP-13 polls for COMPLETED | COMP-11/12/13 contract | Architect decision |
| `wdp.chbk_outbox_row` DB-level UNIQUE on `(file_job_id, event_type, row_number)` and `(c_ntwk_case_id, c_ntwk_phase_id)` | COMP-07/08/09/11 risk | DBA confirmation |
| COMP-08 SKIPPED-marker accumulation boundedness — does Scheduler2 archive SKIPPED rows? | COMP-08 risk | Confirm Scheduler2 archive query |
| COMP-08 update-path PENDING-without-HPAN — defect or design? | COMP-08 risk | Architect decision |
| COMP-08 writer-ACK hazard — accept or remediate? | COMP-08 risk | Architect decision |
| Production replica count for **every** continuously-running component — XL Deploy placeholders | All components | Environment config / team confirmation |
| COMP-21 missing IDP token cache — defect in `CachedTokenServiceImpl`, or intentional? | COMP-21 OQ | Architect decision |
| COMP-21 `asyncExecutor` core=1/max=1/queue=5 sizing — intentional or leftover defaults? | COMP-21 OQ | Architect decision |
| COMP-21 empty `logstash_server_host_port` and stdout-only logging — intentional? | COMP-21 OQ | Team confirmation |
| COMP-22 known callers of `POST /summary` and `POST /documents` | COMP-22 OQ | Team confirmation |
| COMP-22 reason for Kafka publish disablement (PR #93 review) | COMP-22 OQ | Team / commit-author confirmation |
| COMP-22 SFG SFTP filename collision — namespacing required? | COMP-22 OQ | Architect decision |
| COMP-22 unconfigured HikariCP and Tomcat pools | COMP-22 OQ | Architect decision |
| COMP-29 FaxQueueService eViewer License proxy — runbook? | COMP-29 gap | Team confirmation |
| COMP-03 `validateOrgId` method body absent | COMP-03 security | Raise formal ADR |
| COMP-22 INSERT/DELETE owner of `wdp.CASE` — COMP-37 column-level UPDATE has no INSERT/DELETE owner identified | COMP-37 OQ | Cross-component review |
| Production cron values (COMP-12 five schedulers, COMP-07/08 batch crons, COMP-51) | All schedulers | K8s secret confirmation |
| Archive column `c_ntwrk_phase_id` — exists in DDL? | COMP-12 OQ | DBA / schema-owner confirmation |
| OQ-FileJob-COMPLETED-Owner (cross-component contract) | COMP-11/12/13 | Architect decision |
| Production values of `${kafka_retry_count}`, `${kafka_retry_delay}`, plus 5 other retry/delay env vars (CaseManagement, CaseAction, Rule retries) | COMP-04 OQ | Env config / team confirmation |
| Production value of `${idp_token_service_url}` and `${kafka_nap_topic}` | COMP-04 OQ | Env config |
| GUARPAY7 → TIER5 mapping intent — architect confirmation | COMP-04 OQ | Architect decision |
| `@EnableCaching` import without annotation application — intentional placeholder or accidental? | COMP-04 OQ | Architect decision |
| COMP-04 decommission timeline and migration target | COMP-04 OQ | Team confirmation |
| Cross-repo writer of `wdp.disputes_questionnaire_doc_status_rules` | COMP-26 OQ | Cross-repo grep |
| COMP-20 ContestService relationship to QuestionnaireService | COMP-26 OQ | Cross-component review |
| Component that transitions `wdp.lft_report.status` beyond PENDING | COMP-30 OQ | Cross-repo sweep |
| Owners of `nap.nap_parent_entity`, `wdp.case`, `wdp.action` from COMP-30 reader perspective | COMP-30 OQ | Already known (COMP-02 / COMP-23 / COMP-24); confirm reader attribution |
| `logback-spring.xml` runtime source — file absent from repo; Logstash integration non-functional unless mounted at runtime | COMP-30 OQ | Deployment team confirmation |
| `/actionrules` `AuthorizationList` claim absence returns HTTP 500 NPE — defect or accepted? | COMP-32 OQ | Architect decision |
| `/actionrules` empty-string `AuthorizationList` claim splits to `[""]`, intersection empty, silent — defect or accepted? | COMP-32 OQ | Architect decision |
| Three endpoints (`/notification`, `/business-event`, `/cbk-response`) return three different HTTP 200 shapes on no-rule-found while the other 11 throw 404 | COMP-32 OQ | Architect decision |
| `@Cacheable` key computed before in-method blank→"NA" normalisation — duplicate cache entries | COMP-32 OQ | Architect decision |
| `/documentType` and `/eventRule` controller mappings commented out with full backing code intact — abandon or re-enable? | COMP-32 OQ | Architect decision |
| DB-level UNIQUE on `wdp.hash_key_store.hpan` | COMP-35 OQ | DBA confirmation |
| Production value of `${dek_rotation_interval_days}` | COMP-35 OQ | Ops |
| Prometheus scrape configuration — does the scrape job carry a JWT? | COMP-35 OQ | Ops |
| Log-pipeline PAN-scrubbing posture (Logstash target side) for the DEBUG last-4 PAN log | COMP-35 OQ | Ops |
| NAP-DPS authentication mechanism — `napcacrt.jks` is unreferenced orphan; auth must be at Ingress / service mesh / network layer | COMP-39 OQ | Infrastructure team confirmation |
| `NAP.NAP_UPDATE_RESPONSE_RULES` table owner | COMP-39 OQ | Team confirmation |
| COMP-39 probe path mismatch — `resources.yaml` probes hit `/merchant/gcp/update-consumer/nap/livez` but actuator `additional-path: server:/livez` exposes endpoint at server root. Probes resolving correctly? | COMP-39 risk | Runtime observation |
| COMP-39 `POST /event` controller-level exception handling — response codes and body schema | COMP-39 contract | Follow-up |
| COMP-39 concurrency=1 — intentional or default-by-omission? | COMP-39 OQ | Architect decision |
| BEN platform idempotency — does BEN deduplicate at its end on retry-driven re-publish? | COMP-42 OQ | BEN integration team |
| Production runtime values — `${ben_topic}`, `${ben_bootstrap_servers}`, etc. | COMP-42 OQ | Env config |
| COMP-51 cron value, page size, table prefix, replica count | COMP-51 OQ | XLD config inspection |
| Discover hybrid-merchant RE2/REPR special case — permanent regulatory rule or migration-era workaround? | COMP-51 OQ | Architect / regulatory confirmation |
| HTTP 504 retry on POST endpoints (Accept, Add Action, Rules) — safe across all 7 upstream services? | COMP-51 risk | Architect / team confirmation |
| Hardcoded user IDs `WCSEEXPB` (COMP-51) and `WCSEEXPC` (COMP-17) — confirm distinction is intentional | COMP-51 OQ | Confirm with platform user-ID registry |

---

*This document is the session bootstrap reference. Per-component source-verified findings live in WDP-COMP-NN-*.md files. Platform-level architectural questions live in WDP-ARCHITECTURE.md "Open Discussion Points".*
