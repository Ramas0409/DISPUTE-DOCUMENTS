# WDP-COMP-05-NAP-DISPUTE-EVENT-PROCESSOR
**Worldpay Dispute Platform — Component Reference**
*Version: 1.0 DRAFT | April 2026*
*Extracted from: gcp-nap-dispute-event-consumer — WDP-COMPONENTS.md COMPLETE section*
*Architect-confirmed: PENDING*

---

> ⚠️ **DECOMMISSION-SCOPED COMPONENT**
> This component sits on the NAP/WPG inbound path which is planned for
> decommission as CB911 merchant migration completes. No new development
> or design work is planned. The `wdpOnly` runtime flag is the active
> migration gate. This file documents the component as-built.

---

## ━━━ CORE SKELETON ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## Identity

| Field             | Value |
|-------------------|-------|
| **Name**          | `NAPDisputeEventProcessor` |
| **Type**          | `Kafka Consumer + REST API (manual reprocessing endpoint)` |
| **Repository**    | `gcp-nap-dispute-event-consumer` |
| **Maven artifact**| `gcp-nap-dispute-event-consumer v1.6.5` |
| **Technology**    | `Spring Boot 3.5.7 / Java 17 / Spring Kafka / Spring Data JPA` |
| **Owner**         | `Integration Team` |
| **Status**        | `✅ Production` |
| **Doc status**    | `📝 DRAFT` |
| **Decommission scope** | `⚠️ NAP/WPG inbound path — planned for decommission` |
| **Sections present** | `Core \| Block A (REST — manual reprocessing) \| Block B (Kafka Consumer)` |

---

## Purpose

**What it does**

NAPDisputeEventProcessor is the stateful, event-driven downstream processor
on the NAP acquiring platform inbound path. It consumes enriched dispute
notification messages from the `nap-dispute-events` AWS MSK topic (published
by COMP-04 NAPDisputeEventService) and translates them into case management
actions on the WDP case management platform via synchronous REST calls.

The component receives a single unified `NotificationEvent` payload on a
single Kafka topic. At runtime it classifies each message into one of three
sub-types — SRV116 (new chargeback notification), WIN/LOSS (arbitration
outcome), or SRV118 (chargeback response/representment) — by field inspection,
and routes each to a distinct processing path.

The component is in an active hybrid migration state. A `wdpOnly` flag on
each event gates whether the event belongs to a migrated legacy merchant or a
native NAP case. WIN/LOSS and SRV118 flows are skipped entirely for
`wdpOnly = 'Y'` events. The SRV116 path also applies MASTERCARD/MAESTRO-
specific MCM Queue lookup logic not present for other card networks.

This is the only inbound consumer operating on the NAP acquiring platform path.
All other acquiring platforms (Visa batch, file-based) flow through
CaseCreationConsumer (COMP-14).

**What it does NOT do**

- Does not publish to any Kafka topic. No KafkaTemplate or producer exists
  in the codebase. The `SEND_BUSINESS_RULES_INFO_TO_KAFKA` terminal state
  visible in the pipeline is a no-op — infrastructure exists but no publish
  occurs. See Risks.
- Does not perform any NAP platform authorization — that is COMP-02 UAMS.
- Does not enrich events — enrichment (productType, subProductType, fraud
  indemnity) is performed upstream by COMP-04 NAPDisputeEventService before
  the Kafka message is published.
- Does not use a Kafka DLQ topic. All error state is persisted to the
  PostgreSQL `NAP.DISPUTE_EVENT_CONSUMER_ERROR` table.
- Does not process non-NAP acquiring platform events — COMP-14
  CaseCreationConsumer handles all other platforms.
- Does not apply circuit-breaker logic — no Resilience4j dependency is present.

---

## Internal Processing Flow

