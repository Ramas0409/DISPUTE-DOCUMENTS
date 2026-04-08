# WDP-DB.md
**Worldpay Dispute Platform — Database Schema Ownership Map**
*Version: 2.0 | April 2026*
*Updated: All component files uploaded and verified April 2026*

---

## Purpose of This Document

This is the master database reference for the WDP platform. It contains:

1. **Database Instances** — all database systems WDP connects to and
   who owns them
2. **Schema Ownership Map** — every schema and table, which component
   owns it (writes to it), and which components access it read-only
3. **Shared Table Risk Register** — tables written by more than one
   component, flagged as coordination risks
4. **Data Retention and Compliance Notes**

Individual component files (WDP-COMP-[NN]-*.md) contain the Database
Ownership section specific to each component. This file is the
cross-cutting view — the complete data ownership map for the platform.

---

## 1. Database Instances

| Database | Type | Owner | Purpose | WDP Components That Connect |
|----------|------|-------|---------|----------------------------|
| WDP Aurora PostgreSQL (globaldisputedatabase / wpdisputedatabase) | Amazon Aurora PostgreSQL | WDP Team | Primary WDP operational database. Contains all dispute case data, outbox tables, file processing tables, rules, notes, questionnaires, org/user data, and configuration. | Most WDP components (wdp schema) |
| NAP Aurora PostgreSQL | Amazon Aurora PostgreSQL | WDP Team | NAP acquiring platform data — entity hierarchy, MIDs, NAP case data, batch metadata, NAP rules | COMP-02 UAMS, COMP-05 NAPDisputeEventProcessor, COMP-06 NAPDisputeDeclineBatch, COMP-23 CaseManagementService, COMP-24 CaseActionService, COMP-25 NotesService, COMP-16 BusinessRulesProcessor (read-only), COMP-31 BusinessRulesService. COMP-04 NAPDisputeEventService confirmed stateless — no DB connection. |
| IBM DB2 — Core Platform (BC, MD, DC schemas) | IBM DB2 | Enterprise (not WDP owned) | Core enterprise merchant hierarchy and Core platform dispute records. Read-only for most WDP components. COMP-43 CoreNotificationConsumer is the sole WDP writer to the BC schema dispute tables. | COMP-03 CHAS (read-only MD/DC/BC), COMP-15 EvidenceConsumer (read-only BC — V3 path), COMP-22 DisputeService (read-only BC — when coreMigrationStatus=false), COMP-34 MerchantTransactionService (read-only BC — CapOne enrichment), COMP-43 CoreNotificationConsumer (READ+WRITE BC.TBC_DM_CASE, BC.TBC_DM_OCCUR, BC.TBC_DM_NOTES) |
| MS SQL Server — Legacy Fax System | Microsoft SQL Server | External (legacy — not WDP owned) | Legacy fax queue storage shared with legacy fax processing systems outside WDP | COMP-29 FaxQueueService (read + status update — claim, reject, discard operations on dbo.IncomingFaxes) |
| Amazon DynamoDB | DynamoDB | WDP Team | Evidence document metadata. Used exclusively by DocumentManagementService | COMP-37 DocumentManagementService (owns) |
| AWS ElastiCache | Redis | WDP Team | JWT token cache for TokenService. Hash: wdpinternalidptoken:token — written by an external component not yet identified, read by COMP-36. | COMP-36 TokenService (reads; ⚠️ writer identity unknown) |

---

## 2. Schema Ownership Map — WDP Aurora PostgreSQL

### Schema: `wdp`

*The primary WDP schema. Owned by the WDP Core team.*

