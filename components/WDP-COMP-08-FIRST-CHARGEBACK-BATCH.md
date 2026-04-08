# WDP-COMP-08-FIRST-CHARGEBACK-BATCH
**Worldpay Dispute Platform — Component Reference**
*Version: 1.0 DRAFT | April 2026*
*Extracted from: wdp-mcm-first-chargeback-queue-batch using GitHub Copilot CLI | Architect-confirmed: PENDING*

---

## ━━━ CORE SKELETON ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
*Mandatory for every component regardless of type.*

---

## Identity

| Field             | Value                                                        |
|-------------------|--------------------------------------------------------------|
| **Name**          | `FirstChargebackBatch`                                       |
| **Type**          | `Batch/Scheduler`                                            |
| **Repository**    | `wdp-mcm-first-chargeback-queue-batch`                       |
| **Technology**    | Spring Boot 3.5.3 / Spring Batch / Java 17                   |
| **Owner**         | Integration Team                                             |
| **Status**        | ✅ Production                                                 |
| **Doc status**    | 📝 DRAFT                                                     |
| **Sections present** | `Core \| Block D`                                         |

---

## Purpose

**What it does**

FirstChargebackBatch is the origin of all MasterCard first chargeback data in WDP. It runs as a continuously-executing Spring Batch application on a Kubernetes Deployment, driven by an internal `@Scheduled` cron expression. On each firing, it polls the MasterCard Dispute Management (MCM) unworked first chargeback queue via the IBM DataPower Gateway proxy, enriches each claim with settled transaction PAN data, encrypts the cardholder account number via the WDP Encryption Service, and writes one or more rows to the `wdp.chbk_outbox_row` transactional outbox per qualifying chargeback.

If this component fails or falls behind, no new MasterCard first chargeback cases will be created or updated downstream. It is a source-of-truth ingest boundary — downstream processing, case creation, and Kafka publishing are all gated on this component writing PENDING rows to the outbox.

All MasterCard calls are proxied through IBM DataPower, which acts as the API facade to the MasterCard network. Authentication to MCM uses a static Vantiv license key, not OAuth. All three outbound REST integrations (DataPower, WDP Encryption Service, IDP Token Service) use Spring Retry (3 attempts, 2-second fixed delay) but have no timeout configuration and no Resilience4j circuit breaker — this is a platform-wide deviation from DEC-014.

The component deliberately processes one claim per Spring Batch chunk (chunk size = 1), making each claim its own JPA transaction. This enables partial-run progress at the cost of throughput. Two feature flags control operational safety: `readSpecificItemFromQueue` restricts polling to a whitelist of specific claim IDs (surgical testing), and `removeItemFromQueueDisabled` suppresses all ACK PUT calls (safety switch during testing or controlled rollout).

**What it does NOT do**

- Does not publish to Kafka directly — writes PENDING rows to `wdp.chbk_outbox_row` only; the separate InboundDisputeEventScheduler (COMP-12) reads and publishes to Kafka
- Does not handle second or subsequent chargebacks, pre-arbitration, arbitration, or reversal records — first chargeback only
- Does not perform case routing, decisioning, evidence attachment, or response filing
- Does not write to S3 or any file system
- Does not invoke any WDP case management API to create cases
- Does not cache IDP tokens between encryption calls — a fresh token is fetched per claim
- Does not perform encryption on the existing-claim update path — account number is null on all rows written via the existing-claim path

---

## Internal Processing Flow