```mermaid
flowchart TD
    IN["Kafka message received\n nap-dispute-events topic\n NotificationEvent payload"]

    ACK["MANUAL_IMMEDIATE offset commit\n Pre-ACK BEFORE processing\n ⚠️ Deviation from DEC-005"]

    PRIOR{{"Prior unresolved ERROR\nrecords for same ARN\nor NetworkCaseId?"}}

    BLOCK["Block / escalate event\nWrite DRAFT status to\nDISPUTE_EVENT_CONSUMER_ERROR"]

    CLASSIFY{{"Classify event\nby field inspection"}}

    SRV116["SRV116 path\nNew chargeback notification\n(default — all other cases)"]
    WINLOSS["WIN/LOSS path\noutcome field is non-blank"]
    SRV118["SRV118 path\nnapResponseType AND\nnapResponseCode non-blank"]

    WDPONLY_WL{{"wdpOnly = 'Y'?"}}
    WDPONLY_118{{"wdpOnly = 'Y'?"}}

    SKIP_WL["Skip — WIN/LOSS not\nprocessed for wdpOnly events"]
    SKIP_118["Skip — SRV118 not\nprocessed for wdpOnly events"]

    OUTCOME_CHECK{{"outcome = NA\nor CLOSED?"}}
    SKIP_OUTCOME["Skip silently"]

    MCM{{"MASTERCARD\nor MAESTRO?"}}
    MCM_LOOKUP["MCM Queue lookup\nnap.mcm_queue_data\nResolve claim ID"]
    MCM_MULTI{{"Multiple MCM\nclaim IDs?"}}
    MCM_HOLD["Place ONHOLD\nWrite to error table\nAwait napEventExpiryHour"]

    CASE_LOOKUP["Case lookup\nCase Search REST\nVISA: networkCaseId\nOthers: ARN"]

    CASE_EXISTS{{"Existing case\nfound?"}}

    CREATE_CASE["Create new case\nCreate Case REST\nWorkflow Rule Lookup REST\nNew Case Action Rules REST\nPre-Action Status REST"]

    UPDATE_CASE["Update existing case\nAdd Action REST\nCase Action Search REST\nNew Case Action Rules REST\nPre-Action Status REST"]

    NOTES{{"dataRecord\nnon-blank?"}}
    INSERT_NOTES["Insert Notes REST"]

    MISMATCH{{"Field mismatch\ndetected?"}   }
    API_LOG["API Log REST\nLog mismatch diagnostic"]

    WL_PROCESS["Look up existing case\nCase Search REST\nDerive actionCode,\ncaseLiability, owner\nUpdate Case REST"]

    SRV118_PROCESS["CBK Response Rules REST\nConstruct case actions\nwriteOff, representment,\nmerchantCharge, MACP, MDCL\nCreate/Update Case REST"]

    TERMINAL["SEND_BUSINESS_RULES_INFO_TO_KAFKA\nTerminal state — no-op\nNo Kafka publish occurs"]

    ERR_WRITE["Write error record to\nNAP.DISPUTE_EVENT_CONSUMER_ERROR\nRaw Kafka JSON payload stored\nStatus: FAILED1 → FAILED2 → ERROR"]

    DONE["Processing complete\nOffset already committed"]

    IN --> ACK
    ACK --> PRIOR
    PRIOR -->|"Yes"| BLOCK
    PRIOR -->|"No"| CLASSIFY

    CLASSIFY -->|"outcome non-blank"| WINLOSS
    CLASSIFY -->|"napResponseType AND\nnapResponseCode non-blank"| SRV118
    CLASSIFY -->|"all other cases"| SRV116

    WINLOSS --> WDPONLY_WL
    WDPONLY_WL -->|"Yes"| SKIP_WL
    WDPONLY_WL -->|"No"| OUTCOME_CHECK
    OUTCOME_CHECK -->|"Yes"| SKIP_OUTCOME
    OUTCOME_CHECK -->|"No"| WL_PROCESS

    SRV118 --> WDPONLY_118
    WDPONLY_118 -->|"Yes"| SKIP_118
    WDPONLY_118 -->|"No"| SRV118_PROCESS

    SRV116 --> MCM
    MCM -->|"Yes"| MCM_LOOKUP
    MCM -->|"No"| CASE_LOOKUP
    MCM_LOOKUP --> MCM_MULTI
    MCM_MULTI -->|"Yes"| MCM_HOLD
    MCM_MULTI -->|"No"| CASE_LOOKUP

    CASE_LOOKUP --> CASE_EXISTS
    CASE_EXISTS -->|"No"| CREATE_CASE
    CASE_EXISTS -->|"Yes"| UPDATE_CASE

    CREATE_CASE --> NOTES
    UPDATE_CASE --> NOTES
    NOTES -->|"Yes"| INSERT_NOTES
    NOTES -->|"No"| MISMATCH
    INSERT_NOTES --> MISMATCH
    MISMATCH -->|"Yes"| API_LOG
    MISMATCH -->|"No"| TERMINAL

    API_LOG --> TERMINAL
    WL_PROCESS --> TERMINAL
    SRV118_PROCESS --> TERMINAL

    TERMINAL --> DONE
    SKIP_WL --> DONE
    SKIP_118 --> DONE
    SKIP_OUTCOME --> DONE

    WL_PROCESS -->|"REST failure"| ERR_WRITE
    SRV118_PROCESS -->|"REST failure"| ERR_WRITE
    CREATE_CASE -->|"REST failure"| ERR_WRITE
    UPDATE_CASE -->|"REST failure"| ERR_WRITE
    MCM_HOLD --> DONE
    ERR_WRITE --> DONE
```

