# WDP-HANDOVER.md
**Worldpay Dispute Platform — Architecture Session Handover**
*Version: 3.0 | April 2026*
*Component migration complete — all 40 available component files uploaded.*

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
- When architecture evolves, tell Ram which docs need updating
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
| WDP-ARCHITECTURE.md | ✅ Current — v2.0 | Fully rebuilt April 2026. 15 Mermaid diagrams confirmed. |
| WDP-DECISIONS.md | ✅ Current — v2.0 | Rebuilt April 2026. DEC-011 and DEC-014 voided. DEC-019/020 recorded. Deviation maps added. |
| WDP-INTEGRATIONS.md | ✅ Current — v2.0 | Rebuilt April 2026. BEN Kafka correction applied. JustAI planned status confirmed. |
| WDP-NFRS.md | ✅ Current — v2.0 | Rebuilt April 2026. DEC-014/011 void references removed. Risk Register added (23 risks). |
| WDP-HANDOVER.md | ✅ Current — v3.0 | This file. Updated April 2026. |
| WDP-ARCHITECTURE-v1-ARCHIVED.md | 🗑️ Removed | Removed April 2026. Content superseded by individual component files and current Tier 1 documents. BRE checkpointing section was aspirational design only (DEC-011 VOID). |

### Tier 2 — Reference Indexes

| Document | Status | Notes |
|----------|--------|-------|
| WDP-COMP-INDEX.md | ✅ Current — v2.0 | All 50 components registered and statuses updated April 2026. |
| WDP-KAFKA.md | ✅ Current — v2.0 | Fully populated from all component files April 2026. |
| WDP-DB.md | ✅ Current — v2.0 | Fully populated from all component files April 2026. |
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

**Component migration complete.** All 40 available component DRAFT files have been uploaded and verified.
**Next phase:** WDP-DECISIONS.md ✅, WDP-INTEGRATIONS.md ✅, WDP-NFRS.md ✅ all complete. Next: WDP-FLOW documents or Archive WDP-COMPONENTS.md.

### Tier 4 — Workflow Documents

No workflow files exist yet. 11 flows identified in WDP-FLOW-INDEX.md.
Workflow documentation to be done as a parallel work stream.

---

## Current Work Position

**Phase:** Component migration COMPLETE. Ready for knowledge base rebuild phase.

**Completed:**
- All 40 available component files uploaded and verified (April 2026)
- WDP-COMP-INDEX.md v2.0 — all statuses and descriptions updated
- WDP-KAFKA.md v2.0 — fully populated with all confirmed topic, consumer group, and deviation data
- WDP-DB.md v2.0 — fully populated with all confirmed table ownership across wdp, nap, and external schemas

**Next work — Rebuild phase (in priority order):**

1. ~~**WDP-DECISIONS.md**~~ ✅ COMPLETE — rebuilt April 2026. DEC-011 and DEC-014
   formally voided. DEC-019 (clear PAN) and DEC-020 (no idempotency) recorded.
   Deviation maps added to DEC-001, DEC-003, DEC-005. WDP-DECISIONS-ARCHIVE.md
   removed from project.

2. ~~**WDP-INTEGRATIONS.md**~~ ✅ COMPLETE — rebuilt April 2026. BEN delivery
   corrected to Kafka (BEN-owned MSK cluster, not webhook). JustAI confirmed
   planned only — not in codebase. WDP-INTEGRATIONS-ARCHIVE.md removed.

3. ~~**WDP-NFRS.md**~~ ✅ COMPLETE — rebuilt April 2026. Circuit breaker and
   BRE checkpointing references removed (both void). Kafka delivery corrected
   to at-most-once. Section 6 Platform Risk Register added — 23 confirmed
   risks, all cross-referenced to WDP-DECISIONS.md v2.0.

4. **WDP-FLOW-[NAME].md files** — document the 11 core flows identified
   in WDP-FLOW-INDEX.md. Can start in parallel with decisions rebuild.