```mermaid
flowchart TD
    TRIGGER(["@Scheduled cron fires\napp.scheduler.cron\nenv var: scheduler_cron"])
    LAUNCH["Spring Batch JobLauncher\nTimestamp param appended for uniqueness\nAuto-start disabled — cron-only"]

    POLL["Queue Poll — GET DataPower\napp.dataPowerService.unworkedChargebacksQueueUrl\nReturns QueueClaim array\nRetry: 3× 2s fixed delay | No timeout | No CB"]

    EMPTY{{"Queue null\nor empty?"}}
    EXIT_EMPTY(["Job COMPLETED\n0 records processed"])

    FLAG{{"readSpecificItemFromQueue\nenabled?"}}
    FILTER["Filter claim IDs to\nmcmDisputeCaseNumbers whitelist\nenv var: mcm_case_numbers"]

    FOR_EACH["For each QueueClaim\n— chunk size = 1 —"]

    CLAIM_FETCH["Claim Detail Fetch — GET DataPower\napp.dataPowerService.claimDetail\nURL: .../mastercom/v6/claims/{claim-id}\nRetry: 3× 2s | No timeout | No CB"]

    CHBK_FILTER["Filter chargebacks from ClaimDetail:\nchargebackType == 'CHARGEBACK'\n&& !reversed && !reversal\nfields: reversed / reversal (boolean primitives)"]

    ALL_FILTERED{{"chargebackDetailList\nempty after filter?"}}
    FILTER_EXCEPTION["Exception thrown internally\nCaught at process() outer catch\nprocessedItems = empty list"]
    NOT_ACKED_FILTER(["Claim NOT acknowledged\nACK list empty — PUT suppressed\nClaim reappears in MCM queue\nnext run"])

    IDEM_BULK["Bulk idempotency check\nfindByNetworkCaseId(claimId)\nWHERE c_ntwk_case_id = claimId"]

    CLAIM_PATH{{"Rows found\nfor this claimId?"}}

    %% ── NEW CLAIM PATH ──────────────────────────────────────
    NEW_PATH["NEW CLAIM PATH\nisAccountNumberRequired = true\n(resets to false after first chargeback)"]

    FOR_NEW["For each qualifying chargeback\n(ordered)"]

    FIRST_CHK{{"isAccountNumberRequired\n= true?\n(first chargeback in claim)"}}
    SKIP_PAN["Skip all PAN resolution\naccountNumber = null\nMap entity — status = PENDING"]

    SETTLED_CHK{{"transactionId present\non ClaimDetail?\n(StringUtils.isNotBlank)"}
    }
    FALLBACK_PAN["Account number =\nclaimDetail.primaryAccountNum\n(claim-level PAN fallback)"]
    SETTLED_FETCH["Settled Tx Fetch — GET DataPower\napp.dataPowerService.settledTransaction\nURL: .../mastercom/v6/claims/{id}/transactions/clearing/{tx-id}\nRetry: 3× 2s | No timeout | No CB"]
    ACCT_RESOLVE["Resolve account number:\n1st choice: mastercardMappingServiceAccountNumber\nfallback: claimDetail.primaryAccountNum\nTruncate: last 16 chars if > 16"]

    MASK_CHK{{"Account number\ncontains '*'?"}}
    WRITE_MASKED["Write masked value as-is\n(encryption skipped)"]

    IDP_FETCH["IDP Token Fetch — GET\napp.idpService.tokenUrl\nFresh token per call — no caching\nRetry: 3× 2s | No timeout | No CB"]

    ENCRYPT["Encryption POST\napp.disputeService.encryptionUrl\nRequest: {pan} → Response: {hpan}\nRetry: 3× 2s | No timeout | No CB"]

    ENCRYPT_FAIL{{"Encryption or IDP\ntoken call fails?"}}
    ENCRYPT_FAIL_OUT(["Exception caught at process()\nprocessedItems = empty\nACK list empty\nClaim NOT acknowledged\nRemains in MCM queue"])

    MAP_NEW["Map to ChbkOutboxEntity\nstatus = PENDING | HPAN set\nSet isAccountNumberRequired = false\nfor remaining chargebacks in claim"]

    %% ── EXISTING CLAIM PATH ─────────────────────────────────
    EXIST_PATH["EXISTING CLAIM PATH\nisAccountNumberRequired = false (all)\nNo PAN fetch, no encryption"]
    FOR_EX["For each qualifying chargeback\nprocessStatus default = SKIPPED"]
    IDEM_CHBK["Per-chargeback lookup\ncountByNetworkCaseIdAndNetworkPhaseId\nWHERE c_ntwk_case_id = claimId\nAND c_ntwk_phase_id = chargebackId"]
    CHBK_FOUND{{"Count > 0?"}}
    KEEP_SKIPPED["processStatus = SKIPPED\n(existing row found)"]
    SET_PENDING["processStatus = PENDING\n(new chargeback in existing claim)"]
    MAP_EX["Map ChbkOutboxEntity\nstatus = PENDING or SKIPPED\naccountNumber = null\nReset processStatus = SKIPPED\nfor next chargeback"]

    %% ── WRITE AND ACK ───────────────────────────────────────
    WRITE["Outbox Write — JPA save\nwdp.chbk_outbox_row\nSame JPA transaction as idempotency read\n(wdpTransactionManager, chunk boundary)"]
    WRITE_FAIL{{"Write exception?"}}
    WRITE_SKIP(["Exception caught and logged\nItem silently skipped\nBatch continues"])

    ACK_GATE{{"removeItemFromQueueDisabled\n= true?"}}
    SUPPRESS_ACK(["ACK PUT suppressed\nSafety switch active\nClaim stays in MCM queue"])

    ACK_PUT["Batch ACK — PUT DataPower\napp.dataPowerService.claimAcknowledgement\nURL: .../mastercom/v6/chargebacks/acknowledge\nAll (claimId, chargebackId) pairs in chunk\nRetry: 3× 2s | No timeout | No CB"]

    ACK_FAIL{{"ACK fails or\nfailureReason in response?"}}
    ACK_FAIL_OUT(["Exception swallowed\nOutbox rows already committed\nClaim NOT acknowledged\nOn re-run: existing rows → SKIPPED\nAdded to ACK list for retry"])

    ACK_OK["Claim acknowledged\nRemoved from MCM queue"]

    MORE{{"More claims\nin list?"}}
    JOB_DONE(["Job COMPLETED"])

    %% ── WIRING ──────────────────────────────────────────────
    TRIGGER --> LAUNCH --> POLL
    POLL --> EMPTY
    EMPTY -->|"Yes"| EXIT_EMPTY
    EMPTY -->|"No"| FLAG
    FLAG -->|"Yes"| FILTER --> FOR_EACH
    FLAG -->|"No"| FOR_EACH

    FOR_EACH --> CLAIM_FETCH --> CHBK_FILTER --> ALL_FILTERED
    ALL_FILTERED -->|"Yes"| FILTER_EXCEPTION --> NOT_ACKED_FILTER
    ALL_FILTERED -->|"No"| IDEM_BULK --> CLAIM_PATH

    CLAIM_PATH -->|"No rows — new claim"| NEW_PATH --> FOR_NEW --> FIRST_CHK
    FIRST_CHK -->|"false — non-first chargeback"| SKIP_PAN
    FIRST_CHK -->|"true — first chargeback"| SETTLED_CHK
    SETTLED_CHK -->|"Absent"| FALLBACK_PAN --> MASK_CHK
    SETTLED_CHK -->|"Present"| SETTLED_FETCH --> ACCT_RESOLVE --> MASK_CHK
    MASK_CHK -->|"Yes — masked"| WRITE_MASKED --> MAP_NEW
    MASK_CHK -->|"No — clear PAN"| IDP_FETCH --> ENCRYPT --> ENCRYPT_FAIL
    ENCRYPT_FAIL -->|"Yes"| ENCRYPT_FAIL_OUT
    ENCRYPT_FAIL -->|"No"| MAP_NEW
    SKIP_PAN --> FOR_NEW
    MAP_NEW --> FOR_NEW

    CLAIM_PATH -->|"Rows found — existing claim"| EXIST_PATH --> FOR_EX --> IDEM_CHBK --> CHBK_FOUND
    CHBK_FOUND -->|"Yes"| KEEP_SKIPPED --> MAP_EX
    CHBK_FOUND -->|"No"| SET_PENDING --> MAP_EX
    MAP_EX --> FOR_EX

    FOR_NEW --> WRITE
    FOR_EX --> WRITE

    WRITE --> WRITE_FAIL
    WRITE_FAIL -->|"Yes"| WRITE_SKIP
    WRITE_FAIL -->|"No"| ACK_GATE
    WRITE_SKIP --> ACK_GATE

    ACK_GATE -->|"Yes — suppressed"| SUPPRESS_ACK
    ACK_GATE -->|"No"| ACK_PUT --> ACK_FAIL
    ACK_FAIL -->|"Yes"| ACK_FAIL_OUT
    ACK_FAIL -->|"No"| ACK_OK

    ACK_OK --> MORE
    SUPPRESS_ACK --> MORE
    MORE -->|"Yes"| FOR_EACH
    MORE -->|"No"| JOB_DONE
```