---

## Boundaries

### Inbound Interfaces

| Source | Protocol | Endpoint / Topic / Trigger | Payload / Description |
|--------|----------|----------------------------|-----------------------|
| COMP-04 NAPDisputeEventService | Kafka | `nap-dispute-events` (config key: `${kafka_consumer_topic}`) | `NotificationEvent` JSON — unified payload encoding SRV116, WIN/LOSS, and SRV118 sub-types. Partition key: merchantId |
| Operations team / tooling | REST | `POST /merchant/gcp/event-consumer/nap/event` | Manual reprocessing trigger. Accepts a `ConsumerErrorId` to re-trigger processing of a previously failed event from the DB error store |

### Outbound Interfaces

| Target | Protocol | Endpoint / Topic / Resource | Purpose | On failure |
|--------|----------|-----------------------------|---------|------------|
| IDP Token Service | REST GET | IDP token endpoint | Obtain JWT Bearer token for all subsequent outbound REST calls. ThreadLocal pattern, cleared in finally block. | ⚠️ NOT DOCUMENTED — failure handling not confirmed |
| Case Search | REST GET | Case search endpoint | Look up existing dispute case by ARN (non-VISA) or NetworkCaseId (VISA) | Write to DISPUTE_EVENT_CONSUMER_ERROR |
| Create Case | REST POST | Case creation endpoint | Create new dispute case when no existing case found (SRV116, first occurrence) | Write to DISPUTE_EVENT_CONSUMER_ERROR |
| Update Case / Add Action | REST POST | Case update endpoint | Append new action to an existing case | Write to DISPUTE_EVENT_CONSUMER_ERROR |
| Workflow Rule Lookup | REST POST | Workflow rules endpoint | Determine workflow name for new case creation | Write to DISPUTE_EVENT_CONSUMER_ERROR |
| Case Action Search | REST GET | Case action endpoint | Retrieve existing action summaries for a case | Write to DISPUTE_EVENT_CONSUMER_ERROR |
| New Case Action Rules | REST GET | Action rules endpoint | Determine stage code, action code, owner, SLA days | Write to DISPUTE_EVENT_CONSUMER_ERROR |
| CBK Response Rules | REST GET | CBK rules endpoint | Look up chargeback response rules (SRV118 path only) | Write to DISPUTE_EVENT_CONSUMER_ERROR |
| Pre-Action Status Rules | REST POST | Pre-action status endpoint | Determine prior action status before new action | Write to DISPUTE_EVENT_CONSUMER_ERROR |
| Insert Notes | REST POST | Notes endpoint | Add free-text note from `dataRecord` field (when non-blank) | Write to DISPUTE_EVENT_CONSUMER_ERROR |
| API Log | REST POST | API log endpoint | Log field-mismatch diagnostic information (reversalIndicator, merchantId, transactionType) | ⚠️ NOT DOCUMENTED |
| `NAP.DISPUTE_EVENT_CONSUMER_ERROR` | PostgreSQL | NAP schema | Database DLQ — write failed event records with raw Kafka JSON payload for manual reprocessing | N/A — this IS the failure target |
| `nap.mcm_queue_data` | PostgreSQL | NAP schema | MCM Queue lookup for MASTERCARD/MAESTRO SRV116 events — resolve claim ID | ONHOLD + error record |

**Note on REST dependencies:** All 11 REST calls use a bare `RestTemplate` with no connection timeout, read timeout, or connection pool configuration. No circuit breaker is present. A hung downstream call blocks the single consumer thread indefinitely.

---

## Database Ownership

### Tables Owned (written by this component)

