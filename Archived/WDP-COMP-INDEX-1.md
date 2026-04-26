# WDP-COMP-INDEX.md
**Worldpay Dispute Platform — Master Component Registry**
*Version: 1.0 | April 2026*
*Source: WDP-COMPONENTS.md v1.0 DRAFT + architect review*

---

## Purpose of This Document

This is the navigation hub for all WDP component documentation.
It maps every component to its file, records production and
documentation status at a glance, and provides a one-line
responsibility summary for each component.

Claude uses this file to navigate the component knowledge base.
You use it to track migration progress from WDP-COMPONENTS.md
to individual component files, and to assign numbers to new
components added each quarter.

This document does NOT contain component detail. All detail
lives in the individual WDP-COMP-[NN]-[NAME].md files.

---

## Numbering Rules

- Numbers are permanent. Once assigned, a number never changes
  and is never reused, even if the component is retired.
- Numbers are sequential in the order components were first
  documented. They do not imply architectural hierarchy.
- New components added each quarter get the next available number.
- The number is the stable identifier. The name segment of the
  filename is human-readable context and may be updated if a
  component is renamed.
- Format: two-digit zero-padded integer — 01, 02, ... 09, 10, 11...
  Extend to three digits (001...) only if the platform exceeds 99
  components.

---

## What Is and Is Not a Component

**Gets a component number and file:**
Any independently deployable unit — a microservice, a batch job,
a Kubernetes Deployment or CronJob, a UI application.

**Does NOT get a component number:**
- The Kafka Event Bus (Part 3 in WDP-COMPONENTS.md) — this is
  infrastructure configuration and a topic registry. Documented
  in WDP-KAFKA.md.
- Acquiring Platform Integration contexts (Part 6 in
  WDP-COMPONENTS.md) — these are integration patterns, not
  deployable components. Documented in WDP-INTEGRATIONS.md.
- UI sections within a portal (Disputes Section, Queues Section,
  etc.) — these are functional areas within a portal application,
  not separate deployables. Documented as sections within the
  relevant portal component file (WDP-COMP-49 or WDP-COMP-50).

**Special note on FileAcknowledgementProcessor:**
Sections 2.3.4 and 5.3.2 in WDP-COMPONENTS.md both describe
ACK file generation behaviour that maps to the same deployable:
`wdp-evidence-ack-scheduler`. These have been consolidated into
a single component (13). Section 5.3.2 in WDP-COMPONENTS.md
is a cross-reference, not a separate service.

---

## Status Key

**Production status:**
- ✅ Production — live in production
- 🔄 In Progress — being built or partially live
- 🔴 Planned — not yet started

**Documentation status:**
- ✅ COMPLETE — individual component file exists, architect-confirmed
- 📝 MIGRATING — content migrated from WDP-COMPONENTS.md to
  individual file, enrichment (REST contracts, Kafka contracts,
  flow diagrams) in progress
- 📋 PENDING — not yet migrated to individual file;
  content still only in WDP-COMPONENTS.md
- ⬜ NOT STARTED — no content exists yet

---

## Component Registry

### Access & API Layer

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 01 | API Gateway | REST API | ✅ Production | 📝 MIGRATING | WDP-COMP-01-API-GATEWAY.md |
| 02 | UserAccessManagementService | REST API | ✅ Production | 📝 MIGRATING | WDP-COMP-02-UAMS.md |
| 03 | CoreHierarchyAuthorizationService | REST API | ✅ Production | 📝 MIGRATING | WDP-COMP-03-CHAS.md |

**01 — API Gateway**
Single entry point for all WDP traffic. Runs a 4-layer JWT authentication,
request logging, region-role authorization, and case-level entity authorization
filter pipeline before proxying requests to backend microservices. URL-based
routing configuration loaded from database at startup.

**02 — UserAccessManagementService (UAMS)**
Access control and user lifecycle management for the NAP platform. Owns the
merchant entity hierarchy and ACL. Exposes the /authorize endpoint called by
the API Gateway for NAP platform case-level authorization. Proxies all user
lifecycle operations to SunGard IdP.

