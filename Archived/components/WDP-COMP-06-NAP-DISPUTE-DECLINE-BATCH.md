# WDP-COMP-06-NAP-DISPUTE-DECLINE-BATCH
**Worldpay Dispute Platform — Component Reference**
*Version: 1.0 DRAFT | April 2026*
*Extracted from: WDP-COMPONENTS.md (Copilot CLI confirmed content) |
Architect-confirmed: PENDING*

---

## ━━━ CORE SKELETON ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## Identity

| Field             | Value |
|-------------------|-------|
| **Name**          | `NAPDisputeDeclineBatch` |
| **Type**          | `Batch/Scheduler` |
| **Repository**    | `gcp-visa-issuer-decline-batch` |
| **Maven artifact**| `visa-issuer-decline-batch v1.1.1` |
| **Technology**    | `Java 17 · Spring Boot 3.5.7 · Spring Batch` |
| **Status**        | `✅ Production` |
| **Doc status**    | `📝 DRAFT` |
| **Sections present** | `Core · Block D (Batch)` |

> ⚠️ DECOMMISSION-SCOPED — This component sits on the NAP/WPG inbound
> path, which is planned for decommission. No new development or design
> work is planned. All gaps below are ACCEPTED GAPS.

---

## Purpose

**What it does**

NAPDisputeDeclineBatch is a scheduled Spring Batch job that advances
pre-arbitration declined dispute cases on the NAP platform. It operates
entirely within the Visa card network and the NAP acquiring platform —
no other card network or acquiring platform is in scope.

The job polls the `nap.action` database table for open Pre-Arbitration
(PAB) action records that have been migrated. For each qualifying record
it calls the internal Case Management service to confirm the dispute is
a Visa case, then calls the internal Visa Adapter HyperSearch endpoint
to obtain the current Visa network case state. If the Visa network reports
that the case is in one of two specific Pre-Arb Declined states, the job
creates a new Issuer Decline (IDCL) draft action via the Case Actions
service to advance the dispute workflow. All cases that do not match the
required state are silently skipped.

Visa network access is mediated entirely through the internal Visa Adapter
service. The job does not call the Visa API directly.

**What it does NOT do**

- Does not consume from, or publish to, any Kafka topic
- Does not write to, or update, the `nap.action` source rows after processing
- Does not handle any card network other than Visa
- Does not process disputes on any acquiring platform other than NAP
- Does not perform PAN encryption (NAP dispute data contains no full PAN —
  confirmed architectural fact; no EncryptionService call is made)
- Does not write to S3, any file system path, or any outbox table
- Does not follow the standard transactional outbox pattern (DEC-001) —
  this is a deliberate deviation; see Key Architectural Decisions below

---

## Internal Processing Flow

```mermaid
flowchart TD
    TRIGGER["@Scheduled cron fires\nonce per configured interval"]
    TOKEN["IDP Token Service GET\nObtain bearer token once per job run"]
    READ["Read nap.action\nCursor-paginated query\nC_CASE_STAGE=PAB · C_ACTION_TYPE=OPAB\nC_ACTION_STA=OPEN · C_MIGRATION_STA=Y\nZ_INSRT > now minus past_days window"]
    FOREACH["For each qualifying action record"]
    CMGET["Case Management GET\nLook up case by networkCaseId\nVerify cardNetwork field"]
    VISA_CHECK{{"cardNetwork\n= VISA?"}}
    SKIP_NETWORK["Skip record\n(non-VISA case)"]
    VAHYPER["Visa Adapter HyperSearch POST\nQuery Visa network case state\nReturns stageStateDesc\ndisputePreArbResponseId\ndeadlines · amounts"]
    STATE_CHECK{{"stageStateDesc =\nPre-Arb Declined?"}}\
    SKIP_STATE["Skip record\n(state not actionable)"]
    RESP_CHECK{{"disputePreArbResponseId\npresent?"}}\
    SKIP_RESP["Skip record\n(response ID absent)"]
    CAGET["Case Actions GET\nRetrieve existing action\nfinancial details to copy"]
    CAPOST["Case Actions POST\nCreate new IDCL draft action\nstageCode=PAB · actionStatus=DRAFT\nuserId=NPRBDECLB\nfinancial amounts copied from existing action"]
    BATCH_META["Write Spring Batch metadata\nBATCH_JOB_INSTANCE\nBATCH_JOB_EXECUTION\nBATCH_STEP_EXECUTION"]
    LOG["Structured log\nshipped to Logstash / ELK via TCP"]
    DONE["Job run complete"]

    TRIGGER --> TOKEN
    TOKEN --> READ
    READ --> FOREACH
    FOREACH --> CMGET
    CMGET --> VISA_CHECK
    VISA_CHECK -->|"No"| SKIP_NETWORK
    VISA_CHECK -->|"Yes"| VAHYPER
    VAHYPER --> STATE_CHECK
    STATE_CHECK -->|"No match"| SKIP_STATE
    STATE_CHECK -->|"Fraud Dispute - Pre-Arb Declined\nor Authorization Dispute - Pre-Arb Declined"| RESP_CHECK
    RESP_CHECK -->|"Absent"| SKIP_RESP
    RESP_CHECK -->|"Present"| CAGET
    CAGET --> CAPOST
    CAPOST --> FOREACH
    SKIP_NETWORK --> FOREACH
    SKIP_STATE --> FOREACH
    SKIP_RESP --> FOREACH
    FOREACH -->|"All records processed"| BATCH_META
    BATCH_META --> LOG
    LOG --> DONE
```

