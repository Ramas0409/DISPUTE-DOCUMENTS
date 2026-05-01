# WDP-HANDOVER.md
**Worldpay Dispute Platform — Architecture Session Handover**
*Version: 3.2 | April 2026*
*Reconciled: 2026-04-30 — absorbed Pending Entries from WDP-CHANGE-LOG.md for COMP-01, COMP-02, COMP-03, COMP-04, COMP-05, COMP-06, COMP-22, COMP-25, COMP-26, COMP-28, COMP-30, COMP-32, COMP-35, COMP-36, COMP-39, COMP-40, COMP-42, COMP-51.*
*Component file source-verification status: 📝 DRAFT — architect confirmation pending on all 38 source-verified components.*

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
| WDP-ARCHITECTURE.md | ⚠️ Reconciliation pending — v2.1 | Pending updates from 2026-04-28/29/30 entries (file_generation_event correction, NAP migration route live, AMEX/DISCOVER no-op note). Awaiting v2.2 rebuild. |
| WDP-DECISIONS.md | ⚠️ Reconciliation pending — v2.1 | DEC-011 and DEC-014 voided. **30+ candidate ADRs surfaced from 2026-04-25/28/29/30 component audits** — see "Doc reconciliation backlog" below. v2.2 rebuild pending. |
| WDP-INTEGRATIONS.md | ⚠️ Reconciliation pending — v2.1 | New integration contracts pending (COMP-04 enrichment chain detail, COMP-05 CVV finding, COMP-39 NAP-DPS auth gap, COMP-42 BEN MSK cluster). v2.2 rebuild pending. |
| WDP-NFRS.md | ⚠️ Reconciliation pending — v2.1 | **~70 candidate RISK rows** surfaced from 2026-04-25/28/29/30 component audits. v2.2 rebuild pending. |
| WDP-HANDOVER.md | ✅ Current — v3.2 | This file. Reconciled 2026-04-30. |
| WDP-CHANGE-LOG.md | ✅ Current | Reconciliation Marker dated 2026-04-30 — 18 entries moved from Pending to Reconciled. |

### Tier 2 — Reference Indexes

| Document | Status | Notes |
|----------|--------|-------|
| WDP-COMP-INDEX.md | ✅ Current — v2.2 | Reconciled 2026-04-30. All 51 components registered. **38 of 51 components carry 🔍 source-verified marker.** New COMP-51 row added. |
| WDP-KAFKA.md | ✅ Current — v2.2 | Reconciled 2026-04-30. `wdp.outgoing_event_outbox` writers now 5 (COMP-42 BEN_EVENTS and COMP-51 EXPIRY_BATCH added). New "hidden coupling" and "operational footgun" subsections. COMP-06 and COMP-51 added to Kafka-free list. |
| WDP-DB.md | ✅ Current — v2.2 | Reconciled 2026-04-30. New rows for `wdp.queues` / `wdp.queue_criterion` / `wdp.user_queue` (COMP-30), `wdp.hash_key_store` / `wdp.data_enc_key` (COMP-35), `wdp.disputes_questionnaire_doc_status_rules`, 15 COMP-32 reader rule tables, `NAP.BUSINESS_RULE_CONSUMER_ERROR`, `NAP.NAP_UPDATE_RESPONSE_RULES`, `NAP.issuer_raised_dispute_timefrm_rules`, `NAP.mcm_queue_data`, `nap.dispute_nap_new_case_action_rules`. ElastiCache row rewritten (COMP-36 read-only). 5 new shared-table risk rows. CVV PCI-DSS finding documented. |
| WDP-FLOW-INDEX.md | ✅ Current — v1.1 | Reconciled 2026-04-30. FLOW-11 expanded into "Case Expiry — Subsystem Lifecycle". New "Per-Flow Authoring Notes" section captures FLOW-07 Visa contest variant and FLOW-11 case expiry coordination gap. |
| WDP-COMP-TEMPLATE.md | ✅ Created — v1.1 | Master template with 4 type blocks. |

### Tier 3 — Individual Component Files

| Status | Count | Details |
|--------|-------|---------|
| ✅ COMPLETE (individual file created and confirmed) | 0 | — |
| 📝 DRAFT 🔍 (source-verified, architect confirmation pending) | 38 | COMP-01, 02, 03, 04, 05, 06, 07, 08, 09, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 30, 32, 35, 36, 37, 39, 40, 41, 42, 43, 51 |
| 📝 DRAFT (Copilot CLI analysis, source-verification pending) | 4 | COMP-29, 31, 34, 38 |
| 📋 PENDING (enterprise-owned, lower priority) | 1 | COMP-10 DM Mainframe |
| ⬜ NOT STARTED | 6 | COMP-33 (OrgManagementService — no repo found), COMP-44 (EDIAConsumer — planned), COMP-45, COMP-46, COMP-47 (File Generation — planned next sprint), COMP-48 (NYCEFileGenerationProcessor — planned) |
| 🔲 UI — separate action item | 2 | COMP-49 WDP Merchant Portal, COMP-50 WDP Ops Portal |

**Component migration status.** All 51 components are registered. 38 of 51 have undergone source-verified correction passes between 2026-04-18 and 2026-04-30. All carry "📝 DRAFT 🔍 — architect confirmation pending" status. Architect confirmation is the next gate.

### Tier 4 — Workflow Documents

No workflow files exist yet. 11 flows identified in WDP-FLOW-INDEX.md.
Two new "Per-Flow Authoring Notes" added: FLOW-07 Visa contest end-to-end variant; FLOW-11 Case Expiry Subsystem coordination gap.
Workflow documentation to be done as a parallel work stream.

---

## Doc Reconciliation Backlog

The 2026-04-30 reconciliation session rebuilt **WDP-DB.md v2.2**, **WDP-KAFKA.md v2.2**, **WDP-COMP-INDEX.md v2.2**, **WDP-FLOW-INDEX.md v1.1**, and **WDP-HANDOVER.md v3.2** from 18 Pending Entries (COMP-01, 02, 03, 04, 05, 06, 22, 25, 26, 28, 30, 32, 35, 36, 39, 40, 42, 51). The following Tier 1 derivatives still need their own reconciliation passes:

| Document | Pending updates from 2026-04-28/29/30 entries |
|----------|------------------------------------------------|
| WDP-NFRS.md | ~70 candidate RISK rows: COMP-01 (4 — blocking RestTemplate / null correlation / authExceptions trim / unhandled NPE), COMP-02 UAMS (5 — DEC-021 wrong-TM scope / orphan-row / no-test / etc), COMP-03 (9 — `validateOrgId` gone / no-`@Transactional` / minReadySeconds defect / etc), COMP-04 (multi-RestTemplate / base64-in-logs / `@EnableCaching` no-op / etc), COMP-05 (CVV-at-rest / cross-component error scoop / non-uniform retry / no inbound dedup / minReadySeconds defect), COMP-06 (RISK-A/B/C/D/E/F/G), COMP-22 (7 — async SFTP filename collision / dead status code / no correlation / unconfigured pools / etc), COMP-25 (mid-batch atomicity / env-var footgun), COMP-26 (TOCTOU / DriverManagerDataSource / 500-on-bad-platform / minReadySeconds defect), COMP-28 (projection divergence), COMP-30 (DEC-021 wrong-TM / orphan-user / CHAS infra→403 conflation / logback-spring missing), COMP-32 (5 OQs surfaced — null/empty AuthorizationList / 3-shape inconsistency / cache key duplicate / disabled endpoints), COMP-35 (Base64 KMS bug / no decrypt-RBAC / DEBUG PAN-last-4 / multi-pod DEK proliferation), COMP-36 (Redis writer unknown), COMP-39 (probe path mismatch / no-`C_EVENT_TYPE`-filter / concurrency-by-omission / NAP-DPS auth gap), COMP-40 (DEC-001 deviation), COMP-42 (no-probes / unbounded predecessor scan / orphaned outbox-row paths), COMP-51 (EXPIRY_BATCH-no-consumer / shared-write-no-coordination / hand-rolled retry / no-test / 504-retry-on-POST safety) |
| WDP-DECISIONS.md | 30+ candidate ADRs: AcceptService split-brain (HIGH), idempotency pattern variants, writer-ACK hazard pattern, silent exception swallow batch pattern, no-probes batch pattern, no-concurrency-guard batch pattern, kafka-before-commit pattern naming, `@Transactional(rollbackOn=Exception.class)` stopgap atomicity, DEC-020 idempotency-key formal void, IDP token caching pattern (COMP-21/COMP-22/COMP-26), async executor sizing, shared `RestTemplate` mutation, business-rules partition-key formalisation (six publishers all use caseNumber), mark-and-send within `@Transactional` cross-component, feature-flag framework adoption, file_job COMPLETED transition ownership, **DEC-021 scope expansion to 7 wrong-TM methods (COMP-02)**, **DEC-021 second offender (COMP-30)**, **shared-error-table consumption rules (COMP-05/COMP-39 no-`C_EVENT_TYPE`-filter)**, **CVV-at-rest in error tables exception ADR (COMP-05)**, **EXPIRY_BATCH outbox channel terminal-write-only acceptance (COMP-51)**, **Case Expiry Subsystem coordination ADR (COMP-17 + COMP-51 shared write)**, **`minReadySeconds` misplacement remediation (now 8 components confirmed)**, **`v-correlation-id` regenerated-per-call anti-pattern remediation (COMP-17 + COMP-51)** |
| WDP-INTEGRATIONS.md | COMP-04 enrichment chain detail (alternative not sequential — CaseManagement → FraudSwitch → DisplayCodeService); COMP-04 retry-config hidden coupling (CaseManagement / FraudSwitch / DisplayCodeService share Kafka retry vars); COMP-05 CVV-at-rest finding; COMP-22 SFG SFTP filename pattern (no namespacing); COMP-35 KMS round-trip held inside `@Transactional`; COMP-36 Redis hash external-writer-unknown; COMP-39 NAP-DPS auth-at-infra-layer (`napcacrt.jks` orphan); COMP-39 NAP_UPDATE_RESPONSE_RULES owner gap; COMP-42 BEN MSK cluster contract refinement; **JustAI scope correction** (active in COMP-21 inbound `JUSTTAI`, planned in COMP-41 outbound only); **COMP-21 enterprise Cert eAPI Entitlement Lookup** (wired, currently inactive). |
| WDP-ARCHITECTURE.md | Component count update 50 → 51; new "Case Expiry Subsystem" sub-grouping (COMP-17 + COMP-51); Section 7.3 file_generation_event correction (was `wdp.file_notifications`); Section 8.1 AMEX/DISCOVER + MC CHI no-op fall-through note; Data Storage section — clarify multi-datasource patterns (COMP-37 unique S3 + DDB + dual-PostgreSQL; COMP-22 dual datasource; COMP-26 single datasource via `DriverManagerDataSource`; COMP-30 dual datasource; COMP-32 dual datasource read-only). |

