# WDP-HANDOVER.md
**Worldpay Dispute Platform — Architecture Session Handover**
*Version: 2.0 | April 2026*
*Update this file whenever the knowledge base structure or current
work position changes significantly.*

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
| WDP-DECISIONS.md | ⚠️ Needs rebuild | Current version has outdated decisions. Rebuild after component files complete. |
| WDP-INTEGRATIONS.md | ⚠️ Needs rebuild | Rebuild after component files complete. |
| WDP-NFRS.md | ⚠️ Needs rebuild | Contains outdated NFRs. Never apply NFRs without confirming with Ram first. |
| WDP-HANDOVER.md | ✅ Current — v2.0 | This file. Updated April 2026. |
| WDP-ARCHITECTURE-v1-ARCHIVED.md | 📦 Archived | Old architecture. Useful reference for BRE steps, ACK pattern, Kafka config. |

### Tier 2 — Reference Indexes

| Document | Status | Notes |
|----------|--------|-------|
| WDP-COMP-INDEX.md | ✅ Created — v1.0 | 50 components registered. Upload to project folder. |
| WDP-KAFKA.md | ✅ Created — v1.0 Skeleton | Pre-populated from COMPONENTS.md Part 3. Enrichment ongoing. |
| WDP-DB.md | ✅ Created — v1.0 Skeleton | Pre-populated from COMPLETE component sections. Enrichment ongoing. |
| WDP-FLOW-INDEX.md | ✅ Created — v1.0 Skeleton | 11 core flows identified. None documented yet. |
| WDP-COMP-TEMPLATE.md | ✅ Created — v1.1 | Master template with 4 type blocks. Upload to project folder. |

### Tier 3 — Individual Component Files

| Status | Count | Details |
|--------|-------|---------|
| ✅ COMPLETE (individual file created and confirmed) | 1 | COMP-13 FileAcknowledgementProcessor |
| 📝 DRAFT (individual file created, architect confirmation pending) | 16 | COMP-01 API Gateway, COMP-02 UAMS, COMP-03 CHAS, COMP-04 NAPDisputeEventService ⚠️ decommission-scoped, COMP-05 NAPDisputeEventProcessor ⚠️ decommission-scoped, COMP-06 NAPDisputeDeclineBatch ⚠️ decommission-scoped, COMP-07 VisaDisputeBatch, COMP-08 FirstChargebackBatch, COMP-09 CaseFillingBatch, COMP-11 FileProcessor, COMP-12 InboundDisputeEventScheduler, COMP-14 CaseCreationConsumer, COMP-15 EvidenceConsumer, COMP-16 BusinessRulesProcessor, COMP-17 CaseExpiryUpdateConsumer, COMP-18 NotificationOrchestrator |
| 📋 PENDING | 20 | COMP-19 through COMP-50 (excluding above) |
| ⬜ NOT STARTED | 2 | COMP-44 (EDIAConsumer), COMP-48 (NYCEFileGenerationProcessor) |

Phase 1 migration complete — all 13 COMPLETE components from WDP-COMPONENTS.md now have individual files.
Phase 2 begins with COMP-14 CaseCreationConsumer.

### Tier 4 — Workflow Documents

No workflow files exist yet. 11 flows identified in WDP-FLOW-INDEX.md.
Workflow documentation to be done in parallel with component migration.

---

## Current Work Position

**Phase:** Knowledge base restructure — foundation complete,
component migration starting.

**Completed this session:**
- Designed the four-tier knowledge architecture
- Defined the component template (WDP-COMP-TEMPLATE.md v1.1)
  with shared skeleton + four optional type blocks
- Created WDP-COMP-INDEX.md with all 50 components registered,
  numbered, and described
- Created WDP-KAFKA.md skeleton with topic registry and MSK config
- Created WDP-DB.md skeleton with schema ownership map
- Created WDP-FLOW-INDEX.md with 11 core flows identified
- Rewrote WDP-HANDOVER.md to v2.0

**Next work: Component Migration — Phase 1 (COMPLETE components)**

Migrate the 13 COMPLETE components from WDP-COMPONENTS.md to
individual files using WDP-COMP-TEMPLATE.md. These already have
confirmed content — migration is mostly moving and enriching with
REST contracts, Kafka contracts, and flow diagrams via Copilot CLI.