| Schema.Table | Purpose | Key Columns | Retention / Notes |
|--------------|---------|-------------|-------------------|
| `NAP.DISPUTE_EVENT_CONSUMER_ERROR` | Database DLQ for failed Kafka events. Stores raw Kafka JSON payload + error metadata for manual reprocessing. Also used to track prior unresolved errors that block processing of new events for the same ARN/NetworkCaseId | error_id (PK), raw_payload, arn / network_case_id, C_ERROR_REASON, status (FAILED1 / FAILED2 / ERROR / DRAFT / SKIPPED / SKIPPED_FAILED / PROCESSED), error_timestamp | Retention policy ⚠️ NOT DOCUMENTED. Manual reprocessing via REST endpoint clears records to PROCESSED status |

### Tables Read (not owned by this component)

| Schema.Table | Owned by | Why accessed |
|--------------|----------|--------------|
| `nap.mcm_queue_data` | ⚠️ Owner NOT DOCUMENTED | MCM Queue lookup for MASTERCARD/MAESTRO SRV116 events — resolves ARN to MCM claim ID before case lookup |
| `nap.timeframe_rules` (⚠️ name NOT DOCUMENTED) | ⚠️ Owner NOT DOCUMENTED | Timeframe rules lookup — table name not confirmed from WDP-COMPONENTS.md |

---

## Configuration and Scaling

| Parameter | Value | Notes |
|-----------|-------|-------|
| Replica count | XL Deploy placeholder | Injected at deploy time per environment. Actual counts not confirmed |
| HPA | None | Scaling is manual replica count change only. No KEDA or lag-based autoscaling |
| Memory request | 2048Mi | |
| Memory limit | 4096Mi | |
| CPU request | Not set | Pod is Burstable QoS class — first candidate for eviction under node memory pressure |
| CPU limit | Not set | JVM CPU consumption is unbounded |
| Deployment type | Kubernetes Deployment | Not a StatefulSet |
| Rollout strategy | RollingUpdate — maxSurge: 1, maxUnavailable: 0 | |
| PodDisruptionBudget | None | Combined with at-most-once delivery, node drains can cause permanent event loss |
| Topology spread | ScheduleAnyway (soft constraint, max skew 1) | No label mismatch documented (contrast with COMP-01, COMP-03) |
| Kafka consumer concurrency | 1 (single thread per pod, default) | Throughput scales only with pod count |
| Offset commit mode | `MANUAL_IMMEDIATE`, `syncCommits=true` | Pre-ACK before processing — deviation from DEC-005 |
| Auto commit | false | |
| Auto offset reset | latest | |
| MSK auth | IAM (SASL_SSL / AWS_MSK_IAM) | |
| Deserializer | `JsonDeserializer<NotificationEvent>` wrapped in `ErrorHandlingDeserializer` | |
| Retry | Spring Retry `@Retryable` / `@Recover` on `NapEventProcessingService` | Count: `${consumer_retrycount}`, Delay: `${consumer_retrydelay}`. Production values not confirmed from repository |
| Observability | ⚠️ NOT DOCUMENTED | OpenTelemetry agent presence not confirmed from WDP-COMPONENTS.md for this component |

---

## Key Architectural Decisions

| Decision | ADR reference | Notes |
|----------|---------------|-------|
| At-most-once Kafka delivery — offset committed BEFORE processing begins | DEC-005 — **DEVIATION** | Deliberate decision to avoid Kafka redelivery. Error state managed via `DISPUTE_EVENT_CONSUMER_ERROR` DB table instead. Highest severity risk: event permanently lost if pod OOM-killed after ACK but before processing |
| Database-backed DLQ over Kafka DLQ | Platform pattern | No Kafka dead-letter topic. All failed events stored in PostgreSQL with full raw payload for manual reprocessing via REST endpoint |
| No circuit breaker | DEC-014 — **DEVIATION** | No Resilience4j dependency in pom.xml. Bare RestTemplate with no timeouts. A single hung downstream call blocks the entire consumer thread. With concurrency=1 this halts all message processing |
| Bare RestTemplate with no timeouts | Local decision | No connection or read timeout configured. max.poll.interval.ms violation risk under hung downstream calls causes consumer group rebalance |
| Single-threaded processing (concurrency=1) | Local decision | Throughput scales only with pod count. No KEDA or lag-based autoscaling |
| Hybrid migration state gated by wdpOnly flag | Local decision | WIN/LOSS and SRV118 skipped for wdpOnly=Y events. Active CB911 migration in progress |
| No EncryptionService call | Platform — NAP specifics | NAP dispute data contains no full PAN. EncryptionService (COMP-35) is not called on this path |
| SEND_BUSINESS_RULES_INFO_TO_KAFKA is a no-op terminal state | Local decision | Pipeline sets this as final apiName after every successful execution but no Kafka publish occurs. Infrastructure present but feature not implemented |
| Single unified topic for all NAP event sub-types | Local decision | SRV116, WIN/LOSS, and SRV118 all arrive on a single topic as the same `NotificationEvent` payload. Sub-type is determined at runtime by field inspection |