---

## Boundaries

### Inbound Interfaces

| Source | Protocol | Endpoint / Topic / Trigger | Payload / Description |
|--------|----------|----------------------------|-----------------------|
| Kubernetes internal scheduler | `@Scheduled cron` | ⚠️ NOT DOCUMENTED — cron expression not confirmed | Job trigger — no inbound payload |
| `nap.action` table | `PostgreSQL DB poll` | Filtered cursor query — see Block D | Open PAB action records for NAP migrated Visa cases |

### Outbound Interfaces

| Target | Protocol | Endpoint / Topic / Resource | Purpose | On failure |
|--------|-----------|-----------------------------|---------|------------|
| IDP Token Service | REST GET | ⚠️ NOT DOCUMENTED | Obtain bearer JWT once per job run | ⚠️ NOT DOCUMENTED |
| Case Management Service | REST GET | ⚠️ NOT DOCUMENTED | Verify `cardNetwork = VISA` for each action record | Skip record (non-VISA) |
| Visa Adapter HyperSearch | REST POST | ⚠️ NOT DOCUMENTED | Query Visa network case state — returns `stageStateDesc`, `disputePreArbResponseId`, deadlines, amounts | ⚠️ NOT DOCUMENTED |
| Case Actions Service | REST GET | ⚠️ NOT DOCUMENTED | Retrieve existing action financial details to copy to new action | ⚠️ NOT DOCUMENTED |
| Case Actions Service | REST POST | ⚠️ NOT DOCUMENTED | Create new `IDCL` draft action — `stageCode=PAB`, `actionStatus=DRAFT`, `userId=NPRBDECLB` | ⚠️ NOT DOCUMENTED |
| Spring Batch metadata (WDP PostgreSQL) | PostgreSQL write | `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` | Job execution tracking | ⚠️ NOT DOCUMENTED |
| Logstash / ELK | TCP log appender | Logstash endpoint | Structured log shipping | ⚠️ NOT DOCUMENTED |

---

## Database Ownership

### Tables Owned (written by this component)

| Schema.Table | Purpose | Key columns | Retention / Notes |
|--------------|---------|-------------|-------------------|
| `⚠️ NOT DOCUMENTED`.`BATCH_JOB_INSTANCE` | Spring Batch job identity and deduplication | job_instance_id, job_name, job_key | Written to WDP PostgreSQL — separate datasource from NAP PostgreSQL. Schema name not confirmed. |
| `⚠️ NOT DOCUMENTED`.`BATCH_JOB_EXECUTION` | Spring Batch execution status per run | job_execution_id, status, start_time, end_time | As above |
| `⚠️ NOT DOCUMENTED`.`BATCH_STEP_EXECUTION` | Spring Batch step-level progress and counts | step_execution_id, step_name, read_count, write_count | As above |

### Tables Read (not owned by this component)

| Schema.Table | Owned by | Why accessed |
|--------------|----------|--------------|
| `nap.action` | ⚠️ Case action services (write path TBC — not COMP-06) | Polled for open PAB action records matching migration and status filter conditions |
| `nap.case` | ⚠️ Core WDP case management (owning component TBC) | Accessed indirectly via Case Management Service REST call — not a direct DB read by this component |

---

## Configuration and Scaling

| Parameter | Value | Notes |
|-----------|-------|-------|
| Replica count | ⚠️ NOT DOCUMENTED | |
| HPA | ⚠️ NOT DOCUMENTED | |
| Memory request | ⚠️ NOT DOCUMENTED | |
| Memory limit | ⚠️ NOT DOCUMENTED | |
| CPU request | ⚠️ NOT DOCUMENTED | |
| CPU limit | ⚠️ NOT DOCUMENTED | |
| Deployment type | `Kubernetes Deployment` | Internal `@Scheduled` cron — not a Kubernetes CronJob |
| Rollout strategy | ⚠️ NOT DOCUMENTED | |
| PodDisruptionBudget | ⚠️ NOT DOCUMENTED | |
| Database connection pool | ⚠️ NOT DOCUMENTED | Two datasources — NAP PostgreSQL (read) and WDP PostgreSQL (Spring Batch metadata write) |
| Observability | Logstash / ELK via TCP appender | Structured log shipping confirmed |