5. **Archive WDP-COMPONENTS.md** — once DECISIONS/INTEGRATIONS/NFRS
   are rebuilt, archive alongside WDP-ARCHITECTURE-v1-ARCHIVED.md.

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
4. Update WDP-KAFKA.md and WDP-DB.md with entries from the new component

---

## How to Start a New Session

**Option A — Starting a decisions rebuild session:**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first, then WDP-COMP-INDEX.md.
Today I want to rebuild WDP-DECISIONS.md from the
confirmed component files. Start with the decisions
that have the most cross-component impact."
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

**For all sessions:** Claude always searches project knowledge
before answering. The tier order above is the search priority.

---

## Confirmed Architectural Facts

Do not contradict these without explicit confirmation from Ram.

**Access and authorization:**
- Case-level authorization is FAIL-CLOSED for NAP and PIN. Any exception
  including timeouts returns 403. The fail-open branch in WDP-COMPONENTS.md
  is incorrect — confirmed from COMP-01 source.
- No CORE platform value exists in API Gateway source code. How CORE, VAP,
  and LATAM requests receive case-level authorization is an open question.
- validateOrgId() is commented out on GET /orgentity in CHAS (COMP-03). Any
  authenticated PIN-platform caller can query any org hierarchy without scope
  validation. Confirmed security gap — flag for formal ADR when
  WDP-DECISIONS.md is rebuilt.
- COMP-24 CaseActionService has no RBAC enforcement — RestInvoker.authorizeUser()
  exists but is never called from any controller. Any authenticated WDP service
  can modify any case action. Security gap flagged for ADR.

**Kafka — platform-wide patterns:**
- Every confirmed Kafka consumer in WDP uses pre-ACK or mid-flow ACK. This is
  a platform-wide pattern, not an isolated exception. DEC-005 (commit after
  full processing) is aspirational — it is not the current implementation.
- No Kafka DLQ topics exist in WDP. Error handling is via database error tables
  per consumer (e.g. NAP.DISPUTE_EVENT_CONSUMER_ERROR for COMP-05).
- No circuit breakers (Resilience4j) anywhere in WDP. Confirmed absent across
  all 40 component files.
- business-rules topic has five confirmed publishers: COMP-12 Scheduler4,
  COMP-15 EvidenceConsumer, COMP-23 CaseManagementService, COMP-24
  CaseActionService, COMP-25 NotesService. All use caseNumber as partition
  key — DEC-003 deviation platform-wide. COMP-14 CaseCreationConsumer
  candidacy still unverified.
- wdp.bre_orchestration_outbox is shared between COMP-18
  (component=NOTIFICATION_ORCHESTRATOR rows) and COMP-12 Scheduler4
  (component=BUSINESS_RULES rows). The component discriminator column
  determines routing. PUBLISHED-status orphan rows have no automatic
  re-drive — manual intervention required.
- case-action-events publisher confirmed as COMP-18 NotificationOrchestrator
  (Filter 1 EXPIRY_EVENT routing).
- wdp.file_notifications does not exist. The actual table is
  wdp.file_generation_event. Corrected in all documents.
- COMP-43 CoreNotificationConsumer is the sole WDP component that writes to
  IBM DB2 Core Platform (BC.TBC_DM_CASE, BC.TBC_DM_OCCUR, BC.TBC_DM_NOTES).
  All other DB2 access across the platform is read-only.

**Confirmed DEC violations and voids — all now formally recorded in WDP-DECISIONS.md v2.0:**
- DEC-011 ⛔ VOID — BRE step checkpointing confirmed never implemented in COMP-16.
  Formally voided in WDP-DECISIONS.md v2.0. Do not reference as active anywhere.
- DEC-014 ⛔ VOID — Resilience4j confirmed absent across all 40 component files.
  Formally voided in WDP-DECISIONS.md v2.0. Do not reference as active anywhere.
- DEC-004 violation in COMP-23 — clear PAN written to persistent storage on standard
  case creation. Formally recorded as DEC-019 (Accepted Risk) in WDP-DECISIONS.md v2.0.