---

## Boundaries

### Inbound Interfaces

| Source | Protocol | Endpoint / Trigger | Payload / Description |
|--------|----------|--------------------|-----------------------|
| Internal `@Scheduled` cron | Spring Scheduler | `app.scheduler.cron` — env var `scheduler_cron` | No payload — time-driven trigger only |
| MCM via DataPower | REST GET (DataPower proxy) | `app.dataPowerService.unworkedChargebacksQueueUrl` → `.../mastercom/v6/queues?queue-name=AcquirerFirstCBUnworked` | Returns `QueueClaim[]` — each contains a single `claimId` field |

### Outbound Interfaces

| Target | Protocol | Endpoint / Resource | Purpose | On failure |
|--------|----------|---------------------|---------|------------|
| DataPower Gateway (MCM) | REST GET | `app.dataPowerService.claimDetail` — `.../mastercom/v6/claims/{claim-id}` | Fetch full dispute claim detail per claim ID | Retry 3× 2s; if all fail, exception caught, item skipped, claim not ACKed |
| DataPower Gateway (MCM) | REST GET | `app.dataPowerService.settledTransaction` — `.../mastercom/v6/claims/{id}/transactions/clearing/{tx-id}` | Fetch settled transaction to resolve preferred account number | Retry 3× 2s; if skipped (no txId), fallback to claim PAN |
| DataPower Gateway (MCM) | REST PUT | `app.dataPowerService.claimAcknowledgement` — `.../mastercom/v6/chargebacks/acknowledge` | Mark processed chargebacks as worked so they are not re-queued | Retry 3× 2s; on failure exception swallowed; outbox rows already committed; claim re-queued on next run |
| IDP Token Service | REST GET | `app.idpService.tokenUrl` — `http://wdp-idp-token-service.wdp-micro:8082/merchant/gcp/idp-token/token` | Obtain bearer token for Encryption Service call — fetched fresh per claim, no caching | Retry 3× 2s; on failure exception caught, claim not written or ACKed |
| WDP Encryption Service | REST POST | `app.disputeService.encryptionUrl` — `http://wdp-encryption-service.wdp-micro:8082/merchant/gcp/encryption/v1/pan/encrypt` | Encrypt clear PAN to HPAN before outbox write | Retry 3× 2s; on failure exception caught, claim not written or ACKed |
| PostgreSQL `wdp.chbk_outbox_row` | JPA (HikariCP) | Schema: `wdp` | Idempotency read + PENDING/SKIPPED row write | No retry on DB calls; writer exceptions caught and logged per item |