---

## Risks and Constraints

| Severity | Risk | Consequence |
|----------|------|-------------|
| 🔴 HIGH | At-most-once delivery: offset acknowledged BEFORE processing begins. If JVM crashes or pod is OOM-killed after ACK but before processing completes, the event is permanently lost. The error table is only written during processing, not before | Unrecorded chargebacks — dispute events silently dropped with no broker-level safety net. No recovery path. Directly conflicts with DEC-005 |
| 🔴 HIGH | Bare RestTemplate with no connection or read timeout and no circuit breaker (DEC-014 deviation). A single hung downstream REST call blocks the entire consumer thread indefinitely. With concurrency=1 this halts all message processing for all events on that pod | max.poll.interval.ms eventually exceeded causing consumer group rebalance and partition re-assignment. All NAP inbound processing on that pod stops |
| 🟡 MEDIUM-HIGH | Recursive error reprocessing: `checkAndProcessConsumerErrorSrvNotification()` iterates all prior ConsumerError records for an ARN and calls the full pipeline for each. For an ARN with many accumulated error records this creates a large synchronous HTTP call tree inside the single Kafka consumer thread | max.poll.interval.ms violation risk and consumer group rebalance under high accumulated error counts |
| 🟡 MEDIUM | No PodDisruptionBudget configured. Kubernetes node drains will evict all pods simultaneously | Combined with at-most-once delivery, any mid-processing eviction causes permanent event loss |
| 🟡 MEDIUM | No CPU limits or requests. Pod is Burstable QoS class | First candidate for eviction under node memory pressure. JVM CPU consumption is unbounded. CPU-based HPA cannot be configured |
| 🟡 MEDIUM | MCM Queue multi-result silent ONHOLD: if an ARN maps to more than one MCM claim ID, the event is silently placed ONHOLD with a misleading error message ("MCM claim id not exist") even though data does exist | MASTERCARD/MAESTRO disputes silently delayed until napEventExpiryHour timeout — no operator alert |
| 🟡 MEDIUM | SEND_BUSINESS_RULES_INFO_TO_KAFKA is dead code in production. Infrastructure exists (entity, repository, service methods) but no Kafka publish ever occurs | Downstream consumers expecting a business-rules Kafka message will never receive one. Silent incomplete feature |
| 🟢 LOW | Shared RestTemplate error handler mutation: `IdpRestInvoker` calls `restTemplate.setErrorHandler()` on a shared singleton bean. Not thread-safe if concurrency > 1 | Currently benign at concurrency=1 but a latent race condition if concurrency is ever increased |
| 🟢 LOW | ThreadLocal IDP JWT token not cleared on exception in manual reprocessing path. The finally block exists on the Kafka listener path but not on `processConsumerErrorInformation()` REST path | Stale token may be used on subsequent manual reprocessing calls from the same thread |

---

## Planned Changes

- **Active migration — CB911 merchant migration:** `wdpOnly` flag gates ongoing migration. Non-migrated CB911 merchants (srv117 and win-loss event types) still processed via this service. Migration timeline: TBD. Component scope expected to reduce as migration progresses.
- **Commented-out Fraud Switch Lookup:** `FRAUD_SWITCH_LOOKUP` pipeline step exists as a constant and has URL configuration in `application.yaml` but the call site in `caseLookup()` is commented out. URL placeholder must still be provided as env-var even though it is never called.
- **Commented-out VISA Case Lookup:** `visaCaseLookup()` method body and both call sites are commented out. Superseded by general `caseLookup()` handling VISA via `networkCaseId`.
- **Amount calculation logic replacement (US2137854):** Old `calculateWorkableAmount()` replaced by direct field mapping. Dead method body remains in codebase.
- **SEND_BUSINESS_RULES_INFO_TO_KAFKA feature:** Infrastructure present but not implemented. No timeline confirmed.
- **JCB and VISA_ALLOCATION workflow types:** Constants in `CaseServiceSupport` suggest active development for JCB and VISA allocation flows beyond current VISA/MASTERCARD/MAESTRO/AMEX handling. Status not confirmed.
- ⚠️ OPEN QUESTION: Whether application-level authentication should be enforced on the manual reprocessing REST endpoint — currently undocumented.