**03 — CoreHierarchyAuthorizationService (CHAS)**
Read-only authorization service for PIN, CORE, VAP, and LATAM platforms.
Validates JWT entity claims against Core enterprise merchant hierarchy in
IBM DB2. Exposes the /authorize endpoint called by the API Gateway for all
non-NAP platform case-level authorization. Owns no data and performs no writes.

---

### Inbound Processing — NAP/WPG Path

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 04 | NAPDisputeEventService ⚠️ Decommission-scoped | REST API + Kafka Producer | ✅ Production | 📝 MIGRATING | WDP-COMP-04-NAP-DISPUTE-EVENT-SERVICE.md |
| 05 | NAPDisputeEventProcessor ⚠️ Decommission-scoped | Kafka Consumer | ✅ Production | 📝 MIGRATING | WDP-COMP-05-NAP-DISPUTE-EVENT-PROCESSOR.md |
| 06 | NAPDisputeDeclineBatch ⚠️ Decommission-scoped | Batch/Scheduler | ✅ Production | 📝 MIGRATING | WDP-COMP-06-NAP-DISPUTE-DECLINE-BATCH.md |

**04 — NAPDisputeEventService**
REST-to-Kafka bridge for inbound NAP dispute events. Receives SRV-116 events
and operator responses via REST, enriches via synchronous lookups against
internal WDP services, and publishes enriched NapEvent to nap-dispute-events
topic. Document upload path proxies directly to DocumentManagementService
without Kafka involvement. Stateless — no database.

**05 — NAPDisputeEventProcessor**
Kafka consumer on nap-dispute-events topic. Translates NAP dispute notifications
into case management actions via REST calls to downstream services. Classifies
events as SRV116, WIN/LOSS, or SRV118 by field inspection. Uses at-most-once
offset commit (deliberate deviation from DEC-005). Database-backed DLQ — no
Kafka dead-letter topic. Currently in hybrid CB911 migration state.

**06 — NAPDisputeDeclineBatch**
Spring Batch job that polls nap.action table for open PAB actions, queries
Visa network via Visa Adapter HyperSearch, and creates IDCL draft actions for
pre-arbitration declined cases. VISA and NAP platform only. Runs as Kubernetes
Deployment with internal Spring @Scheduled cron. No Kafka involvement.

---

### Inbound Processing — Card Network Batch Path

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 07 | VisaDisputeBatch | Batch/Scheduler | ✅ Production | 📝 DRAFT | WDP-COMP-07-VISA-DISPUTE-BATCH.md |
| 08 | FirstChargebackBatch | Batch/Scheduler | ✅ Production | 📝 DRAFT | WDP-COMP-08-FIRST-CHARGEBACK-BATCH.md |
| 09 | CaseFillingBatch | Batch/Scheduler | ✅ Production | 📝 DRAFT | WDP-COMP-09-CASE-FILLING-BATCH.md |

**07 — VisaDisputeBatch**
Origin of all Visa dispute data in WDP. Polls seven Visa RTSI queue types
sequentially via DataPower Gateway, enriches each event via HyperSearch and
internal WDP services, encrypts PAN, and writes PENDING rows to the
chbk_outbox_row transactional outbox. Does not publish to Kafka directly.
Sequential single-threaded processing — all 7 queues in one job execution.

**08 — FirstChargebackBatch**
Origin of all MasterCard first chargeback data in WDP. Polls MCM unworked
first chargeback queue via DataPower, fetches claim detail and settled
transaction PAN, encrypts PAN, and writes PENDING rows to chbk_outbox_row.
Acknowledges processed items back to MCM. Runs every 5 minutes. Replica
count confirmed as 1 — replicas > 1 creates parallel polling risk.

**09 — CaseFillingBatch**
Handles MasterCard dispute lifecycle stages after first chargeback: PAB,
ARB, PRA, AII, AIM. Polls MCM ReceiverCaseFiling queue via DataPower,
determines dispute stage from claim detail fields, encrypts PAN for new
claims, and writes to chbk_outbox_row. Stage determination uses priority-
ordered branch logic on caseType + rulingDate + issuerAction.

---