**Authentication summary:**
- DataPower Gateway: Static Vantiv license key injected from env var `vantive_license` — passed as `Authorization` header. Not OAuth.
- IDP Token Service: No auth on the token fetch call itself (unauthenticated GET).
- Encryption Service: Short-lived bearer token obtained from IDP Token Service, added as `Authorization: Bearer <token>` header. Token fetched fresh per claim — no in-memory caching.

**Timeout summary:** ⚠️ No connection or read timeouts are configured on any RestTemplate. All three outbound REST integrations use a plain `new RestTemplate()` with no `ClientHttpRequestFactory` timeout configuration. This is a platform-wide gap.

---

## Database Ownership

### Tables Owned (written by this component)

| Schema.Table | Purpose | Key columns written | Notes |
|--------------|---------|---------------------|-------|
| `wdp.chbk_outbox_row` | Transactional outbox for MasterCard first chargeback events — PENDING rows consumed by COMP-12 InboundDisputeEventScheduler for Kafka publish | `id` (auto-sequence `CHBK_OUTBOX_ROW_ID_SEQ`), `c_ntwk_case_id` (claimId), `c_ntwk_phase_id` (chargebackId), `c_case_stage` = `CH1` (hardcoded), `c_case_ntwk` = `MASTERCARD` (hardcoded), `c_acq_platform` = `CORE` (hardcoded), `status` = `PENDING` or `SKIPPED`, `event_type` = `CHARGEBACK_PROCESS`, `i_ntwk_tran_id` (networkSettledTransactionId), `i_acq_refnce_num` (acquirerRefNum), `c_level1_entity` (merchantId), `retry_count` = `0`, `created_at`, `updated_at`, `created_by` = `WMFDPB`, `updated_by` = `WMFDPB`, `payload` (serialized CommonEvent JSON including `enrichmentFailure=true` inside `OriginalTransIdentifier`) | ⚠️ Shared writer — also written by COMP-07, COMP-09, COMP-11. Per-component duplicate check on (claimId, chargebackId) before write. Columns NOT set by this batch: `file_job_id`, `kafka_partition`, `kafka_offset`, `kafka_topic`, `published_at`, `error_code`, `error_message`, `document_type`, `source_event`, `i_case`, `i_action_id`, `c_reason`, `c_migration_sta`, `next_retry_st`, `row_number`, `parent_row_number`. `enrichmentFailure` is inside the payload JSON, not a top-level column. |

### Tables Read (not owned by this component)

| Schema.Table | Owned by | Why accessed |
|--------------|----------|--------------|
| `wdp.chbk_outbox_row` | COMP-08 (self — shared table) | Idempotency check: bulk existence by `claimId` (new vs existing path determination), then per-chargeback count by `(claimId, chargebackId)` on the existing path. Both reads share the same JPA chunk transaction as the subsequent write. |

### Spring Batch Metadata Tables

| Table | Schema / Prefix | Notes |
|-------|----------------|-------|
| `BATCH_JOB_INSTANCE` | Env-var-controlled prefix via `table_prefix` (env var). Same PostgreSQL instance as `wdp` datasource. Schema/prefix not determinable from source alone. | Prod database: `wpdisputedatabase` |
| `BATCH_JOB_EXECUTION` | Same as above | |
| `BATCH_STEP_EXECUTION` | Same as above | |