| Table | Owning Component(s) | Purpose | Key Columns | Accessed Read-Only By | Confirmed |
|-------|---------------------|---------|-------------|----------------------|-----------|
| `wdp.chbk_outbox_row` | COMP-07 VisaDisputeBatch, COMP-08 FirstChargebackBatch, COMP-09 CaseFillingBatch, COMP-11 FileProcessor (INSERT — PENDING rows); COMP-12 InboundDisputeEventScheduler (status transitions + Kafka metadata); COMP-14 CaseCreationConsumer (status updates — FAILED/ERROR/SUCCESS/PENDING_DEFERRED/SKIPPED); COMP-15 EvidenceConsumer (status updates — FAILED/ERROR/SUCCESS); COMP-23 CaseManagementService (conditional — updates status of existing rows when chbkOutbox block present in request) | Shared transactional outbox. Dispute events (CHARGEBACK_PROCESS) and evidence (EVIDENCE_ATTACH) queued for Kafka publishing via COMP-12. Key cols: id, file_job_id, row_number, parent_row_number, event_type, status (LOADING/PENDING/PUBLISHED/SUCCESS/ERROR/BLOCKED/SKIPPED), payload (JSON), c_ntwk_case_id, c_ntwk_phase_id, c_case_stage, c_case_ntwk, c_acq_platform, c_level1_entity, kafka_topic, kafka_partition, kafka_offset, idempotency_id, retry_count, next_retry_at, published_at, source_event, document_type, record_detail (JSONB), created_by | COMP-13 FileAcknowledgementProcessor (per-row ACK detail — read-only). No unique DB constraint confirmed — application-level duplicate check only. COMP-11 created_by = "WPFLEPR". Kafka cols set by COMP-12 only. BLOCKED status used for DCPO evidence rows. SUCCESS rows archived to chbk_outbox_row_archive by COMP-12 Scheduler2 after 30 days. | ✅ Confirmed from COMP-07 through COMP-15, COMP-23 analysis |
| `wdp.chbk_outbox_row_archive` | COMP-12 InboundDisputeEventScheduler (Scheduler2) | Long-term archive of SUCCESS rows moved from chbk_outbox_row. 30-day threshold hardcoded. Mirrors chbk_outbox_row columns + archived_at. No purge policy confirmed for archive table itself. | Mirrors chbk_outbox_row + archived_at | None confirmed | ✅ Confirmed from COMP-12 analysis |
| `wdp.file_job` | COMP-11 FileProcessor (all cols except ACK fields); COMP-12 InboundDisputeEventScheduler (status→COMPLETED, completed_at); COMP-13 FileAcknowledgementProcessor (ack_status, ack_generated_at) | File-level processing ledger. One row per inbound ZIP. Key cols: id, file_name, s3_key, s3_bucket, file_size_bytes, status (PENDING/PROCESSING/ERROR — ⚠️ COMPLETED never written by COMP-11), source (WALMART_SIG_CAP/CORE_BULK_RESP/CAPONE_CMRTR/CAPONE_BJWC), ack_required, ack_status (PENDING/COMPLETED), ack_generated_at, total_rows, successful_rows, failed_rows, error_rows, total_evidences, attached_evidences, failed_evidences, error_code, error_message, completed_at, created_by="WPFLEPR", updated_by | COMP-13 polls for status IN (COMPLETED, ERROR) — ⚠️ COMP-11 never writes COMPLETED (only PROCESSING), so ACK generation for successfully completed files may never trigger. Architect decision required. | ✅ Confirmed from COMP-11, COMP-12, COMP-13 analysis |
| `wdp.file_evidence` | COMP-11 FileProcessor (inserts); COMP-15 EvidenceConsumer (updates attachment_status, appended_file_name, failed_s3_key, attached_at) | Evidence document index. One row per extracted document. Key cols: id, file_job_id (FK), chbk_outbox_row_id (FK), file_name, s3_key (staging path), s3_bucket, attachment_status (PENDING/ATTACHED/FAILED), attached_at, attachment_error, failed_s3_key, i_case, c_ntwk_case_id, created_by="WPFLEPR". Not written for DNWK sources. | COMP-12 InboundDisputeEventScheduler (Scheduler5 — error report only, read-only). COMP-13 does not access this table. COMP-15 reads s3_bucket, s3_key, file_name, attachment_status for document retrieval. | ✅ Confirmed from COMP-11, COMP-12, COMP-15 analysis |
| `wdp.CASE` | COMP-23 CaseManagementService (primary — all case create/update operations via wdpTransactionManager); COMP-24 CaseActionService (case open/close transitions, same transaction as action write); COMP-15 EvidenceConsumer (conditional — Z_UPDT timestamp update on RESPDOC WDP path only) | Central dispute case record for all US-platform disputes (PIN, CORE, VAP, LATAM). Key cols: I_CASE (PK / caseNumber), C_CASE_STA (OPEN/CLOSED), C_CASE_FINAL_LIABILITY, I_CASE_ACTION_MAX_SEQ, Z_UPDT, C_LEVEL1_ENTITY through C_LEVEL5_ENTITY, C_CASE_NTWK, I_ACCT_CDH (⚠️ clear PAN written by COMP-23 on standard create — DEC-004 violation) | COMP-03 CHAS (read-only — caseNumber→merchantId+chainId), COMP-14 CaseCreationConsumer (read-only — Capone REQ duplicate check), COMP-16 BusinessRulesProcessor (read-only — rule evaluation), COMP-22 DisputeService (read-only — dispute summary), COMP-27 CaseSearchService (read-only — case lookup), COMP-43 CoreNotificationConsumer (read-only — enrichment before DB2 write) | ✅ Confirmed from COMP-15, COMP-23, COMP-24 analysis |
| `wdp.ACTION` | COMP-23 CaseManagementService (inserts new actions on case create/update, cascaded via USCaseEntity one-to-many, same transaction as wdp.CASE write); COMP-24 CaseActionService (inserts and updates action status, owner, liability); COMP-15 EvidenceConsumer (conditional — C_OWNR transfer from MERCHANT to WPAYOPS on RESPDOC WDP path only) | WDP case action records. Key cols: I_ACTION_SEQ, C_SOURCE_CASE_ID, C_SOURCE_UNIQUE_ID, C_ACTION_TYPE, C_ACTION_STA (OPEN/CLOSED), C_ACTION_STAGE, C_OWNER, C_CASE_FINAL_LIABILITY, D_EXPIRY_DUE, D_RESPONSE_DUE | COMP-16 BusinessRulesProcessor (read-only — rule evaluation), COMP-22 DisputeService (read-only — dispute summary), COMP-24 CaseActionService (read for update), COMP-27 CaseSearchService (read-only), COMP-43 CoreNotificationConsumer (read-only — enrichment) | ✅ Confirmed from COMP-15, COMP-23, COMP-24 analysis |
| `wdp.NOTES` | COMP-25 NotesService (primary — all non-NAP platform notes via wdpTransactionManager); COMP-24 CaseActionService (conditional — when note field present in request, same transaction as action write) | Dispute case notes for PIN/CORE/VAP/LATAM platforms. Immutable audit trail. Key cols: I_NOTE_ID (PK, seq), I_CASE, I_ACTION_SEQ, C_NOTE_TYPE, T_NOTE, D_NOTE, X_INSRT (userId), Z_INSRT, X_INSRT_DISPLAY, X_UPDT_DISPLAY | ⚠️ Downstream readers TBC | ✅ Confirmed from COMP-24, COMP-25 analysis |
| `wdp.disputes_questionnaire` | COMP-26 QuestionnaireService (primary — all questionnaire creates and updates); COMP-15 EvidenceConsumer (upserts document names on RESPDOC WDP path) | Dispute-response questionnaire store. One row per (caseNumber, actionSeq). Full questionnaire serialised as JSON blob in C_QSTNNAIR. Key cols: I_QSTNNAIR_ID (PK, auto-increment seq), I_CASE, I_ACTION_SEQ, C_CASE_STAGE, C_QSTNNAIR (JSON blob), N_DOCUMENT_NAME, userId, createdAt, updatedAt. ⚠️ SHARED TABLE — no confirmed DB unique constraint on (I_CASE, I_ACTION_SEQ); duplicate POST by COMP-26 inserts new row rather than updating. | ⚠️ Downstream readers TBC | ✅ Confirmed from COMP-15, COMP-26 analysis |
| `wdp.case_expiry` | COMP-17 CaseExpiryUpdateConsumer | Active expiry schedule per case+action. Upserted on non-CLOSED action status; deleted on CLOSED. Key cols: i_case (caseNumber), i_action_seq, c_acq_platform, d_expiry_due, d_response_due, i_retry_count, c_workflow_name, z_insr, z_updt. Each write is its own @Transactional JPA transaction, independent of the outbox write — non-atomic split risk. | ⚠️ Downstream consumers of this table not yet confirmed — Copilot follow-up needed on COMP-17 repo | ✅ Confirmed from COMP-17 source |
| `wdp.outgoing_event_outbox` | COMP-17 CaseExpiryUpdateConsumer (channel_type=EXPIRY_EVENTS, created_by=WCSEEXPC); COMP-18 NotificationOrchestrator (channel_type=notification orchestration rows, component=NOTIFICATION_ORCHESTRATOR); COMP-43 CoreNotificationConsumer (channel_type=CORE_EVENTS) | Consumer-side audit, idempotency, predecessor blocking, and retry management. Shared across three consumers via channel_type discriminator. Key cols: id (seq PK), i_case, i_action_seq, channel_type (EXPIRY_EVENTS/CORE_EVENTS/others), idempotency_id, event_timestamp, status (PUBLISHED/FAILED/ERROR/PENDING_DEFERRED), retry_count, next_retry_at, error_code, error_message, original_event (JSON), created_by, created_at, updated_at | COMP-12 InboundDisputeEventScheduler (Scheduler3 reads FAILED and PENDING_DEFERRED rows for retry — does NOT read PUBLISHED or PENDING rows) | ✅ Confirmed from COMP-17, COMP-18, COMP-43 analysis |
| `wdp.bre_orchestration_outbox` | COMP-18 NotificationOrchestrator (writes component=NOTIFICATION_ORCHESTRATOR rows — idempotency, routing state, retry tracking); COMP-12 InboundDisputeEventScheduler (writes component=BUSINESS_RULES rows via Scheduler4) | Outbox for BRE and notification orchestration events. component discriminator field determines routing. Key cols: id, i_case, i_action_seq, component (NOTIFICATION_ORCHESTRATOR / BUSINESS_RULES), status (PUBLISHED/FAILED/ERROR/PENDING_DEFERRED/SUCCESS), retry_count, next_retry_at, idempotency_id, target_action, published_action, event_time_stamp, original_event (JSON). ⚠️ PUBLISHED-status rows have no automatic re-drive path — manual intervention required. | COMP-12 InboundDisputeEventScheduler (Scheduler4 reads FAILED and PENDING_DEFERRED rows only) | ✅ Confirmed from COMP-18 source |
| `wdp.file_generation_event` | COMP-18 NotificationOrchestrator | Staging table for file-based output requests. Written when Filter 4 routing matches. One row per document name. Key cols: idempotency_id, i_case, i_action_seq, c_case_ntwk, c_acq_platform, c_ntwk_case_id, document_name, file_type (AMEX/AMEX_HYBRID/DISCOVER/DISCOVER_RMO/BJS_PLCC/ISSUER_DOCS), status (hardcoded STAGED). ⚠️ Previously documented as wdp.file_notifications — that table name does not exist in the codebase. | COMP-45 CapitalOneResponseFileProcessor, COMP-46 NetworkResponseFileProcessor, COMP-47 DialoguIssuerDocumentProcessor (read to generate outbound files — not yet documented) | ✅ Confirmed from COMP-18 source. Table name corrected from file_notifications. |
| `wdp.br_case_audit_log` | COMP-16 BusinessRulesProcessor | Audit log of business rules evaluated (matched and not matched) for each US-path (CORE/VAP/PIN) message. Written on every message regardless of rule match outcome. Standalone JPA transaction. Key cols: id (seq), i_case, c_action_seq, rule_grp_name, rule_id, rule_name, is_valid, created_at | COMP-31 BusinessRulesService (read-only — GET /audit-log/case/{caseNumber} endpoint) | ✅ Confirmed from COMP-16, COMP-31 analysis |
| `wdp.display_codes` | DBA scripts / database migrations (not written at runtime by any WDP service) | Reference data — maps internal code values to human-readable short and long descriptions by code domain type and platform. ~40 code domain types. Key cols: i_display_code (PK, seq), c_type (domain), c_code, c_short_desc, c_long_desc, c_platform. Cached in-memory by COMP-28 at JVM startup (no TTL, no eviction). | COMP-28 DisplayCodeService (read-only at runtime); COMP-31 BusinessRulesService (read-only — criteria enrichment on GET /criteria) | ✅ Confirmed from COMP-28 analysis |
| `wdp.dispute_static_tabs_rules` | DBA scripts / database migrations | UI tab permission rules — determines which portal tabs are visible based on user role. Read by COMP-28 on POST /search to resolve userPermission object in response. | COMP-28 DisplayCodeService (read-only) | ✅ Confirmed from COMP-28 analysis |
| `wdp.dispute_event_change_log` | COMP-23 CaseManagementService (written in separate wdpTransactionManager call within NAP create path — non-atomic with napTransactionManager case save) | Audit/event-change log written by COMP-23 during NAP case creation path. ⚠️ Non-atomic with NAP case save — if change-log write fails after case save succeeds, audit row is missing. ⚠️ COMP-24 CaseActionService entity maps this table (EventChangeLogRepository declared) but repository is NEVER injected or called at runtime — dormant. | None confirmed | ✅ Confirmed from COMP-23, COMP-24 analysis |
| `wdp.rules` | COMP-31 BusinessRulesService | Core rule records for US platform (PIN/CORE/VAP/LATAM). Mirror of nap.rules in the wdp schema. Key cols: id (PK, seq), group_id, group_name, name, type, description, sort_order, is_enabled, criteria_summary, action_summary, created_by, created_at, updated_by, updated_at | COMP-16 BusinessRulesProcessor (direct JPA read — bypasses BusinessRulesService REST entirely) | ✅ Confirmed from COMP-31, COMP-16 analysis |
| `wdp.rule_criterion` | COMP-31 BusinessRulesService | Per-rule criterion instances for US platform. OneToMany from wdp.rules. CascadeType.ALL. Key cols: id, rule_id (FK), type, category, name, value, operator_symbol, created_at | COMP-16 BusinessRulesProcessor (direct JPA read, eager-loaded with wdp.rules) | ✅ Confirmed from COMP-31, COMP-16 analysis |
| `wdp.rule_action` | COMP-31 BusinessRulesService | Per-rule action instances for US platform. OneToMany from wdp.rules. CascadeType.ALL. Key cols: id, rule_id (FK), category, action_name, field_name1/2/3, field_value1/2/3, created_at | COMP-16 BusinessRulesProcessor (direct JPA read, eager-loaded with wdp.rules) | ✅ Confirmed from COMP-31, COMP-16 analysis |
| `wdp.rule_group` | ⚠️ Owner TBC — not determinable from COMP-31 source (possibly COMP-32 RulesService or DBA scripts) | Rule group definitions for US platform. Reference/configuration data. Key cols: id, name, description, triggered_by, type, is_enabled | COMP-31 BusinessRulesService (read-only — validates group on rule create, returns group list on GET /rule-group) | ⚠️ PENDING — confirm write owner |
| `wdp.FAX_ACTION` | COMP-29 FaxQueueService | Audit records of all fax operations (claim, reject, discard) performed by operations staff via Ops Portal. Written to WDP PostgreSQL by FaxQueueService when operating on legacy MS SQL Server fax data. | None confirmed | ✅ Confirmed from COMP-29 analysis |
| `wdp.acl` | COMP-02 UAMS | Access control list linking consumers to source systems and entity values. For downstream consumer authorization — not used by UAMS for its own auth. Key cols: i_acl_id (PK), c_consumer_name, c_source_system, c_entity_class, c_entity_type, c_entity_value, c_status, c_created_by, t_created_timestamp, c_updated_by, t_updated_timestamp. Index: acl_search_index on (c_consumer_name, c_status) | ⚠️ Downstream consumers TBC | ⚠️ PENDING — confirm readers |
| `wdp.api_route` | ⚠️ Owner TBC | API Gateway URL routing configuration. 26+ routes loaded at pod startup. Key cols: id, path, original_path_regex, replacement_path, uri, auth_exceptions | COMP-01 API Gateway (read-only at startup) | ⚠️ PENDING — confirm write owner |