---

---

## ━━━ TYPE BLOCK A — REST API CONTRACTS ━━━━━━━━━━━━━━━━━━━

---

## REST API Contracts

**Authentication model:**
⚠️ NOT DOCUMENTED — authentication requirement for the manual reprocessing
endpoint is not confirmed from WDP-COMPONENTS.md. The endpoint is not
described as requiring a Bearer JWT, but this has not been confirmed from
source. The Kafka listener path acquires a JWT via IDP Token Service for
outbound calls only.

**Base URL pattern:**
`/merchant/gcp/event-consumer/nap`

---

### Endpoint: `POST /merchant/gcp/event-consumer/nap/event`

**Purpose:** Manual reprocessing — re-trigger processing of a previously
failed event from the database error store

**Caller(s):** Operations team or tooling. Not called by any WDP component
in the normal processing path.

**Auth required:** ⚠️ NOT DOCUMENTED

**Request**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ConsumerErrorId` | ⚠️ NOT DOCUMENTED | Yes | Identifies the `DISPUTE_EVENT_CONSUMER_ERROR` record to reprocess |

*Exact request body structure (JSON / path param / query param) not
confirmed from WDP-COMPONENTS.md.*

**Response — Success**

| HTTP Status | Condition | Body |
|-------------|-----------|------|
| ⚠️ NOT DOCUMENTED | Successful reprocessing | ⚠️ NOT DOCUMENTED |

**Response — Error**

| HTTP Status | Condition | Body |
|-------------|-----------|------|
| ⚠️ NOT DOCUMENTED | Processing failure on reprocessing path | Error details returned in HTTP response body — component does NOT write a new DB record on manual reprocessing failure; error is returned to caller directly |

**Notes:**
In manual reprocessing mode the component does not write a new DB record
on failure — it returns error details to the caller in the HTTP response.
The IDP JWT ThreadLocal token is NOT cleared in a finally block on this
path (confirmed risk — see Risks and Constraints).

---

---

## ━━━ TYPE BLOCK B — KAFKA CONSUMER CONTRACTS ━━━━━━━━━━━━━

---

## Kafka Consumer Contracts

**Consumer framework:** Spring Kafka `@KafkaListener`
**Offset commit strategy:** `MANUAL_IMMEDIATE` with `syncCommits=true` — Pre-ACK BEFORE processing begins. **Deliberate deviation from DEC-005** (platform standard is manual commit AFTER full processing).
**Error handling strategy:** Database DLQ table (`NAP.DISPUTE_EVENT_CONSUMER_ERROR`). No Kafka DLQ topic. Raw Kafka JSON payload stored for manual reprocessing.

---

### Topic: `nap-dispute-events`

| Parameter | Value |
|-----------|-------|
| **Topic name** | `nap-dispute-events` (config key: `${kafka_consumer_topic}`) |
| **Consumer group** | `${kafka_group_id}` — injected via Kubernetes secret. Actual group ID not confirmed from repository |
| **Partition key** | `merchantId` (set by COMP-04 NAPDisputeEventService as producer) |
| **Concurrency** | 1 (single thread per pod, default) |
| **Max poll records** | ⚠️ NOT DOCUMENTED — default (500) assumed |
| **Max poll interval** | ⚠️ NOT DOCUMENTED — default (5 min) assumed. Risk: recursive error reprocessing may breach this limit |
| **Offset commit** | `MANUAL_IMMEDIATE`, `syncCommits=true` — pre-ACK BEFORE processing. Deviation from DEC-005 |
| **Auto commit** | `false` |
| **Auto offset reset** | `latest` |
| **MSK auth** | IAM (SASL_SSL / AWS_MSK_IAM) |
| **Deserializer** | `JsonDeserializer<NotificationEvent>` wrapped in `ErrorHandlingDeserializer` |
| **Ordering guarantee** | Per partition (by merchantId) — guaranteed within partition |

**Message payload structure**

Single unified `NotificationEvent` payload encoding all three sub-types.
Key fields used for classification and processing:

| Field | Type | Description |
|-------|------|-------------|
| `outcome` | String | Non-blank → WIN/LOSS sub-type. Values: NA, CLOSED (skipped), ARB, other |
| `napResponseType` | String | Non-blank (combined with napResponseCode) → SRV118 sub-type |
| `napResponseCode` | String | Non-blank (combined with napResponseType) → SRV118 sub-type |
| `arn` | String | Acquirer Reference Number — case lookup key for non-VISA networks |
| `networkCaseId` | String | Visa-specific case lookup key |
| `acquirerCaseNumber` | String | Used in SRV116 duplicate detection |
| `uniqueId` | String | Used in SRV116 duplicate detection |
| `stageCode` | String | Used in SRV116 duplicate detection |
| `actionCode` | String | Used in SRV116 duplicate detection |
| `creditDebitIndicator` | String | Used in SRV116 duplicate detection |
| `dataRecord` | String | Free-text notes field — triggers Insert Notes REST call when non-blank |
| `cardNetwork` | String | Routing: MASTERCARD/MAESTRO triggers MCM Queue lookup; VISA uses networkCaseId for case lookup |
| `migrationStatus` | String | WIN/LOSS: events where max action has migrationStatus='Y' are silently skipped |
| `wdpOnly` | String | 'Y' = legacy migrated merchant — WIN/LOSS and SRV118 paths skipped entirely |
| `reversalIndicator` | String | Mismatch detected → API Log REST call |
| `merchantId` | String | Mismatch detected → API Log REST call |
| `transactionType` | String | Mismatch detected → API Log REST call |
| `sourceSystemCaseId` | String | ⚠️ Noted in COMP-04 context — presence in NotificationEvent not confirmed |
| `enrichmentFailure` | Boolean | ⚠️ Noted in COMP-04 as carried on Kafka event — handling by this consumer NOT DOCUMENTED |

**Event classification / routing**

Sub-type is determined at runtime by field inspection on the incoming
`NotificationEvent` payload. Precedence order:

1. If `outcome` field is non-blank → **IN_WIN_LOSS**
2. Else if both `napResponseType` AND `napResponseCode` are non-blank → **IN_SRV118**
3. All other cases → **IN_SRV116** (default — new chargeback notification)

**On processing failure**

| Failure scenario | Behaviour |
|-----------------|-----------|
| Downstream REST call fails (any of the 11 REST dependencies) | Write `ConsumerError` row to `NAP.DISPUTE_EVENT_CONSUMER_ERROR` with raw Kafka JSON payload and C_ERROR_REASON identifying which step failed. Status: FAILED1 → FAILED2 → ERROR on repeated failures. Offset already committed (pre-ACK) |
| SRV116: ARN maps to multiple MCM claim IDs | Place event ONHOLD with error message "MCM claim id not exist". Write error record. Recover after napEventExpiryHour |
| SRV116: Duplicate detected (same acquirerCaseNumber + uniqueId + stageCode + actionCode + creditDebitIndicator) | Skip — no re-processing |
| WIN/LOSS: outcome = NA or CLOSED | Skip silently |
| WIN/LOSS: max action migrationStatus = 'Y' | Skip silently |
| SRV116 / WIN/LOSS / SRV118: wdpOnly = 'Y' (WIN/LOSS or SRV118 only) | Skip — WIN/LOSS and SRV118 paths not processed for wdpOnly events |
| Prior unresolved ERROR records for same ARN/NetworkCaseId | Block new event — write DRAFT status to error table. New event not processed until prior errors resolved |
| Manual reprocessing failure | Error details returned in HTTP response to caller. No new DB record written |
| Deserialization error | Handled by `ErrorHandlingDeserializer` — ⚠️ behaviour on deserialization failure NOT DOCUMENTED from WDP-COMPONENTS.md |

---

*End of WDP-COMP-05-NAP-DISPUTE-EVENT-PROCESSOR.md*
*File status: 📝 DRAFT — pending architect confirmation*
*Update WDP-COMP-INDEX.md status, WDP-KAFKA.md, and WDP-DB.md entries
after confirmation.*