---

## Configuration and Scaling

| Parameter | Value | Notes |
|-----------|-------|-------|
| Replica count | XL Deploy placeholder: `{{ replicas-wdp-mcm-first-chargeback-queue-batch }}` | Resolved at deploy time. Confirmed 1 in production — replicas > 1 creates parallel MCM queue polling risk with no distributed lock. |
| HPA | None | No HorizontalPodAutoscaler configured |
| Memory request | 256Mi | `resources.yaml:40` |
| Memory limit | 2048Mi | `resources.yaml:38` |
| CPU request | Not set | Absent from `resources.yaml` |
| CPU limit | Not set | Absent from `resources.yaml` |
| Deployment type | Kubernetes Deployment | Not a CronJob — JVM stays warm between cron fires |
| Rollout strategy | RollingUpdate — maxSurge: 1, maxUnavailable: 0 | `resources.yaml:10-13` |
| PodDisruptionBudget | None | Only manifest is `resources.yaml`; no PDB resource present |
| Topology spread | None | No `topologySpreadConstraints` in pod spec |
| minReadySeconds | 30 | ⚠️ Placed at `spec.template.spec` level (non-standard — typically at `spec` level for Deployment) |
| Database connection pool | HikariCP defaults — maximumPoolSize: 10, minimumIdle: 10, connectionTimeout: 30s, idleTimeout: 600s, maxLifetime: 1800s | No explicit HikariCP properties configured. Spring Boot auto-configures defaults. `spring.datasource.wdp` prefix. |
| Thread pool | Spring Batch `SyncTaskExecutor` (default) | Single-threaded — no thread pool configured |
| Chunk size | 1 (hardcoded) | Each claim is its own JPA transaction |
| Observability | OpenTelemetry Java agent injected via annotation `instrumentation.opentelemetry.io/inject-java` | OTel operator: `opentelemetry-operator-system/default`. Spring Actuator on port 8082. TLS: corporate CA cert chain mounted from secret `ws-int-infotfps` into JRE truststore. |
| Batch job auto-start | Disabled (`spring.batch.job.enabled: false`) | Job launched only via internal `@Scheduled` cron |
| Spring Batch table prefix | Env var `table_prefix` | `spring.batch.jdbc.tablePrefix: ${table_prefix}` |

---

## Key Architectural Decisions

| Decision | ADR reference | Notes |
|----------|---------------|-------|
| Transactional outbox pattern — writes PENDING rows to `wdp.chbk_outbox_row`; downstream publisher reads and publishes to Kafka | DEC-001 ✅ Compliant | Decouples MasterCard ingestion from Kafka availability. Outbox write and idempotency read share the same JPA chunk transaction (wdpTransactionManager). |
| PAN encrypted at ingestion boundary before any persistence | DEC-004 ✅ Compliant | Clear PAN (`primaryAccountNum` on `ClaimDetail`) annotated `@ToString(exclude)` — excluded from all logs. Encryption delegated to WDP Encryption Service. Clear PAN never written to database. HPAN (not clear PAN) is written to entity — if it were logged via `OriginalTransIdentifier.toString()`, the HPAN would be visible (acceptable, but noted). |
| No Resilience4j circuit breaker on any outbound call | DEC-014 ⚠️ DEVIATION | Zero Resilience4j annotations, beans, or properties exist in this codebase (confirmed by full-source grep). All outbound calls protected only by Spring Retry (3× 2s fixed delay). No circuit breaker, rate limiter, or bulkhead on DataPower, Encryption Service, or IDP Token Service. Platform-wide pattern — not isolated to this component. |
| No timeouts on any RestTemplate | Local decision — ⚠️ RISK | All three REST integrations use a plain `new RestTemplate()` with no `ClientHttpRequestFactory`. Default RestTemplate has no connect or read timeout. A hanging downstream call will block the batch thread indefinitely. |
| Kubernetes Deployment, not CronJob | DEC-PLACEHOLDER | JVM stays warm between cron fires, avoiding cold-start latency. Trade-off: replicas > 1 creates parallel queue polling risk with no distributed lock. Production hardened to replica count = 1 via XL Deploy variable. |
| Chunk size = 1 (deliberate) | Local decision | Each claim is its own JPA transaction. One failure does not roll back previously written claims. Enables partial-run progress at the cost of throughput. |
| IDP token fetched fresh per claim — no caching | Local decision — ⚠️ RISK | `IdpTokenResponse` contains `expiresIn` and `expirationDate` fields but they are never read. A fresh HTTP GET is made to IDP on every encryption call. For a large queue, this multiplies IDP load proportionally. |
| Kafka dependency present in pom.xml but no Kafka code active | ⚠️ Dead dependency | `spring-kafka` and `kafka-clients` imported. Only active reference is a misuse of a Kafka-shaded Protobuf exception class (`org.apache.kafka.shaded.com.google.protobuf.ServiceException`) in the Encryption Service impl. No Kafka producer, consumer, or template wired. `aws-msk-iam-auth` also present with zero usage. |
| `removeItemFromQueueDisabled` flag — second operational safety switch | Local decision | Boolean flag (`app.batchProperties.removeItemFromQueueDisabled`, env var `remove_item_from_queue_disabled`). When true, the ACK PUT to DataPower is completely suppressed for all claims regardless of processing outcome. Intended for testing and controlled rollout. No ADR documented. |

