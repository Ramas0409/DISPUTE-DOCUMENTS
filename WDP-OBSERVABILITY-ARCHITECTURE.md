# WDP Observability Architecture
**Worldpay Dispute Platform — Observability, Alerts & Monitoring Design**
*Version: Draft v1.0 | April 2026*
*Status: For Reviewer Feedback*

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Observability Stack](#2-observability-stack)
3. [Platform Signal Pipeline](#3-platform-signal-pipeline)
4. [Cross-Cutting Principles](#4-cross-cutting-principles)
5. [S1 — Edge & Access Layer](#5-s1--edge--access-layer)
6. [S2 — Inbound Processing](#6-s2--inbound-processing)
7. [S3 — Kafka Event Bus](#7-s3--kafka-event-bus)
8. [S4 — Core Lifecycle Consumers](#8-s4--core-lifecycle-consumers)
9. [S5 — Business & Supporting Services](#9-s5--business--supporting-services)
10. [S6 — Outbound & Notification Consumers](#10-s6--outbound--notification-consumers)
11. [S7 — Data & Integration Boundaries](#11-s7--data--integration-boundaries)
12. [Alert Catalogue](#12-alert-catalogue)
13. [Dashboard Hierarchy](#13-dashboard-hierarchy)
14. [Open Questions & Prerequisites](#14-open-questions--prerequisites)

---

## 1. Executive Summary

This document defines the observability architecture for the Worldpay Dispute Platform (WDP). It covers all 50 components across the full dispute lifecycle — from inbound event ingestion through case creation, business rule evaluation, notification orchestration, and outbound delivery to acquiring platforms and card networks.

The platform is divided into seven observability sections aligned with architectural boundaries. Each section is designed, instrumented, and alertable independently while contributing to a unified platform health view in Grafana.

### Key architectural facts that drive this design

**At-most-once Kafka delivery is the platform reality.** Every confirmed Kafka consumer deviates from the platform standard (DEC-005). All consumers use pre-ACK or mid-flow ACK — committing offsets before processing completes. This means **consumer group lag=0 is not a safe signal for processing success**. Every Kafka metric must be paired with a database-level error table metric.

**There are no Kafka Dead Letter Topics.** All consumer failure handling is database-backed. Errors go to `wdp.chbk_outbox_row`, `wdp.outgoing_event_outbox`, `NAP.DISPUTE_EVENT_CONSUMER_ERROR`, or per-component error tables. Kafka DLT lag is not observable because DTLs do not exist.

**DEC-001 (transactional outbox) is violated by eight components.** COMP-15, COMP-16, COMP-19, COMP-20, COMP-23, COMP-24, COMP-25, and COMP-37 all publish to Kafka synchronously without an outbox. Broker unavailability at the moment of publish causes permanent, unrecoverable message loss for these components. These require their own publish-success-rate alerts as P1 conditions.

**No circuit breakers exist anywhere on the platform.** DEC-014 (Resilience4j) is formally voided. No component uses circuit breaking, rate limiting, or bulkhead patterns. Timeouts are also absent from most outbound REST calls — COMP-14 has 11+ REST dependencies with no timeout configured, creating a thread-starvation risk that can halt all case creation from a single hung call.

**The primary platform SLO** is the `chbk_outbox_row` PENDING row age — the time between a dispute event being written to the transactional outbox and being published to Kafka. Target: < 2 minutes P95. All inbound processing metrics orient around this single threshold.

---

## 2. Observability Stack

| Component | Role |
|-----------|------|
| **OpenTelemetry SDK** | Instrumentation layer in every WDP JVM. Emits metrics, traces, and logs via OTLP to the Collector. |
| **OTel Collector** | Central aggregation and routing. Receives OTLP from all components; exports metrics to Prometheus, traces to Tempo, logs to Loki. Also scrapes AWS CloudWatch metrics (MSK, S3, DynamoDB, Aurora) via the CloudWatch Metrics Receiver. |
| **Prometheus** | Metrics storage and alerting engine. Scrapes from Collector remote_write endpoint. Hosts all PromQL-based alert rules. |
| **Tempo** | Distributed trace storage. Trace correlation by `trace_id` across all WDP services. |
| **Loki** | Structured log aggregation. All WDP component logs correlated with `trace_id`. |
| **Grafana** | Unified observability front-end. Queries Prometheus (metrics), Tempo (traces), Loki (logs). |
| **Alertmanager** | Alert routing from Prometheus. Severity-based routing to PagerDuty (P1), Slack (P2), Jira (P3). |

### Signal types

| Signal | What it covers | Primary source |
|--------|----------------|----------------|
| **Metrics** | Request rates, latencies (P50/P95/P99), error rates, queue depths, outbox row counts, batch job durations | OTel SDK (Micrometer bridge), CloudWatch receiver |
| **Traces** | Distributed spans across REST calls and Kafka produce/consume | OTel SDK with HTTP + Kafka instrumentation |
| **Logs** | Structured event logs, correlated with trace_id | OTel log bridge from SLF4J/Logback |

### Metric naming convention

All custom business metrics follow: `wdp_{section}_{entity}_{measurement}_{unit}`

Examples: `wdp_inbound_outbox_pending_age_seconds`, `wdp_gateway_upstream_call_duration_seconds`, `wdp_bre_kafka_publish_total`

---

## 3. Platform Signal Pipeline

```
WDP Components (50)
      │ OTLP gRPC
      ▼
OTel Collector ──── metrics ──► Prometheus ──► Grafana
                ──── traces ──► Tempo       ──► Grafana
                ──── logs   ──► Loki        ──► Grafana
                ──── (+ CloudWatch scrape for MSK / Aurora / S3 / DynamoDB)
```

**Sampling strategy for traces:**
- 100% capture: all 4xx/5xx responses, all processing durations above per-section P99 thresholds
- 10% capture: healthy 2xx requests for baseline latency analysis
- 100% capture from COMP-14: all requests > 5 seconds processing duration (thread-starvation detection)

---

## 4. Cross-Cutting Principles

These principles apply to all seven sections and must be maintained consistently across the entire alert catalogue.

### P1: Pre-ACK is universal — lag is not sufficient

Every Kafka consumer on the platform commits its offset before processing completes. A consumer group lag of zero does not confirm that messages were successfully processed. Every Kafka consumer section therefore requires **two paired metrics**: consumer group lag (from Kafka) and DB error table row count (from the consumer's own database). An alert fires only when both signals indicate a problem, or when either signal exceeds a safety threshold independently.

### P2: No circuit breakers — timeout and saturation are manual concerns

No component has Resilience4j or any circuit-breaking mechanism. The observability layer must perform the function that circuit breakers normally provide: detect when a dependency is degraded and alert before the degradation cascades to the calling component's thread pool. Key patterns:

- Monitor P99 processing duration for all consumers with synchronous REST dependencies
- Alert on P99 approaching the Netty/thread-pool maximum for reactive components (COMP-01)
- Alert on P99 approaching thread exhaustion thresholds for blocking components (COMP-14)

### P3: DEC-001 violations require publish-success-rate alerts as P1

Eight components publish to Kafka synchronously without an outbox. For these components, a Kafka publish failure is **not retried and not recoverable**. The alert must be P1 on any sustained failure, not P2. The affected components are: COMP-15, COMP-16, COMP-19, COMP-20, COMP-23, COMP-24, COMP-25, COMP-37.

### P4: Last-mile delivery must be observed at the external system level

For all outbound consumers (S6), Kafka lag=0 confirms the message was consumed but does not confirm delivery to the external system (NAP-DPS, Signifyd, BEN MSK, IBM DB2). Each S6 component requires its own external delivery success rate metric derived from HTTP response codes, Kafka produce acknowledgements, or database write outcomes.

### P5: Severity tiers

| Tier | Condition | Response | Route |
|------|-----------|----------|-------|
| **P1** | Platform-impacting: data loss, delivery failure, single-thread stall | Page immediately | PagerDuty |
| **P2** | Degradation: SLO breach, error accumulation, dependency latency | Notify + ticket | Slack + Jira |
| **P3** | Informational: trend anomaly, drift from baseline | Log and observe | Jira |

---

## 5. S1 — Edge & Access Layer

**Components:** API Gateway (COMP-01), UAMS (COMP-02), CHAS (COMP-03), TokenService (COMP-36), Merchant Portal (COMP-49), Ops Portal (COMP-50)

### Architecture risks

| Risk | Severity | Detail |
|------|----------|--------|
| No timeout on UAMS/CHAS calls | P1 | API Gateway Step 4 calls UAMS or CHAS with no timeout. A hung auth service blocks Netty event loop threads indefinitely — thread starvation causes the gateway to become unresponsive to all traffic. |
| Blocking RestTemplate in reactive pipeline | P1 | Step 4 uses a blocking RestTemplate inside Spring WebFlux's non-blocking Netty pipeline. This converts a latency spike into a thread-blocking event. |
| NPE in Step 2 (null route) | P2 | LoggingGlobalFilter has no null guard on the route attribute. An unmatched route produces a 500. |
| NPE in Step 3 (missing AuthorizationList) | P2 | AuthorizationFilter throws NPE if the JWT lacks the AuthorizationList claim. Produces a 500. |
| CHAS DB2 failure masked | P1 | CHAS swallows DB2 connection failures via broad catch(Exception) on the chain lookup path and falls through to merchant-level lookup. DB2 failures are invisible to callers. |
| TokenService unknown Redis writer | P2 | The external component that writes the Redis JWT cache is unidentified. If it fails, all WDP services fall through to live IDP calls. |

### Key metrics

```
# API Gateway
wdp_gateway_requests_total{route, method, http_status_code, region, platform}
wdp_gateway_request_duration_seconds{route, http_status_code}           # P50/P95/P99
wdp_gateway_filter_duration_seconds{filter_step}                         # per filter
wdp_gateway_upstream_call_duration_seconds{upstream, outcome}            # UAMS / CHAS
wdp_gateway_upstream_errors_total{upstream, error_type}
wdp_gateway_active_connections                                           # Netty gauge
wdp_gateway_npe_total{source}                                            # route_null | auth_list_absent

# UAMS / CHAS
wdp_uams_authorize_duration_seconds{outcome}
wdp_uams_authorize_error_total{error_type}
wdp_chas_authorize_duration_seconds{lookup_path}                         # chain_success | chain_fallback
wdp_chas_db2_exception_total{query_type}                                 # chain | merchant — CRITICAL

# TokenService
wdp_token_cache_requests_total{cache_layer}                              # redis | memory | idp
wdp_token_redis_available                                                # gauge 0/1
wdp_token_idp_call_rate                                                  # should be near-zero normally
```

### Alert thresholds

| Alert | Severity | Condition |
|-------|----------|-----------|
| Gateway UAMS/CHAS call P95 > 3s | P1 | Pre-warning of thread starvation |
| Netty active connections > 80% of pool max | P1 | Thread saturation imminent |
| Gateway 5xx rate > 2% over 5 min | P1 | NPE or upstream failure |
| `wdp_chas_db2_exception_total` > 0 | P1 | DB2 failure masked — compliance risk |
| `wdp_token_cache_requests_total{cache_layer="idp"}` > 5% of total | P1 | Redis writer has failed |
| Gateway 4xx rate > 10% over 10 min | P2 | JWT storm or routing misconfiguration |
| `wdp_gateway_npe_total{source="route_null"}` > 0 | P2 | Route missing from wdp.api_route |
| Portal P95 response time > 2s on dispute detail | P2 | UI SLO breach |

### Distributed tracing for S1

Trace every request end-to-end: NGINX → API Gateway → filter pipeline → UAMS/CHAS → backend service. Instrument all four filter steps as named spans. Capture 100% of 4xx/5xx responses and 100% of upstream calls exceeding 1 second. Sample 10% of healthy requests.

---

## 6. S2 — Inbound Processing

**Components:** NAPDisputeEventService (COMP-04), NAPDisputeEventProcessor (COMP-05), NAPDisputeDeclineBatch (COMP-06), VisaDisputeBatch (COMP-07), FirstChargebackBatch (COMP-08), CaseFillingBatch (COMP-09), FileProcessor (COMP-11), InboundDisputeEventScheduler (COMP-12), FileAcknowledgementProcessor (COMP-13)

### Architecture risks

| Risk | Severity | Detail |
|------|----------|--------|
| Silent null-return in VisaDisputeBatch | P2 | Enrichment failures return null — item skipped with ELK log only. No dead-letter store. Visa disputes are permanently lost. |
| `removeItemFromQueueDisabled` flag active | P2 | Global MarkAsRead killswitch. When true, items are consumed from Visa/MC queues but never acknowledged. Confirmed active production configuration in COMP-07 and COMP-08. |
| FileProcessor sequential processing | P2 | maxConcurrentMessages=1. SQS queue growth = file processing backlog. |
| DNWK NPE on unregistered sub-source | P1 | NetworkServiceImpl returns null for AMEXOPTB / MC_REVREJ / DISCHYB → NPE → file_job=ERROR, file silently lost. |
| SQS visibility timeout no heartbeat | P2 | Extended once at start — no heartbeat. Large files risk double-processing if processing time exceeds timeout. |
| No distributed locking on Scheduler1 | P2 | Multiple replicas can drain the same PENDING rows simultaneously. DEC-005 pre-ACK means race creates duplicate publishes or missed publishes. |
| BLOCKED evidence rows never unblocking | P2 | Scheduler1 sets PUBLISHED, not SUCCESS. Downstream consumer (COMP-14, unconfirmed) must set SUCCESS for evidence gate to open. |
| chbk_outbox_row archive no purge policy | P3 | Archive table grows unboundedly. No purge is confirmed. |

### Primary SLO metric

`wdp_inbound_outbox_pending_age_seconds` — P95 < 120 seconds.

This single metric is the most important inbound health signal on the platform. It measures the age of PENDING rows in `wdp.chbk_outbox_row` from write timestamp to Kafka publish. When this breaches 120 seconds, cases are being delayed. When it breaches 300 seconds, something is seriously wrong with the outbox relay.

### Key metrics

```
# Outbox (primary SLO)
wdp_inbound_outbox_pending_count                                         # gauge
wdp_inbound_outbox_pending_age_seconds                                   # histogram
wdp_inbound_outbox_failed_count{event_type}                              # gauge — retry queue
wdp_inbound_outbox_error_count{event_type}                               # gauge — terminal failures
wdp_inbound_outbox_blocked_count                                         # gauge — evidence gate
wdp_inbound_outbox_publish_rate                                          # counter

# VisaDisputeBatch (COMP-07)
wdp_visa_batch_last_run_timestamp                                        # gauge — batch heartbeat
wdp_visa_batch_items_total{outcome}                                      # outbox_written | null_return_skipped | error
wdp_visa_batch_mark_as_read_suppressed_total                             # flag active indicator
wdp_visa_batch_queue_fetch_duration_seconds{queue_name}                  # DataPower latency

# FileProcessor (COMP-11)
wdp_file_processor_sqs_queue_depth                                       # gauge — from CloudWatch
wdp_file_processor_file_job_total{source_type, outcome}                  # DNWK error = P1
wdp_file_processor_rows_written_total{source_type}

# InboundEventScheduler (COMP-12)
wdp_scheduler1_kafka_publish_total{outcome, topic}
wdp_scheduler1_evidence_unblocked_total                                  # Phase 2 gate opens
wdp_scheduler2_stuck_processing_jobs                                     # gauge — staleness guard absent
```

### Alert thresholds

| Alert | Severity | Condition |
|-------|----------|-----------|
| `wdp_inbound_outbox_pending_age_seconds` P95 > 300s | P1 | Scheduler1 broken or severely behind |
| `wdp_file_processor_file_job_total{source_type="DNWK", outcome="error"}` > 0 | P1 | File silently lost — NPE on DNWK sub-source |
| Scheduler1 `kafka_publish_total{outcome="failure"}` > 0 sustained 3 min | P1 | Outbox relay broken |
| `wdp_visa_batch_last_run_timestamp` silence > 5 min | P1 | VisaDisputeBatch cron stopped |
| `wdp_inbound_outbox_error_count` growing over 10 min | P1 | Terminal failures accumulating |
| `wdp_inbound_outbox_pending_age_seconds` P95 > 120s | P2 | SLO breach |
| SQS queue depth > 10 and growing | P2 | File processing backlog |
| `wdp_visa_batch_items_total{outcome="null_return_skipped"}` > 5/hr | P2 | Silent Visa dispute drops |
| `wdp_inbound_outbox_blocked_count` > 500 and stable > 15 min | P2 | Evidence gate stuck |
| `wdp_scheduler2_stuck_processing_jobs` > 0 for > 30 min | P2 | File jobs stuck in PROCESSING |

---

## 7. S3 — Kafka Event Bus

**Infrastructure:** AWS MSK (9 active topics, IAM auth, WDP-owned)

### The pre-ACK monitoring problem

Every WDP Kafka consumer deviates from the DEC-005 platform standard. The complete confirmed deviation map:

| Component | Topic | ACK timing |
|-----------|-------|------------|
| COMP-05 NAPDisputeEventProcessor | nap-dispute-events | Pre-ACK |
| COMP-12 InboundDisputeEventScheduler | (producer: mark before send) | Mark-before-send |
| COMP-14 CaseCreationConsumer | new-case-events | Pre-ACK |
| COMP-15 EvidenceConsumer | case-evidence-events | Pre-ACK + syncCommits |
| COMP-16 BusinessRulesProcessor | business-rules | Pre-ACK |
| COMP-17 CaseExpiryUpdateConsumer | case-action-events | Inconsistent (Path A: pre / Path B: mid) |
| COMP-18 NotificationOrchestrator | outgoing-events | Mid-flow (after outbox INSERT, before publishes) |
| COMP-39 NAPOutcomeProcessor | internal-integration-events | Pre-ACK |
| COMP-40 VisaResponseQuestionnaire | internal-integration-events | Pre-ACK |

**Alert design consequence:** Alert on lag ONLY when lag is rising AND offset commit rate is declining simultaneously. Rising lag with stable commit rate = production surge (capacity issue). Rising lag with zero commit rate = consumer dead or stalled.

### Topic registry with risk flags

| Topic | Publisher(s) | Consumer(s) | Risk flags |
|-------|-------------|-------------|------------|
| nap-dispute-events | COMP-04 (DEC-001) | COMP-05 | decom · DEC-001 |
| new-case-events | COMP-12 Scheduler1 | COMP-14 | pre-ACK · concurrency=1 |
| case-evidence-events | COMP-12 Scheduler1 | COMP-15 | pre-ACK · concurrency=1 |
| business-rules | 5 publishers (4× DEC-001) | COMP-16 | 5 pub · 4× DEC-001 |
| outgoing-events | COMP-16 (DEC-001) | COMP-18 | DEC-001 |
| internal-integration-events | COMP-18 | COMP-39, COMP-40 | 2 consumers · pre-ACK |
| case-action-events | COMP-18 | COMP-17 | inconsistent ACK |
| external-request-events | COMP-18 | COMP-41, COMP-42, COMP-44 | 3 consumers · ACK TBC |
| core-request-events | COMP-18 | COMP-43 | ACK TBC |

### Key metrics

```
# MSK infrastructure (from CloudWatch via OTel Collector)
aws_msk_broker_offline_partitions_count                                  # must be 0
aws_msk_broker_under_replicated_partitions                               # must be 0
aws_msk_broker_active_controller_count                                   # must be 1
aws_msk_broker_bytes_in_per_sec{broker}
aws_msk_broker_bytes_out_per_sec{broker}

# Consumer group lag (from Kafka consumer group offset API)
kafka_consumer_group_lag{topic, partition, group}                        # per consumer group
kafka_consumer_group_lag_sum{topic, group}                               # aggregated
kafka_consumer_group_offset_commits_per_sec{topic, group}               # paired with lag

# Per-topic production rate
kafka_topic_messages_in_per_sec{topic}                                   # leading indicator
```

### Alert thresholds

| Alert | Severity | Condition |
|-------|----------|-----------|
| `aws_msk_broker_offline_partitions_count` > 0 | P1 | Broker data unavailable |
| `aws_msk_broker_active_controller_count` = 0 | P1 | No active controller — all ops fail |
| Production rate on `new-case-events` or `case-evidence-events` drops to 0 for > 2 min | P1 | Scheduler1 stopped relaying |
| Consumer lag on critical topics > 1,000 messages | P1 | Consumer severely behind |
| `kafka_topic_messages_in_per_sec{topic="outgoing-events"}` drops to 0 while BRP consuming | P2 | COMP-16 DEC-001 publish failing |
| `aws_msk_broker_under_replicated_partitions` > 0 sustained > 5 min | P2 | Broker health degrading |
| Consumer lag rising AND offset commit rate declining on any topic | P2 | Consumer stalled |

### Kafka/DB dual-view principle

For each topic, the S3 Grafana panel shows:
- **Left Y-axis:** consumer group lag (from Kafka)
- **Right Y-axis:** DB error table count for that consumer's errors (from PostgreSQL)

Both near-zero = healthy. Lag near-zero but DB errors growing = pre-ACK gap materialised.

---

## 8. S4 — Core Lifecycle Consumers

**Processing spine:** new-case-events → business-rules → outgoing-events → case-action-events

**Components (in pipeline order):**
1. CaseCreationConsumer (COMP-14) — new-case-events
2. EvidenceConsumer (COMP-15) — case-evidence-events → publishes to business-rules
3. BusinessRulesProcessor (COMP-16) — business-rules → publishes to outgoing-events
4. NotificationOrchestrator (COMP-18) — outgoing-events → fans out to 4 topics
5. CaseExpiryUpdateConsumer (COMP-17) — case-action-events

### Architecture risks

| Risk | Component | Severity | Detail |
|------|-----------|----------|--------|
| No timeout on 11+ REST deps, concurrency=1 | COMP-14 | P1 | Single hung REST call halts ALL case creation for ALL merchants platform-wide. |
| DEC-001: business-rules publish inside @Transactional | COMP-15 | P2 | Ghost event possible if JPA transaction rolls back after Kafka send. |
| MISCDOC/DRFTDOC silent drop | COMP-15 | P2 | Documents of these types fall through all upload conditionals with no error and no metric. Custom counter must be added. |
| DEC-001: direct publish to outgoing-events | COMP-16 | P1 | Broker unavailability = permanent loss of outgoing-events message — no recovery path. |
| No idempotency on BRP | COMP-16 | P2 | Message redelivery causes rule re-evaluation and re-application. |
| Mid-flow ACK before downstream publishes | COMP-18 | P1 | Crash after ACK = PUBLISHED orphan row in bre_orchestration_outbox — Scheduler4 will never re-drive. |
| Inconsistent ACK: Path A pre / Path B mid | COMP-17 | P2 | Path B crash gap: case_expiry write after mid-flow ACK is unguarded — expiry record lost with no re-drive. |

### Key metrics

```
# COMP-14 CaseCreationConsumer
wdp_case_creation_processing_duration_seconds                            # histogram — thread starvation indicator
wdp_case_creation_outcomes_total{outcome}                                # case_created | failed | error | pending_deferred
wdp_case_creation_outbox_error_accumulation                             # gauge — terminal failures
wdp_case_creation_prior_check_blocked_total

# COMP-15 EvidenceConsumer
wdp_evidence_outcomes_total{outcome}
wdp_evidence_silent_drop_total{reason}                                   # MUST BE ADDED to codebase — gap today
wdp_evidence_s3_download_duration_seconds
wdp_evidence_business_rules_publish_total{outcome}

# COMP-16 BusinessRulesProcessor
wdp_bre_kafka_publish_total{topic, outcome}                              # P1 on failure
wdp_bre_rule_evaluation_total{outcome, platform}                         # matched | no_match
wdp_bre_processing_duration_seconds{platform}                            # DB read per message
wdp_bre_db_query_duration_seconds                                        # rules table health

# COMP-18 NotificationOrchestrator
wdp_notif_orch_orphaned_published_rows                                   # gauge — crash window indicator
wdp_notif_orch_publish_total{target, outcome}                            # per fan-out destination
wdp_notif_orch_routing_flags                                             # coreMigration | disputesAPIMigration
wdp_notif_orch_deferred_total

# COMP-17 CaseExpiryUpdateConsumer
wdp_expiry_ack_path_total{path}                                          # A_pre_ack | B_mid_flow
wdp_expiry_case_expiry_write_total{outcome}
wdp_expiry_silent_discard_total{reason}
```

### Alert thresholds

| Alert | Severity | Condition |
|-------|----------|-----------|
| COMP-14 processing duration P99 > 60s | P1 | Thread stalled — case creation halted |
| COMP-16 `kafka_publish_total{outcome="failure"}` on outgoing-events | P1 | DEC-001 permanent loss |
| COMP-18 `orphaned_published_rows` growing over 15 min | P1 | Mid-flow crash window materialised |
| COMP-14 `outbox_error_accumulation` growing | P1 | Terminal failures on case creation |
| COMP-15 `silent_drop_total` > 0 | P2 | Evidence documents disappearing |
| COMP-16 rule no-match rate > 20% over 10 min | P2 | Rules config change or DB degradation |
| COMP-17 `case_expiry_write_total{outcome="failure"}` on Path B traffic | P2 | Crash gap materialised |
| COMP-18 `routing_flags` state change detected | P2 | Feature flag changed mid-traffic |

### Spine throughput correlation

The most valuable S4 dashboard panel is a multi-line time series showing processing rate for COMP-14, COMP-15, COMP-16, COMP-18, and COMP-17 on the same chart. In normal operation, these rates are broadly correlated. A rate divergence between consecutive consumers is the earliest signal that the spine is degrading.

---

## 9. S5 — Business & Supporting Services

**16 REST microservices in three tiers**

### Tier 1 — Shared infrastructure (called by many, P95 = platform SLO)

| Service | COMP | Key risk |
|---------|------|----------|
| EncryptionService | 35 | KMS call on every decrypt — no cache. P95 governs all case creation and evidence decryption. DEK rotation in days (not 6 hours as documented). No timeout on KMS calls. |
| CaseManagementService | 23 | Authoritative case record. No idempotency on case creation (DEC-020). Clear PAN written on standard create (DEC-019). No timeout on enrichment REST calls. |
| DocumentManagementService | 37 | S3 + DynamoDB, dual-region. DEC-001 on business-rules publish. DynamoDB write failure after S3 success = orphaned S3 object. |

**Latency amplification principle:** EncryptionService and CaseManagementService sit on the hot path of every dispute action. Their P95 is not a service-level SLO — it is a platform-level SLO. A 200ms increase in EncryptionService decrypt propagates to every component that calls it.

### Tier 2 — Domain action services (all DEC-001)

| Service | COMP | Risk |
|---------|------|------|
| AcceptService | 19 | DEC-001 direct publish to internal-integration-events. AMEX/Discover dead code paths. |
| ContestService | 20 | DEC-001 direct publish. |
| CaseActionService | 24 | DEC-001 direct publish. RBAC gap (DEC-018 accepted risk) — no server-side RBAC enforcement. |
| NotesService | 25 | DEC-001: publishes business-rules inside @Transactional boundary. |
| QuestionnaireService | 26 | POST idempotency gap — concurrent POSTs insert duplicate rows. |

### Tier 3 — Support and specialised services

| Service | COMP | Risk |
|---------|------|------|
| APILogService | 38 | Centralised error sink. Callers catch and continue if POST /log fails — audit gap with no alert. 4 confirmed callers: COMP-19, 20, 24, 37. |
| TokenService | 36 | Covered in S1. |
| DisputeService | 22 | Read-only. Kafka producer wired to business-rules but call site commented out — effectively Kafka-free. |
| BusinessRulesService | 31 | NOT called by COMP-16 BusinessRulesProcessor. COMP-16 reads rules directly from DB. |

### Key metrics

```
# EncryptionService (COMP-35)
wdp_encryption_encrypt_duration_seconds                                  # < 50ms P95 (NFR target)
wdp_encryption_decrypt_duration_seconds                                  # < 75ms P95 (NFR target)
wdp_encryption_kms_call_total{outcome}                                   # P1 on failure
wdp_encryption_hmac_key_age_seconds                                      # key never refreshed after startup
wdp_encryption_decrypt_total{caller_service}                             # unauthorised caller detection

# CaseManagementService (COMP-23)
wdp_case_mgmt_request_duration_seconds{endpoint, platform}
wdp_case_mgmt_case_create_total{platform, outcome}
wdp_case_mgmt_kafka_publish_total{outcome}                               # rollback on failure = safe here

# Tier 2 DEC-001 services
wdp_accept_kafka_publish_total{outcome}
wdp_contest_kafka_publish_total{outcome}
wdp_case_action_kafka_publish_total{outcome}
wdp_notes_kafka_publish_total{outcome}
wdp_accept_network_dead_path_total{card_network}                         # AMEX | DISCOVER — should be 0

# APILogService (COMP-38)
wdp_api_log_write_total{caller_service, outcome}
wdp_api_log_caller_failure_total{caller_service}                         # must be added to COMP-19,20,24,37
```

### Platform cascade alert (composite)

Alert P1 when all three conditions are simultaneously true:
- `wdp_encryption_decrypt_duration_seconds` P95 > 150ms
- `wdp_case_mgmt_request_duration_seconds{endpoint="create"}` P95 > 300ms
- `wdp_case_creation_outcomes_total` rate decreasing > 20% from baseline

This detects a latency cascade before it fully manifests as user-visible failures.

---

## 10. S6 — Outbound & Notification Consumers

**Last-mile delivery to external systems**

### The last-mile delivery problem

Every S6 consumer commits its Kafka offset before external delivery completes. WDP considers the event processed — the external system may not have received it. **Kafka lag=0 does not confirm delivery to NAP-DPS, BEN, Signifyd, or IBM DB2.**

### Component delivery map

| Consumer | COMP | Topic | External target | ACK timing | Recovery |
|----------|------|-------|-----------------|------------|----------|
| NAPOutcomeProcessor | 39 | internal-integration-events | NAP-DPS (SRV117/118) | Pre-ACK | NAP.DISPUTE_EVENT_CONSUMER_ERROR table |
| ThirdPartyNotificationConsumer | 41 | external-request-events | Signifyd REST API | Mid (after outbox INSERT) | wdp.outgoing_event_outbox |
| BENConsumer | 42 | external-request-events | BEN MSK cluster (BEN-owned) | Mid | wdp.outgoing_event_outbox |
| CoreNotificationConsumer | 43 | core-request-events | IBM DB2 (BC schema) | Pre-ACK (Path B: mid) | wdp.outgoing_event_outbox (CORE_EVENTS) |
| CapOneResponseFileProcessor | 45 | (reads file_generation_event) | S3 → ControlM → CapitalOne | N/A | file_generation_event status |
| NetworkResponseFileProcessor | 46 | (reads file_generation_event) | S3 → ControlM → Amex/Discover | N/A | file_generation_event status |
| DialoguIssuerDocumentProcessor | 47 | (reads file_generation_event) | S3 → Sterling → Dialogu | N/A | file_generation_event status |

### Architecture risks

| Risk | Component | Severity | Detail |
|------|-----------|----------|--------|
| NAP-DPS SRV118 failure = missed money movement | COMP-39 | P1 | Merchant accepts dispute but NAP-DPS never notified — funds not moved. |
| `notesLookup()` commented out | COMP-39 | P3 | `dataRecord` always null in SRV118 payload. Known functional gap. |
| EDIA migration not implemented | COMP-39 | P2 | No feature flag or route for EDIA exists in COMP-39 source. Migration path is aspirational only. |
| JustAI not implemented | COMP-41 | N/A | No JustAI reference in codebase. Signifyd is the sole live vendor. |
| BEN delivery is Kafka, not REST | COMP-42 | Doc | Prior documentation described BEN as REST webhook. BEN-owned MSK cluster. WDP has no visibility into BEN cluster health. |
| BEN display codes cached at startup | COMP-42 | P3 | No refresh mechanism. Stale codes if display table updated without pod restart. |
| COMP-43 is sole DB2 writer on platform | COMP-43 | P1 | DB2 unavailability stops all CORE platform notifications. No fallback. |
| COMP-43 enriches via 5 REST calls + KMS | COMP-43 | P2 | EncryptionService latency (S5) directly delays DB2 writes for new case events. |
| File gen: DiscoverHybrid uses special SFTP | COMP-46 | P2 | Different delivery path from other network files. Must be monitored separately. |

### Key metrics

```
# COMP-39 NAPOutcomeProcessor
wdp_nap_srv118_call_total{outcome}
wdp_nap_srv117_call_total{outcome}
wdp_nap_dps_response_duration_seconds
wdp_nap_error_table_count{status}                                        # FAILED1 | FAILED2 | ERROR
wdp_nap_prior_error_reprocessed_total{result}

# COMP-41 ThirdPartyNotificationConsumer
wdp_signifyd_api_call_total{api_type, outcome}                           # create | stage | representment
wdp_signifyd_outbox_error_count{status}
wdp_signifyd_api_duration_seconds

# COMP-42 BENConsumer
wdp_ben_kafka_publish_total{outcome}
wdp_ben_external_msk_available                                           # gauge 0/1 (derived)
wdp_ben_display_code_cache_age_seconds

# COMP-43 CoreNotificationConsumer
wdp_core_db2_write_total{table, outcome}                                 # case | occur | notes
wdp_core_db2_connection_available                                        # gauge 0/1
wdp_core_enrichment_duration_seconds
wdp_core_outbox_error_count{status}

# File generation (COMP-45/46/47)
wdp_file_gen_event_pending_count
wdp_file_gen_event_pending_age_seconds
wdp_file_output_total{processor, network, outcome}
wdp_file_s3_put_total{folder, outcome}
```

### Alert thresholds

| Alert | Severity | Condition |
|-------|----------|-----------|
| COMP-43 DB2 write failure > 0 sustained 1 min | P1 | CORE platform not receiving notifications |
| `wdp_core_db2_connection_available` = 0 | P1 | DB2 unavailable |
| COMP-39 SRV118 failure rate > 5% over 10 min | P1 | NAP money movement at risk |
| `wdp_ben_kafka_publish_total{outcome="failure"}` > 10% sustained 5 min | P1 | BEN cluster unavailable |
| `wdp_file_gen_event_pending_age_seconds` > 30 min | P1 | File generation pipeline stalled |
| COMP-41 `outbox_error_count{status="ERROR"}` growing | P2 | Signifyd terminal failures |
| COMP-39 `nap_error_table_count{status="FAILED2"}` > 20 | P2 | Prior error reprocessing failing |
| COMP-43 enrichment duration P95 > 1s | P2 | Upstream dependency (EncryptionService/CaseMgmt) degrading |
| `wdp_ben_display_code_cache_age_seconds` > 24h | P3 | Stale codes risk — pod restart may be needed |

---

## 11. S7 — Data & Integration Boundaries

### Instrumentation split

| Category | Instrumentation approach |
|----------|--------------------------|
| WDP-owned databases (Aurora, DynamoDB, Redis) | Direct OTel + AWS CloudWatch metrics via OTel Collector |
| WDP-owned storage (S3, AWS MSK) | AWS CloudWatch metrics + OTel SDK for application-level signals |
| External systems (IBM DB2, DataPower, NAP-DPS, Signifyd, BEN MSK, Sterling, ControlM) | **Inferred** from calling component outcome metrics |

### Aurora PostgreSQL (primary and NAP)

```
# Connection pools (from HikariCP via Micrometer/OTel)
wdp_db_connection_pool_active{component, schema}
wdp_db_connection_pool_wait_duration_seconds{component}

# From AWS CloudWatch (via OTel Collector)
aws_aurora_db_load{table}
aws_aurora_replica_lag_seconds
aws_aurora_storage_used_bytes
aws_aurora_connections{instance}

# Business-level (from application queries)
wdp_outbox_table_row_count{table, status}                                # growth monitoring
wdp_questionnaire_duplicate_rows_count                                   # HIGH risk table
```

**Aurora alert thresholds:**

| Alert | Severity | Condition |
|-------|----------|-----------|
| Connection pool active = pool max sustained > 2 min | P2 | Pool saturation |
| Aurora replica lag > 5s | P2 | Stale reads in rule evaluation |
| `chbk_outbox_row_archive` row count > 10M | P3 | Plan purge |
| `wdp_questionnaire_duplicate_rows_count` > 0 | P2 | COMP-26 idempotency gap materialised |

### DynamoDB

```
# From AWS CloudWatch
aws_dynamodb_successful_request_latency{table, region, operation}
aws_dynamodb_throttled_requests_total{table, region}                     # P1 on nonzero
aws_dynamodb_consumed_write_capacity_units{table}
```

Monitor both regions (eu-west-2 NAP, us-east-2 US) independently. A throttled request means evidence documents are being dropped. Alert P1.

### Redis ElastiCache

```
wdp_redis_connected                                                      # gauge 0/1
wdp_redis_cache_entry_age_seconds                                        # proxy for unknown writer health
# Primary proxy: TokenService IDP call rate spike (see S1 metrics)
```

**Critical open question:** The external component that writes `wdpinternalidptoken:token` to Redis is unidentified. Until confirmed, the IDP call rate spike from S1 is the only reliable Redis failure indicator.

### S3

```
# From AWS CloudWatch
aws_s3_put_requests_total{bucket, prefix}
aws_s3_error_rate{bucket, prefix}

# Application-derived
wdp_file_processor_sqs_queue_depth                                       # inbound proxy
wdp_s3_outbound_file_uncollected_age_seconds{folder}                     # ControlM collection proxy
```

### Shared table risk register

Three confirmed HIGH severity concurrent-write risks requiring active monitoring:

**1. wdp.ACTION (HIGH)** — concurrent writes from COMP-23 and COMP-24 with no confirmed optimistic locking.

Monitor: `wdp_action_concurrent_write_collisions_total` — detected when two different components update the same action row within a narrow time window (< 1 second). Alert P2 on any collision.

**2. wdp.disputes_questionnaire (HIGH)** — COMP-26 POST idempotency gap creates duplicate rows for the same `(I_CASE, I_ACTION_SEQ)` on concurrent requests.

Monitor: `wdp_questionnaire_duplicate_rows_count` — periodic query counting `(I_CASE, I_ACTION_SEQ)` pairs with more than one row. Alert P2 on any nonzero value.

**3. nap.nap_child_entity (HIGH — confirmed bug)** — COMP-02 `saveChildWithMerchant` uses `wdpTransactionManager` instead of `napTransactionManager`. A failure during a multi-table NAP write may not roll back NAP schema tables. Monitor for partial writes via UAMS error log correlation with NAP child entity existence.

---

## 12. Alert Catalogue

### P1 Alerts — Page immediately

| # | Alert | Section | Metric | Threshold |
|---|-------|---------|--------|-----------|
| P1-01 | Gateway UAMS/CHAS call P95 > 3s | S1 | `wdp_gateway_upstream_call_duration_seconds` | P95 > 3s sustained 2 min |
| P1-02 | CHAS DB2 exception nonzero | S1 | `wdp_chas_db2_exception_total` | > 0 |
| P1-03 | TokenService IDP call rate > 5% | S1 | `wdp_token_cache_requests_total{layer="idp"}` | > 5% of total |
| P1-04 | Outbox PENDING age P95 > 300s | S2 | `wdp_inbound_outbox_pending_age_seconds` | P95 > 300s |
| P1-05 | DNWK file silently lost (NPE) | S2 | `wdp_file_processor_file_job_total{type="DNWK",outcome="error"}` | > 0 |
| P1-06 | Scheduler1 Kafka publish failure | S2 | `wdp_scheduler1_kafka_publish_total{outcome="failure"}` | > 0 sustained 3 min |
| P1-07 | VisaDisputeBatch cron stopped | S2 | `wdp_visa_batch_last_run_timestamp` | Silence > 5 min |
| P1-08 | Terminal outbox errors growing | S2 | `wdp_inbound_outbox_error_count` | Growing over 10 min |
| P1-09 | MSK offline partitions | S3 | `aws_msk_broker_offline_partitions_count` | > 0 |
| P1-10 | No active MSK controller | S3 | `aws_msk_broker_active_controller_count` | = 0 |
| P1-11 | new-case-events production stops | S3 | `kafka_topic_messages_in_per_sec{topic="new-case-events"}` | = 0 sustained 2 min |
| P1-12 | COMP-14 thread stall | S4 | `wdp_case_creation_processing_duration_seconds` | P99 > 60s |
| P1-13 | COMP-16 DEC-001 publish failure | S4 | `wdp_bre_kafka_publish_total{topic="outgoing-events",outcome="failure"}` | > 0 |
| P1-14 | COMP-18 PUBLISHED orphan rows growing | S4 | `wdp_notif_orch_orphaned_published_rows` | Growing > 15 min |
| P1-15 | Cascading latency: Encryption + CaseMgmt + creation rate | S5 | Composite (3-signal rule) | All 3 simultaneously |
| P1-16 | DEC-001 publish failure (COMP-19/20/24/25) | S5 | Per-service `kafka_publish_total{outcome="failure"}` | > 0 |
| P1-17 | EncryptionService KMS call failure | S5 | `wdp_encryption_kms_call_total{outcome="failure"}` | > 0 |
| P1-18 | COMP-43 DB2 write failure | S6 | `wdp_core_db2_write_total{outcome="failure"}` | > 0 sustained 1 min |
| P1-19 | IBM DB2 connection unavailable | S6 | `wdp_core_db2_connection_available` | = 0 |
| P1-20 | COMP-39 NAP-DPS failure rate > 5% | S6 | `wdp_nap_srv118_call_total{outcome!="success"}` | > 5% sustained 10 min |
| P1-21 | BEN MSK publish failure > 10% | S6 | `wdp_ben_kafka_publish_total{outcome="failure"}` | > 10% sustained 5 min |
| P1-22 | File generation backlog > 30 min | S6 | `wdp_file_gen_event_pending_age_seconds` | > 1800s |
| P1-23 | DynamoDB throttled requests | S7 | `aws_dynamodb_throttled_requests_total` | > 0 |

### P2 Alerts — Notify and ticket

| # | Alert | Section | Condition |
|---|-------|---------|-----------|
| P2-01 | Gateway 4xx rate > 10% | S1 | JWT storm or routing misconfiguration |
| P2-02 | Gateway NPE: route_null > 0 | S1 | Route missing from wdp.api_route |
| P2-03 | Portal P95 > 2s on dispute detail | S1 | UI SLO breach |
| P2-04 | Outbox PENDING age P95 > 120s | S2 | SLO breach |
| P2-05 | SQS queue depth > 10 and growing | S2 | File processing backlog |
| P2-06 | Visa silent drops > 5/hr | S2 | `wdp_visa_batch_items_total{outcome="null_return_skipped"}` |
| P2-07 | BLOCKED evidence rows > 500 stable 15 min | S2 | Evidence gate stuck |
| P2-08 | MSK under-replicated partitions > 5 min | S3 | Broker health degrading |
| P2-09 | COMP-15 silent drops > 0 | S4 | Evidence documents disappearing |
| P2-10 | COMP-16 rule no-match > 20% | S4 | Rules config or DB degradation |
| P2-11 | COMP-17 crash gap materialised | S4 | case_expiry write failure on Path B |
| P2-12 | COMP-18 routing flag change | S4 | coreMigration or disputesAPIMigration changed |
| P2-13 | DEC-001: COMP-37 Kafka publish failure | S5 | document-evidence → business-rules publish failed |
| P2-14 | COMP-26 duplicate questionnaire rows > 0 | S5/S7 | idempotency gap materialised |
| P2-15 | EncryptionService decrypt P95 > 75ms | S5 | NFR breach / KMS degrading |
| P2-16 | CaseManagementService create P95 > 300ms | S5 | NFR breach or enrichment latency |
| P2-17 | COMP-41 Signifyd ERROR rows growing | S6 | Terminal failures — fraud intel gap |
| P2-18 | COMP-39 FAILED2 count > 20 | S6 | Prior error reprocessing failing |
| P2-19 | COMP-43 enrichment P95 > 1s | S6 | EncryptionService or CaseMgmt degrading |
| P2-20 | Aurora connection pool saturation | S7 | Pool active = pool max sustained 2 min |
| P2-21 | Aurora replica lag > 5s | S7 | Stale reads in rule evaluation |
| P2-22 | S3 outbound file uncollected > 2× ControlM schedule | S7 | ControlM not collecting |

### P3 Alerts — Observe

| Alert | Section | Condition |
|-------|---------|-----------|
| BEN display code cache > 24h | S6 | May need pod restart after code table update |
| chbk_outbox_row_archive > 10M rows | S7 | Plan purge |
| DiscoverHybrid file rate drops while other networks normal | S6 | Special SFTP path issue |
| COMP-17 silent discard rate rising | S4 | Upstream payload structure change |
| removeItemFromQueueDisabled flag active + queue depth growing | S2 | ACK suppression + backlog |

---

## 13. Dashboard Hierarchy

### Level 1 — Platform health overview (NOC view)

Single dashboard showing platform-wide health in 6 panels:

1. **Spine throughput correlation** — COMP-14/15/16/18/17 processing rates as a multi-line time series. Divergence = spine degradation.
2. **Primary SLO: outbox PENDING age** — P95 time series with 120s (SLO) and 300s (P1) threshold lines.
3. **External delivery health** — 4 gauges: NAP-DPS success rate, Signifyd success rate, BEN publish rate, DB2 write rate.
4. **Kafka topic production rates** — 9 topics as sparklines. Zero = producing component failure.
5. **P1 alert count** — single stat, should always be 0.
6. **Infrastructure health** — MSK offline partitions, Aurora replica lag, DynamoDB throttles.

### Level 2 — Section-level dashboards (on-call view)

One dashboard per section (S1–S7), each with 6 panels as defined in the individual section specifications above.

### Level 3 — Component drilldown (deep investigation)

Per-component dashboards for the highest-criticality services:

- **EncryptionService** — encrypt vs decrypt latency, KMS call rate, DEK rotation events
- **CaseManagementService** — request duration by endpoint and platform, case creation rate
- **COMP-14 CaseCreationConsumer** — processing duration with P99/P999, REST dependency latency breakdown via Tempo
- **COMP-16 BusinessRulesProcessor** — rule evaluation outcomes by platform, publish success rate, DB query duration
- **COMP-18 NotificationOrchestrator** — fan-out success rate per target, orphaned PUBLISHED rows

### Grafana data source configuration

| Data source | Type | Target |
|-------------|------|--------|
| Prometheus | Prometheus | Platform metrics |
| Tempo | Tempo | Distributed traces |
| Loki | Loki | Structured logs |
| AWS CloudWatch | CloudWatch | MSK, Aurora, DynamoDB, S3 metrics |

---

## 14. Open Questions & Prerequisites

The following items must be resolved before this observability design can be fully implemented. Each is an architectural gap or unconfirmed fact from the component survey.

### Architecture gaps requiring confirmation

| # | Question | Impact on observability |
|---|----------|------------------------|
| OQ-01 | Which component writes `wdpinternalidptoken:token` to Redis? | Cannot design Redis writer health monitoring until confirmed |
| OQ-02 | Does COMP-14 CaseCreationConsumer publish to the business-rules Kafka topic after case creation? | Topic producer count and DEC-001 violation assessment incomplete |
| OQ-03 | Which component sets `chbk_outbox_row.status = SUCCESS`? | Evidence unblocking gate (S2) cannot be reliably monitored until confirmed |
| OQ-04 | What is the actual Kafka topic name for COMP-24 CaseActionService ActionEvent? | Alert cannot reference correct topic name |
| OQ-05 | What are the consumer group IDs for COMP-41, COMP-42, COMP-43? | Consumer group lag alerts require exact group IDs |
| OQ-06 | What is the SQS visibility timeout configured for FileProcessor (COMP-11)? | Double-processing risk alert threshold depends on this value |
| OQ-07 | What is the ControlM collection schedule for S3 outbound files? | Cannot set uncollected-file age alert threshold without this |
| OQ-08 | Are COMP-03 CHAS health probes checking DB2 connectivity? | If not, DB2 failure is invisible until COMP-43 write fails |
| OQ-09 | Does COMP-12 Scheduler3 re-drive FAILED rows for COMP-41 ThirdPartyNotificationConsumer? | Recovery path for Signifyd failures assumed but not confirmed |

### Instrumentation gaps requiring code changes

| # | Gap | Component | Severity |
|---|-----|-----------|----------|
| IG-01 | `wdp_evidence_silent_drop_total` metric does not exist | COMP-15 | P2 — MISCDOC/DRFTDOC documents vanish with no visibility |
| IG-02 | `wdp_api_log_caller_failure_total` does not exist in callers | COMP-19, 20, 24, 37 | P2 — APILogService silent failure is undetected |
| IG-03 | No per-publisher metric for business-rules topic | COMP-12, 15, 23, 24, 25 | P2 — cannot attribute production rate drops to specific publisher |
| IG-04 | No ACK path label (A vs B) in COMP-17 | COMP-17 | P2 — cannot quantify crash gap exposure |
| IG-05 | No `wdp_chas_db2_exception_total` metric emitted on catch block | COMP-03 | P1 — DB2 failures masked with no signal |
| IG-06 | No DiscoverHybrid separate monitoring tag in file generation | COMP-46 | P2 — special SFTP path cannot be monitored independently |

### NFR thresholds to confirm

All NFR targets from WDP-NFRS.md v2.0 are used as alert thresholds in this document. Before finalising alert rules, the following specific targets should be confirmed with the product team as current:

- Outbox publish lag: < 2 min P95 (used as P2 threshold)
- Internal API response time: < 200ms P95 (used in S5)
- EncryptionService encrypt: < 50ms P95, decrypt: < 75ms P95
- Portal dispute detail load: < 2s P95

---

*This document was produced as part of the WDP architecture observability design session, April 2026.*
*Architecture partner: Claude (Anthropic) · Solution Architect: Ram*
*Next steps: reviewer feedback → NFR threshold confirmation → instrumentation gap remediation → alert rule implementation*