---

## Current Work Position

**Phase:** Source-verification reconciliation phase. Cross-cutting derivatives being rebuilt from accumulated Pending Entries.

**Reconciliation history:**

| Date | Derivatives rebuilt | Entries consumed | Notes |
|------|---------------------|------------------|-------|
| 2026-04-25 | WDP-DB.md v2.1, WDP-KAFKA.md v2.1, WDP-HANDOVER.md v3.1 | 20 entries (COMP-07, 08, 09, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 23, 24, 27, 37, 41, 43) | First reconciliation session. NFRS / DECISIONS / INTEGRATIONS / ARCHITECTURE / COMP-INDEX still pending. |
| 2026-04-30 | WDP-DB.md v2.2, WDP-KAFKA.md v2.2, WDP-COMP-INDEX.md v2.2, WDP-FLOW-INDEX.md v1.1, WDP-HANDOVER.md v3.2 | 18 entries (COMP-01, 02, 03, 04, 05, 06, 22, 25, 26, 28, 30, 32, 35, 36, 39, 40, 42, 51) | Second reconciliation session. NFRS / DECISIONS / INTEGRATIONS / ARCHITECTURE rebuild still pending — see backlog above. |

**Next work — Reconcile remaining Tier 1 derivatives in priority order:**

1. **WDP-NFRS.md v2.2** — Section 6 Risk Register expansion (~70 candidate RISK rows from 2026-04-28/29/30 audits)
2. **WDP-DECISIONS.md v2.2** — 30+ candidate ADRs surfaced; deviation map enrichment for DEC-001/003/004/005/019/020/021/023
3. **WDP-INTEGRATIONS.md v2.2** — Enrichment-chain detail, CVV finding, BEN cluster contract, JustAI scope, NAP-DPS auth gap
4. **WDP-ARCHITECTURE.md v2.2** — Minimal updates: component count, Case Expiry Subsystem grouping, file_generation_event correction

**After reconciliation backlog cleared:**

5. **Architect confirmation rounds** — promote 📝 DRAFT 🔍 files to ✅ COMPLETE as architect signs off (38 components ready)
6. **Source-verification of remaining 4 components** — COMP-29, 31, 34, 38
7. **WDP-FLOW-[NAME].md files** — document the 11 core flows identified in WDP-FLOW-INDEX.md
8. **COMP-33 OrgManagementService** — document when GitHub repo is found
9. **COMP-45, 46, 47 File Generation** — document when sprint work is complete
10. **UI Portal documentation** — COMP-49 and COMP-50 as a separate action item

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

**Option A — Continuing the reconciliation backlog:**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first, then WDP-CHANGE-LOG.md.
Today I want to reconcile WDP-NFRS.md against the
already-Reconciled entries from 2026-04-30."
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
- No CORE platform value exists in API Gateway source code. **CORE/VAP/LATAM authorization is a confirmed gap (2026-04-30)** — these platform types receive no role-level or case-level authorization at the gateway layer. Architect decision required.
- **(2026-04-30)** COMP-01 listen port is `8082` — confirmed in both `application.yml:L2` and K8s manifest. Prior uncertainty about port 9082 was a documentation error.
- **(2026-04-30)** COMP-01 `v-correlation-id` header on all UAMS and CHAS calls is always null — `RequestCorrelation.setId()` is never called anywhere in the codebase, and `ThreadLocal` is incompatible with the reactive WebFlux threading model.
- **(2026-04-30)** COMP-01 null-platform auto-authorize path in `AuthorizationServiceImpl` is **unreachable from `CaseNumberFilter`** — the filter guards `platform != null && caseId != null` before calling the service. Latent dead code, not active bypass.
- **(2026-04-30)** COMP-01 `wdp-internal-auth` dead config confirmed across all 7 environment YAMLs (local, dev, test, uat, stg, cert, prod) — all use the wrong namespace `spring.security.oauth2.resourceserver.client.*`.
- **(2026-04-30)** COMP-01 Ingress — all 7 hostname entries use path-based routing at `/api/merchant/gcp` (`pathType: ImplementationSpecific`) to port 8082.
- `validateOrgId()` is commented out on GET /orgentity in CHAS (COMP-03). Any authenticated PIN-platform caller can query any org hierarchy without scope validation. **(2026-04-29 update)** The method body is itself absent from `RequestValidator` — remediation requires reimplementation, not uncomment.
- **(2026-04-29)** COMP-03 CHAS exposes **nine** REST endpoints across two controllers (`AuthorizationController`, `MerchantController`, `ProductEntitlementController`, `MerchantControllerV2`). v1.0 documented six.
- **(2026-04-29)** COMP-03 internal firm bypass (`iss` contains `us_worldpay_fis_int`) applies to **BOTH** `/authorize` AND `/entity-authorize` — both share `AuthorizationServiceImpl.authorizeEntity`.
- **(2026-04-29)** COMP-03 internal firm **partial-bypass** on `POST /{platform}/merchant/entitytype` (V1 and V2): internal callers skip JWT `iqentities` extraction and use `orgId` from request body directly. DB queries still execute.
- **(2026-04-29)** COMP-03 `validatePlatform` accepts **both PIN and CORE** (not PIN-only as v1.0 stated). Affects all four V1 hierarchy data endpoints.
- COMP-24 CaseActionService has no RBAC enforcement — `RestInvoker.authorizeUser()` exists but is never called from any controller. Recorded as DEC-018 (Accepted Risk).
- **(2026-04-24)** COMP-27 CaseSearchService `POST /lft` derives internal-vs-external authorization from a request-body field (`LftSearchParams.isInternal`), not from the JWT. Flagged for architect review.
- **(2026-04-24)** External VAP and LATAM callers are not routed through any entity-scope authorization service in COMP-27. Only NAP (→ UAMS) and PIN/CORE (→ CHAS) are. Confirm intent.
- **(2026-04-24)** Internal PIN regular-user role exists (`WDP_PIN_REGULAR` in `AuthorizationList`) alongside the already-known `WDP_NAP_REGULAR`. COMP-27 confirmed.
- **(2026-04-25)** COMP-21 ChargebackService `/actuator/prometheus` requires JWT — NOT in security whitelist (corrected from prior draft).
- **(2026-04-23)** COMP-19 / COMP-20 liveness/readiness probe paths end in `livez` / `readyz` (with `z`), not `liver` / `lives` / `ready` as v1.0 stated.
- **(2026-04-29)** COMP-29 / COMP-35 / COMP-30 / others — `/actuator/prometheus` JWT-protected pattern confirmed widespread; **scrape-side configuration must carry JWT or whitelisting needs platform-wide remediation.**

### Kafka — Platform-Wide Patterns

- Every confirmed Kafka consumer in WDP uses pre-ACK or mid-flow ACK. DEC-005 (commit after full processing) is aspirational — it is not the current implementation.
- No Kafka DLQ topics exist in WDP. Error handling is via database error tables per consumer (e.g. `NAP.DISPUTE_EVENT_CONSUMER_ERROR` for COMP-05; `wdp.outgoing_event_outbox` for COMP-17/41/42/43/51; `wdp.bre_orchestration_outbox` for COMP-18; `wdp.chbk_outbox_row` for COMP-14/15).
- **(2026-04-18+)** Empty anonymous `CommonErrorHandler{}` is registered platform-wide on multiple consumers (COMP-05, COMP-14, COMP-15, COMP-16, COMP-18, COMP-39, COMP-40, COMP-41, COMP-42, COMP-43). Combined with `ErrorHandlingDeserializer`, deserialisation exceptions and unhandled application exceptions are silently swallowed — no DLT, no log, no counter.
- No circuit breakers (Resilience4j) anywhere in WDP. Confirmed absent across all 38 source-verified component files. DEC-014 formally voided.
- **business-rules topic has SIX confirmed publishers:** COMP-12 Scheduler4 (BUSINESS_RULES rows), COMP-15 EvidenceConsumer (WDP path, isMultiDocPending=false), COMP-23 CaseManagementService (kafka-before-commit pattern), COMP-24 CaseActionService (BRE inside `@Transactional`), COMP-25 NotesService (inside `@Transactional`), COMP-37 DocumentManagementService. All use `caseNumber` as partition key — DEC-003 uniform deviation.
- **(2026-04-18)** **COMP-14 CaseCreationConsumer is NOT a publisher of `business-rules`** — confirmed by absence audit (no `KafkaTemplate`, no `ProducerFactory`, no reference to topic in source).
- **(2026-04-28)** **COMP-22 DisputeService is NOT a Kafka producer at runtime** — wired but commented out (commit `c29018cd` by Shringi Nitin (WP), 2025-08-08, message "code changes (#93)"). No in-source rationale.
- **(2026-04-28+)** **`wdp.outgoing_event_outbox` has FIVE writers** (was three pre-2026-04-30): COMP-17 (EXPIRY_EVENTS, `WCSEEXPC`), COMP-41 (GP_EVENTS, `WNEC`), COMP-42 (BEN_EVENTS, `WBENC`), COMP-43 (CORE_EVENTS, `PCSECRTC`), COMP-51 (EXPIRY_BATCH, `WCSEEXPB`). Each writer uses a distinct `channel_type` discriminator. **No DB-level UNIQUE constraint visible in any of the five repos** — DBA confirmation required.
- `wdp.bre_orchestration_outbox` is shared between COMP-18 (`component=NOTIFICATION_ORCHESTRATOR` rows) and COMP-12 Scheduler4 (`component=BUSINESS_RULES` rows). PUBLISHED-status orphan rows have no automatic re-drive — manual intervention required.
- **(2026-04-18)** **COMP-18 does NOT write to `wdp.outgoing_event_outbox`** — WDP-DB.md v2.0 was incorrect on this point. Source grep confirms zero references.
- **(2026-04-18)** COMP-12 InboundDisputeEventScheduler uses **mark-and-send within a single `@Transactional`** — broker ACK precedes TX commit. **At-least-once with duplicate-possible crash window** (NOT at-most-once). Consumer-side `idempotency-key` dedup is the contracted mitigation.
- **(2026-04-18)** COMP-12 channelTypeTopicMap non-prod default keys: `EXPIRY_EVENTS`, `CORE_EVENTS`, `GP_EVENTS`, `BEN_EVENTS` (NOT `SEN_EVENTS`).
- **(2026-04-18)** COMP-12 Kafka-metadata write-back asymmetry: `new-case-events` and `case-evidence-events` paths persist `kafka_offset`/`kafka_partition`/`kafka_topic` back to the source outbox row; the other three paths do not. Incident correlation requires log-side join on `idempotencyId`.
- **(2026-04-25)** COMP-41 ThirdPartyNotificationConsumer writes `channel_type=GP_EVENTS` (NOT `GF_EVENTS`). Three distinct PUBLISHED-orphan paths confirmed.
- **(2026-04-29)** COMP-04 NAPDisputeEventService partition-key per-endpoint mapping: `merchantId` on case-update and outcome paths; `cardAcceptorCodeId` on POST `/event` (new-dispute path) only.
- **(2026-04-29)** COMP-04 `${kafka_retry_count}` and `${kafka_retry_delay}` env vars **also govern** CaseManagement (new-event path), FraudSwitch, and DisplayCodeService outbound REST retries. Hidden coupling — tuning Kafka retries silently retunes those three REST dependencies.
- case-action-events publisher confirmed as COMP-18 NotificationOrchestrator (Filter 1 EXPIRY_EVENT routing). cert environment uses distinct topic `case-action-events-cert` and group `case-action-events-group-cert`.
- **(2026-04-25)** COMP-43 CoreNotificationConsumer is the sole WDP component that writes to IBM DB2 Core Platform. COMP-43 reads only `BC.TBC_DM_CASE` (enrichment fully REST-driven). Clear-PAN column is `I_ACCT_CDH` (NOT `I_ACCT_CDR`). PAN-last-4 column is `I_ACCT_CDH_LST`.
- `wdp.file_notifications` does not exist. The actual table is `wdp.file_generation_event`. COMP-18 is the sole writer (`created_by/updated_by=WDPOUCU`, INSERT-only, status hardcoded `STAGED`).
- **(2026-04-29)** **COMP-39 is the consumer of `internal-integration-events` for the NAP outbound path** — single `@KafkaListener` confirmed. COMP-39 is therefore **NOT** the consumer of COMP-24's `${kafka.topic}` ActionEvent topic; that consumer remains unidentified.
- **(2026-04-29)** Operational footgun pattern: env vars without YAML defaults — if unset, service fails to start. Confirmed: `${kafka_business_event_topic}` (COMP-25), `${kafka_retry_count}` and others (COMP-04), `gcp_async_corepoolsize` / `core_migration_status` (COMP-22), feature flags on COMP-07.