---

## Risks and Constraints

| Severity | Risk | Consequence |
|----------|------|-------------|
| 🔴 HIGH | No timeouts on any RestTemplate. DataPower, Encryption Service, and IDP Token Service calls have no connect or read timeout configured. | A hanging downstream call will block the single batch thread indefinitely. The `@Scheduled` cron will not fire again until the current execution completes. MCM ingestion halts. |
| 🔴 HIGH | No Resilience4j circuit breaker on any outbound integration (DEC-014 deviation). | A persistently degraded DataPower or Encryption Service will not be detected or isolated. Retry storms possible under sustained downstream failure. |
| 🟡 MEDIUM | Replica count must be exactly 1. No distributed lock prevents parallel MCM queue polling. | If XL Deploy resolves the replica variable to > 1, two pods will poll the same MCM queue simultaneously. This risks duplicate PENDING rows if both process the same claimId before either writes — the bulk idempotency check uses `findByNetworkCaseId` and the per-chargeback count uses `countByNetworkCaseIdAndNetworkPhaseId`, both without a DB-level lock. |
| 🟡 MEDIUM | IDP token fetched fresh on every encryption call with no caching. | For large queue batches, IDP token fetch volume equals the number of claims with new first chargebacks. High IDP load at peak ingestion. TTL fields (`expiresIn`, `expirationDate`) exist on the response object but are never read. |
| 🟡 MEDIUM | Exception swallowing on encryption failure — Spring Batch step is not marked FAILED. | If encryption fails for a claim, the processor catches the exception, returns an empty list, and the step continues with the next claim. The batch job completes COMPLETED even if many claims failed encryption. Failed claims remain in the MCM queue and will be retried on the next run, but there is no alerting or failure count visible at the job level. |
| 🟡 MEDIUM | `OriginalTransIdentifier` has no `@ToString` exclusion for `accountNumber`. The HPAN (not clear PAN) would be included if this object were logged via `toString()`. | Not a DEC-004 violation (HPAN is not clear PAN), but HPAN leakage in logs is worth reviewing from a data minimisation perspective. |
| 🟡 MEDIUM | Claims with all chargebacks filtered (all reversed or reversals) are NOT acknowledged to MCM. | These claims reappear in the MCM queue on every run, are fetched again, filtered again, and silently ignored. This can inflate MCM queue polling overhead if reversal-only claims accumulate. |
| 🟢 LOW | `spring-boot-starter-oauth2-client` on classpath with no OAuth2 client registration or usage. | No runtime effect. Unnecessary security surface on the classpath. Should be removed if OAuth2 is not planned for this service. |
| 🟢 LOW | `minReadySeconds: 30` placed at `spec.template.spec` level in `resources.yaml` (non-standard — typically at `spec` level for a Deployment). | Non-standard placement. Verify whether Kubernetes honours it in this position or silently ignores it. If ignored, pod health checks may not have the intended buffer during rolling updates. |
| 🟢 LOW | Dead Kafka dependencies (`spring-kafka`, `kafka-clients`, `aws-msk-iam-auth`) and OAuth2 dependency imported in `pom.xml` with no active usage. | Unnecessary classpath bloat. `spring-kafka` misuse as a Protobuf exception source (`org.apache.kafka.shaded.com.google.protobuf.ServiceException`) creates a confusing dependency. |
| 🟢 LOW | `CheckmarkUtil.java`, `CaseEvent.java`, `WebResponse.java`, `MerchantDetails.java`, `HistoricalDisputeDetail.java` — dead domain/utility classes with no production instantiation sites. | Code noise. `CaseEvent.java` appears to be residual scaffolding from an earlier event design. Risk: a future developer may attempt to use these assuming they are active. |
| 🟢 LOW | `app.batchProperties.userId` property (`WMFDPB`) is bound in `ApplicationProps.Batch` but never read at runtime. The constant `ApplicationConstants.USEERID` (`WMFDPB`) is used directly instead. | Dead configuration. No functional impact. |

