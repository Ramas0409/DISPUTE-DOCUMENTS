# WDP-DB.md
**Worldpay Dispute Platform — Database Schema Ownership Map**
*Version: 1.0 SKELETON | April 2026*
*Source: WDP-COMPONENTS.md COMPLETE sections + component file enrichment (in progress)*

---

## Purpose of This Document

This is the master database reference for the WDP platform. It contains:

1. **Database Instances** — all database systems WDP connects to and
   who owns them
2. **Schema Ownership Map** — every schema and table, which component
   owns it (writes to it), and which components access it read-only
3. **Shared Table Risk Register** — tables written by more than one
   component, flagged as coordination risks
4. **Cross-Component Data Flow** — how data moves between components
   via shared database tables (outbox pattern, lookup tables)

This document is populated in two passes:
- **Pass 1 (this skeleton):** Database instances, schemas, and table
  ownership extracted from COMPLETE component sections in
  WDP-COMPONENTS.md.
- **Pass 2 (ongoing):** Table-level detail (key columns, retention,
  indexes) populated as each WDP-COMP-[NN]-*.md file is completed.

Individual component files (WDP-COMP-[NN]-*.md) contain the Database
Ownership section specific to each component. This file is the
cross-cutting view — the complete data ownership map for the platform.

---

## How to Update This Document

When a WDP-COMP-[NN]-*.md file is completed:
1. Add or update rows in the Schema Ownership Map for all tables
   that component owns (writes to)
2. Add the component to the `Accessed by (read-only)` column for
   any tables it reads but does not own
3. Check if any newly confirmed table appears in the Shared Table
   Risk Register — if a table is written by more than one component,
   flag it
4. Mark table rows ✅ CONFIRMED once the owning component file is
   architect-confirmed

---

## 1. Database Instances

| Database | Type | Owner | Purpose | WDP Components That Connect |
|----------|------|-------|---------|----------------------------|
| WDP Aurora PostgreSQL (globaldisputedatabase) | Amazon Aurora PostgreSQL | WDP Team | Primary WDP operational database. Contains all dispute case data, outbox tables, file processing tables, and configuration. | Most WDP components |
| NAP PostgreSQL | Amazon Aurora PostgreSQL | WDP Team | NAP acquiring platform data — entity hierarchy, MIDs, NAP case data, batch metadata | COMP-02 UAMS, COMP-05 NAPDisputeEventProcessor, COMP-06 NAPDisputeDeclineBatch. COMP-04 NAPDisputeEventService confirmed stateless — no database connection of any kind. |
| IBM DB2 (Core merchant master) | IBM DB2 | Enterprise (not WDP owned) | Core enterprise merchant hierarchy. Entity relationships, chain hierarchy, merchant master data | COMP-03 CHAS (read-only) |
| Amazon DynamoDB | DynamoDB | WDP Team | Evidence document metadata. Used exclusively by DocumentManagementService | COMP-37 DocumentManagementService (owns) |
| AWS ElastiCache | Redis | WDP Team | JWT token cache for TokenService | COMP-36 TokenService (owns) |

---

## 2. Schema Ownership Map — WDP Aurora PostgreSQL

### Schema: `wdp`

*The primary WDP schema. Owned by the WDP Core team.*