### Inbound Processing — File-Based Path

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 10 | DM Mainframe | External Infrastructure | ✅ Production | 📋 PENDING | WDP-COMP-10-DM-MAINFRAME.md |
| 11 | FileProcessor | Batch/Scheduler | ✅ Production | 📝 DRAFT | WDP-COMP-11-FILE-PROCESSOR.md |
| 12 | InboundDisputeEventScheduler | Batch/Scheduler + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-12-INBOUND-EVENT-SCHEDULER.md |
| 13 | FileAcknowledgementProcessor | Batch/Scheduler | ✅ Production | ✅ COMPLETE | WDP-COMP-13-FILE-ACK-PROCESSOR.md |

**10 — DM Mainframe**
On-premise enterprise mainframe that bridges file-based card network and
merchant integrations with WDP. Not owned by the WDP team — shared enterprise
infrastructure. Receives inbound files from Amex, Discover, PIN Networks,
CapitalOne, Walmart, and Meijer via Sterling Mailbox. Routes outbound response
files back to those partners. All transfers via mainframe-to-mainframe protocol
through Sterling Mailbox. WDP dependency on enterprise SLA.

**11 — FileProcessor**
Primary inbound file-ingestion component. Triggered by SQS events when ZIP
files land in S3. Identifies source from filename, parses six source types
(DWSG, DBLK, DISR, MFAD, DCPO, DNWK), writes to three PostgreSQL outbox
tables (file_job, chbk_outbox_row, file_evidence), stages evidence documents
to S3, and encrypts PAN for DNWK sources. Does not publish to Kafka — the
outbox is the handoff to InboundDisputeEventScheduler.

**12 — InboundDisputeEventScheduler**
Transactional outbox publisher. Runs five schedulers that read PENDING rows
from four outbox tables and publish to the appropriate Kafka topics. Serves
as the Kafka publishing layer for VisaDisputeBatch, FirstChargebackBatch,
CaseFillingBatch, and FileProcessor — none of those write to Kafka directly.
Uses mark-before-send (at-most-once) delivery pattern. Replica count > 1
creates race condition risk — no distributed locking.

**13 — FileAcknowledgementProcessor**
Generates outbound acknowledgement files for inbound file jobs that required
confirmation. Polls file_job table for completed jobs with ack_required=true,
reads per-row detail from chbk_outbox_row, formats ACK files in four merchant-
specific formats (Walmart SigCap, Meijer, CapitalOne CMRTR, CapitalOne BJWC),
and uploads to S3. Documented in WDP-COMPONENTS.md sections 2.3.4 and 5.3.2
— these are the same deployable (wdp-evidence-ack-scheduler).

---

### Core Processing — Event Consumers

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 14 | CaseCreationConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-14-CASE-CREATION-CONSUMER.md |
| 15 | EvidenceConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-15-EVIDENCE-CONSUMER.md |
| 16 | BusinessRulesProcessor | Kafka Consumer + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md |
| 17 | CaseExpiryUpdateConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-17-CASE-EXPIRY-CONSUMER.md |
| 18 | NotificationOrchestrator | Kafka Consumer + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md |

**14 — CaseCreationConsumer**
Primary case creation component for all non-NAP dispute events. Consumes
from new-case-events topic. Enriches events with merchant and transaction
data from the relevant acquiring platform API (CORE via MerchantTransactionService,
LATAM and VAP via direct API calls). Creates dispute cases in WDP Core via
CaseManagementService. Handles transient PAN decryption for enrichment calls —
clear PAN never persisted.

**15 — EvidenceConsumer**
Consumes from case-evidence-events topic and attaches evidence documents to
existing dispute cases. Retrieves documents from S3 /staging using path
reference in the event, then calls DocumentManagementService to store the
document in S3 with metadata in DynamoDB.

**16 — BusinessRulesProcessor**
Kafka consumer that executes configured business rules against dispute events.
The execution engine — distinct from BusinessRulesService which manages rule
definitions. Reads rules directly from Aurora PostgreSQL (no API call to
BusinessRulesService). Publishes processed events to outgoing-events topic
for downstream routing. BRE crash recovery via step checkpointing (DEC-011).