---

## Planned Changes

- Clean up dead Kafka dependencies (`spring-kafka`, `kafka-clients`, `aws-msk-iam-auth`) — Kafka publishing responsibility confirmed to lie with the outbox publisher (COMP-12), not this batch.
- Remove `spring-boot-starter-oauth2-client` if OAuth2 integration is not planned for this service.
- Formally document or remove `removeItemFromQueueDisabled` flag — currently an undocumented operational safety switch with no ADR.
- Formally document or remove `readSpecificItemFromQueue` flag — currently an undocumented surgical testing tool with no ADR.
- Add IDP token caching — token TTL fields exist on `IdpTokenResponse` but are never read. Implementing a short-lived token cache would reduce IDP load at peak ingestion volumes.
- Add RestTemplate timeout configuration — all three outbound REST integrations lack connect and read timeouts. A platform-wide fix is needed.
- ⚠️ OPEN QUESTION: Confirm whether `minReadySeconds: 30` at `spec.template.spec` level in `resources.yaml` is honoured by Kubernetes or silently ignored.
- ⚠️ OPEN QUESTION: Confirm actual XL Deploy variable value for replica count in each environment (dev, staging, prod). Source confirms it is a placeholder — the production count of 1 is a team convention, not a code-enforced constraint.
- ⚠️ OPEN QUESTION: Is there a plan to add Resilience4j circuit breakers to this component as part of a platform-wide DEC-014 remediation?

---

---

## ━━━ TYPE BLOCK D — BATCH AND SCHEDULER CONTRACTS ━━━━━━━━
*This component is a Spring Batch job running as a long-lived Kubernetes Deployment.*

---

## Batch and Scheduler Contracts

**Batch framework:** Spring Batch (Spring Boot 3.5.3 integration)
**Deployment type:** Kubernetes Deployment (not CronJob) — JVM stays warm between cron firings
**Trigger mechanism:** Internal `@Scheduled` cron expression (`BatchScheduler.java`) — no HTTP trigger, no Actuator job-launch endpoint, no external trigger wired
**Job uniqueness:** Timestamp parameter appended to `JobParameters` on every launch — ensures each cron firing creates a unique Spring Batch `JobInstance`

---

### Job: MCM First Chargeback Ingest Job

**Purpose:** Poll the MCM unworked first chargeback queue via DataPower, enrich each claim with PAN data and encryption, and write PENDING rows to `wdp.chbk_outbox_row` for downstream Kafka publishing by COMP-12.

**Schedule**

| Parameter | Config key | Value / Source |
|-----------|------------|----------------|
| Cron expression | `app.scheduler.cron` | Injected from env var `scheduler_cron` — not hardcoded in any YAML. Value not determinable from source alone. Production value: every 5 minutes. |
| Job auto-start | `spring.batch.job.enabled` | `false` — disabled. Cron-driven only. |
| Chunk size | Hardcoded | 1 — each `QueueClaim` is one Spring Batch chunk |

**Input source**

| Source | Type | Query / Filter | Pagination |
|--------|------|----------------|------------|
| MCM via DataPower — `app.dataPowerService.unworkedChargebacksQueueUrl` | REST GET (queue poll) | Queue name: `AcquirerFirstCBUnworked`. Returns full array in one call. | None implemented. No page-size parameter or loop over pages. Server-side limit (if any) not determinable from source. |
| `readSpecificItemFromQueue` flag | Optional filter applied to queue poll result | If enabled (`read_specific_item_from_queue = true`), claim IDs are filtered to `mcmDisputeCaseNumbers` whitelist (`mcm_case_numbers` env var) | N/A |

**Processing steps**

| Step | Name | Description | Chunk size | On failure |
|------|------|-------------|------------|------------|
| 1 | `BatchItemReader.read()` | Queue poll — GET DataPower for unworked claim IDs. Apply `readSpecificItemFromQueue` filter if enabled. Return one `QueueClaim` per `read()` call. | 1 | Returns null if queue empty — job exits COMPLETED with 0 records. Exception on poll failure propagates; item not processed. |
| 2 | `BatchItemProcessor.process()` | Claim detail fetch; chargeback filter; idempotency check; PAN resolution + encryption (new claims); entity mapping (new or existing path). All exceptions caught — returns empty list on failure; step does NOT fail. | 1 | Exception caught at outer try/catch. Returns empty `processedItems`. Batch continues with next claim. Claim not ACKed if empty result. |
| 3 | `BatchItemWriter.write()` | JPA save each `ChbkOutboxEntity` to `wdp.chbk_outbox_row`. Then batch ACK all processed chargebacks to MCM via DataPower PUT (conditional on `removeItemFromQueueDisabled` flag). | 1 | Write exception: caught and logged per item, silently skipped. ACK exception: swallowed — outbox rows already committed. |