| Table | Owning Component | Purpose | Key Columns | Accessed Read-Only By | Confirmed |
|-------|-----------------|---------|-------------|----------------------|-----------|
| `wdp.chbk_outbox_row` | COMP-07, COMP-08, COMP-09, COMP-11 (writers — PENDING rows); COMP-12 InboundDisputeEventScheduler (status transitions + Kafka metadata); COMP-14 CaseCreationConsumer (status updates only — FAILED/ERROR/SUCCESS/PENDING_DEFERRED/SKIPPED); COMP-15 EvidenceConsumer (status updates only — FAILED/ERROR/SUCCESS) | Shared transactional outbox. Dispute events (CHARGEBACK_PROCESS) and evidence (EVIDENCE_ATTACH) queued for Kafka publishing. Key cols: id, file_job_id, row_number, parent_row_number, event_type, status (LOADING/PENDING/PUBLISHED/SUCCESS/ERROR/BLOCKED/SKIPPED), payload (JSON), c_ntwk_case_id, c_ntwk_phase_id, c_case_stage, c_case_ntwk, c_acq_platform, c_level1_entity, kafka_topic, kafka_partition, kafka_offset, idempotency_id, retry_count, next_retry_at, published_at, source_event, document_type, record_detail (JSONB), created_by | COMP-13 FileAcknowledgementProcessor (reader — per-row ACK detail). No unique constraint confirmed — application-level duplicate check only. COMP-11 created_by = "WPFLEPR". Kafka cols set by COMP-12 only. BLOCKED status used for DCPO evidence rows. SUCCESS rows archived to chbk_outbox_row_archive by COMP-12 Scheduler2 after 30 days. | ✅ Confirmed from COMP-07, COMP-08, COMP-09, COMP-11, COMP-12, COMP-13 analysis |
| `wdp.file_job` | COMP-11 FileProcessor (all cols except ACK fields); COMP-12 InboundDisputeEventScheduler (status→COMPLETED, completed_at); COMP-13 FileAcknowledgementProcessor (ack_status, ack_generated_at) | File-level processing ledger. One row per inbound ZIP. Key cols: id, file_name, s3_key, s3_bucket, file_size_bytes, status (PENDING/PROCESSING/ERROR — ⚠️ COMPLETED never written by COMP-11), source (WALMART_SIG_CAP / CORE_BULK_RESP / CAPONE_CMRTR / CAPONE_BJWC — confirmed ApplicationConstants values, not DWSG/DBLK/DCPO), ack_required, ack_status (PENDING/COMPLETED), ack_generated_at, total_rows, successful_rows, failed_rows, error_rows, total_evidences, attached_evidences, failed_evidences, error_code, error_message, completed_at, created_by="WPFLEPR", updated_by | COMP-13 FileAcknowledgementProcessor polls for status IN (COMPLETED, ERROR) — ⚠️ COMP-11 never writes COMPLETED, only PROCESSING. ACK generation for successfully completed files may never trigger. Architect decision required. | ✅ Confirmed from COMP-11, COMP-12, COMP-13 analysis |
| `wdp.file_evidence` | COMP-11 FileProcessor (inserts); COMP-15 EvidenceConsumer (updates attachment_status, appended_file_name, failed_s3_key) | Evidence document index. One row per extracted document. Key cols: id, file_job_id (FK), chbk_outbox_row_id (FK), file_name, s3_key (staging path), s3_bucket, attachment_status (PENDING/ATTACHED/FAILED), attached_at, attachment_error, failed_s3_key, i_case, c_ntwk_case_id, created_by="WPFLEPR". Not written for DNWK sources. | COMP-12 InboundDisputeEventScheduler (Scheduler5 — error report only, read-only). COMP-13 does not access this table. COMP-15 reads s3_bucket, s3_key, file_name, attachment_status for document retrieval. | ✅ Confirmed from COMP-11, COMP-12, COMP-15 analysis |
| `wdp.ACTION` | ⚠️ Write path owner TBC — multiple components suspected | WDP case action records. COMP-15 EvidenceConsumer writes: transfers ownership field (C_OWNR) from MERCHANT to WPAYOPS for RESPDOC document type on WDP path only. | COMP-15 EvidenceConsumer (conditional write — RESPDOC WDP path only) | ⚠️ PENDING — ⚠️ SHARED TABLE — confirm all other writers |
| `wdp.CASE` | ⚠️ Write path owner TBC — CaseManagementService primary writer | Central case record. COMP-15 EvidenceConsumer writes: updates Z_UPDT (updated timestamp) for RESPDOC document type on WDP path only. | COMP-03 CHAS (read-only), COMP-14 CaseCreationConsumer (read-only — Capone REQ check), COMP-15 EvidenceConsumer (conditional write — RESPDOC WDP path only) | ⚠️ PENDING — ⚠️ SHARED TABLE — confirm all writers |
| `wdp.disputes_questionnaire` | ⚠️ Owner TBC | Dispute questionnaire answers and attached file names. COMP-15 EvidenceConsumer upserts attached file names for RESPDOC document type on WDP path only. | COMP-15 EvidenceConsumer (upsert — RESPDOC WDP path only) | ⚠️ PENDING — confirm full schema and other writers |
| `wdp.case_expiry` | COMP-17 CaseExpiryUpdateConsumer | Active expiry schedule per case+action. Stores due dates consumed by downstream expiry-driven workflows. Upserted on non-CLOSED action status; deleted on CLOSED. Key cols: i_case (caseNumber), i_action_seq, c_acq_platform, d_expiry_due, d_response_due, i_retry_count, c_workflow_name, z_insr, z_updt. Each write is its own @Transactional JPA transaction, independent of the outbox write. | ⚠️ Downstream consumers of this table not yet confirmed — Copilot follow-up needed | ✅ Confirmed from COMP-17 source |
| `nap.br_case_audit_log` | COMP-16 BusinessRulesProcessor | Audit log of business rules evaluated (matched and not matched) for each UK/NAP path message. Written on every message regardless of rule match outcome. | case_id, rule_id, matched, evaluated_at, sort_order | None confirmed | ✅ Confirmed from COMP-16 source |
| `wdp.br_case_audit_log` | COMP-16 BusinessRulesProcessor | Audit log of business rules evaluated for each US path (CORE/VAP/PIN) message. Structural mirror of nap.br_case_audit_log in the wdp schema. | case_id, rule_id, matched, evaluated_at, sort_order | None confirmed | ✅ Confirmed from COMP-16 source |
| `wdp.chbk_outbox_row_archive` | COMP-12 InboundDisputeEventScheduler (Scheduler2) | Long-term archive of SUCCESS rows moved from chbk_outbox_row. 30-day threshold hardcoded. Mirrors chbk_outbox_row columns + archived_at. No purge policy confirmed for archive table itself. | None confirmed | ✅ Confirmed from COMP-12 analysis |
| `wdp.outgoing_event_outbox` | COMP-17 CaseExpiryUpdateConsumer (channel_type = EXPIRY_EVENTS, created_by = WCSEEXPC); COMP-18 NotificationOrchestrator (channel_type = notification orchestration rows); ⚠️ additional writers TBC |Consumer-side audit, idempotency, and predecessor blocking store for EXPIRY_EVENTS messages. Also used as outgoing event relay by Scheduler3. Key cols: id, i_case, i_action_seq, channel_type, idempotency_id, event_timestamp, status (PUBLISHED/FAILED/ERROR/PENDING_DEFERRED), retry_count, next_retry_at, error_code, error_message, original_event (JSON), created_by, created_at, updated_at | COMP-12 InboundDisputeEventScheduler (Scheduler3 reads FAILED and PENDING_DEFERRED rows for retry) | ⚠️ PENDING — full set of writers not yet confirmed |
| `wdp.bre_orchestration_outbox` | COMP-18 NotificationOrchestrator (confirmed — writes component=NOTIFICATION_ORCHESTRATOR rows; COMP-12 Scheduler4 also writes component=BUSINESS_RULES rows) | Outbox for BRE and notification orchestration events. component field determines routing. Key cols: id, i_case, i_action_seq, component, status, retry_count, next_retry_at, idempotency_id, original_event (JSON). | COMP-12 InboundDisputeEventScheduler (Scheduler4 reads FAILED and PENDING_DEFERRED rows) | ✅ Confirmed from COMP-18 source |
| `wdp.notification_orchestration_outbox` | ⚠️ Publisher TBC | Outbox for notification orchestration events — Scheduler4 reads PROCESSING rows | status, payload | COMP-12 InboundDisputeEventScheduler (Scheduler4) | ⚠️ PENDING |
| `wdp.file_generation_event` | COMP-18 NotificationOrchestrator | Staging table for file-based output requests. Written when Filter 4 matches. ⚠️ CORRECTION: previously documented as `wdp.file_notifications` — that name does not exist in the COMP-18 codebase. Key cols: i_case, i_action_seq, c_ntwk_case_id, c_file_type, c_acq_platform, status. fileType derived from caseNetwork + hybridMerchant flag. | COMP-45 CapitalOneResponseProcessor, COMP-46 DialoguIssuerDocumentProcessor, COMP-47 NetworkResponseFileProcessor (read to generate outbound files) | ✅ Confirmed from COMP-18 source. Table name corrected. |
| `wdp.acl` | COMP-02 UAMS | Access control list linking consumers to source systems and entity values. For downstream consumers — not used by UAMS for its own auth | i_acl_id (PK), c_consumer_name, c_source_system, c_entity_class, c_entity_type, c_entity_value, c_status, c_created_by, t_created_timestamp, c_updated_by, t_updated_timestamp, c_activated_by, c_activated_timestamp, c_deactivated_by, t_deactivated_timestamp. Index: acl_search_index on (c_consumer_name, c_status)| ⚠️ Downstream consumers TBC | ⚠️ PENDING |
| `wdp.api_route` | ⚠️ Owner TBC | API Gateway URL routing configuration. 26+ routes loaded at pod startup | id, path, original_path_regex, replacement_path, uri, auth_exceptions — writer still unconfirmed | COMP-01 API Gateway (read at startup) | ⚠️ PENDING |
| `wdp.chbk_outbox_row_archive` | COMP-12 InboundDisputeEventScheduler | Archive table for SUCCESS rows from chbk_outbox_row (Scheduler2) | ⚠️ Mirrors chbk_outbox_row | None | ⚠️ PENDING |
| `wdp.case` | Write path unconfirmed (CaseManagementService — COMP not yet migrated) | Central case record — created by CaseManagementService, read by COMP-03 CHAS and COMP-14 CaseCreationConsumer for case lookup (create vs update routing) | i_case_id (PK), case_number, c_level1_entity, c_level4_entity | COMP-03 CHAS (read-only, caseNumber→merchantId+chainId), COMP-14 CaseCreationConsumer (read-only, case lookup for NEW vs UPDATE routing) | ⚠️ PENDING — owning component (CaseManagementService) not yet migrated |