- DEC-001 (transactional outbox) — violated by COMP-04, COMP-15, COMP-16,
  COMP-19, COMP-20, COMP-23, COMP-24, COMP-25. Deviation map recorded in
  WDP-DECISIONS.md v2.0.
- No idempotency on case creation (COMP-23) — formally recorded as DEC-020
  (Accepted Risk) in WDP-DECISIONS.md v2.0.

**Key confirmed platform facts:**
- BusinessRulesProcessor (COMP-16) makes direct DB calls to nap.rules /
  wdp.rules — does NOT call BusinessRulesService (COMP-31).
- NotificationOrchestrator (COMP-18) does NOT publish to
  internal-integration-events — that topic is published by AcceptService
  (COMP-19) and ContestService (COMP-20) only.
- COMP-22 DisputeService is read-only — it owns no database state, performs
  no writes. Kafka producer to business-rules is wired but commented out.
- COMP-28 DisplayCodeService does NOT determine TIER1 eligibility — it
  returns raw code lists. Eligibility logic belongs to calling services.
- COMP-38 APILogService has no AOP inside it. Callers hold catch-blocks and
  invoke this service directly via REST.
- TokenService (COMP-36) is JWT management only — no relation to PAN
  tokenisation. Redis hash wdpinternalidptoken:token must be written by an
  external component not yet identified.
- COMP-33 OrgManagementService — GitHub repository not found.
  Documentation deferred.
- removeItemFromQueueDisabled flag is a second active operational safety
  switch present in both COMP-07 VisaDisputeBatch and COMP-08
  FirstChargebackBatch. When true, suppresses ALL MarkAsRead/ACK PUT calls
  globally. Confirmed active production configuration.
- saveChildWithMerchant in UAMS (COMP-02) uses @Primary wdpTransactionManager
  instead of napTransactionManager. All three NAP tables it writes are in the
  NAP schema. Rollback on failure is not guaranteed. Confirmed bug — assess
  production impact.
- BEN notification is delivered via Kafka publish to a BEN-owned MSK cluster
  using separate SASL/JAAS credentials — NOT via REST or webhook. Confirmed
  from COMP-42 source and WDP-INTEGRATIONS.md v2.0. WDP-COMP-INDEX.md
  description corrected.
- JustAI integration is planned only — no JustAI reference exists anywhere in
  the COMP-41 codebase. Signifyd is the sole live third-party vendor.
  WDP-COMP-INDEX.md description corrected. WDP-INTEGRATIONS.md v2.0 records
  JustAI as planned under Section 5.2.

**Enterprise shared services (not WDP owned):**
- Akamai — CDN and edge security (Merchant Portal only)
- APIGEE — B2B API gateway for external merchants
- IDP — enterprise OAuth 2.0 identity provider (SunGard for NAP)
- Sterling Mailbox — universal file aggregation hub
- ControlM — on-premise file transfer agent (bridges Sterling and S3)
- EDIA platform — enterprise Kafka streaming (not owned by WDP)
- MS SQL Server (legacy fax system) — shared with legacy systems outside WDP

**WDP owned on-premise:**
- DM Mainframe — mainframe-to-mainframe file transfers
- File Transfer Batch — DiscoverHybrid special pull via SFTP

**Planned work (not yet built):**
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
- JustAI third-party notification integration (COMP-41 planned extension)

---

## Open Questions Requiring Architect Confirmation