**Downstream calls per claim (worst case — new claim with settled transaction)**

1. GET DataPower — claim detail
2. GET DataPower — settled transaction (if `transactionId` present on claim)
3. GET IDP Token Service — bearer token for encryption (first chargeback in claim only)
4. POST WDP Encryption Service — encrypt PAN to HPAN (first chargeback in claim only)
5. JPA `save()` to `wdp.chbk_outbox_row` — once per qualifying chargeback
6. PUT DataPower — batch ACK (once per chunk, conditional on `removeItemFromQueueDisabled`)

**Outputs**

| Target | Type | What is written | On failure |
|--------|------|-----------------|------------|
| `wdp.chbk_outbox_row` | JPA write (wdpTransactionManager) | One row per qualifying chargeback: status `PENDING` (new) or `SKIPPED` (duplicate existing). Idempotency read and outbox write share the same JPA chunk transaction. | Exception caught and logged per item; silently skipped |
| MCM via DataPower — ACK | REST PUT | All `(claimId, chargebackId)` pairs processed in the chunk acknowledged as worked | Exception swallowed; outbox rows already committed; claim re-queued in MCM; on re-run existing rows get SKIPPED status and are re-added to ACK list |

**Failure and recovery**

The job is safe to re-run. The bulk idempotency check (`findByNetworkCaseId`) determines whether each claim has been seen before, routing it to the new-claim or existing-claim path. On the existing-claim path, each chargeback is looked up individually (`countByNetworkCaseIdAndNetworkPhaseId`) — if a row exists it gets `SKIPPED`, if not it gets `PENDING`. This means a partial run followed by a re-run produces at most duplicate `SKIPPED` rows for already-processed chargebacks, and `PENDING` rows for newly seen chargebacks.

ACK failures are self-healing: unacknowledged claims re-appear in the MCM queue on the next run, go through the existing-claim path (all chargebacks found → SKIPPED), and are re-added to the ACK list for another PUT attempt.

No manual reprocessing path is documented. The `readSpecificItemFromQueue` flag can serve as a targeted re-processing tool for specific case IDs during operational incidents.

Spring Batch metadata tables (BATCH_JOB_INSTANCE, BATCH_JOB_EXECUTION, BATCH_STEP_EXECUTION) record job and step status per run. A COMPLETED job with 0 items processed is indistinguishable at the batch metadata level from a COMPLETED job with a successful run — failed items are swallowed, not surfaced as FAILED step executions.

**Spring Batch metadata**

| Table | Schema / Prefix | Purpose |
|-------|----------------|---------|
| `{table_prefix}JOB_INSTANCE` | Env-var prefix `table_prefix`. Same PostgreSQL instance as `wdp` datasource. Schema not determinable from source. | Job identity and deduplication |
| `{table_prefix}JOB_EXECUTION` | Same | Execution status per run |
| `{table_prefix}STEP_EXECUTION` | Same | Step-level progress and counts |

---

**Commented-out code — planned or deferred work**

| File | What was commented out | Status |
|------|------------------------|--------|
| `ChbkOutboxRepository.java:12-22` | Two `@Query` JPQL methods — replaced by Spring Data derived query methods. Custom queries match derived queries exactly. | Dead scaffolding from refactor. Safe to remove. |
| `ProcessorUtil.java:96` | `documentRequired` field setter — replaced by `networkDocInd` setter | Migration complete. Safe to remove. |
| `ProcessorUtil.java:105` | `caseType` field setter (`isPartialChargeback() ? PART : null`) | `caseType` field on `CommonEvent` is never populated at runtime. Appears intentionally deferred or removed. Requires decision. |
| `RestInvoker.java:49` | Detailed exception format in encryption POST handler — replaced with plain `throw e` | Logging detail was removed. Safe to remove the comment. |

---

*End of WDP-COMP-08-FIRST-CHARGEBACK-BATCH.md*
*File status: 📝 DRAFT — awaiting architect confirmation*
*Remember to update WDP-COMP-INDEX.md (doc status), WDP-KAFKA.md (confirm Section 5 outbox entry), and WDP-DB.md (enrich chbk_outbox_row row and Spring Batch metadata rows)*