*Add rows as component files are completed and table ownership is confirmed.*

---

### Schema: `nap`

*NAP acquiring platform schema. Managed by WDP team for NAP integration.*

| Table | Owning Component | Purpose | Key Columns | Accessed Read-Only By | Confirmed |
|-------|-----------------|---------|-------------|----------------------|-----------|
| `nap.nap_parent_entity` | COMP-02 UAMS | Top-level merchant groupings for NAP platform | entity_id, name, status | ⚠️ TBC | ⚠️ PENDING |
| `nap.nap_child_entity` | COMP-02 UAMS | Sub-groupings beneath a NAP parent entity | child_id, parent_id, status | ⚠️ TBC | ⚠️ PENDING |
| `nap.nap_merchant` | COMP-02 UAMS | Individual merchant IDs with MCC and WPG ID | mid, mcc, wpg_id, parent_id | ⚠️ TBC | ⚠️ PENDING |
| `nap.nap_entity_rel` | COMP-02 UAMS | Merchant-to-entity relationships — primary authorization lookup | merchant_id, parent_entity, child_entities | COMP-02 UAMS /authorize endpoint | ✅ Confirmed from UAMS analysis |
| `nap.case` | ⚠️ Owned by core WDP — written by case management | Case lookup table for NAP platform — used by UAMS to resolve caseNumber to merchantId | case_number, merchant_id | COMP-02 UAMS (read-only, caseNumber→merchantId), COMP-03 CHAS (read-only, caseNumber→chainId+merchantId), COMP-06 NAPDisputeDeclineBatch (read-only) | ⚠️ PENDING — owning component to confirm |
| `nap.action` | ⚠️ Case action services (write path TBC) | NAP case actions — queried by NAPDisputeDeclineBatch for open PAB actions | C_CASE_STAGE, C_ACTION_TYPE, C_ACTION_STA, C_MIGRATION_STA | COMP-06 NAPDisputeDeclineBatch (read-only, filtered query) | ⚠️ PENDING |
| `NAP.DISPUTE_EVENT_CONSUMER_ERROR` | COMP-05 NAPDisputeEventProcessor | Database DLQ for failed Kafka events — raw payload stored for manual reprocessing | error_id, raw_payload, arn, error_timestamp | ⚠️ Manual reprocessing path only | ✅ Confirmed from COMP-05 analysis |