Migration order (priority):
```
COMP-01  WDP-COMP-01-API-GATEWAY.md
COMP-02  WDP-COMP-02-UAMS.md
COMP-03  WDP-COMP-03-CHAS.md
COMP-04  WDP-COMP-04-NAP-DISPUTE-EVENT-SERVICE.md
COMP-05  WDP-COMP-05-NAP-DISPUTE-EVENT-PROCESSOR.md
COMP-06  WDP-COMP-06-NAP-DISPUTE-DECLINE-BATCH.md
COMP-07  WDP-COMP-07-VISA-DISPUTE-BATCH.md
COMP-08  WDP-COMP-08-FIRST-CHARGEBACK-BATCH.md
COMP-09  WDP-COMP-09-CASE-FILLING-BATCH.md
COMP-11  WDP-COMP-11-FILE-PROCESSOR.md
COMP-12  WDP-COMP-12-INBOUND-EVENT-SCHEDULER.md
COMP-13  WDP-COMP-13-FILE-ACK-PROCESSOR.md
```

*(COMP-10 DM Mainframe is enterprise-owned infrastructure — lower priority)*

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

**Option A — Starting a component migration session:**
```
"You are my senior architecture partner for WDP.
Read WDP-HANDOVER.md first, then WDP-COMP-INDEX.md.
Today I want to migrate [COMP-NN ComponentName] from
WDP-COMPONENTS.md to its individual component file.
I will use Copilot CLI to extract the missing sections."
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

**For all sessions:** Claude always searches project knowledge
before answering. The tier order above is the search priority.

---

## Copilot CLI Workflow for Component Migration

For each component migration, Ram will:

1. Open the component repository in IntelliJ with Copilot CLI
2. Identify the component type (REST / Kafka Consumer / Producer / Batch)
3. Ask type-appropriate questions from WDP-COMP-TEMPLATE.md
4. Paste Copilot responses into the chat
5. Claude extracts architecture signal, discards implementation detail,
   and completes the component file sections
6. For each completed component, Claude also provides:
   - Rows to add to WDP-KAFKA.md (if Kafka involved)
   - Rows to add to WDP-DB.md (table ownership)
   - Flags for any deviations from platform standards
7. Ram confirms or corrects
8. Component file marked ✅ COMPLETE

---

## Confirmed Architectural Facts

- Case-level authorization is FAIL-CLOSED for NAP and PIN.
  Any exception including timeouts returns 403. The fail-open
  branch in WDP-COMPONENTS.md is incorrect — Copilot CLI
  confirmed this from source. Update WDP-COMPONENTS.md when
  API Gateway section is retired.
- No CORE platform value exists in gateway source code. How
  CORE, VAP, and LATAM requests receive case-level authorization
  is an open question — to be confirmed during platform
  integration analysis.
- removeItemFromQueueDisabled flag is a second active operational
  safety switch present in both COMP-07 VisaDisputeBatch and
  COMP-08 FirstChargebackBatch. When true, suppresses ALL MarkAsRead /
  ACK PUT calls globally — items are processed and written to the
  outbox but remain in the MCM/Visa queue. Confirmed active production
  configuration. Operations teams must be aware both flags exist.

Do not contradict these without explicit confirmation from Ram.
- wdp.file_notifications does not exist. The actual table written by COMP-18 NotificationOrchestrator is wdp.file_generation_event. All prior references to file_notifications have been corrected in WDP-ARCHITECTURE.md, WDP-COMP-INDEX.md, and WDP-DB.md.
- wdp.bre_orchestration_outbox is a shared table with a component discriminator column. COMP-18 writes component=NOTIFICATION_ORCHESTRATOR rows. COMP-12 Scheduler4 writes component=BUSINESS_RULES rows and reads FAILED/PENDING_DEFERRED rows of both types for retry.
- case-action-events publisher confirmed as COMP-18 NotificationOrchestrator (Filter 1 EXPIRY_EVENT routing). Resolved open question from COMP-17.

**Enterprise shared services (not WDP owned):**
- Akamai — CDN and edge security
- APIGEE — B2B API gateway for external merchants
- IDP — enterprise OAuth 2.0 identity provider (SunGard for NAP)
- Sterling Mailbox — universal file aggregation hub
- ControlM — on-premise file transfer agent (bridges Sterling and S3)
- EDIA platform — enterprise Kafka streaming (not owned by WDP)

**WDP owned on-premise:**
- DM Mainframe — mainframe-to-mainframe file transfers
- File Transfer Batch — DiscoverHybrid special pull via SFTP

**Key confirmed platform decisions:**
- No circuit breakers in WDP currently — Resilience4j referenced but
  not active across all components. COMP-05 and COMP-04 confirmed
  no circuit breaker.
- No Kafka DLQ topics — error handling via database error tables
  per consumer (COMP-05 uses NAP.DISPUTE_EVENT_CONSUMER_ERROR)
- BusinessRulesProcessor (COMP-16) makes direct DB calls — does NOT
  call BusinessRulesService (COMP-31)
- NotificationOrchestrator (COMP-18) does NOT publish to
  internal-integration-events — that topic is published by AcceptService
  (COMP-19) and ContestService (COMP-20) only
- TokenService (COMP-36) is JWT management only — nothing to do with
  PAN tokenisation
- NAP dispute data has no full PAN — no EncryptionService call needed
  for NAP inbound path
- Ops Portal connects directly to API Gateway — no Akamai
- Merchant Portal routes through Akamai
- ChargebackService (COMP-21) is the only WDP service exposed
  externally via APIGEE
- WPG 119-response documents go directly to DocumentManagementService
  via NAPDisputeEventService — bypass Kafka entirely
- All components run on the same AWS EKS cluster
- InboundDisputeEventScheduler (COMP-12) polls every 2 minutes

**Planned work (not yet built):**
- COMP-44 EDIA Consumer (strategic outbound route)
- COMP-48 NYCE File Generation Processor
- Dashboard Section in both portals
- NAP inbound migration to common chbk_outbox_row path
- NAP outbound migration from direct NAP-DPS API to EDIA route
- LATAM full integration
- VAP full integration

**Open questions requiring architect confirmation:**
- business-rules topic publishers — COMP-15 EvidenceConsumer (WDP path, isMultiDocPending=false) and COMP-12 InboundDisputeEventScheduler (Scheduler4, bre_orchestration_outbox rows) both confirmed. COMP-14 CaseCreationConsumer candidacy still unverified — Copilot follow-up required on COMP-14 repo.
- Publisher of wdp.outgoing_event_outbox — COMP-17 confirmed for EXPIRY_EVENTS channel; other channel writers still TBC
- wdp.bre_orchestration_outbox publisher — RESOLVED: COMP-18 NotificationOrchestrator writes component=NOTIFICATION_ORCHESTRATOR rows; COMP-12 Scheduler4 writes component=BUSINESS_RULES rows and reads both for retry
- PROCESSING-only query in Schedulers 3 and 4 (COMP-12) — intentional
  or processing gap?
- Scheduler 3 topic routing via channelTypeTopicMap — confirm topics
- Discover vs DiscoverHybrid inbound/outbound differences
- Amex vs AmexHybrid differences
- ACK file rules per source (which sources require ACK, which do not)
- Queue and skill-based routing logic owner — suspected COMP-30
  UserQueueSkillService (flagged in COMP-02 UAMS analysis)
- business-rules topic publishers — COMP-15 EvidenceConsumer (WDP path,  
  isMultiDocPending=false) and COMP-12 InboundDisputeEventScheduler (Scheduler4, 
  bre_orchestration_outbox rows) both confirmed. COMP-14 CaseCreationConsumer candidacy still 
  unverified — Copilot follow-up required on COMP-14 repo.
- DEC-011 BRE step checkpointing — confirmed NOT IMPLEMENTED in COMP-16. The named steps (VALIDATE, ENRICH, ATTACH_ISSUER_DOC) and checkpoint mechanism described in WDP-ARCHITECTURE-v1-ARCHIVED.md and WDP-DECISIONS.md are aspirational design, not current code. Must be corrected when WDP-DECISIONS.md is rebuilt. Do not reference DEC-011 as active.
- saveChildWithMerchant in UAMS (COMP-02) uses @Primary
  wdpTransactionManager instead of napTransactionManager.
  All three tables it writes (nap_child_entity, nap_merchant,
  nap_entity_rel) are in the NAP schema. Rollback on failure
  is not guaranteed. Confirm whether this has caused data
  integrity issues in production and whether a fix is planned.
  - validateOrgId() is commented out on GET /orgentity in CHAS
  (COMP-03). Any authenticated PIN-platform caller can query
  any org hierarchy without scope validation. Confirmed security
  gap — flag to team and consider formal ADR when
  WDP-DECISIONS.md is rebuilt.

---

## After Component Migration is Complete

Rebuild in this order once all 50 component files are confirmed:

1. **WDP-DECISIONS.md** — rebuild from component-level decision entries.
   ~34 decisions identified in COMPLETE components. Many DEC-PLACEHOLDER
   entries in DRAFT components will surface more.

2. **WDP-INTEGRATIONS.md** — rebuild external system contracts from
   confirmed component boundary sections.

3. **WDP-NFRS.md** — rebuild from confirmed risk registers across all
   component files. Platform-wide patterns will be visible once all 50
   are complete.

4. **WDP-FLOW-[NAME].md files** — document the 11 core flows identified
   in WDP-FLOW-INDEX.md. Can start in parallel with migration.

5. **Archive WDP-COMPONENTS.md** — once all content is migrated and
   confirmed in individual files, archive alongside
   WDP-ARCHITECTURE-v1-ARCHIVED.md.