---

### Schema: `nap`

*NAP acquiring platform schema. Managed by WDP team for NAP integration.*

| Table | Owning Component(s) | Purpose | Key Columns | Accessed Read-Only By | Confirmed |
|-------|---------------------|---------|-------------|----------------------|-----------|
| `nap.nap_parent_entity` | COMP-02 UAMS | Top-level merchant groupings for NAP platform | entity_id, name, status | ⚠️ TBC | ⚠️ PENDING |
| `nap.nap_child_entity` | COMP-02 UAMS | Sub-groupings beneath a NAP parent entity | child_id, parent_id, status | ⚠️ TBC | ⚠️ PENDING |
| `nap.nap_merchant` | COMP-02 UAMS | Individual merchant IDs with MCC and WPG ID | mid, mcc, wpg_id, parent_id | ⚠️ TBC | ⚠️ PENDING |
| `nap.nap_entity_rel` | COMP-02 UAMS | Merchant-to-entity relationships — primary NAP authorization lookup | merchant_id, parent_entity, child_entities | COMP-02 UAMS /authorize endpoint | ✅ Confirmed from COMP-02 analysis |
| `nap.case` | COMP-23 CaseManagementService (primary — all NAP case create/update via napTransactionManager); COMP-24 CaseActionService (NAP case open/close transitions, same transaction as nap.action write) | NAP/UK platform case record. Mirror of wdp.CASE in the nap schema. Separate napTransactionManager — never in same transaction as wdp schema writes. Key cols: I_CASE (PK), C_CASE_STA, C_CASE_FINAL_LIABILITY, I_CASE_ACTION_MAX_SEQ, Z_UPDT, I_ACCI_CDH (⚠️ clear PAN written by COMP-23 on standard create — DEC-004 violation) | COMP-02 UAMS (read-only — caseNumber→merchantId), COMP-03 CHAS (read-only — caseNumber→chainId+merchantId), COMP-05 NAPDisputeEventProcessor (read-only), COMP-06 NAPDisputeDeclineBatch (read-only), COMP-16 BusinessRulesProcessor (read-only — UK rule evaluation) | ✅ Confirmed from COMP-23, COMP-24 analysis |
| `nap.action` | COMP-23 CaseManagementService (inserts new actions, cascaded via UKCaseEntity one-to-many); COMP-24 CaseActionService (inserts and updates NAP action status, owner, liability) | NAP/UK platform action records. Mirror of wdp.ACTION in the nap schema. Same napTransactionManager transaction as nap.case write. | C_CASE_STAGE, C_ACTION_TYPE, C_ACTION_STA, C_MIGRATION_STA, C_OWNER | COMP-05 NAPDisputeEventProcessor (read-only), COMP-06 NAPDisputeDeclineBatch (read-only — filtered query for open PAB actions), COMP-16 BusinessRulesProcessor (read-only — UK rule evaluation, eager-loaded with nap.case) | ✅ Confirmed from COMP-23, COMP-24 analysis |
| `nap.NOTES` | COMP-25 NotesService | Dispute case notes for NAP/UK platform. Separate napTransactionManager — never mixed with wdp schema. Identical column structure to wdp.NOTES. Key cols: I_NOTE_ID (PK, seq), I_CASE, I_ACTION_SEQ, C_NOTE_TYPE, T_NOTE, D_NOTE, X_INSRT, Z_INSRT, X_INSRT_DISPLAY, X_UPDT_DISPLAY | ⚠️ Readers TBC | ✅ Confirmed from COMP-25 analysis |
| `nap.br_case_audit_log` | COMP-16 BusinessRulesProcessor | Audit log of business rules evaluated (matched and not matched) for each UK/NAP path message. Written on every message regardless of rule match outcome. Standalone JPA transaction. Key cols: id (seq), i_case, c_action_seq, rule_grp_name, rule_id, rule_name, is_valid, created_at | COMP-31 BusinessRulesService (read-only — GET /audit-log/case/{caseNumber} endpoint) | ✅ Confirmed from COMP-16, COMP-31 analysis |
| `nap.rules` | COMP-31 BusinessRulesService | Core rule records for NAP/UK platform. Key cols: id (PK, seq), group_id, group_name, name, type, description, sort_order, is_enabled, criteria_summary, action_summary, created_by, created_at, updated_by, updated_at | COMP-16 BusinessRulesProcessor (direct JPA read — bypasses BusinessRulesService REST entirely at execution time) | ✅ Confirmed from COMP-31, COMP-16 analysis |
| `nap.rule_criterion` | COMP-31 BusinessRulesService | Per-rule criterion instances for NAP platform. OneToMany from nap.rules. CascadeType.ALL, LAZY. Key cols: id, rule_id (FK → nap.rules.id), type, category, name, value, operator_symbol, operator_readable, description, created_at | COMP-16 BusinessRulesProcessor (direct JPA read, eager-loaded with nap.rules at execution) | ✅ Confirmed from COMP-31, COMP-16 analysis |
| `nap.rule_action` | COMP-31 BusinessRulesService | Per-rule action instances for NAP platform. OneToMany from nap.rules. CascadeType.ALL, LAZY. Key cols: id, rule_id (FK), category, action_name, field_name1/2/3, field_value1/2/3, field_value1/2/3_description, created_at | COMP-16 BusinessRulesProcessor (direct JPA read, eager-loaded with nap.rules at execution) | ✅ Confirmed from COMP-31, COMP-16 analysis |
| `nap.rule_group` | ⚠️ Owner TBC — not determinable from COMP-31 source (possibly COMP-32 RulesService or DBA scripts) | Rule group definitions for NAP platform. Reference/configuration data. Key cols: id, name, description, triggered_by, type, is_enabled, c_display_group_name | COMP-31 BusinessRulesService (read-only — validates group on rule create, returns group list on GET /rule-group) | ⚠️ PENDING — confirm write owner |
| `nap.rule_criteria` | ⚠️ Owner TBC — NOT determinable from COMP-31 source | Reference/lookup table for available criterion definitions. Distinct from nap.rule_criterion (which holds per-rule instances). Read only by COMP-31 for GET /criteria endpoint. | id, category, type, criteria definition fields | COMP-31 BusinessRulesService (read-only) | ⚠️ PENDING — confirm write owner |
| `nap.rule_action_field` | ⚠️ Owner TBC — NOT determinable from COMP-31 source | Reference/lookup table for available action field definitions. Read only by COMP-31 for GET /actions endpoint. | id, category, action_name, action_display_name, field_name, field_display_name, type, is_required | COMP-31 BusinessRulesService (read-only) | ⚠️ PENDING — confirm write owner |
| `NAP.DISPUTE_EVENT_CONSUMER_ERROR` | COMP-05 NAPDisputeEventProcessor | Database DLQ for failed Kafka events on nap-dispute-events — raw payload stored for manual reprocessing | error_id, raw_payload, arn, error_timestamp | ⚠️ Manual reprocessing path only | ✅ Confirmed from COMP-05 analysis |