---

### Spring Batch Metadata Tables

*Written by all Spring Batch jobs. Schema varies by component — some use dedicated schemas, some share.*

| Table | Written by | Purpose |
|-------|------------|---------|
| `BATCH_JOB_INSTANCE` | COMP-06, COMP-07, COMP-08, COMP-09 | Job identity and deduplication |
| `BATCH_JOB_EXECUTION` | COMP-06, COMP-07, COMP-08, COMP-09 | Execution status per run |
| `BATCH_STEP_EXECUTION` | COMP-06, COMP-07, COMP-08, COMP-09 | Step-level progress and counts |

COMP-07 VisaDisputeBatch confirmed: Spring Batch metadata written to 
WDP Aurora PostgreSQL. Schema name not confirmed from source — 
injected via K8s secret. Separate from Visa RTSI datasource.
COMP-08 FirstChargebackBatch confirmed: Spring Batch metadata written to
WDP Aurora PostgreSQL (wpdisputedatabase). Table prefix injected via
env var table_prefix (K8s secret). Schema/prefix value not auditable
from source. Uses wdpTransactionManager — same datasource as
wdp.chbk_outbox_row writes.
COMP-09 CaseFillingBatch confirmed: Spring Batch metadata written to
WDP Aurora PostgreSQL. Table prefix injected via K8s secret
(scheduler_cron secret — same secret as cron). Schema/prefix value
not auditable from source alone. Uses wdpTransactionManager — same
datasource as wdp.chbk_outbox_row writes.