**17 — CaseExpiryUpdateConsumer**
Consumes from case-action-events (expiry) topic and updates case expiry status
in WDP Core database. Terminal consumer — no further downstream processing
triggered from this component.

**18 — NotificationOrchestrator**
Central outbound routing component. Consumes from outgoing-events topic and
routes each event to the appropriate outbound Kafka topic based on code-defined
routing logic (no config table). Publishes to external-request-events,
core-request-events, and case-action-events (expiry). Also writes to
wdp.file_generation_event table (⚠️ previously documented as wdp.file_generation_event
— that table does not exist) to stage requests for file-based output components.
Does NOT publish to internal-integration-events (that is AcceptService and
ContestService).

---

### Core Processing — Case Action Services

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 19 | AcceptService | REST API + Kafka Producer | ✅ Production | 📋 PENDING | WDP-COMP-19-ACCEPT-SERVICE.md |
| 20 | ContestService | REST API + Kafka Producer | ✅ Production | 📋 PENDING | WDP-COMP-20-CONTEST-SERVICE.md |
| 21 | ChargebackService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-21-CHARGEBACK-SERVICE.md |
| 22 | DisputeService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-22-DISPUTE-SERVICE.md |
| 23 | CaseManagementService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-23-CASE-MANAGEMENT-SERVICE.md |
| 24 | CaseActionService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-24-CASE-ACTION-SERVICE.md |
| 25 | NotesService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-25-NOTES-SERVICE.md |
| 26 | QuestionnaireService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-26-QUESTIONNAIRE-SERVICE.md |

**19 — AcceptService**
Processes merchant decisions to accept a dispute. Calls Visa and MasterCard
APIs directly and synchronously to notify the network of acceptance. For NAP
disputes across all networks, publishes to internal-integration-events for
NAP Outcome Processor to notify NAP-DPS.

**20 — ContestService**
Processes merchant decisions to contest a dispute. Calls Visa and MasterCard
APIs directly and synchronously. Publishes two event categories to
internal-integration-events: NAP contest events (all networks) for NAP-DPS
notification, and Visa contest events (all acquiring platforms) to trigger
Visa questionnaire retrieval by VisaResponseQuestionnaire.

**21 — ChargebackService**
The only WDP Core service accessible externally to merchants via APIGEE and
Akamai. Supports both dispute read and action operations. Also serves as the
callback endpoint for third-party systems (SignifyD, JustAI) and BEN merchant
notification platform after they receive dispute notifications.

**22 — DisputeService**
Manages the core dispute lifecycle — state transitions, dispute data retrieval,
and dispute-level operations. Authoritative service for dispute state.

**23 — CaseManagementService**
Owns the dispute case record. Responsible for case creation, case updates, and
maintaining the integrity of the case state machine. Authoritative source for
all case data in WDP. Called by CaseCreationConsumer and NAPDisputeEventProcessor
as the primary case write target.

**24 — CaseActionService**
Handles specific case-level actions taken by operations teams — routing a case
to a queue, writing off a case, splitting a case, or advancing a case through
the dispute lifecycle. Called by NAPDisputeDeclineBatch to create IDCL draft
actions.

**25 — NotesService**
Allows operations teams and merchants to add notes to a dispute case. Notes
are retained as part of the immutable case audit trail.

**26 — QuestionnaireService**
Manages the response questionnaire merchants complete before submitting a
contest. Captures structured evidence and reasoning submitted to the card
network as part of the contest response.

---

### Core Processing — Search & Display

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 27 | CaseSearchService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-27-CASE-SEARCH-SERVICE.md |
| 28 | DisplayCodeService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-28-DISPLAY-CODE-SERVICE.md |

**27 — CaseSearchService**
Provides dispute search capability across the platform. Supports extensive
filter criteria. Serves both portal UIs and programmatic search requests.
Also called by VisaDisputeBatch and NAPDisputeDeclineBatch for case lookup
during ingestion and batch processing.

**28 — DisplayCodeService**
Manages mapping between internal WDP codes and human-readable display values
shown in UI — reason codes, network codes, status codes, action codes. Called
by NAPDisputeEventService during enrichment to determine TIER1 sub-product
eligibility from fraud and INR reason code lists.

---