---

### Spring Batch Metadata Tables

*Written by all Spring Batch jobs. Schema and prefix vary by component — injected via K8s secrets.*

| Table | Written by | Purpose |
|-------|------------|---------|
| `BATCH_JOB_INSTANCE` | COMP-06, COMP-07, COMP-08, COMP-09 | Job identity and deduplication |
| `BATCH_JOB_EXECUTION` | COMP-06, COMP-07, COMP-08, COMP-09 | Execution status per run |
| `BATCH_STEP_EXECUTION` | COMP-06, COMP-07, COMP-08, COMP-09 | Step-level progress and counts |

⚠️ Schema location (which PostgreSQL database and schema) for Spring Batch
metadata tables varies per component and must be confirmed from each
component's Copilot CLI analysis. Table prefix injected via K8s secret.

---

## 3. External Database Dependencies

### IBM DB2 — Core Platform (BC schema) — WDP Writes

COMP-43 CoreNotificationConsumer is the **sole WDP component that writes to IBM DB2**. All other WDP components that access DB2 are read-only.

| Table | WDP Writer | Purpose | Key Columns | Notes |
|-------|-----------|---------|-------------|-------|
| `BC.TBC_DM_CASE` | COMP-43 CoreNotificationConsumer | Parent dispute case record for CORE platform. One row per WDP case. | I_CASE_ID (IDENTITY PK generated by DB2), I_CASE (patched = leftPad(I_CASE_ID, 10, "9")), c_wdp_case (WDP case number backlink) | Two-phase INSERT: first save generates I_CASE_ID; second save patches I_CASE. DB2 reads use WITH UR (uncommitted read isolation). Written on CREATE events; UPDATE on UPDATE events. |
| `BC.TBC_DM_OCCUR` | COMP-43 CoreNotificationConsumer | Action/occurrence records as children of TBC_DM_CASE | I_OCCUR_ID (IDENTITY PK), I_CASE_ID (FK), I_CASE_OCCUR (occurrence number / actionSequence) | Same @Transactional(coreTransactionManager) as TBC_DM_CASE. INSERT on CREATE, UPDATE on UPDATE. |
| `BC.TBC_DM_NOTES` | COMP-43 CoreNotificationConsumer | First note for a case/occurrence. Written only on CREATE + actionSequence=01 when notes list non-empty. | I_NOTE_ID (IDENTITY PK), I_CASE_ID (FK), I_OCCUR_ID | Same transaction as TBC_DM_CASE and TBC_DM_OCCUR. INSERT only. |