⚠️ Schema location (which PostgreSQL database and schema) for Spring Batch
metadata tables varies per component and must be confirmed from each
component's Copilot CLI analysis.

---

## 3. External Database Dependencies

These databases are not owned by WDP but are accessed by WDP components.

| Database | Accessed by | Access type | Purpose |
|----------|-------------|-------------|---------|
| IBM DB2 — Core merchant master (MD, DC, BC schemas) | COMP-03 CHAS; COMP-15 EvidenceConsumer (V3 path only — BC.TBC_DM_CASE and BC.TBC_DM_OCCUR read with UNCOMMITTED READ isolation) | Read-only | COMP-03: Entity scope validation and hierarchy data. COMP-15: V3 legacy CORE path case lookup — BC.TBC_DM_CASE joined to BC.TBC_DM_OCCUR to resolve case and occurrence records. JPA entities: MD.TMD_ENTY, MD.TMD_ENTY_REL, DC.TDC_VIQ_ORG_ENTY, BC.TBC_MRCHNT_MAST_BO, MD.TMD_DISPLAY_CODES, MD.TMD_MRCHNT_WOMPLY, BC.TMD_CHAIN. ⚠️ MD.TMD_CHAIN_ANALY declared via JPA but never called at runtime. Native SQL (MerchantDaoImpl): MD.TMD_CHAIN (TMD1), MD.TMD_SUPR_CHAIN (TMD4), MD.TMD_STORE (TMD3), MD.TMD_MRCHNT (TMD5), MD.TMD_DIVISION (TMD2), MD.TMD_ADDRESS. |

---

## 4. Shared Table Risk Register

Tables written by more than one component require coordination to avoid
race conditions, duplicate writes, or conflicting state. Every entry here
is a potential architecture risk.

| Table | Writers | Risk | Mitigation in place | Severity |
|-------|---------|------|---------------------|----------|
| `wdp.chbk_outbox_row` | COMP-07 VisaDisputeBatch, COMP-08 FirstChargebackBatch, COMP-09 CaseFillingBatch, COMP-11 FileProcessor | Multiple batch jobs write PENDING rows simultaneously. No cross-job coordination. | Per-component duplicate check on (networkCaseId + networkPhaseId + disputeStage) before write | 🟡 MEDIUM — duplicate check is item-level only, no write-lock |
| `wdp.file_job` | COMP-11 FileProcessor (creates and updates), COMP-13 FileAcknowledgementProcessor (updates ack fields only) | Two components write to same row at different lifecycle stages | Non-overlapping field sets — FileProcessor owns status fields, FileAckProcessor owns ack_status fields | 🟢 LOW — field-level separation reduces risk |
| ⚠️ Additional shared tables | To be identified as component files are completed | | | |
| `nap.nap_child_entity` | UAMS createEntity + UAMS saveChildWithMerchant | saveChildWithMerchant uses wrong transaction manager (wdpTransactionManager instead of napTransactionManager) — rollback on failure may not cover nap schema writes | None — bug confirmed | 🔴 HIGH |

---

## 5. Data Retention and Compliance Notes

| Data type | Retention requirement | Enforced by | Reference |
|-----------|----------------------|-------------|-----------|
| Dispute case audit trail | 7 years — immutable | Core WDP platform | PCI-DSS compliance |
| PAN data | Never stored in plaintext | COMP-35 EncryptionService | DEC-004, DEC-007 |
| HPAN (hashed PAN) | Retained with case data | Core WDP platform | DEC-007 |
| EPAN (encrypted PAN) | Retained in EPAN→HPAN mapping table | COMP-35 EncryptionService | DEC-007 |
| Evidence documents | ⚠️ Retention policy to be confirmed | COMP-37 DocumentManagementService | TBC |
| API audit logs | ⚠️ Retention policy to be confirmed | COMP-38 APILogService | TBC |
| Outbox rows (chbk_outbox_row) | SUCCESS rows archived by COMP-12 Scheduler2 | COMP-12 InboundDisputeEventScheduler | Operational |
| Spring Batch metadata | ⚠️ Retention policy to be confirmed | Per batch component | TBC |

---

*This document is a living registry. Populate it as each component
file is completed. Flag any table that gains a second writer immediately
as a shared table risk.*

*Last updated: April 2026*
*Enrichment in progress — populating as WDP-COMP-[NN]-*.md files are completed*