### Core Processing — Queue & Workflow Services

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 29 | FaxQueueService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-29-FAX-QUEUE-SERVICE.md |
| 30 | UserQueueSkillService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-30-USER-QUEUE-SKILL-SERVICE.md |

**29 — FaxQueueService**
Handles fax communications sent by merchants to WDP. Operations teams act on
merchant faxes through this service. Available to Ops Portal users only.

**30 — UserQueueSkillService**
Manages the relationship between users, queues, and skills. Determines which
queues a user can access and which cases within those queues are eligible
based on assigned skills. Likely owns the queue and skill-based routing logic
referenced in the platform architecture (follow-up required — see UAMS note
in WDP-COMP-02).

---

### Core Processing — Rules & Configuration

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 31 | BusinessRulesService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-31-BUSINESS-RULES-SERVICE.md |
| 32 | RulesService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-32-RULES-SERVICE.md |

**31 — BusinessRulesService**
Provides CRUD management of business rules used in dispute processing. Called
by portal UIs via API Gateway to add, modify, retrieve, and delete rules.
Manages rule definitions only — does NOT execute rules. Execution is handled
by BusinessRulesProcessor (16). Called by VisaDisputeBatch for queue status
rule lookups.

**32 — RulesService**
Manages additional configuration rules for dispute routing, queue assignment,
and case handling behaviour. Works alongside BusinessRulesService to provide
full rules configuration capability.

---

### Core Processing — Organisation & User Services

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 33 | OrgManagementService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-33-ORG-MANAGEMENT-SERVICE.md |
| 34 | MerchantTransactionService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-34-MERCHANT-TRANSACTION-SERVICE.md |

**33 — OrgManagementService**
Manages organisational hierarchies within WDP. Handles merchant account
configuration, org-level settings, and routing rule configuration at the
organisation level.

**34 — MerchantTransactionService**
Provides merchant and transaction data enrichment for CORE acquiring platform
disputes. Called by CaseCreationConsumer during dispute case creation to fetch
merchant and transaction details from the CORE platform. LATAM and VAP
enrichment is handled by CaseCreationConsumer calling those platform APIs directly.

---

### Core Processing — Supporting Services

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 35 | EncryptionService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-35-ENCRYPTION-SERVICE.md |
| 36 | TokenService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-36-TOKEN-SERVICE.md |
| 37 | DocumentManagementService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-37-DOCUMENT-MANAGEMENT-SERVICE.md |
| 38 | APILogService | REST API | ✅ Production | 📋 PENDING | WDP-COMP-38-API-LOG-SERVICE.md |

**35 — EncryptionService**
Sole component authorised to handle plaintext PAN data in WDP. Encrypts PANs
at point of ingestion. Provides transient decryption to authorised callers
(CaseCreationConsumer only). Implements two-token PAN strategy (DEC-007) —
HPAN for deterministic lookups, EPAN for reversible storage. Uses AWS KMS
with 6-hour DEK cache (DEC-008). Global dependency — all inbound processing
halts if this service is unavailable.

**36 — TokenService**
Centralised JWT token management for all WDP consumers and batch jobs.
Caches JWT tokens in AWS ElastiCache and refreshes from IDP only on expiry.
Eliminates per-component IDP integration. Note: has no relation to PAN
tokenisation — JWT management only.

**37 — DocumentManagementService**
Handles evidence document storage and retrieval. Documents stored in S3 with
metadata in DynamoDB. Called by EvidenceConsumer (inbound file evidence),
VisaResponseQuestionnaire (Visa contest questionnaires), NAPDisputeEventService
(NAP document attachments), and merchant UI submissions. Only WDP component
with a DynamoDB dependency.

**38 — APILogService**
Provides API-level audit logging across the platform. Captures inbound API
calls for audit, compliance, and operational troubleshooting. Called via AOP
by components that throw LoggingException (e.g. VisaDisputeBatch).

---

### Outbound Processing — Card Network Response

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 39 | NAPOutcomeProcessor | Kafka Consumer | ✅ Production | 📋 PENDING | WDP-COMP-39-NAP-OUTCOME-PROCESSOR.md |
| 40 | VisaResponseQuestionnaire | Kafka Consumer | ✅ Production | 📋 PENDING | WDP-COMP-40-VISA-RESPONSE-QUESTIONNAIRE.md |