### IBM DB2 — Core Platform — Read-Only Access

| Table(s) | Accessed by | Purpose |
|----------|------------|---------|
| MD.TMD_ENTY, MD.TMD_ENTY_REL, DC.TDC_VIQ_ORG_ENTY, BC.TBC_MRCHNT_MAST_BO, MD.TMD_DISPLAY_CODES, MD.TMD_MRCHNT_WOMPLY, BC.TMD_CHAIN, and others | COMP-03 CHAS | Entity scope validation and Core hierarchy data |
| BC.TBC_DM_CASE, BC.TBC_DM_OCCUR | COMP-15 EvidenceConsumer (V3 path only — WITH UR isolation) | V3 legacy CORE path case lookup |
| BC.TBC_DM_CASE, BC.TBC_DM_OCCUR | COMP-22 DisputeService (when coreMigrationStatus=false) | Legacy CORE dispute summary aggregation |
| BC.TBC_CC_TR07, BC.TBC_MRCHNT_MAST_BO | COMP-34 MerchantTransactionService (CapOne enrichment only) | CapOne transaction and merchant lookup |
| BC.TBC_DM_CASE, BC.TBC_DM_OCCUR, BC.TBC_DM_NOTES | COMP-43 CoreNotificationConsumer (also writer — see above) | Case/occurrence/notes lookup for enrichment before DB2 write |