**Component-specific parameters**

| Parameter | Config key | Value |
|-----------|------------|-------|
| Look-back window | `${past_days}` | ⚠️ NOT DOCUMENTED — injected from environment |
| Page size | `${number_of_records}` | ⚠️ NOT DOCUMENTED — injected from environment |
| Cron expression | ⚠️ NOT DOCUMENTED | ⚠️ NOT DOCUMENTED |

---

## Key Architectural Decisions

| Decision | ADR reference | Notes |
|----------|---------------|-------|
| No transactional outbox — direct REST call to Case Actions | DEC-001 — DEVIATION | IDCL action is created via direct REST POST. No outbox write. Acceptable for a decommission-scoped component. |
| No PAN encryption at ingestion boundary | DEC-004 — DEVIATION | NAP dispute data contains no full PAN. EncryptionService is not called. Confirmed architectural fact. |
| No Kafka involvement — no partitioning by merchant | DEC-003 — NOT APPLICABLE | This component has no Kafka surface. |
| No Kafka offset commit | DEC-005 — NOT APPLICABLE | This component has no Kafka consumer. |
| No circuit breakers active | DEC-014 — DEVIATION | Resilience4j referenced in the platform but confirmed not active in this component. Downstream REST failures have no circuit-breaker protection. |
| Visa-only, NAP-only scope | Local decision | All non-Visa cases are skipped immediately after Case Management GET. No other card network or platform is served. |
| Runs as Kubernetes Deployment with internal @Scheduled cron | Local decision | Not a Kubernetes CronJob. Scheduling is managed within the Spring application. |
| Does not update nap.action rows after processing | Local decision | Source records are left unchanged. Re-query on the next run will re-evaluate the same records unless their status changes through another path. |

---

## Risks and Constraints

| Severity | Risk | Consequence |
|----------|------|-------------|
| 🟡 MEDIUM | No circuit breaker on Visa Adapter HyperSearch calls. If the Visa Adapter becomes slow or unavailable, every record in the batch will block until timeout before skipping or failing. | Job run duration degrades significantly under Visa Adapter instability; may breach scheduling interval. |
| 🟡 MEDIUM | nap.action records are not marked as processed after a successful IDCL creation. If the downstream Case Actions POST succeeds but the job re-runs before the action status propagates back, duplicate IDCL draft actions may be created for the same case. | Duplicate draft actions on a single dispute case — requires manual remediation. |
| 🟡 MEDIUM | Two separate PostgreSQL datasources (NAP read, WDP Spring Batch write) in a single job. Failure behaviour across datasource boundaries is not documented — it is unclear whether a failure to write Spring Batch metadata after a successful IDCL creation is recoverable. | Inconsistency between Spring Batch execution records and actual case state. |
| 🟢 LOW | Cron schedule, look-back window, and page size are all injected from environment configuration. These are not documented in the knowledge base. Misconfiguration (e.g. excessively large look-back window) could cause unexpectedly large batch volumes. | Throughput spike; downstream REST services under unexpected load. |
| 🟢 LOW | Component is decommission-scoped. It will be removed when the NAP/WPG inbound path is decommissioned. No active maintenance is planned. Any production defects discovered will require a scoping decision. | Latent bugs may remain unresolved until decommission. |

---

## Planned Changes

- ⚠️ Decommission — this component will be removed as part of the
  NAP/WPG inbound path decommission. Timeline: TBD. No decommission
  date has been confirmed as of April 2026.
- No other planned changes confirmed. Review at decommission planning.

---

---

## ━━━ TYPE BLOCK D — BATCH AND SCHEDULER CONTRACTS ━━━━━━━

---

## Batch and Scheduler Contracts

**Batch framework:** Spring Batch  
**Deployment type:** Kubernetes Deployment (internal `@Scheduled` cron)  
**Trigger mechanism:** Internal Spring `@Scheduled` cron  
**Job uniqueness:** ⚠️ NOT DOCUMENTED — Spring Batch job deduplication
strategy (JobParameters timestamp or equivalent) not confirmed

---

### Job: Visa Issuer Pre-Arbitration Decline Batch

**Purpose:** For each open PAB action on a migrated NAP Visa dispute,
query the Visa network and create an IDCL draft action if the case is in
a Pre-Arb Declined state.

**Schedule**