**39 — NAPOutcomeProcessor**
Consumes from internal-integration-events and delivers dispute outcomes to
NAP-DPS for NAP acquiring platform money movement. Processes NAP accept events
(from AcceptService) and NAP contest events (from ContestService) for all
networks. Direct API call to NAP-DPS — planned migration to EDIA route.

**40 — VisaResponseQuestionnaire**
Consumes from internal-integration-events and retrieves the Visa questionnaire
submitted as part of a merchant contest. Triggered for all acquiring platforms
whenever a merchant contests a Visa dispute. Calls Visa API, then stores the
questionnaire via DocumentManagementService (S3 + DynamoDB).

---

### Outbound Processing — Notification Consumers

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 41 | ThirdPartyNotificationConsumer | Kafka Consumer | ✅ Production | 📋 PENDING | WDP-COMP-41-THIRD-PARTY-NOTIFICATION-CONSUMER.md |
| 42 | BENConsumer | Kafka Consumer | ✅ Production | 📋 PENDING | WDP-COMP-42-BEN-CONSUMER.md |
| 43 | CoreNotificationConsumer | Kafka Consumer | ✅ Production | 📋 PENDING | WDP-COMP-43-CORE-NOTIFICATION-CONSUMER.md |
| 44 | EDIAConsumer | Kafka Consumer + Kafka Producer | 🔴 Planned | ⬜ NOT STARTED | WDP-COMP-44-EDIA-CONSUMER.md |

**41 — ThirdPartyNotificationConsumer**
Consumes from external-request-events and delivers dispute event notifications
to SignifyD and JustAI via REST API. After receiving notification, those systems
call back ChargebackService to get dispute details and act on disputes.

**42 — BENConsumer**
Consumes from external-request-events and delivers dispute lifecycle
notifications to BEN merchant notification platform via webhook. Merchants
receive notifications via BEN and call back ChargebackService for dispute
details and actions.

**43 — CoreNotificationConsumer**
Consumes from core-request-events and delivers dispute outcome notifications
to the CORE acquiring platform via IBM DB2 direct connection. CORE is WDP-owned
so has a direct DB2 connection, bypassing the EDIA route used by external
platforms. Potential future migration to EDIA as EDIA adoption matures.

**44 — EDIAConsumer** *(Planned)*
WDP-owned anti-corruption layer between WDP internal event format and the
enterprise EDIA Kafka streaming platform. Will consume from external-request-events,
translate WDP internal dispute events to EDIA enterprise format, and publish
to the EDIA platform Kafka topic. Strategic outbound route for NAP, LATAM,
and VAP acquiring platforms.

---

### Outbound Processing — File Generation

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 45 | CapitalOneResponseFileProcessor | Batch/Scheduler | ✅ Production | 📋 PENDING | WDP-COMP-45-CAPONE-RESPONSE-FILE-PROCESSOR.md |
| 46 | NetworkResponseFileProcessor | Batch/Scheduler | ✅ Production | 📋 PENDING | WDP-COMP-46-NETWORK-RESPONSE-FILE-PROCESSOR.md |
| 47 | DialoguIssuerDocumentProcessor | Batch/Scheduler | ✅ Production | 📋 PENDING | WDP-COMP-47-DIALOGU-ISSUER-DOC-PROCESSOR.md |
| 48 | NYCEFileGenerationProcessor | Batch/Scheduler | 🔴 Planned | ⬜ NOT STARTED | WDP-COMP-48-NYCE-FILE-GENERATION-PROCESSOR.md |

**45 — CapitalOneResponseFileProcessor**
Reads from wdp.file_generation_event DB table and generates CapitalOne response files.
Places files in S3 /outbound/capitalOne/ for ControlM to transfer to Sterling
Mailbox for delivery to CapitalOne via DM Mainframe.