### MS SQL Server — Legacy Fax System

| Table | Accessed by | Access type | Purpose |
|-------|------------|-------------|---------|
| `dbo.IncomingFaxes` | COMP-29 FaxQueueService | Read + conditional status update | Legacy fax queue. COMP-29 reads fax records, updates their status on claim/reject/discard. Shared with legacy fax processing systems outside WDP. Not WDP-owned. |

---

## 4. Shared Table Risk Register

Tables written by more than one component require coordination to avoid
race conditions, duplicate writes, or conflicting state.

| Table | Writers | Risk | Mitigation in place | Severity |
|-------|---------|------|---------------------|----------|
| `wdp.chbk_outbox_row` | COMP-07, COMP-08, COMP-09, COMP-11 (INSERT PENDING rows); COMP-12, COMP-14, COMP-15, COMP-23 (status updates) | Multiple components write simultaneously. Batch jobs INSERT rows; consumers update status of pre-existing rows. | Per-component duplicate check on (networkCaseId + networkPhaseId + disputeStage) before INSERT. Status updates operate on pre-existing rows by PK — no insert race. No DB unique constraint confirmed. | 🟡 MEDIUM |
| `wdp.file_job` | COMP-11 FileProcessor (creates + updates status), COMP-12 (status→COMPLETED), COMP-13 (ack fields only) | Three components write to same row at different lifecycle stages. | Non-overlapping field sets — COMP-11 owns status fields, COMP-12 owns completed_at, COMP-13 owns ack_status fields. | 🟢 LOW — field-level separation |
| `wdp.CASE` | COMP-23 CaseManagementService (primary), COMP-24 CaseActionService (open/close transitions), COMP-15 EvidenceConsumer (Z_UPDT conditional) | Multiple services update the central case record. | COMP-23 uses case-level transactions. COMP-24 and COMP-15 updates are conditional and scoped. No optimistic locking confirmed. | 🟠 MEDIUM-HIGH — confirm locking strategy |
| `wdp.ACTION` | COMP-23 CaseManagementService (INSERT + UPDATE), COMP-24 CaseActionService (INSERT + UPDATE), COMP-15 EvidenceConsumer (UPDATE C_OWNR conditional) | Multiple services write action records. No RBAC enforcement in COMP-24 — any authenticated caller can modify any action. | No write-lock or optimistic lock confirmed from source. | 🔴 HIGH — no RBAC + concurrent write risk |
| `wdp.disputes_questionnaire` | COMP-26 QuestionnaireService (primary), COMP-15 EvidenceConsumer (upsert document names) | Two components write to same table. COMP-26 has a POST idempotency gap — duplicate POSTs insert new rows rather than updating. | No DB unique constraint confirmed on (I_CASE, I_ACTION_SEQ). | 🔴 HIGH — duplicate row risk on COMP-26 POST |
| `wdp.outgoing_event_outbox` | COMP-17 (EXPIRY_EVENTS), COMP-18 (notification orchestration), COMP-43 (CORE_EVENTS) | Three consumers write to same table using channel_type as discriminator. | channel_type column discriminates between component domains. Each component only reads/writes its own channel_type rows. | 🟢 LOW — channel_type provides logical isolation |
| `nap.case` | COMP-23 CaseManagementService, COMP-24 CaseActionService | Same shared-write risk as wdp.CASE, on the NAP schema. | Same mitigation pattern as wdp.CASE. | 🟠 MEDIUM-HIGH |
| `nap.action` | COMP-23 CaseManagementService, COMP-24 CaseActionService | Same shared-write risk as wdp.ACTION, on the NAP schema. | Same pattern. | 🔴 HIGH — same RBAC absence |
| `nap.nap_child_entity` | COMP-02 UAMS (createEntity + saveChildWithMerchant) | saveChildWithMerchant uses wrong transaction manager (wdpTransactionManager instead of napTransactionManager) — rollback on failure may not cover nap schema writes | None — confirmed bug | 🔴 HIGH |