| Question | Source | Action needed |
|----------|--------|---------------|
| COMP-14 CaseCreationConsumer — does it publish to business-rules topic after case creation? | COMP-14 open question | Copilot follow-up on COMP-14 repo: "Does CaseCreationConsumer publish to the business-rules Kafka topic after case creation?" |
| COMP-24 CaseActionService ActionEvent topic — what is the actual topic name (${kafka.topic}) and which component consumes it? | COMP-24 Kafka gap | Confirm from deployment config; likely COMP-39 NAPOutcomeProcessor |
| wdp.case_expiry downstream consumers — which components read this table? | COMP-17 open question | Copilot follow-up on COMP-17 repo: "Which components read from wdp.case_expiry?" |
| COMP-36 TokenService — which external component writes the Redis hash wdpinternalidptoken:token? | COMP-36 open question | Team confirmation — the TokenService repo does not contain the Redis writer |
| PROCESSING-only query in Schedulers 3 and 4 (COMP-12) — intentional or gap? | COMP-12 risk | Architect decision required — Scheduler3/4 query FAILED/PENDING_DEFERRED only, not PROCESSING or PENDING rows |
| Scheduler 3 topic routing via channelTypeTopicMap — confirm all topic mappings | COMP-12 gap | Confirm EXPIRY_EVENTS→case-action-events; confirm other channel types |
| Discover vs DiscoverHybrid inbound/outbound differences | COMP-11/file gen | Architect confirmation needed |
| Amex vs AmexHybrid differences | COMP-11/file gen | Architect confirmation needed |
| COMP-23 non-atomic cross-datasource write (wdp.dispute_event_change_log within NAP create path) — accepted risk or gap? | COMP-23 risk | Architect decision required — WDP-DECISIONS.md rebuilt but this item not yet formally recorded |
| COMP-23 no idempotency on case creation — accepted risk? | COMP-23 risk | Architect decision required — concurrent identical requests produce duplicate case records |
| COMP-26 POST idempotency gap — duplicate POSTs insert new rows | COMP-26 risk | Architect decision — DB unique constraint on (I_CASE, I_ACTION_SEQ) needed? |
| COMP-24 RBAC gap — RestInvoker.authorizeUser() never called | COMP-24 security | Recorded as DEC-018 (Accepted Risk) in WDP-DECISIONS.md v2.0. Remediation timeline TBC with team. |
| COMP-03 validateOrgId() commented out on GET /orgentity | COMP-03 security | Raise formal ADR — not yet recorded in WDP-DECISIONS.md v2.0. |
| nap.rule_group, nap.rule_criteria, nap.rule_action_field write ownership | COMP-31 gap | Confirm whether COMP-32 RulesService or DBA scripts own these |
| PIN platform enrichment path in COMP-14 — same as CORE (MerchantTransactionService) or distinct? | COMP-14 open question | Architect decision + Copilot follow-up |
| COMP-29 FaxQueueService eViewer License proxy — is it documented and is there a production runbook? | COMP-29 gap | Team confirmation |
| wdp.api_route table — which component owns writes? | COMP-01 gap | Confirm from team or infrastructure config |
| File job terminal status PROCESSING vs COMPLETED — COMP-11 never writes COMPLETED, COMP-13 polls for COMPLETED | COMP-11/13 gap | Architect decision — is ACK generation for successful file jobs broken? |

---

## After Knowledge Base is Complete

Rebuild in this order:

1. ~~**WDP-DECISIONS.md**~~ ✅ COMPLETE — rebuilt April 2026. 29 decisions
   registered. DEC-011 and DEC-014 voided. DEC-016 through DEC-023 added.

2. ~~**WDP-INTEGRATIONS.md**~~ ✅ COMPLETE — rebuilt April 2026. All external
   integration contracts confirmed from component files.

3. ~~**WDP-NFRS.md**~~ ✅ COMPLETE — rebuilt April 2026. 23 platform risks
   documented in new Section 6 Risk Register.

4. **WDP-FLOW-[NAME].md files** — document the 11 core flows identified
   in WDP-FLOW-INDEX.md. Can start in parallel.

5. **Archive WDP-COMPONENTS.md** — once DECISIONS/INTEGRATIONS/NFRS
   are rebuilt, archive alongside WDP-ARCHITECTURE-v1-ARCHIVED.md.

6. **COMP-33 OrgManagementService** — document when GitHub repo is found.

7. **COMP-45, 46, 47 File Generation** — document when sprint work is
   complete and repos are available.

8. **UI Portal documentation** — COMP-49 and COMP-50 as a separate
   action item.