### Confirmed DEC Violations and Voids

- DEC-011 ⛔ VOID — BRE step checkpointing confirmed never implemented in COMP-16. Formally voided in WDP-DECISIONS.md v2.0.
- DEC-014 ⛔ VOID — Resilience4j confirmed absent across all 38 source-verified component files. Formally voided in WDP-DECISIONS.md v2.0.
- DEC-004 violation in COMP-23 — clear PAN written to persistent storage on standard case creation (`wdp.CASE.I_ACCT_CDH` and `nap.case.I_ACCT_CDH`). Formally recorded as DEC-019 (Accepted Risk).
- **(2026-04-25)** DEC-004 / DEC-019 violation in COMP-43 — clear PAN written to `BC.TBC_DM_CASE.I_ACCT_CDH` (DB2) on CREATE + actionSeq=01 path. Architect decision required (OQ-COMP-43-6).
- **🔴 (2026-04-29) NEW PCI-DSS finding — COMP-05 CVV at rest:** `NAP.DISPUTE_EVENT_CONSUMER_ERROR` persists CVV in two places per error row — `C_CVV` column directly mapped on the entity, AND raw `NotificationEvent` JSON serialised into `C_KAFKA_EVENT` which also contains the `cvv` field. Cross-link: COMP-04 logs the CVV via Lombok `toString()`; COMP-05 persists it. Architect decision required.
- DEC-001 (transactional outbox) — confirmed deviations:
  - COMP-04 (direct publish on HTTP thread) — confirmed source-verified 2026-04-29
  - COMP-15 (synchronous publish inside `@Transactional` — ghost-event window)
  - COMP-16 (direct synchronous publish, no outbox)
  - COMP-18 ⚠️ PARTIAL (uses `wdp.bre_orchestration_outbox` but no `@Transactional`; four distinct write points are independent auto-commits)
  - COMP-19 (direct synchronous publish; HTTP 500 on Kafka failure does not undo case action)
  - COMP-20 (direct synchronous publish)
  - COMP-23 — `kafka-before-commit` pattern: publish inside `@Transactional` BEFORE commit
  - COMP-24 ⚠️ PARTIAL — BRE publish IS inside `@Transactional`. ActionEvent publish is OUTSIDE `@Transactional` on EP 2/8/9
  - COMP-25 (synchronous publish inside `@Transactional`; **mid-batch atomicity** — per-event loop produces deterministic Kafka-orphan events on partial-batch failure, added 2026-04-28)
  - COMP-37 (direct synchronous publish post-DDB on primary upload; questionnaire path mitigates with `@Transactional(rollbackOn=Exception.class)`)
  - COMP-43 (separate transaction managers — outbox INSERT on `wdpTransactionManager` commits before DB2 write on `coreTransactionManager`; crash window wide open)
- **🔴 (2026-04-29) DEC-021 wrong-TM scope expansion — COMP-02 UAMS:** previously documented as a one-method offence (`saveChildWithMerchant`). Source verification confirms **7 methods across 4 NAP tables** use `wdpTransactionManager` instead of `napTransactionManager` — `saveChildWithMerchant`, `createEntity`, `updateEntity`, `updateMerchantRelationships`, `deleteMerchantByParent`, `deleteChildEntity`, `deleteParentEntity`. Affects `nap_parent_entity`, `nap_child_entity`, `nap_merchant`, `nap_entity_rel`. Same severity class as DEC-019.
- **🔴 (2026-04-28) DEC-021 second offender — COMP-30 UserQueueSkillService:** service-level `@Transactional` on `createQueue`/`updateQueue` binds to `@Primary usTransactionManager` — UK writes to `nap.queues`, `nap.queue_criterion`, `nap.user_queue` are NOT covered by the outer TX. Same root cause class as COMP-02.
- DEC-020 (Accepted Risk) — no idempotency on case creation (COMP-23). Formally recorded.

### Key Confirmed Platform Facts

- BusinessRulesProcessor (COMP-16) makes direct DB calls to `nap.rules` / `wdp.rules` — does NOT call BusinessRulesService (COMP-31).
- NotificationOrchestrator (COMP-18) does NOT publish to `internal-integration-events` — that topic is published by AcceptService (COMP-19), ContestService (COMP-20), and conditionally COMP-16 (UK/NAP path).
- **(2026-04-28)** **COMP-22 DisputeService is read-only — confirmed source-verified zero writes.** No `INSERT`, `UPDATE`, `DELETE`, no Spring Data repositories, no `@Modifying`, no `JdbcTemplate`. Kafka producer to business-rules is wired but commented out — effectively Kafka-free at runtime. The COMP-37 column-level UPDATE on `wdp.CASE` is now confirmed orphan to COMP-22; cross-component review item recorded for COMP-22/COMP-37 is **closed on the COMP-22 side**. The COMP-37 INSERT/DELETE owner question stands.
- COMP-28 DisplayCodeService does NOT determine TIER1 eligibility — returns raw code lists. Eligibility logic belongs to calling services (e.g. COMP-04 NAPDisputeEventService).
- COMP-38 APILogService has no AOP inside it. Callers hold catch-blocks and invoke this service directly via REST.
- **(2026-04-29)** **COMP-36 TokenService is read-only on Redis** — source verified zero write operations. The `wdpinternalidptoken:token` Redis hash is populated by an UNKNOWN EXTERNAL COMPONENT not present in any audited WDP repo. External writer identity remains an open question.
- COMP-33 OrgManagementService — GitHub repository not found. Documentation deferred.
- `removeItemFromQueueDisabled` flag is a second active operational safety switch in COMP-07 and COMP-08. When true, suppresses ALL MarkAsRead/ACK PUT calls globally.
- BEN notification is delivered via Kafka publish to a BEN-owned MSK cluster using separate SASL/JAAS credentials — NOT via REST or webhook. Confirmed from COMP-42 source.
- **(2026-04-25 — corrected)** **JustAI integration scope:** JustAI is **active in COMP-21 ChargebackService** (consumer name `JUSTTAI` triggers partner branch in `isPartner` — note double-T spelling). It **remains absent from COMP-41**. The platform-wide statement "JustAI is planned only" was correct *for outbound third-party notification (COMP-41)* but not platform-wide. Signifyd is the sole live third-party vendor for outbound notification.
- **(2026-04-28)** **COMP-32 RulesService is a pure read-only REST API** — zero writes, zero Kafka, zero PAN, zero outbound REST/HTTP. Sole outbound dependency is two PostgreSQL datasources.
- **(2026-04-29)** **COMP-35 EncryptionService DEK rotation interval is days** (`${dek_rotation_interval_days}`) — prior platform docs stating "6-hour DEK cache" are incorrect.
- **(2026-04-29)** **COMP-39 hosts the manual NAP error reprocessing REST endpoint** (`POST /event`, JWT-authenticated). This resolves the long-standing question of which component hosts the manual reprocessing surface referenced in COMP-05's documentation — it is COMP-39, not COMP-05.
- **(2026-04-30)** **`minReadySeconds` misplacement** is a platform-wide pattern. Eight components confirmed with the defect: COMP-03, 04 (untouched), COMP-08, COMP-09, COMP-12, COMP-25, COMP-26, COMP-28, COMP-34, COMP-40. `minReadySeconds: 30` is placed under `spec.template.spec` instead of `spec` — silently ignored by Kubernetes. Candidate for platform-wide remediation pass.
- **(2026-04-25 + 2026-04-30)** **`v-correlation-id` regenerated-per-call anti-pattern** — confirmed in COMP-17 (write half) and COMP-51 (read half) of the Case Expiry Subsystem. End-to-end audit trail across the 4–5 calls per record cannot be reconstructed from headers alone. Same anti-pattern likely exists in other components — platform-wide remediation candidate.
- **(2026-04-30)** **Case Expiry Subsystem confirmed:** COMP-17 (writer half — Kafka consumer maintaining `wdp.case_expiry`) + COMP-51 (reader half — scheduled batch acting on past-due rows). They share `wdp.case_expiry` as their coordination surface with **no row-level lock, no version column, no SELECT FOR UPDATE**. Race possible.

### Component-Specific Source-Verified Findings (2026-04-18 / 23 / 24 / 25 / 28 / 29 / 30)

#### COMP-01 API Gateway