| Parameter | Config key | Value / Source |
|-----------|------------|----------------|
| Cron expression | ⚠️ NOT DOCUMENTED | ⚠️ NOT DOCUMENTED — injected from K8s environment |
| Look-back window | `${past_days}` | ⚠️ NOT DOCUMENTED — injected from K8s environment |
| Page size | `${number_of_records}` | ⚠️ NOT DOCUMENTED — injected from K8s environment |
| Timezone | ⚠️ NOT DOCUMENTED | ⚠️ NOT DOCUMENTED |

**Input source**

| Source | Type | Query / Filter | Pagination |
|--------|------|----------------|------------|
| `nap.action` (NAP PostgreSQL) | DB poll | `C_CASE_STAGE = 'PAB'` AND `C_ACTION_TYPE = 'OPAB'` AND `C_ACTION_STA = 'OPEN'` AND `C_MIGRATION_STA = 'Y'` AND `Z_INSRT > (now - ${past_days})` | Cursor-based — page size `${number_of_records}` |

**Processing steps**

| Step | Name | Description | Chunk size | On failure |
|------|------|-------------|------------|------------|
| 1 | Token bootstrap | IDP Token Service GET — obtain bearer JWT once for the entire job run | N/A (once per job) | ⚠️ NOT DOCUMENTED |
| 2 | Card network filter | Case Management GET — verify `cardNetwork = VISA`; skip record if not Visa | ⚠️ NOT DOCUMENTED | Skip record |
| 3 | Visa network lookup | Visa Adapter HyperSearch POST — retrieve `stageStateDesc`, `disputePreArbResponseId`, deadlines, amounts | ⚠️ NOT DOCUMENTED | ⚠️ NOT DOCUMENTED |
| 4 | State filter | Check `stageStateDesc` — only `"Fraud Dispute - Pre-Arb Declined"` or `"Authorization Dispute - Pre-Arb Declined"` proceed; all others skipped | N/A | Skip record |
| 5 | Response ID check | Verify `disputePreArbResponseId` is present; skip if absent | N/A | Skip record |
| 6 | Financial detail fetch | Case Actions GET — retrieve existing action financial details to copy into new action | ⚠️ NOT DOCUMENTED | ⚠️ NOT DOCUMENTED |
| 7 | IDCL action creation | Case Actions POST — create new `IDCL` draft action: `stageCode=PAB`, `actionStatus=DRAFT`, `userId=NPRBDECLB`, financial amounts copied from step 6 | ⚠️ NOT DOCUMENTED | ⚠️ NOT DOCUMENTED |

**Downstream calls per record**

Each qualifying record triggers up to four serial REST calls:

1. Case Management GET — verify `cardNetwork = VISA` (skip if not Visa)
2. Visa Adapter HyperSearch POST — query Visa network case state (skip if state not actionable or response ID absent)
3. Case Actions GET — retrieve financial details from existing action
4. Case Actions POST — create new IDCL draft action

Records filtered at steps 1 or 2 generate only one or two REST calls.
Records that reach step 4 generate four serial REST calls. No parallelism
within the record processing loop is documented.

**Outputs**

| Target | Type | What is written | On failure |
|--------|------|-----------------|------------|
| Case Actions Service | REST POST | New `IDCL` draft action — `stageCode=PAB`, `actionStatus=DRAFT`, `userId=NPRBDECLB` | ⚠️ NOT DOCUMENTED |
| Spring Batch metadata (WDP PostgreSQL) | DB write | `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` records | ⚠️ NOT DOCUMENTED |

**Failure and recovery**

⚠️ NOT DOCUMENTED — the following questions are ACCEPTED GAPS:

- Whether the job is safe to re-run (idempotency) — specifically, whether
  a duplicate IDCL draft action would be created if the job re-runs before
  the case action status has propagated back to `nap.action`
- Whether Spring Batch checkpointing allows resumption from the last
  committed chunk or requires a full restart
- Whether partial results (some records processed, job then fails) are
  committed or rolled back
- Where job failures are recorded beyond Spring Batch metadata
- Whether there is a manual reprocessing path

**Spring Batch metadata**

| Table | Schema | Purpose |
|-------|--------|---------|
| `BATCH_JOB_INSTANCE` | ⚠️ NOT DOCUMENTED | Job identity and deduplication |
| `BATCH_JOB_EXECUTION` | ⚠️ NOT DOCUMENTED | Execution status per run |
| `BATCH_STEP_EXECUTION` | ⚠️ NOT DOCUMENTED | Step-level progress and counts |

Written to WDP PostgreSQL. This is a **separate datasource** from the
NAP PostgreSQL database from which `nap.action` is read.

---

*End of WDP-COMP-06-NAP-DISPUTE-DECLINE-BATCH.md*