**46 — NetworkResponseFileProcessor**
Reads from wdp.file_generation_event DB table and generates four network response
files: Amex, AmexHybrid, Discover, and DiscoverHybrid. Each placed in its
own dedicated S3 /outbound folder. DiscoverHybrid uses a special on-premise
File Transfer Batch pull via SFTP rather than the standard ControlM push flow.

**47 — DialoguIssuerDocumentProcessor**
Reads from wdp.file_generation_event DB table and generates a ZIP file containing
all issuer documents received by WDP. Places ZIP in S3 /outbound/dialogu/
for delivery to merchants via Sterling SFTP to Dialogue.

**48 — NYCEFileGenerationProcessor** *(Planned)*
Will generate outbound files for PIN Networks (NYCE) and place them in
S3 /outbound/pinNetworks/ for ControlM to transfer to Sterling → DM Mainframe
→ PIN Networks.

---

### UI Layer

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 49 | WDP Merchant Portal | UI Application | ✅ Production | 📋 PENDING | WDP-COMP-49-MERCHANT-PORTAL.md |
| 50 | WDP Ops Portal | UI Application | ✅ Production | 📋 PENDING | WDP-COMP-50-OPS-PORTAL.md |

**49 — WDP Merchant Portal**
Merchant-facing UI. Allows merchants to view disputes, take actions, submit
evidence, and manage their organisation and users. Routes through Akamai for
CDN and edge security. Includes Disputes, User Management, and Org Management
sections. Dashboard section planned. UI sections 7.3–7.6 from WDP-COMPONENTS.md
are documented within this file.

**50 — WDP Ops Portal**
Internal operations-facing UI. Allows WDP operations teams to manage disputes,
configure queues, manage users and organisations. Connects directly to API
Gateway — does not route through Akamai. Includes Disputes, Queues, User
Management, and Org Management sections. Dashboard section planned. Queue
and Fax Queue functionality documented within this file.

---
## Migration Progress Summary

| Status | Count | Components |
|--------|-------|------------|
| ✅ COMPLETE (individual file created and confirmed) | 1 | COMP-13 FileAcknowledgementProcessor |
| 📝 DRAFT (individual file created, architect confirmation pending) | 12 | COMP-01 through COMP-14 (excluding COMP-10) |
| 📋 PENDING (not yet migrated to individual file) | 25 | COMP-15 through COMP-50 (excluding above) |
| ⬜ NOT STARTED (no content yet) | 2 | COMP-44 (EDIAConsumer), COMP-48 (NYCEFileGenerationProcessor) |

**Phase 1 migration complete.** All 13 COMPLETE components from WDP-COMPONENTS.md now have individual files.
**Next: Phase 2 — DRAFT components.** Start with COMP-14 CaseCreationConsumer.

---

## Cross-Reference: Where Platform Sections Are Documented

| Section in WDP-COMPONENTS.md | Documented in |
|------------------------------|---------------|
| Part 3 — Kafka Event Bus (3.1, 3.2) | WDP-KAFKA.md (to be created) |
| Part 6 — Acquiring Platform Integrations (6.1–6.5) | WDP-INTEGRATIONS.md |
| Part 7.3 — Disputes Section (UI) | WDP-COMP-49 and WDP-COMP-50 |
| Part 7.4 — Queues Section (UI) | WDP-COMP-50 (Ops Portal only) |
| Part 7.5 — User Management Section (UI) | WDP-COMP-49 and WDP-COMP-50 |
| Part 7.6 — Org Management Section (UI) | WDP-COMP-49 and WDP-COMP-50 |
| Part 7.7 — Dashboard Section (UI, Planned) | WDP-COMP-49 and WDP-COMP-50 |
| Appendix — Copilot CLI Question Sets | WDP-COMP-TEMPLATE.md (as usage guidance) |

---

## Adding New Components (Quarterly Update Protocol)

1. Assign the next sequential number (current last: 50)
2. Add a row to the registry table in the correct logical section
3. Write the one-line responsibility summary below the table
4. Create the component file using WDP-COMP-TEMPLATE.md
5. After the file is complete, update WDP-KAFKA.md and WDP-DB.md
   with entries from the new component
6. Update WDP-HANDOVER.md to note the new component and its number

---

*Last updated: April 2026*
*Current component count: 50*
*Next available number: 51*