- Listen port `8082` (not 9082) confirmed in `application.yml:L2` and K8s manifest.
- 7-environment deployment (local, dev, test, uat, stg, cert, prod). All Ingress hostname entries use path-based routing at `/api/merchant/gcp` (`pathType: ImplementationSpecific`).
- `v-correlation-id` always null on UAMS and CHAS calls — `RequestCorrelation.setId()` never called; `ThreadLocal` incompatible with reactive WebFlux threading model.
- Null-platform auto-authorize path in `AuthorizationServiceImpl` is unreachable from `CaseNumberFilter` — latent dead code, not active bypass.
- `wdp-internal-auth` config dead across all 7 environment YAMLs (wrong namespace `spring.security.oauth2.resourceserver.client.*`).
- 26+ routes loaded from `wdp.api_route` at pod startup. Column set: `id`, `path`, `original_path_regex`, `replacement_path`, `uri`, `auth_exceptions`. Write owner still TBC.
- Blocking `RestTemplate` on Netty event loop — UAMS/CHAS calls have no timeout. Single degraded auth service can exhaust gateway threads. 🔴 HIGH.
- `authExceptions` trim bug — leading-space entries after comma-split silently miss whitelist match. 🟡 MEDIUM.
- Two unhandled NPE paths → 500 (route attr null; `AuthorizationList` claim absent). 🟡 MEDIUM.

#### COMP-02 UserAccessManagementService (UAMS)

- Source-verified 2026-04-29 — first source-verified pass.
- 3-layer authorization model on `/authorize`: consumer authorization → entity-class gate → entity-value lookup.
- Owns four NAP tables (`nap_parent_entity`, `nap_child_entity`, `nap_merchant`, `nap_entity_rel`) plus `wdp.acl`.
- 🔴 **DEC-021 wrong-TM scope = 7 methods** across the four NAP tables (see DEC Violations section above for full list). Same severity class as DEC-019.
- Kafka-free at runtime — vestigial `KAFKA_SERVICE_ERROR` enum exists but is application-layer only.
- Proxies all user lifecycle operations to SunGard IdP.
- DBA confirmation required for: UNIQUE on `nap.nap_parent_entity.c_name`; UNIQUE on `(nap.nap_child_entity.i_parent_entity_id, c_name)`; UNIQUE on `(nap.nap_entity_rel.i_parent_entity_id, i_child_entity_id, i_mid)`; UNIQUE composite on `wdp.acl.(c_consumer_name, c_source_system, c_entity_type, c_entity_value)`.

#### COMP-03 CoreHierarchyAuthorizationService (CHAS)

- 9 REST endpoints across two controllers (was 6 in v1.0).
- Internal firm bypass (`iss` contains `us_worldpay_fis_int`) applies to BOTH `/authorize` AND `/entity-authorize`.
- Internal firm partial-bypass on `POST /{platform}/merchant/entitytype` (V1 and V2): internal callers skip JWT `iqentities` extraction and use `orgId` from request body directly. DB queries still execute.
- `validatePlatform` accepts both PIN and CORE (not PIN-only).
- `validateOrgId()` method body is itself absent from `RequestValidator` — remediation requires reimplementation, not uncomment.
- Zero `@Transactional` annotations anywhere in `src/main`. Both transaction managers (`wdpTransactionManager`, `coreTransactionManager`) bean-wired but never enclose any service method.
- `minReadySeconds: 30` misplaced — eighth component confirmed with this defect.
- Clean DEC-004 / DEC-019 attestation — zero matches across source for PAN field names; zero persistent writes.
- Component-level alias corrections in `MerchantDaoImpl` native SQL (`MD.TMD_STORE` alias `TMD` not `TMD3`; `MD.TMD_MRCHNT` alias `TMD3` not `TMD5`). `MD.TMD_CHAIN_ANALY` is a declared-but-unused read target via dead `ProductEntitlementRepository` injection.

#### COMP-04 NAPDisputeEventService

- Source-verified 2026-04-29 via Copilot CLI.
- **Stateless — confirmed no database connection.**
- Enrichment chain is **alternative**, not sequential: CaseManagement is tried first; FraudSwitch only runs when CaseManagement returns null; DisplayCodeService only runs on the GUARPAY1/GUARPAY4 + fraudIndemnified branch.
- Bypass condition (`enrichment_failure=true OR function_code=603`) only skips field validation in `EventBusinessValidator`. Enrichment still runs.
- Bypass is honoured **only on POST /event**. Case-update and outcome paths do not pass through `EventBusinessValidator.validateRequest`.
- `enrichmentFailure` on outbound `NapEvent` is a **pass-through copy** of inbound `message.enrichment_failure` — service never sets it itself. Own enrichment failure → HTTP 500, no event published.
- IDP token fetch is **lazy and request-scoped** via `RequestTokenHolder`. No `@Retryable` — single attempt; IDP failure halts the request.
- `RestServiceInvoker` is the primary REST client (actively used). Three `RestTemplate` instances exist, all default-constructor — including a local instance inside `getDisplayCodeDetails` that shadows the field-level one.
- New tier mapping: **GUARPAY8 → TIER8** (not previously documented).
- POST `/response/document` logs the inbound DTO including the full base64 `file` field at controller entry via Lombok-generated `toString()`. Checkmarx sanitizer invoked but does not strip the base64 payload. Distinct from CVV finding, same family.
- `@EnableCaching` imported in the application class but the annotation is **not applied**. Spring caching is not active.
- K8s: maxSurge=1 maxUnavailable=0, topology spread, liveness `/livez:8082` (delay 35s), readiness `/readyz:8082` (delay 25s), no startup probe, no PDB, no HPA.

#### COMP-05 NAPDisputeEventProcessor

- Source-verified 2026-04-29 against `gcp-nap-dispute-event-consumer`. v1.0 → v2.0.
- 🔴 **CVV persisted at rest** in `NAP.DISPUTE_EVENT_CONSUMER_ERROR` — `C_CVV` column AND inside `C_KAFKA_EVENT` raw JSON. PCI-DSS concern. Architect decision required.
- 🔴 **Prior-error scan does NOT filter on `C_EVENT_TYPE`** — same cross-component shared-table consumption hazard as COMP-39. Decision needed on whether COMP-05 and COMP-39 share a uniform recovery model or whether the missing filter is a defect on both.
- Touches **4 DB tables** (primary: `NAP.DISPUTE_EVENT_CONSUMER_ERROR`; corrected: `NAP.issuer_raised_dispute_timefrm_rules` (NOT `nap.timeframe_rules`); also `NAP.BUSINESS_RULE_CONSUMER_ERROR` (a fourth table not in v1.0); read-only: `NAP.mcm_queue_data`).
- K8s manifest in repo (`resources.yaml`). Container port 8082. Liveness `/livez`, Readiness `/readyz`. No startup probe. No HPA. No PDB. `minReadySeconds: 30` misplaced (silently ignored).
- Retry pattern non-uniform — most steps `maxAttempts=1` (no retry); only Create/Update Case + Insert Notes + API Log use `${napevent.retrycount}`. Insert Notes and API Log are non-halting (write SKIPPED_FAILED, continue).
- Does NOT read the `idempotency-key` Kafka header. No inbound dedup. DEC-020 deviation confirmed.
- No Kafka producer wiring — dead `SEND_BUSINESS_RULES_INFO_TO_KAFKA` terminal state.
- Pre-ACK confirmed: `acknowledgment.acknowledge()` is the FIRST call in the listener.

#### COMP-06 NAPDisputeDeclineBatch

- Source-verified 2026-04-30. v1.0 → v1.1.
- Spring Batch single-step chunk-1 component running inside a long-lived K8s Deployment via internal `@Scheduled` cron (`spring.batch.job.enabled=false` — scheduler is sole launcher). Same deployment pattern as COMP-07/08/09 — *not* a K8s CronJob.
- Polls `nap.action` for open PAB/OPAB actions on migrated NAP Visa cases, queries Visa network state via internal Visa Adapter HyperSearch, creates IDCL draft actions for Pre-Arb Declined cases. VISA + NAP only.
- Two PostgreSQL datasources: NAP (read), WDP (Spring Batch metadata).
- Zero `RestTemplate` timeouts, zero K8s probes, zero Resilience4j, zero `@Retryable`. Same operational pattern class as COMP-21.
- Kafka client libraries on classpath but unused — only component in the audit set so far where Kafka libraries are vestigial rather than actively used or absent.
- Decommission-scoped — all MEDIUM risks accepted.
- Confirms COMP-23 inbound `GET /nap/case` and COMP-24 Endpoint 1 callers from the COMP-06 side — round-trip confirmation.

#### COMP-07 VisaDisputeBatch

- `skipCase` path writes no outbox row at all (previously implied SKIPPED was written).
- Writer save failures and MarkAsRead retry-exhaustion are caught-and-logged at `BatchItemWriter.java:139-142`, never propagated. Spring Batch sees chunk completion as successful.
- No `@Transactional` anywhere. Chunk transaction is the only transactional boundary.
- No liveness, readiness, or startup probes wired.
- Feature flags (`coreDataNeedToMigrate`, `removeItemFromQueueDisabled`, `readSpecificItemFromQueue`) have no defaults — startup fails if K8s Secret absent.
- JobLauncher is default `SyncTaskExecutor` — no custom TaskExecutor.
- `created_by` / `updated_by` constant: `"WVDPB"`.

#### COMP-08 FirstChargebackBatch