---

## 5. Data Retention and Compliance Notes

| Data type | Retention requirement | Enforced by | Reference |
|-----------|----------------------|-------------|-----------|
| Dispute case audit trail | 7 years — immutable | Core WDP platform | PCI-DSS compliance |
| PAN data | Never stored in plaintext — ⚠️ DEC-004 violation confirmed in COMP-23 (clear PAN on standard case create) | COMP-35 EncryptionService (enrichment path); gap on standard create path | DEC-004, DEC-007 |
| HPAN (hashed PAN) | Retained with case data | Core WDP platform | DEC-007 |
| EPAN (encrypted PAN) | Retained in EPAN→HPAN mapping table | COMP-35 EncryptionService | DEC-007 |
| Evidence documents | ⚠️ Retention policy to be confirmed | COMP-37 DocumentManagementService (S3) | TBC |
| API audit logs | ⚠️ Retention policy to be confirmed | COMP-38 APILogService | TBC |
| Questionnaire data | ⚠️ Retention policy to be confirmed | COMP-26 QuestionnaireService | TBC |
| Outbox rows (chbk_outbox_row) | SUCCESS rows archived by COMP-12 Scheduler2 after 30 days. Archive table (chbk_outbox_row_archive) has no confirmed purge policy. | COMP-12 InboundDisputeEventScheduler | Operational |
| Spring Batch metadata | ⚠️ Retention policy to be confirmed | Per batch component | TBC |

---

*Last updated: April 2026*
*Version 2.0 — fully populated from all 40 component DRAFT files*