- `created_by` / `updated_by` constant: `"WMFDPB"`.
- Duplicate-check path writes SKIPPED marker rows (insert-a-marker pattern, distinct from COMP-07's suppress-insert).
- Update-path writes `status=PENDING` with `accountNumber=null` on newly-arrived chargebacks on existing claims — suspected defect.
- **Writer-ACK hazard** — mid-chunk JPA save failure does not prevent the subsequent ACK PUT to MCM; ACK PUT exceptions are swallowed silently. Items acknowledged off MCM with no corresponding outbox row are unrecoverable.
- Currency exponent hardcoded (`USD_EXPONENT=2`, `CAD_EXPONENT=2`) — actual currency never consulted.

#### COMP-09 CaseFillingBatch

- `created_by` / `updated_by` constant: `"WMPAPB"`.
- **Update path always INSERTs new row** — existing rows never UPDATEd. Multiple rows per claim through phase lifecycle (PENDING / SKIPPED).
- Writer swallows all save exceptions — chunk reports success even on DB save failure.
- Skip paths write no row at all.
- `c_case_stage` concatenation: base stage EXCEPT when `issuerCaseStatus = withdraw` → `{stage}_withdraw`.
- `network_notes` column written conditionally only when `schemeRef.notes` is non-empty.
- `c_ntwk_case_id` semantics: `standardClaimId` per claimType.

#### COMP-11 FileProcessor

- `created_by` constant: `"FILE_PROCESSOR"` (corrected 2026-04-18 from `"WPFLEPR"`).
- DNWK silent-file-loss paths are **two**: `DISCHYB` and `AMEXOPTB`. **`MC_REVREJ` IS registered.**
- Initial CHARGEBACK_PROCESS status PENDING via `@PrePersist`. LOADING is transient. DCPO evidence rows BLOCKED; DCPO parent stays PENDING.
- Only **two `@Transactional` sites** in the whole repo.
- DCPO CHARGEBACK_PROCESS save is auto-commit AND the per-image evidence loop has empty catch — orphan-parent risk.
- DNWK per-record failure path is one-retry-then-silent-row-loss.
- DEC-004 non-numeric-`acctNum` edge case bypasses encryption.
- `S3ServiceImpl` silently swallows get/move/put failures and returns null.

#### COMP-12 InboundDisputeEventScheduler

- Mark-and-send within single `@Transactional` — at-least-once duplicate-possible.
- No K8s probes (liveness, readiness, startup all absent).
- Bare `RestTemplate` in `config/CommonConfig.java` for Scheduler5 email POST — no timeouts, no pool.
- Shared `ThreadPoolTaskScheduler` with `poolSize=5` across five schedulers — sibling-starvation risk.
- channelTypeTopicMap non-prod default keys: `EXPIRY_EVENTS`, `CORE_EVENTS`, `GP_EVENTS`, `BEN_EVENTS`.
- `idempotencyId` typed as `UUID` on `chbk_outbox_row` entity but `String` on `outgoing_event_outbox` entity — cross-table type inconsistency.
- Archive INSERT SQL references column `c_ntwrk_phase_id` not mapped on `ChbkOutboxArchiveEntity` — DDL verification required.

#### COMP-13 FileAcknowledgementProcessor

- Writes EXACTLY four columns to `wdp.file_job` enforced by `@DynamicUpdate`: `ack_status`, `ack_generated_at`, `updated_at`, `updated_by`.
- Cross-component COMPLETED-transition contract: COMP-11 never writes COMPLETED; COMP-12 Scheduler2 owns it; COMP-13 polls for `status IN (COMPLETED, ERROR)`. Contract undocumented in all three repos.
- Three latent runtime bugs and one third `headObject` failure branch surfaced 2026-04-18.
- Zero tests of any kind.

#### COMP-14 CaseCreationConsumer

- Zero `@Transactional` annotations anywhere — all DB writes auto-commit.
- `auto.offset.reset = latest` — cold starts skip messages.
- ACK at `KafkaConsumer.java:38` precedes `processKafkaNotificationEvent()` at `:43`.
- Empty `CommonErrorHandler` registered.
- No liveness probe, no startup probe — only readiness.
- Layer-1 prior-chargeback dedup is in-memory stream filter — safe only by `concurrency=1`.
- Cert profile naming: `new-case-events-group-cert`.
- Confirmed NOT a publisher of `business-rules`.

#### COMP-15 EvidenceConsumer

- V3 path defect: MISCDOC / DRFTDOC / RESPQDOC / ISSRQDOC document types silently mark `file_evidence.attachment_status=ATTACHED` without any upload — data-integrity defect.
- V3 RESPDOC path: failed ownership-transfer PATCH leaves the document uploaded to V3 Core but the WDP DB unmarked — ghost-upload.
- WDP-path RESPDOC publishes **two** BR events per success.
- DSNOTDOC WDP events publish with `source=""` — builder has no DSNOTDOC branch.
- V3 Update Action PATCH mutates the shared `RestTemplate` bean's request factory — order-dependent global state.
- Inbound Kafka key (`caseNumber`) and `idempotency-key` header are both `@Nullable`.
- WDP-path gating is narrower than prior documentation implied.
- Platform string is uppercased on publish.

#### COMP-16 BusinessRulesProcessor

- Pre-ACK at `KafkaConsumer.java:38`; processing at `:41`.
- FCMG guard correction: actual guard is `FCHG / RCAL` pairing.
- US DB bean name: `wdpdataSource` (lowercase `d`).
- UK transaction manager: `ukTransactionManager`.
- Token cleanup asymmetric — UK INSIDE finally; US OUTSIDE finally.
- US-only `DOCUMENT_ATTACHED_TO_OPEN_CASE` recursive fallback.
- `ErrorLogService` class does not exist in source tree — autowire and call sites commented out.
- `isErrorOccured` flag from outgoing publish is ignored by caller.
- Audit-log sequences: `nap.br_case_audit_log_id_seq`, `wdp.br_case_audit_log_id_seq`, both `allocationSize=1`.

#### COMP-17 CaseExpiryUpdateConsumer

- Kafka headers are kebab-case on the wire: `idempotency-key`, `event-timestamp`.
- `max.poll.records=500` and `max.poll.interval.ms=600000` (10 min).
- Cert environment uses distinct topic `case-action-events-cert` and group `case-action-events-group-cert`.
- SASL: `AWS_MSK_IAM` over `SASL_SSL`.
- Predecessor query is **not scoped on `i_action_seq`** — cross-action interference.
- Predecessor blocking: any non-ERROR prior row maps to PENDING_DEFERRED.
- Outbox writes have **no service-level `@Transactional`**; `case_expiry` writes are `@Transactional REQUIRED`. Non-atomic split.
- Poison-message handling: deserialisation failure NPEs at path-selection branch BEFORE any ACK; redelivers indefinitely.
- `v-correlation-id` not propagated on IDP token call — only on case-search call.
- No liveness, readiness, or startup probes wired.
- Six fully unused + 1 partially-wired pom dependencies.
- Dedup detection key fields: `(idempotencyId, channelType, eventTimestamp)`.
- No `@SchedulerLock`, advisory lock, or singleton guard.
- **(2026-04-30 cross-link)** Paired with COMP-51 as the writer half of the Case Expiry Subsystem; shared `wdp.case_expiry` write is unguarded.

#### COMP-18 NotificationOrchestrator

- Step 6 previous-event guard is two-stage: SQL `WHERE caseId=?` returns all rows; `id<eventId`, `component`, and `status NOT IN (SUCCESS, SKIPPED)` filters applied in-memory.
- Filter 4 Sub-condition B stage+action set: `{RE2+REPR, PAB+MDCL, REQ+RRSP, ARB+MDCL}`.
- `documentNameList` IS published on `PublishedNotificationEvent` when set.
- Has `readinessProbe` and `livenessProbe` configured; NO `startupProbe`.
- Default Actuator exposure only.
- Zero `@Transactional` annotations. Four distinct outbox write points are independent auto-commits.
- Sole writer of `wdp.file_generation_event`.
- MDC correlation not wired — `HttpInterceptor` puts MDC but is never registered.
- Dead code to remove: `findByIdempotencyIdAndComponent`, `HttpInterceptor` class, `RequestCorrelation` ThreadLocal, `org.apache.httpcomponents:httpclient` dependency.
- FNS file-notification-service code is live awaiting downstream COMP-44 EDIA Consumer.

#### COMP-19 AcceptService

- Stage code is **CHI** throughout all three validators.
- VISA PAB permits **WDNL** (in addition to IDCL/IPAB/CHGM). VISA APC uses **PCMP**.
- `expiryFlow=true` EACP override applies to first action of outgoing `AddActionRequest` for ALL card networks (not Visa-only).
- MC CHI silent no-op applies to **both NAP and PIN**.
- `errorLogService.saveErrorLog` — 2 of 9 call sites are ACTIVE.
- Liveness probe: `GET /merchant/gcp/accept/livez`; readiness: `/readyz`.
- 🔴 **NEW HIGH split-brain finding:** MC CHI on NAP can publish `AcceptEvent` without MC network notification.
- 🔴 **NEW HIGH split-brain finding:** AMEX/DISCOVER on NAP can publish `AcceptEvent` without network notification.
- 🟠 **NEW HIGH:** Compensation has no inner try/catch.
- Add-case-action runs AFTER the network call. Compensation `updateAction(ERROR)` targets the original action sequence.
- Kafka gate actionCode is sourced from `CaseLookupResponse`.

#### COMP-20 ContestService

- Max Kafka publishes per request is **2**, not 3.
- Card-network failures return **HTTP 400** (not HTTP 500). Only Kafka publish exhaustion and internal `RuntimeException` paths return HTTP 500.
- Liveness probe path is `/livez`.
- ErrorLogService commented-out call sites: 8.
- DEC-014 reclassified: absence-of-library fact, not a deviation.
- MCM submission contract: `createSecondPresentmentChargeback` (CH1/RE2 — POST) and `retrieveClaim + updateCase(action=REJECT)` (PAB — GET then PUT).
- DataPower non-NAP path uses same VANTIV license header as Visa DataPower (shared `licenseKey` secret).

#### COMP-21 ChargebackService

- 38 distinct downstream call sites across 12 target applications. 4 are dead code.
- **No IDP token cache** — `CachedTokenServiceImpl` delegates straight through to a fresh GET on every call.
- `asyncExecutor` is core=1, max=1, queue=5.
- `IdpRestInvoker` mutates the shared `RestTemplate` `setErrorHandler` per token call.
- Logstash appender effectively broken — production secret `logstash_server_host_port` is empty.
- Surfaces full `cardNumber` in two model classes; `cardNumberLast4` in eight others.
- Dead config: `wdp.caseUpdateUrl` injected as field but never read.
- LATAM silent fall-through.
- `JustAI` IS active (consumer name `JUSTTAI` triggers partner branch).
- Rollout `maxSurge=4`.
- `/actuator/prometheus` requires JWT — NOT in security whitelist.
- Confirmed Kafka-free.
- DB-free (no JPA, JdbcTemplate, DataSource references).
- Enterprise Cert eAPI Entitlement Lookup wired and ready (`entitlementFlag=false` currently).

#### COMP-22 DisputeService

- Source-verified 2026-04-28. v1.0 → v2.0.
- **Confirmed source-verified zero writes** — no `INSERT`, `UPDATE`, `DELETE`, no Spring Data repositories, no `@Modifying`, no `JdbcTemplate`. The COMP-37 column-level UPDATE on `wdp.CASE` is now confirmed orphan to COMP-22; cross-component review item closed on the COMP-22 side.
- Kafka publish disablement attribution: commit `c29018cd`, Shringi Nitin (WP), 2025-08-08, message "code changes (#93)". No in-source rationale.
- Runtime is **Spring Boot 3.5.12** (not 3.5).
- Readiness probe is **port 8082 path `/readyz`** (not port 8052).
- SFG SFTP fallback is **fully `@Async`** end-to-end. HTTP 200 returned before SFTP write completes. Exceptions inside the executor may be lost.
- SFG SFTP filename is `file.getOriginalFilename()` raw — no case-number prefix, no action-sequence, no timestamp. Filename collision possible.
- Internal-firm enforcement is application-layer (not Spring Security), `contains` check on JWT `iss` claim against `us_worldpay_fis_int`. `ForbiddenException` constructor uses `HttpStatus.UNAUTHORIZED` but global handler returns 403 — dead status code.
- Does **not** propagate `v-correlation-id` on any outbound REST call.
- HikariCP pools and Tomcat threads are unconfigured — Spring Boot defaults apply.
- `@Async` SFTP executor sized via env vars with **no defaults** — startup-fails-if-absent.
- Ships exactly two application endpoints. Sweep confirmed zero `@KafkaListener`, zero `@Scheduled`, zero `@JmsListener`, zero `@RabbitListener`, zero WebSocket / SSE handlers.
- Platform set on `/documents` path: `NAP`, `VAP`, `CORE`, `LATAM`. NAP comparisons use `equalsIgnoreCase`.
- Swagger / OpenAPI exposed in non-prod environments only.
- Response field names: `dsptSummary`, `creditDsptSummary`, `debitDsptSummary` (not `dept*`); `outstandingDsptAmount` / `outstandingDsptCount`.

#### COMP-23 CaseManagementService

- `kafka-before-commit` pattern: publish inside `@Transactional` BEFORE commit.
- **Owns 6 tables, not 7** — `wdp.dispute_event_change_log` REMOVED.
- NAP `nap.case` PAN column is `I_ACCT_CDH`.
- US create path duplicate-`wdp.NOTES`-insert defect: two identical `USNotesEntity` save blocks execute when `notesRequest != null`.
- NAP create path blind-merge on `NAP.DISPUTE_EVENT_CONSUMER_ERROR` — cross-component write into COMP-05-owned table with no owner check.
- `wdp.chbk_outbox_row` update path uses `findById(...).isPresent()` guard. No terminal-status guard — can overwrite PROCESSED row.
- No Flyway / Liquibase / `schema.sql` / `data.sql` in repo.
- Owner of case-number sequences: `nap.case_i_case_sequence` (NAP) and `wdp.pin_case_i_case_sequence` (shared by PIN, CORE, VAP, LATAM).
- `RequestCorrelation` ThreadLocal leak.
- `RestTemplate` mutation pattern.
- `spring-boot-devtools` shipping in COMP-23 prod image.
- `getRandomDigits` returns null once sequence length + prefix + random alpha reaches 12 — NPE risk.

#### COMP-24 CaseActionService

- BRE Kafka publish IS INSIDE `@Transactional`.
- ActionEvent publish is OUTSIDE `@Transactional` on EP 2 / 8 / 9 — genuine post-commit split-brain when `napUpdateEvent=true`.
- FULL_CTM action code is **CHGM** (not `CHMR`).
- NAP conditional outbox writes to `nap.DISPUTE_EVENT_CONSUMER_ERROR`.
- `wdp.dispute_event_change_log` does not exist in this repo.
- Open-action constraint enforced in-memory only.
- Last-write-wins on shared case/action tables.
- NAP vs US functional asymmetry on EP 5: NAP silently ignores `chbkOutbox`.
- EP 9 three-transaction sequence with no compensation.
- No connection or read timeouts on any outbound REST.
- UAT IDP token URI hardcoded as default in `application.yml` line 42.
- `idempotency-key` captured by HttpInterceptor and forwarded but never validated server-side.

#### COMP-25 NotesService

- Source-verified 2026-04-28. v1.0 → v1.1.
- Mid-batch atomicity: per-event loop inside `@Transactional` produces deterministic Kafka-orphan events on partial-batch failure (not just JVM-crash window).
- `kafka_business_event_topic` env var has no default — service fails to start if unset.
- Payload is the 11-field `AddNotesBREvent` structure (`eventType="NOTE_ADDED"`, `startRuleGroup="NOTE_ADDED_TO_CASE"`, `previousActionSequence` and `documentNameList` always null).
- COMP-25 has no awareness of COMP-23 / COMP-24 co-writers on `wdp.NOTES`.

#### COMP-26 QuestionnaireService

- Source-verified 2026-04-28. v1.0 → v1.1.
- NO `@Transactional` annotation anywhere in service or controller code. Multi-step flows (PUT, B1 UPSERT) span multiple per-call default transactions — TOCTOU race exposure.
- Uses `DriverManagerDataSource` (no HikariCP, no application-tier pool). Every JPA call opens a new JDBC connection.
- 20 endpoints (15 Visa + 5 Dispute).
- Error body field name: `errorMessage` (not `message`).
- HTTP 405 via `HttpRequestMethodNotSupportedException` handler. Status set: {200, 201, 400, 401, 404, 405, 500}; no 403, 409, 415, 422, 503 paths.
- Maps `HttpMessageNotReadableException` to HTTP 500 (not conventional 400). Invalid platform string also returns HTTP 500.
- No Liquibase / Flyway / SQL migrations. DB-level constraints not determinable from source.
- `minReadySeconds: 30` mis-indented under `spec.template.spec` — silently ignored.
- A2 Pre-Compliance cross-field validation is symmetric.
- `StandardEntityError` and `createDuplicateResponseEntity()` helpers in `GlobalExceptionHandler` are dead code.
- POST idempotency gap confirmed — no application-level pre-existing-row check, no DB-level UNIQUE visible.

#### COMP-27 CaseSearchService

- **15 REST endpoints** across 6 controllers (not 14 as v1.0 stated).
- **Two write operations** — V1 PATCH and V2 POST case-lock endpoints. Concurrent-locker race unguarded.
- Platform-wide instance of DEC-014 VOID pattern.
- `migrationStatus` mapping has TWO `as migrationStatus` aliases on different source columns — second overrides first.
- Endpoint count corrections: V1 default `pageSize` is 25; V2 `startRecordNumber`/`pageSize` declared as String; JWT claim is `iqorgid`.
- Ingress host count: 4. Rollout `maxSurge=4`.
- `/progress` endpoint creates 4 futures but joins only 2 — orphans `caseDetails` and `documentResponse`.
- NAP/WDP lock-timestamp column-name drift: NAP `x_locked_at`; WDP `z_locked_at`.
- `RECEIVED_DOCUMENT` queue criterion enum returns literal SQL identifier with space character.
- `value` column content concatenated into SQL by `QueueSupport.buildQueueCriterion`.

#### COMP-28 DisplayCodeService

- Source-verified 2026-04-28. v1.0 → v1.1.
- Does NOT determine TIER1 sub-product eligibility — service returns raw code lists; eligibility logic is the caller's responsibility.
- Two repository projections on `wdp.dispute_static_tabs_rules`: `findByRoles` (POST /search) returns 11 Y/N flag columns; `getPermissions` (GET /privileges) returns 17 Y/N flag columns. The 6-column delta (`fax_match`, `fax_report`, `trans_detail`, `auth_detail`, `settle_detail`, `dispute_history`) is the source of permission-shape divergence between endpoints.

#### COMP-30 UserQueueSkillService

- Source-verified 2026-04-28. v1.0 → v1.1.
- Confirmed dual-datasource design with `ukTransactionManager` (UK / nap) and `usTransactionManager` (US / wdp, `@Primary`). No XA.
- Confirmed Kafka-free at runtime.
- Maven artifactId is `user-queue-skill-service`.
- Auto-enrol on POST /user targets queues of type `SKILLS_INTERNAL` **OR** `DEFAULT_QUEUE`.
- 🔴 **DEC-021 wrong-TM** — service-level `@Transactional` on `createQueue`/`updateQueue` binds to `@Primary usTransactionManager`; UK writes NOT covered by outer TX.
- POST /user has no `@Transactional` — auto-enrol side-effect runs in a separate implicit transaction; failure leaves orphaned `nap.users` row.

#### COMP-32 RulesService

- Source-verified 2026-04-28. v1.0 → v1.1.
- Pure read-only REST API. Zero writes, zero Kafka, zero PAN, zero outbound REST/HTTP, zero `@Transactional` mutations. Sole outbound dependency is two PostgreSQL datasources (`wdpdataSource` `@Primary` + `napdataSource`), via `DataSourceBuilder.create().build()` with no explicit pool tuning.
- Hosts 14 active REST endpoints + 2 controller-disabled endpoints with full backing code intact. The 14 active endpoints map 1:1 to 14 distinct `@Cacheable` cache names backed by Spring's default `ConcurrentMapCacheManager`.
- `/actionrules` is the sole endpoint that performs caller-identity-aware response shaping, via JWT `AuthorizationList` claim ∩ `app.action.roles` config list.
- `/actionrules` `migrationStatus="N"` is an active production migration kill-switch — DB bypassed, hardcoded all-N response. Constant `MIGRATION_STATUS = "Y"` is misleadingly named.
- K8s probes via Spring Boot Actuator health groups — `/livez` (initialDelay 40s) and `/readyz` (initialDelay 30s) under servlet context `/merchant/gcp/rules`.

#### COMP-35 EncryptionService

- Source-verified 2026-04-29. v1.0 → v1.1.
- Owns `wdp.hash_key_store` and `wdp.data_enc_key` — sole writer and sole reader of both. No shared-table writes.
- DEK rotation interval is **days** (`${dek_rotation_interval_days}`). Prior platform docs stating "6-hour DEK cache" are incorrect.
- Decrypt `@Transactional` boundary brackets the KMS network call. A Hikari connection is held during the KMS round-trip on every decrypt request.
- `/actuator/prometheus` and `/actuator/info` are NOT in the security permit-all list; they require JWT.
- 🔴 `rotateDEK()` reuse-existing-DEK branch carries a Base64 bug — `dek_enc` is passed to KMS as raw UTF-8 bytes of the Base64 string instead of decoded ciphertext. Symptom only manifests on pod restart that finds a recent-enough DEK row to reuse. Failure is silently swallowed.
- No method-level authorization on the decrypt endpoint. Decrypt-caller policy (COMP-14 only) is convention, not technical control.
- `PANHashingServiceImpl` logs last-4 PAN at DEBUG.
- Spring Boot version is 3.5.6 (precision).

#### COMP-36 TokenService

- Source-verified 2026-04-29. v1.0 → v1.1.
- **Source-verified read-only on Redis hash** — performs HGET only; zero write operations exist anywhere in the wdp-idp-token-service repository.
- The `wdpinternalidptoken:token` Redis hash is populated by an UNKNOWN EXTERNAL COMPONENT not present in any audited WDP repo. External writer identity is an open question.
- Two-layer cache: AWS ElastiCache Redis (primary) + Spring in-memory OAuth2 store (secondary). Falls through to IDP `client_credentials` grant only on full cache miss.

#### COMP-37 DocumentManagementService

- **IS a Kafka producer** (6th publisher of `business-rules`).
- Primary upload step order: `S3 → DDB → desk update → Kafka publish → action-indicator update`. Action-indicator update is LAST step.
- Endpoint 11 (POST questionnaire) is the only publish path in the platform that wraps Kafka send inside `@Transactional(rollbackOn=Exception.class)`.
- Endpoint 8 NAP base64 path NEVER publishes (controller forces `notifyBRQueue=false`).
- `USCaseEntity` and `UKCaseEntity` are column-level UPDATE co-writers only (desk blanking). INSERT/DELETE owner question stands (closed on the COMP-22 side per 2026-04-28 audit).
- Five DynamoDB GSIs declared but queries none from Java.
- Endpoint 11 HTTP verb is POST.
- DynamoDB duplicate check is application-level query + in-memory compare, non-atomic.
- `idempotency-key` header accepted, logged, MDC-tagged, echoed, propagated as outbound Kafka record header but NOT used for deduplication.

#### COMP-39 NAPOutcomeProcessor

- Source-verified 2026-04-29. v1.0 → v2.0.
- **Hosts the manual NAP error reprocessing REST endpoint** (`POST /event`, JWT-authenticated via `JwtIssuerAuthenticationManagerResolver` resolved from `${trusted_issuers}`). Resolves the long-standing question of which component hosts the manual reprocessing surface — it is COMP-39, not COMP-05.
- Fourth confirmed writer of `NAP.DISPUTE_EVENT_CONSUMER_ERROR` (alongside COMP-05 primary, COMP-23, COMP-24). Discriminator: `C_ACQ_PLATFORM` (mapped from `sourceSystemName`); COMP-39 sets `"NAP"` constant on insert. `C_EVENT_TYPE` values for COMP-39 rows: `OUT_SRV118`, `OUT_SRV117`.
- 🔴 **Prior-error scan filters on `caseNumber` + `sourceSystemName="NAP"` + status IN (FAILED1, FAILED2) with no `C_EVENT_TYPE` filter** — error records written by COMP-05/23/24 against the same case may be reprocessed through COMP-39's SRV118/SRV117 outbound pipeline. Same defect class as COMP-05 — uniform ADR required.
- `@EnableRetry` wired on the application class — Spring Retry is live (distinct from COMP-41 where the imports were dead).
- Confirms at-most-once pre-ACK as a second component with this pattern (alongside COMP-05). Both write to the same shared error table.
- Audit user constant: `"NCSEUPDTC"` (corrected from v1.0 transcription `"NCSEUDPTC"`).
- No `application-{env}.yml` profiles in repo — every `${...}` placeholder resolves at runtime from environment variables only.
- Single `@KafkaListener` confirmed; therefore COMP-39 is **NOT** the consumer of COMP-24's `${kafka.topic}` ActionEvent topic.

#### COMP-40 VisaResponseQuestionnaire

- Source-verified 2026-04-29. v1.0 → v1.1.
- Pre-ACK (DEC-005 deviation). `acknowledge()` at `KafkaConsumer.java:L30` precedes `processContestEvent()` at `:L32`.
- AckMode `MANUAL_IMMEDIATE`. Concurrency=1. Container factory `notificationListener`. Stateless. Single `@KafkaListener` in codebase. Consumer group `internal-integration-events-ques-group`.
- No DLQ, no local error table.
- Filters by non-null `visaResponseIds` only — silently discards AcceptService events.

#### COMP-41 ThirdPartyNotificationConsumer

- Writes `channel_type=GP_EVENTS` (NOT `GF_EVENTS`).
- Three distinct PUBLISHED-orphan paths invisible to COMP-12 Scheduler3.
- `@Cacheable("displaycodedetails")` and `@Cacheable("notificationRule")` are silent no-ops — `@EnableCaching` absent.
- Zero `@Transactional` annotations.
- No K8s liveness, readiness, or startup probes.
- Spring Retry imports dead — never applied.
- `auto.offset.reset = latest`.
- Predecessor lookup loads ALL outbox rows for the case, then filters in Java memory.
- Two `RestTemplate` construction patterns.
- **Zero references to JustAI anywhere.**

#### COMP-42 BENConsumer

- Source-verified 2026-04-29. v1.0 → v2.0.
- **Fourth writer of `wdp.outgoing_event_outbox`** (`channel_type=BEN_EVENTS`, `created_by=WBENC`). Previous handover fact list and WDP-DB.md v2.1 omitted COMP-42 from the writers list — gap closed.
- Zero `@Transactional` in `src/main` — all outbox writes are bare auto-commit JPA `save()` calls.
- All three K8s probes ABSENT — no liveness, no readiness, no startup.
- BEN product cache is `@Cacheable` lazy with `@Scheduled` `@CacheEvict` on `${app.cache-delay-hours}` fixed delay — NOT pre-warmed at startup.
- IDP token is dual-mode — lazy ThreadLocal for per-event CMS/CAS calls, eager (`fetchNewToken=true`) for the Display Code Service startup call only.
- Predecessor lookup loads ALL channel_types for the caseNumber from DB then filters to BEN_EVENTS in-memory — unbounded query per caseNumber.
- Outbound BEN producer key is `merchantId` (DEC-003 compliant).
- DEC-019 / DEC-004 compliant — `cardNumberLast4` is the only card field forwarded; full PAN never persisted, never published.
- `spring-retry` is transitive via `spring-kafka` — not declared in pom.xml.
- Both `spring-boot-starter-oauth2-resource-server` and `spring-boot-starter-oauth2-client` are dead — zero usage in `src/main`.

#### COMP-43 CoreNotificationConsumer

- DB2 PAN column name is `I_ACCT_CDH`. PAN-last-4 column: `I_ACCT_CDH_LST`.
- DB2 reads only `BC.TBC_DM_CASE` for enrichment. Enrichment is fully REST-driven.
- Step 6 4xx classification: only HTTP 400 OR 404 → ERROR; all other status codes → FAILED.
- STOP_DUP path: silent skip + ACK + no outbox write.
- `eventId` field type: `Long` (boxed, nullable).
- No K8s liveness / readiness / startup probes. Pod readiness governed only by `minReadySeconds: 30`.
- No `management:` block in any YAML profile — Spring Boot defaults expose only `health` and `info`.
- `created_by` / `updated_by` hardcoded to `"PCSECRTC"`.
- `next_retry_at` set on every non-SUCCESS update.
- `retry_count` increments only on incoming `status==FAILED`; ERROR escalation on third FAILED write.
- `original_event` JSON column carries fields not used by COMP-43.
- No Kafka-metadata write-back columns — incident correlation requires log-side join on `idempotencyId`.
- `actionSequence == "01"` comparison is string equality. No leading-zero normalisation.
- DataSource bean qualifiers `coredataSource` / `wdpdataSource` (lowercase, single-word) plus `@Primary` on WDP datasource.
- Idempotency check SELECT and INSERT run in separate JPA short transactions.
- UPDATE path runs `coreCaseRepository.save` outside any explicit `@Transactional`.

#### COMP-51 CaseExpiryProcessor *(NEW component, source-verified 2026-04-25)*

- Standalone Spring Batch Deployment in `gcp-case-expiry-processor-batch` repository. **Reader half of the case-expiry subsystem; COMP-17 is the writer half.**
- Trigger is a Spring `@Scheduled` cron — NOT a Kubernetes CronJob. Cron expression and page size are env-templated.
- Chunk size = 1; reader paginates with `PageRequest.of(0, pageSize, Sort.by("z_insrt"))` plus monotonic-id cursor.
- Calls 7 distinct upstream services: IDP token, Case Action API, Case Management API, Expiry Rules API, Update Action API, Accept API, Add Action API.
- Retry mechanism is hand-rolled — increment `i_retry_count` on `wdp.case_expiry` if < 3, write outbox + delete row on 3rd consecutive failure. `spring-retry` is on classpath but unused.
- Writes outbox rows with `channel_type=EXPIRY_BATCH` (distinct from COMP-17's `EXPIRY_EVENTS`), `created_by=WCSEEXPB` (distinct from COMP-17's `WCSEEXPC`), and `status=ERROR` direct (no PUBLISHED→FAILED transition).
- No `@SchedulerLock`, no ShedLock, no advisory lock. DEC-023 operational-only — replicas must be 1.
- No liveness/readiness/startup probes wired.
- No test coverage. `src/test/` is empty.
- Generates a fresh random `v-correlation-id` UUID per outbound REST call — same anti-pattern as COMP-17.
- IDP token call does NOT carry `v-correlation-id` — same gap as COMP-17.
- Uses `DriverManagerDataSource` (not pooled) — acceptable for low-frequency batch.
- Outbox INSERT and `wdp.case_expiry` DELETE share the Spring Batch chunk transaction (atomic by chunk boundary).
- COMP-17 and COMP-51 share `wdp.case_expiry` — **no lock or version column between them.**

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

### Resolved 2026-04-28 / 29 / 30 (removed from open list)

The following previously-open questions are now resolved by the 2026-04-28/29/30 source-verification round:

- COMP-04 token fetch timing → **resolved: lazy, request-scoped via `RequestTokenHolder`**
- COMP-04 `enrichmentFailure` service-set vs pass-through → **resolved: pass-through only**
- COMP-04 bypass scope across endpoints → **resolved: POST `/event` only**
- COMP-04 `RestServiceInvoker` dead-code claim → **resolved: NO, primary REST client**
- COMP-04 GUARPAY tier table → **resolved including GUARPAY8 → TIER8**
- COMP-04 decommission marker presence in source → **resolved: none, status is architectural**
- COMP-22 / COMP-37 cross-component ownership of `USCaseEntity` / `UKCaseEntity` → **resolved on COMP-22 side: zero writes from COMP-22; COMP-37 INSERT/DELETE owner question stands as a single-component question**
- COMP-26 POST idempotency gap → **confirmed: no application-level pre-existing-row check, no DB-level UNIQUE visible** (now narrows to architect remediation decision)
- COMP-30 routing decision-maker question → **resolved: COMP-30 is data provider only**
- Manual NAP error reprocessing surface owner → **resolved: COMP-39 hosts `POST /event`**
- COMP-39 as ActionEvent topic consumer hypothesis → **resolved: NO, single `@KafkaListener` confirmed; consumer of COMP-24's `${kafka.topic}` remains unidentified**
- COMP-42 outbox involvement → **resolved: COMP-42 IS a 4th writer (BEN_EVENTS / WBENC)**
- COMP-42 BEN delivery transport → **confirmed: Kafka-only via BEN-owned MSK cluster**
- COMP-42 partition key on outbound BEN producer → **resolved: `merchantId`, DEC-003 compliant**
- COMP-42 PAN handling → **resolved: DEC-019 / DEC-004 compliant**
- COMP-42 spring-retry direct-dependency → **resolved: transitive via spring-kafka**
- COMP-42 OAuth2 starters usage → **resolved: dead, zero usage**
- Downstream consumers of `wdp.case_expiry` → **partially resolved: COMP-51 confirmed as primary reader; cross-component sweep on remaining WDP repos still pending**
- COMP-01 port discrepancy (8082 vs 9082) → **resolved: 8082 confirmed; documentation error only, no real discrepancy**

(Plus the previously-resolved 2026-04-18/23/24/25 items from the v3.1 carryover.)

### Currently Open

| Question | Source | Action needed |
|----------|--------|---------------|
| COMP-24 ActionEvent topic name (`${kafka.topic}`) and consumer identity — **NOT COMP-39** | COMP-24 / COMP-39 | Confirm from deployment config |
| `wdp.case_expiry` other downstream consumers beyond COMP-51 | COMP-17 / COMP-51 OQ | Cross-repo sweep on remaining WDP components |
| COMP-36 TokenService — which external component writes Redis hash `wdpinternalidptoken:token`? | COMP-36 OQ | Team confirmation |
| Scheduler3/4 query FAILED/PENDING_DEFERRED only — does any process re-drive PUBLISHED orphans? | COMP-12 risk | Architect decision |
| Scheduler3 channel_type filter behaviour — does it filter consistently across EXPIRY_EVENTS, GP_EVENTS, BEN_EVENTS, CORE_EVENTS, EXPIRY_BATCH? Does it ever read PUBLISHED? | COMP-41 / COMP-42 OQ | Architect / Claude Code follow-up on COMP-12 source |
| `nap.queues` / `nap.queue_criterion` other-writer ownership beyond COMP-30 | COMP-27 / COMP-30 / COMP-31 OQ | Cross-repo sweep |
| External VAP/LATAM entity-scope authorization absent in COMP-27 — intentional or gap? | COMP-27 OQ | Architect decision |
| COMP-27 `/lft` `isInternal` derives from request body, not JWT | COMP-27 security finding | Architect review |
| COMP-23 PAN-clear remediation timeline (DEC-019) | COMP-23 risk | Team confirmation |
| COMP-43 PAN-clear in DB2 — intentional approved exception or remediate? (OQ-COMP-43-6) | COMP-43 risk | Architect decision |
| COMP-19 split-brain on MC CHI / AMEX / DISCOVER on NAP — fail-close, accept-and-document, or remediate? (OQ-COMP-19-9) | COMP-19 finding | Architect decision |
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
| COMP-22 reason for Kafka publish disablement on 2025-08-08 (PR #93 review) | COMP-22 OQ | Team / commit-author confirmation |
| COMP-22 SFG SFTP filename collision — namespacing required? | COMP-22 OQ | Architect decision |
| COMP-22 unconfigured HikariCP and Tomcat pools | COMP-22 OQ | Architect decision |
| COMP-29 FaxQueueService eViewer License proxy — runbook? | COMP-29 gap | Team confirmation |
| COMP-03 `validateOrgId()` method body absent | COMP-03 security | Raise formal ADR |
| COMP-22 INSERT/DELETE owner of `wdp.CASE` — COMP-37 column-level UPDATE has no INSERT/DELETE owner identified | COMP-37 OQ | Cross-component review |
| Production cron values (COMP-12 five schedulers, COMP-07/08 batch crons, COMP-51) | All schedulers | K8s secret confirmation |
| Archive column `c_ntwrk_phase_id` — exists in DDL? | COMP-12 OQ | DBA / schema-owner confirmation |
| OQ-FileJob-COMPLETED-Owner (cross-component contract) | COMP-11/12/13 | Architect decision |
| **NEW (COMP-04)**: Production values of `${kafka_retry_count}`, `${kafka_retry_delay}`, plus 5 other retry/delay env vars (CaseManagement, CaseAction, Rule retries) | COMP-04 OQ | Env config / team confirmation |
| **NEW (COMP-04)**: Production value of `${idp_token_service_url}` and `${kafka_nap_topic}` | COMP-04 OQ | Env config |
| **NEW (COMP-04)**: GUARPAY7 → TIER5 mapping intent — architect confirmation | COMP-04 OQ | Architect decision |
| **NEW (COMP-04)**: `@EnableCaching` import without annotation application — intentional placeholder or accidental? | COMP-04 OQ | Architect decision |
| **NEW (COMP-04)**: Decommission timeline and migration target | COMP-04 OQ | Team confirmation |
| **NEW (COMP-05)**: CVV-at-rest in `NAP.DISPUTE_EVENT_CONSUMER_ERROR` — remediate or document approved exception? | COMP-05 risk | Architect decision (DEC-019-class) |
| **NEW (COMP-05/COMP-39)**: Shared-error-table consumption — both components scan `NAP.DISPUTE_EVENT_CONSUMER_ERROR` without `C_EVENT_TYPE` filter; cross-component reprocessing hazard. Defect on both, or intentional? | COMP-05 / COMP-39 risk | Architect ADR — uniform decision required |
| **NEW (COMP-26)**: Cross-repo writer of `wdp.disputes_questionnaire_doc_status_rules` | COMP-26 OQ | Cross-repo grep |
| **NEW (COMP-26)**: COMP-20 ContestService relationship to QuestionnaireService | COMP-26 OQ | Cross-component review |
| **NEW (COMP-30)**: Component that transitions `wdp.lft_report.status` beyond PENDING | COMP-30 OQ | Cross-repo sweep |
| **NEW (COMP-30)**: Owners of `nap.nap_parent_entity`, `wdp.case`, `wdp.action` from COMP-30 reader perspective | COMP-30 OQ | Already known (COMP-02 / COMP-23 / COMP-24); confirm reader attribution |
| **NEW (COMP-30)**: `logback-spring.xml` runtime source — file absent from repo; Logstash integration non-functional unless mounted at runtime | COMP-30 OQ | Deployment team confirmation |
| **NEW (COMP-32)**: `/actionrules` `AuthorizationList` claim absence returns HTTP 500 NPE — defect or accepted? | COMP-32 OQ | Architect decision |
| **NEW (COMP-32)**: `/actionrules` empty-string `AuthorizationList` claim splits to `[""]`, intersection empty, silent — defect or accepted? | COMP-32 OQ | Architect decision |
| **NEW (COMP-32)**: Three endpoints (`/notification`, `/business-event`, `/cbk-response`) return three different HTTP 200 shapes on no-rule-found while the other 11 throw 404 | COMP-32 OQ | Architect decision |
| **NEW (COMP-32)**: `@Cacheable` key computed before in-method blank→"NA" normalisation — duplicate cache entries | COMP-32 OQ | Architect decision |
| **NEW (COMP-32)**: `/documentType` and `/eventRule` controller mappings commented out with full backing code intact — abandon or re-enable? | COMP-32 OQ | Architect decision |
| **NEW (COMP-35)**: DB-level UNIQUE on `wdp.hash_key_store.hpan` | COMP-35 OQ | DBA confirmation |
| **NEW (COMP-35)**: Production value of `${dek_rotation_interval_days}` | COMP-35 OQ | Ops |
| **NEW (COMP-35)**: Prometheus scrape configuration — does the scrape job carry a JWT? | COMP-35 OQ | Ops |
| **NEW (COMP-35)**: Log-pipeline PAN-scrubbing posture (Logstash target side) for the DEBUG last-4 PAN log | COMP-35 OQ | Ops |
| **NEW (COMP-39)**: NAP-DPS authentication mechanism — `napcacrt.jks` is unreferenced orphan; auth must be at Ingress / service mesh / network layer | COMP-39 OQ | Infrastructure team confirmation |
| **NEW (COMP-39)**: `NAP.NAP_UPDATE_RESPONSE_RULES` table owner | COMP-39 OQ | Team confirmation |
| **NEW (COMP-39)**: COMP-39 probe path mismatch — `resources.yaml` probes hit `/merchant/gcp/update-consumer/nap/livez` but actuator `additional-path: server:/livez` exposes endpoint at server root. Probes resolving correctly? | COMP-39 risk | Runtime observation |
| **NEW (COMP-39)**: COMP-39 `POST /event` controller-level exception handling — response codes and body schema | COMP-39 contract | Follow-up |
| **NEW (COMP-39)**: COMP-39 concurrency=1 — intentional or default-by-omission? | COMP-39 OQ | Architect decision |
| **NEW (COMP-42)**: Scheduler3 channel_type filter behaviour for BEN_EVENTS — same class as COMP-41 question | COMP-42 OQ | Claude Code follow-up on COMP-12 |
| **NEW (COMP-42)**: BEN platform idempotency — does BEN deduplicate at its end on retry-driven re-publish? | COMP-42 OQ | BEN integration team |
| **NEW (COMP-42)**: Production runtime values — `${ben_topic}`, `${ben_bootstrap_servers}`, etc. | COMP-42 OQ | Env config |
| **NEW (COMP-51)**: EXPIRY_BATCH outbox rows have no identified consumer — define consumer or accept as audit-only? | COMP-51 risk | Architect decision |
| **NEW (COMP-51)**: COMP-17 ↔ COMP-51 race conditions on `wdp.case_expiry` — accept as risk or remediate via row-version / SELECT FOR UPDATE? | COMP-51 risk | Architect decision |
| **NEW (COMP-51)**: COMP-51 cron value, page size, table prefix, replica count | COMP-51 OQ | XLD config inspection |
| **NEW (COMP-51 / COMP-17)**: `v-correlation-id` regenerated-per-call anti-pattern — platform-wide remediation candidate | COMP-51 / COMP-17 OQ | Architect decision |
| **NEW (COMP-51)**: Discover hybrid-merchant RE2/REPR special case — permanent regulatory rule or migration-era workaround? | COMP-51 OQ | Architect / regulatory confirmation |
| **NEW (COMP-51)**: HTTP 504 retry on POST endpoints (Accept, Add Action, Rules) — safe across all 7 upstream services? | COMP-51 risk | Architect / team confirmation |
| **NEW (COMP-51)**: Hardcoded user IDs `WCSEEXPB` (COMP-51) and `WCSEEXPC` (COMP-17) — confirm distinction is intentional | COMP-51 OQ | Confirm with platform user-ID registry |
| **NEW (Platform)**: `minReadySeconds` misplacement — eight components confirmed (COMP-03, 08, 09, 12, 25, 26, 28, 34, 40). Platform-wide DevOps remediation pass? | Platform OQ | DevOps |

---

*End of WDP-HANDOVER.md v3.2.*
*Reconciled 2026-04-30 from 18 Pending Entries in WDP-CHANGE-LOG.md.*
