# WDP-COMPONENTS.md
**Worldpay Dispute Platform — Component Architecture Reference**
*Version: 1.0 DRAFT | April 2026*
*Built from: WDP-ARCHITECTURE.md v2.0 + GitHub Copilot CLI code analysis*

---

## How to Use This Document

This document captures architect-level component information for every WDP component.
It is built iteratively — each component section is completed using GitHub Copilot CLI
code analysis combined with architect confirmation.

**Status indicators:**
- ✅ Production — component is live
- 🔄 In Progress — being built or partially live
- 🔴 Planned — not yet started
- 📝 DRAFT — template not yet filled from Copilot analysis
- ✅ COMPLETE — filled and architect-confirmed

**What this document covers per component:**
1. What it does — responsibility, not implementation
2. Boundaries — inputs and outputs
3. Key architectural decisions — references to WDP-DECISIONS.md
4. Risks and constraints — NFRs, compliance, dependencies
5. Planned changes — migrations, open questions

**What this document does NOT cover:**
- Code, classes, methods, or implementation patterns
- Database schemas or SQL
- Configuration values or secrets
- Deployment specs or Kubernetes manifests
These belong in the Claude Code project per component.

---

## Table of Contents

**Part 1 — Access & API Layer**
- 1.1 API Gateway
- 1.2 UserAccessManagementService
- 1.3 CoreHierarchyAuthorizationService

**Part 2 — Inbound Processing**
- 2.1 NAP/WPG Path
  - 2.1.1 NAPDisputeEventService
  - 2.1.2 NAPDisputeEventProcessor
  - 2.1.3 NAPDisputeDeclineBatch
- 2.2 Card Network Batch Path
  - 2.2.1 VisaDisputeBatch
  - 2.2.2 FirstChargebackBatch
  - 2.2.3 CaseFillingBatch
- 2.3 File-Based Inbound Path
  - 2.3.1 DM Mainframe
  - 2.3.2 FileProcessor
  - 2.3.3 InboundDisputeEventScheduler
  - 2.3.4 FileAcknowledgementProcessor

**Part 3 — Kafka Event Bus**
- 3.1 AWS MSK Configuration
- 3.2 Topic Reference

**Part 4 — Core Processing**
- 4.1 Event Consumers
  - 4.1.1 CaseCreationConsumer
  - 4.1.2 EvidenceConsumer
  - 4.1.3 BusinessRulesProcessor
  - 4.1.4 CaseExpiryUpdateConsumer
  - 4.1.5 NotificationOrchestrator
- 4.2 Case Action Services
  - 4.2.1 AcceptService
  - 4.2.2 ContestService
  - 4.2.3 ChargebackService
  - 4.2.4 DisputeService
  - 4.2.5 CaseManagementService
  - 4.2.6 CaseActionService
  - 4.2.7 NotesService
  - 4.2.8 QuestionnaireService
- 4.3 Search & Display Services
  - 4.3.1 CaseSearchService
  - 4.3.2 DisplayCodeService
- 4.4 Queue & Workflow Services
  - 4.4.1 FaxQueueService
  - 4.4.2 UserQueueSkillService
- 4.5 Rules & Configuration Services
  - 4.5.1 BusinessRulesService
  - 4.5.2 RulesService
- 4.6 Organisation & User Services
  - 4.6.1 OrgManagementService
  - 4.6.2 MerchantTransactionService
- 4.7 Supporting Services
  - 4.7.1 EncryptionService
  - 4.7.2 TokenService
  - 4.7.3 DocumentManagementService
  - 4.7.4 APILogService

**Part 5 — Outbound Processing**
- 5.1 Card Network Response
  - 5.1.1 NAP Outcome Processor
  - 5.1.2 VisaResponseQuestionnaire
- 5.2 Notification Consumers
  - 5.2.1 ThirdPartyNotificationConsumer
  - 5.2.2 BEN Consumer
  - 5.2.3 CoreNotificationConsumer
  - 5.2.4 EDIA Consumer 🔴 Planned
- 5.3 Outbound File Generation
  - 5.3.1 CapitalOne Response file processor
  - 5.3.2 FileAcknowledgementProcessor (outbound)
  - 5.3.3 NetworkResponseFileProcessor
  - 5.3.4 Dialogu Issuer document Processor
  - 5.3.5 NYCE File Generation Processor 🔴 Planned

**Part 6 — Acquiring Platform Integration**
- 6.1 NAP Platform Integration
- 6.2 CORE Platform Integration
- 6.3 LATAM Platform Integration 🔴 In Progress
- 6.4 VAP Platform Integration 🔴 In Progress
- 6.5 EDIA Platform 🔴 Planned

**Part 7 — UI Layer**
- 7.1 WDP Merchant Portal
- 7.2 WDP Ops Portal
- 7.3 Disputes Section
- 7.4 Queues Section
- 7.5 User Management Section
- 7.6 Org Management Section
- 7.7 Dashboard Section 🔴 Planned

---

## PART 1 — ACCESS & API LAYER

---

### 1.1 API Gateway
**Owner:** Core WDP Team
**Status:** ✅ Production | ✅ COMPLETE
**Technology:** Spring Cloud Gateway
**Repository:** wdp-gateway

**Enterprise dependencies (not WDP owned):**
- Akamai — CDN and edge security for Merchant Portal
  and B2B external merchant traffic
- APIGEE — B2B API gateway for external merchant
  system-to-system integration
- IDP — enterprise OAuth 2.0 identity provider

---

**What it does**
The API Gateway is the single entry point for all
traffic into WDP regardless of origin — Merchant
Portal, Ops Portal, or external merchant systems.

It runs a 4-layer filter pipeline on every request:

1. **JWT Authentication** — validates Bearer tokens
   against trusted IDP issuers using cached public
   keys (not a per-request IDP call). Invalid or
   missing tokens are rejected with 401.

2. **Request Logging** — logs inbound path and
   matched route. Logs response status on return.
   Never short-circuits.

3. **Region-Based Role Authorization** — extracts
   region from the URL path (NAP/UK or PIN/US),
   decodes the JWT, and checks the caller holds
   a valid WDP role for that region and firm type
   (internal or external). Rejects with 403 if no
   matching role. Skipped if no region in path.

4. **Case-Level Entity Authorization** — if the URL
   contains a valid case ID, calls an external
   authorization service to confirm the caller has
   access to that specific case. For NAP platform
   calls UserAccessManagementService. For PIN
   platform calls CoreHierarchyAuthorizationService.
   Internal firms are auto-authorized and bypass
   the external call. Rejects with 403 on failure.
   Skipped if no case ID in path.

After all filters pass, the gateway rewrites the
URL path (strips the /api/merchant/gcp prefix) and
proxies the request to the appropriate backend
microservice. No other request or response
transformation is applied.

**Boundaries**

Inputs:
- Merchant Portal requests via Akamai
- Ops Portal requests — direct connection
  (does not go through Akamai)
- External merchant system requests
  via Akamai → APIGEE → API Gateway

Outputs:
- Authenticated and authorized requests
  routed to WDP Core microservices
  via URL path rewrite and proxy
- UserAccessManagementService — case-level
  authorization call for NAP platform requests
- CoreHierarchyAuthorizationService — case-level
  authorization call for PIN platform requests

**Configuration & Scaling**
- Replica count is environment-specific, resolved
  at deploy time via XL Deploy variable. Actual
  counts per environment to be confirmed from
  XL Deploy configuration.
- Memory: 2048Mi limit / 256Mi request.
  No CPU limits configured.
- URL-based routing configuration stored in
  wdp.api_route database table (26+ routes),
  loaded into memory at pod startup.
  Routing changes require pod restart.
- IDP public key cached by Spring Security —
  not fetched per request.
- No rate limiting at gateway level —
  delegated to Akamai and APIGEE upstream.
- NGINX ingress capped at 25MB request bodies.

**Authentication & Authorization failure handling**
- 401 Unauthorized — returned by Spring Security
  with JSON error body and WWW-Authenticate header.
  Triggered by missing, expired, or untrusted JWT.
- 403 Forbidden — returned by custom filter beans
  with empty response body (no JSON). Triggered by
  insufficient role or no case access. Clients must
  handle the empty body case.
- Whitelisted paths (health endpoints) bypass JWT
  authentication entirely. Whitelist is loaded once
  at startup from the routing table.

**Key architectural decisions**
- Single API Gateway for all traffic paths —
  portal, ops, and B2B
- JWT validation via IDP public key cache —
  not a per-request IDP call
- URL-based routing configuration in database
  table — loaded at startup, not hot-reloaded
- Two-tier authorization model — region/role
  check followed by case-level entity check
- Case-level authorization split by platform —
  NAP uses UserAccessManagementService,
  PIN uses CoreHierarchyAuthorizationService

**Risks & Constraints**
- No timeouts configured on external authorization
  service calls — a slow or unresponsive UAMS or
  CoreHierarchyAuthorizationService will block the
  gateway thread indefinitely
- Case-level authorization is fail-open on null
  response — a null result from the external
  service permits the request through
- AuthorizationFilter and CaseNumberFilter are
  both at Order 0 — execution order between them
  is not guaranteed by configuration
- No HPA configured — pod count does not scale
  automatically under load
- No JVM heap tuning — running on JVM defaults
  with OpenTelemetry agent adding unaccounted
  overhead
- Routing configuration changes require pod
  restart to reload from database
- No rate limiting at gateway level

**Planned changes**
- Case-level authorization to be consolidated to
  a single service regardless of platform,
  replacing the current NAP/PIN split between
  UserAccessManagementService and
  CoreHierarchyAuthorizationService

---

### 1.2 UserAccessManagementService
**Owner:** Core WDP Team
**Status:** ✅ Production | ✅ COMPLETE
**Technology:** Spring Boot 3.5 / Java 17
**Repository:** gcp-user-access-management-service

---

**What it does**
UserAccessManagementService is the central access
control and user lifecycle management service for
the WDP NAP ecosystem. It has two distinct
responsibilities that are served by the same
deployable:

1. **Reference data management** — owns and
   manages the merchant entity hierarchy
   (parent → child entities), Merchant IDs (MIDs),
   their relationships, and an Access Control List
   (ACL) used by downstream consumers.

2. **Runtime authorization** — exposes a POST
   /authorize endpoint called by the API Gateway
   for NAP platform requests. Validates whether a
   JWT-bearer's entity claims grant access to a
   given merchant ID or dispute case number by
   cross-referencing the entity relationship data
   it owns.

It also acts as a proxy/orchestrator to an external
SunGard Identity Provider (IdP) for all user
lifecycle operations — creating, updating,
activating, deactivating, suspending, and resetting
passwords for NAP merchant portal users. No user
credentials are stored by this service.

**What it does NOT do:**
- Issue or mint JWT tokens
- Store user credentials or passwords
- Process financial transactions or dispute cases
- Enforce authorization for PIN platform — the
  /authorize endpoint only implements NAP.
  CORE, VAP, LATAM, and PIN platform requests
  return 400 (stub code currently in production)
- Process bulk onboarding file content — uploads
  CSV/XLSX to S3 only; downstream processing
  happens elsewhere
- Use the ACL data for its own authorization
  decisions — ACL is maintained purely for
  downstream consumers

⚠️ FOLLOW-UP NEEDED: The original architecture
review noted that UAMS "enforces access control
rules across the platform — users may only act on
disputes belonging to their org hierarchy, queue
access is restricted by role and skill." Copilot
CLI analysis did not surface any queue or
skill-based routing logic within this service.
This behaviour may live in a different service
(e.g. UserQueueSkillService). To be confirmed
and this note updated before WDP-COMPONENTS.md
is considered final.

**Boundaries**

Inputs:
- API Gateway — POST /authorize calls for NAP
  platform case-level authorization checks
- WDP Merchant Portal users (external NAP) —
  entity and user management operations scoped
  to their own entity hierarchy
- Merchant Fraud/Disputes Portal users —
  user lifecycle operations scoped to their
  napParentEntity
- Internal Worldpay systems — full access to
  all entity, merchant, user, and ACL management
  operations; all entity-scoping bypassed

Outputs:
- SunGard IdP (two instances, firm-based routing)
  — all user lifecycle operations proxied via REST
- AWS S3 (eu-west-2) — bulk entity onboarding
  files written to RECEIVED/{ENV}/{filename}
- PostgreSQL nap schema — entity hierarchy,
  MIDs, entity relationships, case lookup (read)
- PostgreSQL wdp schema — ACL entries

**Authorization model**
Access control is enforced in two layers:

Layer 1 — Spring Security JWT validation.
Every request requires a valid Bearer JWT from
a trusted issuer. Requests without a valid JWT
are rejected with 401 before reaching any
application logic.

Layer 2 — Application-level scope enforcement.
Five distinct rules applied after JWT validation:

- **Internal vs external split** — JWT iss claim
  containing us_worldpay_fis_int grants full
  access; all entity-scoping and role checks
  bypassed for internal callers.
- **Role-based action gating** — user management
  operations require WDP_NAP_ADMIN or
  WDP_PIN_ADMIN in the JWT AuthorizationList
  claim.
- **Entity scope enforcement** — external callers
  are restricted to data within their own
  napParentEntity and napChildEntities JWT
  claims. All queries and mutations are filtered
  accordingly.
- **Runtime merchant access check** — POST
  /authorize cross-references JWT entity claims
  against nap.nap_entity_rel to confirm access
  to a specific merchant or case.
- **Self-action prevention** — external users
  cannot update, reset password, or change
  status of their own account.

**The /authorize endpoint**
Called by the API Gateway for every NAP platform
request containing a case ID.

- Request: platform (NAP only), plus either a
  caseNumber or merchantId
- If caseNumber provided: resolves to merchantId
  via nap.case (read-only lookup)
- Validates JWT napParentEntity and
  napChildEntities claims against
  nap.nap_entity_rel
- Internal callers always pass — no DB lookup
- Returns HTTP 200 with empty body on success.
  Returns 401 if JWT entity claims do not cover
  the merchant. Returns 400 if platform is
  anything other than NAP (stub behavior)
- The 200 status code itself is the authorization
  signal — no body is returned

**Data owned**
- nap.nap_parent_entity — top-level merchant
  groupings
- nap.nap_child_entity — sub-groupings beneath
  a parent; default child auto-created when
  parent is created
- nap.nap_merchant — individual Merchant IDs
  with MCC and WPG ID; one MID can only belong
  to one parent entity
- nap.nap_entity_rel — merchant-to-entity
  relationships; primary lookup table for
  authorization checks
- nap.case — read-only; used only to resolve
  caseNumber to merchantId during authorization
- wdp.acl — access control list entries linking
  named consumers to source systems and entity
  values; maintained for downstream consumers,
  not used for this service's own auth decisions

**Configuration & Scaling**
- Replica count is environment-specific, resolved
  at deploy time via XL Deploy variable. Actual
  counts per environment to be confirmed from
  XL Deploy configuration.
- Memory: 2048Mi limit / 1024Mi request.
  No CPU limits or requests configured.
- No HPA — scaling is purely static.
- Two PostgreSQL datasources (nap and wdp schemas)
  each with HikariCP default pool of 10
  connections per pod. Not explicitly configured.
- Async thread pool configured via environment
  variables — actual values in Kubernetes secret,
  not visible in source.
- OpenTelemetry Java agent injected at runtime —
  adds approximately 100-200MB to JVM non-heap
  footprint. No explicit JVM heap tuning
  configured.
- Topology spread constraint configured but
  non-functional — label mismatch means all pods
  can schedule to same node (see Risks).

**Key architectural decisions**
- UAMS serves as both the reference data store
  for the entity hierarchy AND the runtime
  authorization check service — single service
  with two distinct responsibilities
- Authorization for NAP platform only — PIN
  platform case-level authorization is handled
  by CoreHierarchyAuthorizationService
- User credential management fully delegated
  to SunGard IdP — no credentials stored
- Internal callers bypass all entity-scoping
  and role checks via JWT issuer claim
- ACL maintained as a service to downstream
  consumers — not used for this service's own
  authorization decisions
- Two separate PostgreSQL datasources (nap and
  wdp schemas) within the same service

**Risks & Constraints**
- Topology spread constraint has a label mismatch
  — pods can all schedule to the same node with
  no warning, defeating the spread intent
- NPE bug in updateEntity — entity existence
  guard is inverted, causing unhandled 500
  instead of 400 when entity is not found
- No timeouts on outbound RestTemplate — IdP
  calls can block threads indefinitely, same
  risk as documented in API Gateway (1.1)
- No HPA — no automated response to load
- No CPU limits or requests — poor scheduling
  and risk of node CPU starvation
- S3 region hardcoded to eu-west-2 regardless
  of deployment environment — cross-region
  latency risk if bucket is in a different region
- @Async methods do not specify the configured
  thread pool by name — may fall back to
  unbounded SimpleAsyncTaskExecutor under load
- Platform stubs (CORE, VAP, LATAM, PIN) return
  400 "Implementation is in progress" in
  production — misleading error for any caller
  attempting non-NAP authorization
- In-memory aggregation of entity/merchant
  counts — large data volumes will cause heap
  pressure

**Planned changes**
- Case-level authorization to be consolidated
  to a single service regardless of platform,
  replacing the current NAP/PIN split between
  UserAccessManagementService and
  CoreHierarchyAuthorizationService (noted
  also in 1.1 API Gateway planned changes)

---

### 1.3 CoreHierarchyAuthorizationService
**Owner:** Core WDP Team
**Status:** ✅ Production | ✅ COMPLETE
**Technology:** Spring Boot / Java 17
**Repository:** gcp-core-hierarchy-authorization-service

---

**What it does**
CoreHierarchyAuthorizationService is a pure
authorization and read-only data service for the
PIN platform. It has two responsibilities:

1. **Runtime authorization** — validates whether
   a JWT-bearer's entity claims grant access to
   a specific merchant, chain, or hierarchy entity
   by cross-referencing token claims against the
   Core merchant hierarchy database. Called by
   the API Gateway for all PIN platform requests
   containing a case ID.

2. **Merchant hierarchy data API** — exposes
   read-only lookups of the Core merchant
   hierarchy structure, entity-type searches,
   and product entitlement data. Available to
   PIN platform callers only.

The service owns no data, performs no writes,
and has no state between requests. All data it
queries belongs to either the Core enterprise
merchant hierarchy system (DB2) or the WDP
dispute platform (PostgreSQL case table).

**What it does NOT do:**
- Issue or mint JWT tokens
- Write to any database — all access is
  read-only
- Handle NAP platform authorization — NAP
  requests to /authorize are explicitly rejected
  with 400. NAP platform is handled by
  UserAccessManagementService (1.2)
- Manage users, entities, or merchants — no
  CRUD operations of any kind
- Cache authorization results — every request
  results in live database queries

**Boundaries**

Inputs:
- API Gateway — POST /authorize calls for PIN,
  CORE, VAP, and LATAM platform case-level
  authorization checks
- PIN platform callers — merchant hierarchy
  data lookups and product entitlement queries

Outputs:
- PostgreSQL (WDP globaldisputedatabase) —
  read-only case lookup to resolve caseNumber
  to merchantId and chainId
- IBM DB2 (Core merchant master) — read-only
  entity relationship queries to validate JWT
  entity scope against merchant hierarchy.
  Tables: MD.TMD_ENTY, MD.TMD_ENTY_REL,
  DC.TDC_VIQ_ORG_ENTY, BC.TBC_MRCHNT_MAST_BO
  and related schemas. This is an enterprise
  system not owned by WDP.

**Authorization model**
Access control is enforced in three sequential
layers. A request must pass all three to succeed.

Layer 1 — Spring Security JWT validation.
Every request requires a valid Bearer JWT from
a trusted issuer. Requests without a valid JWT
are rejected with 401 before reaching any
application logic.

Layer 2 — Internal firm bypass.
If the JWT iss claim contains us_worldpay_fis_int
the caller is treated as an internal Worldpay
system and authorization is granted immediately
without any database lookup. Same pattern as
UserAccessManagementService (1.2).

Layer 3 — Entity scope check against Core DB2.
For all external callers, the service extracts
two JWT claims:
- iqorgid — the caller's org ID
- iqentities — semicolon/comma-delimited list
  of entity IDs the caller has access to

It then queries the Core DB2 entity relationship
tables to confirm the requested merchant or chain
is a descendant of the entities in the caller's
JWT scope. Chain (CH type) is checked first;
merchant (MT type) is used as fallback if no
chain is present. If no matching rows are found,
403 Forbidden is returned with a JSON error body.

Note: these JWT claims (iqorgid/iqentities) are
distinct from the NAP platform claims
(napParentEntity/napChildEntities) used by
UserAccessManagementService — different identity
models for PIN vs NAP platforms.

**The /authorize endpoint**
Called by the API Gateway for PIN, CORE, VAP,
and LATAM platform requests containing a case ID.

- Request: platform (not NAP), plus one of
  caseNumber, merchantId, or level4Entity
  (chain ID)
- If caseNumber provided: resolves to merchantId
  and chainId via WDP PostgreSQL case table
- Validates JWT iqorgid/iqentities claims against
  Core DB2 entity relationship tables
- Internal callers always pass — no DB lookup
- Returns HTTP 200 with empty body on success
- Returns 403 with JSON error body on failure
- Returns 400 if platform is NAP or if all of
  caseNumber, merchantId, and level4Entity
  are absent
- The 200 status code itself is the authorization
  signal — no body is returned on success

**Configuration & Scaling**
- Replica count is environment-specific, resolved
  at deploy time via XL Deploy variable. Actual
  counts per environment to be confirmed from
  XL Deploy configuration.
- Memory: 2048Mi limit / 1024Mi request.
  No CPU limits or requests configured.
- No HPA — scaling is purely static.
- Two database connections per pod — PostgreSQL
  (WDP case data) and IBM DB2 (Core merchant
  master). Both use HikariCP default pool of
  10 connections. Not explicitly configured.
- OpenTelemetry Java agent injected at runtime
  — adds memory and CPU overhead not accounted
  for in resource limits.
- Topology spread constraint configured but
  non-functional — label mismatch means all
  pods can schedule to the same node
  (see Risks).

**Key architectural decisions**
- Authorization only — no data ownership,
  no writes, no CRUD. Single responsibility
  contrast to UserAccessManagementService (1.2)
  which combines authorization with data
  management
- PIN/CORE/VAP/LATAM platform only —
  NAP platform explicitly rejected. Clear
  platform boundary between this service
  and UserAccessManagementService
- Dual database dependency — WDP PostgreSQL
  for case resolution plus IBM DB2 Core
  enterprise merchant master for entity
  scope validation. DB2 is not WDP-owned
- Internal firm bypass — consistent with
  platform-wide pattern; internal callers
  bypass all entity scope checks
- Chain-first, merchant-fallback — authorization
  attempts chain-level validation before
  falling back to merchant-level

**Risks & Constraints**
- No circuit breaker on Core DB2 dependency —
  if DB2 is slow or unavailable, all
  authorization requests fail with 500. No
  graceful degradation. Liveness probe does
  not check DB2 connectivity so pods will not
  restart. This is the highest severity risk
  in this service.
- Topology spread constraint has a label
  mismatch — same issue as 1.2 UAMS. All pods
  can schedule to the same node with no warning
- No CPU limits or requests — poor scheduling
  and risk of node CPU starvation
- No HPA — no automated response to load
- No connection pool tuning on either database
  — default 10 connections per datasource per
  pod. A single /authorize request may consume
  connections from both pools simultaneously
- No caching on authorization paths — every
  /authorize call results in live DB queries
  against both PostgreSQL and DB2. High-frequency
  callers generate proportionally high DB load
- Exception used for control flow in
  authorization fallback — genuine DB errors
  on the chain lookup path are silently swallowed
  and fall through to the merchant lookup,
  making it impossible to distinguish a missing
  chain from a DB failure
- Logstash TCP appender has no neverBlock or
  fallback configuration — if Logstash is
  unavailable, application threads can block
  waiting for the log socket buffer to drain

**Planned changes**
- Case-level authorization to be consolidated
  to a single service regardless of platform,
  replacing the current NAP/PIN split between
  UserAccessManagementService and
  CoreHierarchyAuthorizationService (noted
  also in 1.1 and 1.2)

---

## PART 2 — INBOUND PROCESSING

---

### 2.1 NAP/WPG Path

---

#### 2.1.1 NAPDisputeEventService
**Owner:** Integration Team
**Status:** ✅ Production | ✅ COMPLETE
**Technology:** Spring Boot 3.x / Java
**Repository:** mdvs-gcp-nap-dispute-event-service
**Maven artifact:** nap-event-service v1.4.4

---

**What it does**

NAPDisputeEventService is the inbound integration boundary between the NAP 
(Network Acquirer Processing) acquiring platform and the WDP internal event 
architecture. It acts as a pure REST-to-Kafka event bridge with enrichment 
logic — it receives raw dispute lifecycle notifications over REST, enriches 
each event by making synchronous lookups against internal WDP services, then 
publishes the enriched event as a message to AWS MSK (Kafka) for downstream 
consumption.

It handles two fundamentally different inbound paths:

**Path 1 — NAP-DPS dispute events (Kafka path)**
Receives new disputes, case updates, and win/loss outcomes from NAP-DPS and 
the WDP OPS Portal. Each event is validated, enriched via synchronous REST 
lookups against internal WDP services, and published to the `kafka.nap-topic` 
Kafka topic. The enrichment pipeline resolves product type, sub-type, fraud 
indemnity status, and reason codes before publishing.

**Path 2 — WPG response document uploads (direct proxy path)**
Receives base64-encoded evidence documents from the WDP OPS Portal and proxies 
them directly to DocumentManagementService to attach to NAP dispute cases. 
This path has no Kafka involvement, no enrichment, and no retry.

The service owns no persistent storage. It is entirely stateless — no database, 
no cache, no Kafka consumer side.

---

**Boundaries**

Inputs:
- From NAP Platform (NAP-DPS):
  `POST /event` — new chargeback dispute notification (SRV-116 message format).
  Sends new dispute events for migrated merchants.

- From WDP OPS Portal (`gcp-ops-portal-dev`):
  `POST /{caseId}/{uniqueId}` — case update (operator response actions:
    write-off, representment, merchant charge)
  `POST /outcome/{caseId}/{uniqueId}` — win/loss outcome recording
  `POST /response/document` — evidence document upload (base64-encoded,
    proxied to DocumentManagementService)

Outputs:
- Publishes enriched `NapEvent` objects to `kafka.nap-topic` on AWS MSK.
  Partition key: `merchantId` (or `cardAcceptorCodeId` for new events).
  Covers: new dispute events, case update events, win/loss outcome events.
  Document uploads do NOT publish to Kafka.

- Calls DocumentManagementService directly (REST POST) to attach evidence
  documents to NAP dispute cases — bypasses Kafka entirely.

Synchronous outbound REST dependencies (enrichment pipeline, Kafka path only):
- CaseManagementService — case lookup by ARN, networkCaseId, or caseNumber.
  Returns productType, subProductType, fraudIndemnityReason, merchantId.
- CaseActionService — resolves internal WDP case number from NAP
  sourceSystemCaseId + sourceSystemUniqueId.
- FraudSwitch (Transaction Lookup) — fraud transaction lookup by merchantId
  + gatewayTranId. Returns productEnrolled (GUARPAY1–GUARPAY5),
  fraudIndemnified, fraudIndemnityReason.
- DisplayCodeService — returns fraud and INR reason code lists for a given
  userId and displayCodeTypes. Used to determine TIER1 sub-product eligibility.
- ChargebackResponseRulesService — validates operator response actions
  (write-off, representment, merchant charge) against allowed rules for a
  given responseCode + responseType.
- IDPTokenService — fetches a bearer token per inbound request, used to
  authenticate all other outbound REST calls.

Transport:
- Listens on port 8082, context path `/merchant/gcp/nap`.
- Exposed via nginx Ingress in Kubernetes (TLS/HTTPS termination at ingress).
- Three Ingress hostnames configured: external, internal WPS, NAP-specific.
- Correlation ID (`v-correlation-id`) accepted on all endpoints, propagated
  via MDC through all outbound calls. UUID generated if absent.

---

**Key architectural decisions**

- **REST-to-Kafka bridge, not a processor** — this service routes and enriches;
  it makes no business decisions about dispute outcomes. Downstream Kafka
  consumers own all processing logic.

- **Push model from NAP** — NAP-DPS pushes SRV-116 events to WDP rather than
  WDP polling NAP. This is the inbound integration contract for migrated
  NAP merchants.

- **Two-path design with asymmetric behaviour** — the document upload path
  (`POST /response/document`) is architecturally distinct from all other paths:
  no enrichment, no Kafka, no retry. This is a deliberate direct-proxy pattern,
  not an omission.

- **Enrichment before publish, not after** — productType, subProductType, and
  fraudIndemnityStatus are resolved synchronously before the Kafka message is
  published. Downstream consumers receive a fully enriched event and do not
  need to call back to WDP services for this data.

- **Degraded-mode ingestion supported** — validation is conditionally bypassed
  when `enrichment_failure=true` OR `function_code=603`, allowing events that
  could not be enriched to still be ingested. The `enrichmentFailure` flag is
  carried on the Kafka event to signal this to consumers.

- **Stateless by design** — no database, JPA, cache, or Kafka consumer
  dependency. All state lives in the inbound request and outbound Kafka event.

- **Idempotent Kafka producer** — configured with `ENABLE_IDEMPOTENCE_CONFIG=true`
  and `MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION=1` to prevent duplicate messages
  on retry and preserve ordering within a partition.

- **Replica count managed externally via XLDeploy** — the pod replica count
  is an XLDeploy template placeholder resolved at deploy time per environment.
  No HPA is configured; scaling is entirely static and environment-managed.

- **Security enforcement delegated to network layer** — despite having OAuth2
  dependencies, all HTTP endpoints are whitelisted (`/**`). Authentication
  relies entirely on Ingress/firewall controls, not application-level token
  validation.

---

**Risks & Constraints**

🔴 HIGH — No HTTP timeout on RestTemplate
RestTemplate is instantiated with default constructor — no connection or read
timeout configured. If any downstream service (CaseManagement, FraudSwitch,
IDP, etc.) hangs, the inbound HTTP thread blocks indefinitely. Under load this
exhausts the embedded Tomcat thread pool and causes cascading unavailability.
No Resilience4j, Hystrix, or circuit-breaker library is present.

🔴 HIGH — Blocking Kafka publish on HTTP thread
`kafkaTemplate.send(...).get()` is called synchronously with no timeout.
The HTTP request thread is held for the full Kafka broker roundtrip. Combined
with `@Retryable`, worst-case blocking time per request =
`kafka_retry_count × kafka_retry_delay`. No async or timeout-bounded pattern
is used.

🔴 HIGH — CVV logged in plain text at INFO level (PCI-DSS)
`NapEvent.toString()` explicitly includes the CVV field. The full NapEvent
object is logged at INFO level immediately before Kafka publish
(`log.info("Kafka event: {} ", napevent)`). CVV is PCI-DSS 3.2.1 restricted
data — logging to Logstash constitutes effective persistence.
`CheckmarxUtil.sanitizeString()` is used elsewhere but not on this log call.

🔴 HIGH — All endpoints unauthenticated
Despite OAuth2 dependencies, SecurityConfig whitelists all paths (`/**`).
Any caller that can reach the Ingress can invoke dispute events, case updates,
win/loss outcomes, or document uploads without authentication. The
`gcp-ops-portal-dev` OAuth client registration ID indicates token validation
was intended but not implemented.

🟡 MEDIUM — Up to 5 serial synchronous REST hops per new dispute request
The `POST /event` enrichment pipeline can trigger: caseLookup →
fraudTransactionLookup → displayCodeDescription → plus IDP token fetch →
Kafka publish — all serial, all on the same HTTP thread. Each has its own
`@Retryable` loop. No overall request deadline exists. A single slow
downstream service degrades the entire dispute ingestion pipeline with no
partial enrichment fallback.

🟡 MEDIUM — No CPU limits or requests configured
Only memory limits (3072Mi) and requests (1024Mi) are configured. No CPU
limit or request is set. This prevents CPU-based HPA, creates noisy-neighbour
risk on the node, and gives the Kubernetes scheduler no CPU guarantee for
this pod.

🟡 MEDIUM — GUARPAY7 silently maps to TIER5 (same as GUARPAY5)
The product sub-type mapping sets `productSubType = "TIER5"` for both
GUARPAY5 and GUARPAY7. No `PRODUCT_SUB_TYPE_GUARPAY7` constant exists.
Downstream Kafka consumers cannot distinguish GUARPAY5 from GUARPAY7
merchants via the `productSubType` field alone. Whether this is intentional
is undocumented.

🟢 LOW — `RestServiceInvoker` autowired but unused in NapEventServiceImpl
A `@Autowired RestServiceInvoker restInvoker` dependency is declared in
NapEventServiceImpl but never called — all REST calls route through
`eventService`. Dead code that adds confusion to the dependency graph.

🟢 LOW — Silent field drop on BeanUtils.copyProperties
`BeanUtils.copyProperties(eventRequestDTO, napevent)` silently skips fields
where the property name does not match exactly between source and target.
Future additions to `EventRequestDTO` may be silently lost in `NapEvent`
with no exception or warning.

---

**Planned changes**
- No planned changes confirmed at time of writing.
- Open question: whether authentication should be enforced at application
  level (resolving the OAuth2 dependency vs. SecurityConfig whitelist
  inconsistency).
- Open question: GUARPAY7 → TIER5 mapping — confirm if intentional and
  document the decision.

---

#### 2.1.2 NAPDisputeEventProcessor
**Owner:** Integration Team
**Status:** ✅ Production | ✅ COMPLETE
**Repository:** gcp-nap-dispute-event-consumer
**Maven artifact:** gcp-nap-dispute-event-consumer v1.6.5
**Technology:** Spring Boot 3.5.7, Java 17, Spring Kafka, Spring Data JPA

---

**What it does**

NAPDisputeEventProcessor is a stateful, event-driven Kafka consumer that ingests
dispute notification messages from AWS MSK and translates them into case management
actions on the downstream NAP case management platform. It consumes a single unified
`NotificationEvent` payload from a single Kafka topic, classifies each event into one
of three sub-types (SRV116, WIN/LOSS, SRV118), and either creates a new dispute case
or appends an action to an existing one via synchronous REST calls to downstream
services.

The service does NOT produce to any Kafka topic. All outbound communication is HTTP
REST. It is the only inbound consumer operating on the NAP acquiring platform path —
all other acquiring platforms flow through CaseCreationConsumer.

It is currently in a hybrid migration state: a `wdpOnly` flag on each event gates
whether WDP-only (migrated legacy) records are processed differently from native NAP
cases. Win/Loss and SRV118 flows are entirely skipped for `wdpOnly = 'Y'` events.

---

**Boundaries**

Inputs:
- Consumes from: AWS MSK Kafka topic (configured via `${kafka_consumer_topic}`)
  Consumer group: `${kafka_group_id}`
  Event type: `NotificationEvent` JSON (single unified payload encoding all
  three sub-types: SRV116, WIN/LOSS, SRV118)
- Manual reprocessing: `POST /merchant/gcp/event-consumer/nap/event`
  Accepts a `ConsumerErrorId` to re-trigger processing of a previously
  failed event from the database error store.

Outputs:
- Creates new dispute cases — downstream Case Management REST API
  (first occurrence of a dispute; IN_SRV116 event, no existing case found)
- Updates existing dispute cases / appends actions — downstream Case
  Management REST API (subsequent events for an existing case)
- Inserts case notes — downstream Case Notes REST API
  (when `dataRecord` field is non-blank)
- Writes API log entries — downstream API Log REST API
  (when field mismatches detected: reversalIndicator, merchantId,
  transactionType)
- Writes error records — PostgreSQL `NAP.DISPUTE_EVENT_CONSUMER_ERROR`
  (on any downstream API failure or business rule failure; raw Kafka JSON
  payload stored for manual reprocessing)

Does NOT produce to any Kafka topic. No Kafka producer or KafkaTemplate
exists in the codebase.

---

**Event types processed**

All three sub-types arrive on the same Kafka topic as the same NotificationEvent
payload. Sub-type is determined at runtime by field inspection:

| Sub-type    | Identification condition                                              |
|-------------|-----------------------------------------------------------------------|
| IN_WIN_LOSS | `outcome` field is non-blank                                          |
| IN_SRV118   | Both `napResponseType` AND `napResponseCode` are non-blank            |
| IN_SRV116   | All other cases (default — new chargeback notification)               |

**IN_SRV116** — New Chargeback Notification. Creates or updates a case. For
MASTERCARD/MAESTRO, performs an MCM Queue lookup against the local database before
the new case action lookup. For VISA, uses `networkCaseId` for case lookup; all
other networks use `arn`. Duplicate detection prevents re-processing if the same
`acquirerCaseNumber` + `uniqueId` + `stageCode` + `actionCode` +
`creditDebitIndicator` already exists as a case action.

**IN_WIN_LOSS** — Arbitration Win/Loss Outcome. Updates an existing case with outcome.
Derives `actionCode`, `caseLiability`, and `owner` from the win/loss result and
counterparty type (ARB vs other). Events with `outcome = NA` or `CLOSED` are silently
skipped. Events where the max action has `migrationStatus = 'Y'` are silently skipped.

**IN_SRV118** — Chargeback Response (Representment / Write-off / Merchant Charge).
Looks up CBK response rules to construct one or more case actions (writeOff,
representment, merchantCharge, MACP, MDCL). Duplicate SRV118 action detection
prevents re-processing. Special case: response code 9952 + ARB stage forces the
action stage to ARB regardless of rules lookup result.

---

**Outbound REST dependencies**

All REST calls authenticate via a Bearer JWT token obtained from IDP Token Service
per message processing thread (ThreadLocal pattern, cleared in a `finally` block
after each event).

| # | Dependency                | Method | Purpose                                               |
|---|---------------------------|--------|-------------------------------------------------------|
| 1 | IDP Token Service         | GET    | Obtain JWT Bearer token for all subsequent calls      |
| 2 | Case Search               | GET    | Look up existing dispute case by ARN or NetworkCaseId |
| 3 | Create Case               | POST   | Create new dispute case                               |
| 4 | Update Case / Add Action  | POST   | Append new action to existing case                    |
| 5 | Workflow Rule Lookup      | POST   | Determine workflow name for new case                  |
| 6 | Case Action Search        | GET    | Retrieve existing action summaries for a case         |
| 7 | New Case Action Rules     | GET    | Determine stage code, action code, owner, SLA days    |
| 8 | CBK Response Rules        | GET    | Look up chargeback response rules (SRV118)            |
| 9 | Pre-Action Status Rules   | POST   | Determine prior action status before new action       |
| 10| Insert Notes              | POST   | Add free-text note from `dataRecord` field            |
| 11| API Log                   | POST   | Log field-mismatch diagnostic information             |

Database dependencies (via JPA / JDBC):
- PostgreSQL `NAP` schema — error tracking (`DISPUTE_EVENT_CONSUMER_ERROR`),
  MCM queue lookup (`mcm_queue_data`), timeframe rules

---

**Error handling and reprocessing**

There is no Kafka-level DLQ. The service uses a database-backed error queue:

On any downstream API failure or business rule failure, a `ConsumerError` row is
written to `NAP.DISPUTE_EVENT_CONSUMER_ERROR` with the raw Kafka JSON payload stored
for later reprocessing. The `C_ERROR_REASON` column records which pipeline step
failed and the exception message.

Error status lifecycle:
`FAILED1` → `FAILED2` → `ERROR` (on repeated failures)
`DRAFT` (blocked on prior error for same ARN)
`SKIPPED` / `SKIPPED_FAILED` (business-rule-skipped events)
`PROCESSED` (on successful manual reprocessing)

Manual reprocessing is triggered via `POST /merchant/gcp/event-consumer/nap/event`
with a `ConsumerErrorId`. In reprocessing mode the service does not write a new DB
record on failure — it returns error details to the caller in the HTTP response.

Before processing any new event, the service checks for prior unresolved ERROR-status
records for the same ARN/NetworkCaseId and blocks or escalates the new event if
prior errors exist.

---

**Kafka consumer configuration**

| Parameter              | Value / Config key                    |
|------------------------|---------------------------------------|
| Topic                  | `${kafka_consumer_topic}` (single)    |
| Consumer group         | `${kafka_group_id}`                   |
| Bootstrap servers      | `${kafka_bootstrap_servers}` (AWS MSK)|
| Offset commit mode     | `MANUAL_IMMEDIATE`, `syncCommits=true`|
| Auto commit            | `false`                               |
| Auto offset reset      | `latest`                              |
| Concurrency            | 1 (single thread per pod, default)    |
| Auth to MSK            | IAM (SASL_SSL / AWS_MSK_IAM)          |
| Deserializer           | `JsonDeserializer<NotificationEvent>` |
|                        | wrapped in `ErrorHandlingDeserializer`|

---

**Retry configuration**

Retry implemented via Spring Retry (`@Retryable` / `@Recover`) on
`NapEventProcessingService`. Retry count and delay are environment-variable driven
(`${consumer_retrycount}` / `${consumer_retrydelay}`). Production values not visible
in repository.

No circuit breaker (Resilience4j, Hystrix) is configured. No resilience4j or
spring-cloud-circuitbreaker dependency is present in `pom.xml`.

---

**Scaling and deployment**

| Parameter         | Value                                                  |
|-------------------|--------------------------------------------------------|
| Replica count     | XL Deploy placeholder — externally injected at deploy |
| HPA               | None — scaling is manual (replica count change only)   |
| Memory request    | 2048Mi                                                 |
| Memory limit      | 4096Mi                                                 |
| CPU request       | Not set                                                |
| CPU limit         | Not set                                                |
| Deployment type   | Kubernetes Deployment (not StatefulSet)                |
| Rollout strategy  | RollingUpdate — maxSurge: 1, maxUnavailable: 0         |
| PodDisruptionBudget | None                                                 |
| Topology spread   | ScheduleAnyway (soft constraint, max skew 1)           |

---

**Key architectural decisions**

- **At-most-once Kafka delivery (deliberate deviation from DEC-005)** — offset is
  acknowledged BEFORE processing begins, not after. This is the opposite of the
  platform standard defined in DEC-005 (manual commit after full processing). The
  deliberate intent is to avoid Kafka redelivery and manage all error state via the
  `DISPUTE_EVENT_CONSUMER_ERROR` database table instead. ⚠️ If the JVM crashes or
  pod is OOM-killed after ACK but before processing completes, the event is
  permanently lost — the error table is only written during processing, not before.
  This is the highest severity risk in this component.

- **Database-backed DLQ over Kafka DLQ** — no Kafka dead-letter topic. All failed
  events are stored in PostgreSQL with full raw payload for manual reprocessing.
  This is a deliberate alternative to the Kafka DLQ pattern used elsewhere.

- **No circuit breaker (deviation from DEC-014)** — this component has no
  Resilience4j dependency. A single slow downstream call will block the entire
  consumer thread indefinitely given the bare `RestTemplate` with no timeout
  configuration. With concurrency=1, this halts all message processing.

- **Bare RestTemplate with no timeouts** — `RestTemplate` is instantiated as a
  no-arg bean with no connection timeout, read timeout, or connection pool. Under
  hung downstream calls, `max.poll.interval.ms` will eventually be exceeded,
  causing a consumer group rebalance and partition re-assignment.

- **Single-threaded processing (concurrency=1)** — throughput scales only with pod
  count. No KEDA or lag-based autoscaling. Capacity increase requires manual replica
  count change via XL Deploy.

- **Hybrid migration state** — `wdpOnly` / `migrationStatus` flag is active in
  production. Some cases exist in a legacy system and are processed differently from
  native NAP cases. Win/Loss and SRV118 flows skip entirely for `wdpOnly = 'Y'`
  events. Migration is ongoing and gated by this runtime flag.

- **SEND_BUSINESS_RULES_INFO_TO_KAFKA is a no-op terminal state** — the pipeline
  sets this as the final `apiName` after every successful execution but no code
  ever acts on it. No Kafka produce occurs. Infrastructure exists for a planned
  business-rules downstream publish feature but is not yet implemented.

---

**Risks and constraints**

🔴 HIGH — At-most-once delivery: event permanently lost if pod crashes after
offset ACK but before processing completes. No broker-level safety net. Directly
conflicts with DEC-005. Unrecorded chargebacks are the failure consequence.

🔴 HIGH — Bare RestTemplate with no timeouts and no circuit breaker: a single
hung downstream call blocks the entire consumer thread indefinitely. With
concurrency=1 this halts all message processing. Directly conflicts with DEC-014.

🟡 MEDIUM-HIGH — Recursive error reprocessing: `checkAndProcessConsumerErrorSrvNotification()`
iterates all prior ConsumerError records for an ARN and calls the full pipeline for
each. For an ARN with many accumulated error records, this creates a large
synchronous HTTP call tree inside the single Kafka consumer thread, risking
`max.poll.interval.ms` violation and consumer group rebalance.

🟡 MEDIUM — No PodDisruptionBudget: Kubernetes node drains will evict all pods
simultaneously. Combined with at-most-once delivery, any mid-processing eviction
causes permanent event loss.

🟡 MEDIUM — No CPU limits or requests: pod is Burstable QoS class; first candidate
for eviction under node memory pressure. JVM CPU consumption is unbounded.

🟡 MEDIUM — MCM Queue multi-result silent ONHOLD: if an ARN maps to more than one
MCM claim ID, the event is silently placed on ONHOLD with a misleading error message
("MCM claim id not exist") even though data does exist. Recovers only after
`napEventExpiryHour` timeout.

🟡 MEDIUM — SEND_BUSINESS_RULES_INFO_TO_KAFKA is dead: downstream consumers
expecting a business-rules Kafka message will never receive one. Silent incomplete
feature.

🟢 LOW — Shared RestTemplate error handler mutation: `IdpRestInvoker` calls
`restTemplate.setErrorHandler()` on a shared singleton bean. Not thread-safe if
concurrency > 1; currently benign at concurrency=1 but a latent race condition.

🟢 LOW — ThreadLocal token not cleared on exception in manual reprocessing path:
the `finally` block clearing the IDP JWT token exists on the Kafka listener path
but not on the `processConsumerErrorInformation()` REST path.

---

**Planned changes and migration work**

- **Active migration:** `wdpOnly` flag gates ongoing CB911 migration. srv117 and
  vin-loss event types (non-migrated CB911 merchants) still processed via this
  service. Migration timeline: TBD.

- **Commented-out Fraud Switch Lookup:** `FRAUD_SWITCH_LOOKUP` pipeline step exists
  as a constant and has URL configuration in `application.yaml` but the call site
  in `caseLookup()` is commented out. Deferred — URL placeholder must still be
  provided as an env-var even though it is never called.

- **Commented-out VISA Case Lookup:** `visaCaseLookup()` method body and both
  call sites are commented out. Removed in favour of the general `caseLookup()`
  handling VISA via `networkCaseId`.

- **Amount calculation logic replacement (US2137854):** Old `calculateWorkableAmount()`
  method replaced by direct field mapping. Dead method body remains in codebase
  (lines 1067-1101 of `CaseServiceImpl`).

- **SEND_BUSINESS_RULES_INFO_TO_KAFKA feature:** Infrastructure exists (entity,
  repository, service methods) for publishing enriched business-rules data
  downstream. Feature not yet implemented.

- **JCB and VISA_ALLOCATION workflow types:** Constants referenced in
  `CaseServiceSupport` suggest active development for JCB card network and VISA
  allocation flows beyond current VISA/MASTERCARD/MAESTRO/AMEX handling.

---

#### 2.1.3 NAPDisputeDeclineBatch
**Owner:** Integration Team
**Status:** ✅ Production | ✅ COMPLETE
**Repository:** gcp-visa-issuer-decline-batch
**Maven artifact:** visa-issuer-decline-batch v1.1.1
**Technology:** Java 17, Spring Boot 3.5.7, Spring Batch

---

**What it does**

NAPDisputeDeclineBatch is a scheduled Spring Batch job that processes
Visa issuer pre-arbitration decline cases on the NAP platform. It reads
open PAB (Pre-Arbitration) action records from the NAP PostgreSQL database,
verifies each case is VISA via the Case Management service, queries the
Visa network via the internal Visa Adapter HyperSearch, and — if the Visa
case is in a Pre-Arb Declined state — creates a new IDCL (Issuer Decline)
draft action via the Case Actions service to advance the dispute workflow.

It is VISA-only and NAP-platform-only. All non-VISA cases are immediately
skipped. It does NOT poll the Visa API directly — Visa network access is
mediated entirely through the internal Visa Adapter service.

It does NOT produce to any Kafka topic, write to any file, or update the
`nap.action` source rows after processing.

---

**Boundaries**

Inputs:
- PostgreSQL `nap.action` table (read only)
  Query filters: `C_CASE_STAGE = 'PAB'`, `C_ACTION_TYPE = 'OPAB'`,
  `C_ACTION_STA = 'OPEN'`, `C_MIGRATION_STA = 'Y'`,
  `Z_INSRT > (now - ${past_days})`
  Cursor-based pagination — page size `${number_of_records}`
- IDP Token Service (GET) — bearer token bootstrap once per job run
- Case Management Service (GET) — case lookup to verify `cardNetwork = VISA`
- Visa Adapter HyperSearch (POST) — Visa network case lookup returning
  `stageStateDesc`, `disputePreArbResponseId`, deadlines, amounts
- Case Actions Service (GET) — retrieve existing action's financial details
  to copy into the new action

Outputs:
- Case Actions Service (POST) — creates new `IDCL` draft action on the
  dispute case with `stageCode = PAB`, `actionStatus = DRAFT`,
  `userId = NPRBDECLB`, copying financial amounts from the existing action
- Spring Batch metadata tables — `BATCH_JOB_INSTANCE`,
  `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` etc. written to WDP
  PostgreSQL (separate datasource from NAP PostgreSQL)
- Logstash / ELK — structured log shipping via TCP appender

Does NOT write to Kafka. Does NOT update `nap.action` rows after
processing. Does NOT write to any file or S3 path.

---

**Processing logic**

For each qualifying `nap.action` record:

1. Call Case Management GET — if `cardNetwork != VISA` → skip record
2. Call Visa Adapter HyperSearch POST using `networkCaseId`
3. Check `stageStateDesc` — only two values trigger action creation:
   - `"Fraud Dispute - Pre-Arb Declined"`
   - `"Authorization Dispute - Pre-Arb Declined"`
   All other values → skip record
4. Verify `disputePreArbResponseId` is present → skip if absent
5. Call Case Actions GET — retrieve existing action financial details
6. Call Case Actions POST — create IDCL draft action

Owner assignment on the new action:
- `"SIGNIFYD"` if `subProductType == "TIER1"` (case-insensitive)
- `"MERCHANT"` in all other cases

Each record is processed and committed individually (chunk size = 1).
Each record triggers up to 4 serial REST calls.

---

**Trigger and schedule**

Deployed as a Kubernetes **Deployment** (not a CronJob). The container
runs continuously with an internal Spring `@Scheduled` cron loop.

| Parameter      | Config key             | Value                          |
|----------------|------------------------|--------------------------------|
| Cron schedule  | `app.scheduler.cron`   | `${scheduler_cron}` — K8s secret|
| Look-back window | `app.batchProperties.pastDays` | `${past_days}` — K8s secret |
| Page size      | `app.batchProperties.numberOfRecords` | `${number_of_records}` — K8s secret |
| Chunk size     | `app.batchProperties.chunk` | `1` — hardcoded all envs     |
| Batch user ID  | `app.batchProperties.userId` | `NPRBDECLB` — hardcoded      |

Actual cron schedule and volume parameters are not visible in source —
injected at runtime from Kubernetes secret
`gcp-visa-issuer-decline-batch-secrets`.

Each job execution is made unique by appending a timestamp parameter
(`date=yyyyMMdd_HHmmss.SSS`) to `JobParameters` so Spring Batch does
not deduplicate runs at the infrastructure level.

---

**Scaling and deployment**

| Parameter            | Value                                              |
|----------------------|----------------------------------------------------|
| Replica count        | XL Deploy placeholder — externally injected        |
| HPA                  | None — scaling is manual                           |
| Memory request       | 512Mi                                              |
| Memory limit         | 2048Mi                                             |
| CPU request          | Not set                                            |
| CPU limit            | Not set                                            |
| Deployment type      | Kubernetes Deployment (not CronJob)                |
| Rollout strategy     | RollingUpdate — maxSurge: 1, maxUnavailable: 0     |
| PodDisruptionBudget  | None                                               |
| Topology spread      | ScheduleAnyway (soft constraint, max skew 1)       |
| Observability        | OpenTelemetry Java agent injected via K8s annotation|

---

**Error handling**

Spring Batch chunk size = 1. Failures are isolated per record:

- `BatchItemProcessor` catches all exceptions, logs them, and returns
  `null` — Spring Batch treats `null` as skip. One record failure does
  NOT stop the batch.
- `BatchItemWriter` catches and logs per-item exceptions within the
  for-each loop. Processing continues to the next record.
- **No retry mechanism** — `spring-retry` is declared in `pom.xml` but
  unused. No `@Retryable`, `RetryTemplate`, or Spring Batch
  `.faultTolerant().retry()` is configured.
- **No dead-letter queue or error table.** Failed records are not
  tracked anywhere in the application.
- **No status update on `nap.action` after failure.** Failed records
  retain `C_ACTION_STA = 'OPEN'` and will be re-queued on the next
  scheduled run if still within the `pastDays` look-back window.

If the IDP token call fails, all records in that run will fail silently
— there is no job-level abort on token failure.

---

**Idempotency**

There is no explicit idempotency mechanism in this service. Specifically:

- No "processed" status flag is set on `nap.action` rows after success.
- Records will be re-queried on every run while `C_ACTION_STA = 'OPEN'`
  and within the `pastDays` window.
- If multiple replicas are deployed, each fires the cron independently,
  reads the same records, and may create duplicate IDCL actions. No
  distributed locking exists.
- The only partial de-duplication relies on the downstream Case Actions
  service enforcing its own duplicate prevention — not this service.

---

**Key architectural decisions**

- **DB-first, not push-driven** — the job reads from `nap.action`
  (PostgreSQL) rather than being triggered by Kafka or a card network
  push event. This is a pull/poll pattern scoped entirely to the NAP
  platform.

- **Visa access via internal adapter, not direct** — Visa network
  queries go through `mdvs-gcp-visa-adapter` HyperSearch, not a direct
  Visa API call. The architecture diagram in WDP-ARCHITECTURE.md
  describing this as "polls Visa API" is a simplification.
  ⚠️ WDP-ARCHITECTURE.md needs correction on this point.

- **Migration-gated processing** — `C_MIGRATION_STA = 'Y'` is a hard
  filter on all queries. Only records explicitly flagged as migrated are
  eligible. This is a transitional component in the CB911 → WDP
  migration. As migration completes, this filter becomes a no-op.

- **Continuous Deployment, not CronJob** — running as a Deployment with
  an internal scheduler keeps the JVM warm between cron fires. The
  trade-off is that replica count directly controls whether the cron
  fires once or multiple times simultaneously, creating duplicate
  processing risk with replicas > 1.

- **Bare RestTemplate with no timeouts** — same pattern as
  NAPDisputeEventProcessor. With chunk size = 1 and 4 serial REST calls
  per record, a single hung downstream call blocks that record
  indefinitely with no timeout or circuit breaker.

---

**Risks and constraints**

🔴 HIGH — No idempotency + no status update: if replicas > 1, or if
the job runs twice in overlapping windows, duplicate IDCL actions will
be created. No application-level guard exists. Entirely dependent on
downstream Case Actions service for de-duplication.

🟡 MEDIUM — No retry mechanism: `spring-retry` dependency is present
but unused. Transient failures (network blip, downstream timeout) cause
silent record skip with no retry attempt. Records must wait for the
next scheduled run to be re-tried.

🟡 MEDIUM — No error tracking: failed records are not written to any
dead-letter store or error table. There is no operational visibility
into which records failed or why, beyond log entries in ELK.

🟡 MEDIUM — Bare RestTemplate with no timeouts and no circuit breaker:
a hung downstream call (Case Management, Visa Adapter, Case Actions)
blocks the processing thread for that record indefinitely.

🟡 MEDIUM — No PodDisruptionBudget: node drains will terminate the
running pod mid-batch with no constraint. Any in-flight record at the
time of eviction will be re-processed on the next run (duplication
risk compounds with the idempotency gap).

🟡 MEDIUM — No CPU limits or requests: pod is Burstable QoS; first
candidate for eviction under node memory pressure.

🟢 LOW — STG and Test environments share dev namespace URLs:
`application-stg.yaml` and `application-test.yaml` both resolve to
`*.develop.gcp-ff:8082` endpoints. STG and Test are not isolated from
dev for this service.

🟢 LOW — Unused `AddActionResponse`: the POST response from Case
Actions (containing `caseNumber`, `actionSequence`) is captured but
never read, logged, or validated. Success is assumed if no exception
is thrown.

---

**Planned changes and in-progress work**

- **Planned Kafka event publish (not yet implemented):** Strong
  structural evidence of a planned feature — an unused `Event` model
  (`platform`, `caseNumber`, `actionSequence`, `startRuleGroup`,
  `source`), two unused constants (`VISA_ISS_DEC = "BRVISD"`,
  `START_RULE_GROUP = "ISSUER_DOCUMENTS"`), and three staged Kafka POM
  dependencies (`spring-kafka`, `kafka-clients`, `aws-msk-iam-auth`)
  all point to a planned post-IDCL Kafka event to trigger downstream
  `ISSUER_DOCUMENTS` rule evaluation. No publishing code is implemented.
  Needs an ADR or decision record.

- **Planned retry logic (not yet implemented):** `spring-retry` and
  `spring-aspects` declared in POM but unused. Intended to add
  `@Retryable` to REST call methods. Not yet wired.

- **Migration flag removal:** Once CB911 migration is complete,
  `C_MIGRATION_STA = 'Y'` filter becomes a no-op and should be removed.
  Timeline: TBD — tied to CB911 migration programme.

---

### 2.2 Card Network Batch Path

---

#### 2.2.1 VisaDisputeBatch
**Owner:** Integration Team
**Status:** ✅ Production | ✅ COMPLETE
**Repository:** gcp-visa-disputes-processor-batch
**Maven artifact:** com.wp.gcp:visa-disputes-processor-batch v1.2.5
**Technology:** Spring Boot 3.5.3 / Spring Batch / Java 17

---

**What it does**

VisaDisputeBatch is the sole inbound batch responsible for ingesting
all Visa dispute lifecycle events into WDP. It polls Visa's RTSI
(Real-Time Settlement Interface) batch queues on a cron schedule,
processes all seven Visa queue types sequentially in a single job
execution, enriches each event against multiple internal WDP services,
and writes structured case events to the `wdp.chbk_outbox_row`
transactional outbox for downstream Kafka publishing. PAN is encrypted
before any persistence occurs.

This batch is the **origin of all Visa dispute data in WDP**. If this
batch fails or falls behind, no new Visa dispute cases are created or
updated downstream.

**What it explicitly does NOT do:**
- Does not process Mastercard disputes — Visa-only
- Does not publish to Kafka directly — writes to outbox only; a
  separate Kafka publisher reads PENDING rows and publishes
- Does not send dispute responses or outcomes back to Visa — only
  acknowledgement via mark-as-read
- Does not perform PAN encryption itself — delegates entirely to
  wdp-encryption-service before writing

---

**The Seven Visa Queues**

All seven queues are processed in a single job execution, strictly
sequentially. Each queue maps to a distinct Visa dispute lifecycle stage:

| Queue             | Visa Queue Name                   | Dispute Stage                          |
|-------------------|-----------------------------------|----------------------------------------|
| FIRSTCHARGEBACK   | AWAITING_ACTION_BQ_DISPUTE        | First chargeback — acquirer must respond |
| PREARB            | AWAITING_ACTION_BQ_PRE_FILING     | Pre-arbitration filing pending         |
| ARB               | INCOMING_BQ_ARBITRATIONS          | Incoming arbitration filing            |
| ACCEPT            | INCOMING_BQ_ACCEPTANCES_RECEIVED  | Issuer acceptance of chargeback        |
| RECALL            | INCOMING_BQ_RECALLS               | Issuer recall of previously filed dispute |
| PRECOMP           | INCOMING_BQ_PRECOMPLIANCES        | Incoming pre-compliance                |
| COMP              | INCOMING_BQ_COMPLIANCES           | Incoming compliance filing             |

Queue order and list are controlled by the `queueNameList` environment
variable injected at runtime. Queue exhaustion is detected via Visa RTSI
status code `I-300600000` — when returned, the reader advances to the
next queue without failing the job.

---

**Processing Pipeline**

For each item read from Visa RTSI, the processor executes these steps
in sequence:

1. **Platform mapping** — derives platform from networkId:
   Visa networkId → CORE platform; networkId 0002 → PIN platform (initial)

2. **IDP token acquisition** — obtains Bearer token from
   wdp-idp-token-service for all subsequent internal WDP calls

3. **Queue status rules check** — calls mdvs-gcp-rules-service to
   determine if the item should be skipped and to obtain the
   assigned dispute stage. If no rule found, item is skipped.

4. **HyperSearch enrichment** — calls Visa RTSI HyperSearch to retrieve
   rich case details: dispute IDs, transaction details, merchant info,
   document lists, filing item IDs, reason codes, and fraud classification

5. **PIN validation** (networkId 0002 only) — calls Visa Transaction
   Search to confirm `CardholderIdMethodCode = "2"` for PIN routing.
   If not PIN and not CORE migration eligible, item is skipped and
   not marked as read

6. **Migration status** — migration status hardcoded to `"Y"` on all items

7. **Case lookup** — calls mdvs-gcp-case-search-service to determine
   whether to create a new WDP case or update an existing one.
   PRECOMP and COMP queues have queue-specific lookup logic.

8. **PAN encryption** — if account number is present, calls
   wdp-encryption-service. Encrypted hpan replaces plain PAN on the
   case event before any persistence

9. **Write to outbox** — writes case event to `wdp.chbk_outbox_row`
   with duplicate check (except RECALL queue). Calls Visa RTSI
   MarkAsRead for eligible platform items.

---

**Boundaries**

Inputs — Visa RTSI APIs (all via DataPower Gateway, auth via vantiveLicense):
- **GetBatchQueue** — polls each of the 7 queues; server-driven
  pagination; all pages loaded into memory before processing begins
- **HyperSearch** — rich Visa case detail enrichment per item
- **MarkAsRead** — acknowledges item consumption; called only for PIN
  platform items and CORE items where coreDataNeedToMigrate=true
- **VisaTransSearch** — PIN platform routing confirmation (networkId
  0002 only)
- **PreArbDetail** — configured and implemented but currently inactive
  (commented out in processor). PAB and FAR stage dispute detail.
- **PreArbResponseDetail** — configured and implemented but currently
  inactive (commented out in processor)

Inputs — Internal WDP Services (all Bearer token authenticated via IDP):
- **wdp-idp-token-service** — IDP token, called once per item
- **mdvs-gcp-rules-service** — queue status rules and skip decision
- **mdvs-gcp-case-search-service** — case lookup for create vs update
- **wdp-encryption-service** — PAN encryption before persistence
- **gcp-api-log-service** — error logging via AOP on all
  LoggingException throws (up to 10,000 chars of stack trace)
- **gcp-merchant-transaction-service** — implemented but permanently
  inactive (commented-out code path replaced by Visa TX Search)

Outputs:
- **wdp.chbk_outbox_row** (PostgreSQL) — dispute case events written
  with status PENDING (ready for Kafka), SKIPPED (duplicate detected),
  or ERROR (duplicate check could not be performed due to missing data)
- **Visa RTSI MarkAsRead** — conditional; not called for CORE platform
  items where coreDataNeedToMigrate=false, allowing a separate core
  processor to consume them

The outbox table holds full case event JSON payload, Visa case ID,
phase ID, dispute stage, platform (PIN or CORE), card network (always
VISA), network transaction ID, and audit fields. Kafka topic, partition,
and offset columns are populated by the downstream Kafka publisher,
not by this batch.

---

**Concurrency and ordering**

All seven queues are processed **strictly sequentially in a single
thread** within a single Spring Batch job execution. There is no
parallelism, partitioning, or multi-threading. Items within a queue
are processed in the order returned by Visa RTSI — no re-ordering is
applied. One job execution = one cron fire = all queues processed end
to end.

---

**Trigger and schedule**

Deployed as a Kubernetes **Deployment** (not a CronJob). A long-running
pod runs continuously, with an embedded Spring `@Scheduled` cron
triggering job execution at the configured interval.

| Parameter         | Value                                              |
|-------------------|----------------------------------------------------|
| Cron schedule     | Externalized via K8s secret `scheduler_cron`       |
| Kubernetes type   | Deployment (not CronJob)                           |
| Job auto-start    | Disabled — driven by internal cron only            |
| Chunk size        | 1 (hardcoded) — every item its own transaction     |
| Run uniqueness    | Timestamp parameter appended to JobParameters      |

The actual cron expression is not visible in source — injected via the
`gcp-visa-disputes-processor-batch-secrets` Kubernetes secret.

---

**Scaling and deployment**

| Parameter           | Value                                            |
|---------------------|--------------------------------------------------|
| Replica count       | XL Deploy placeholder — externally injected      |
| HPA                 | None — scaling is entirely manual                |
| Memory limit        | 2048Mi (2 GiB)                                   |
| Memory request      | 256Mi                                            |
| CPU limits          | Not set — Burstable QoS class                    |
| Rollout strategy    | RollingUpdate: maxSurge=1, maxUnavailable=0      |
| minReadySeconds     | 30 seconds                                       |
| PodDisruptionBudget | None defined                                     |
| Topology spread     | None configured                                  |
| Observability       | OpenTelemetry Java agent injected via annotation |

K8s secrets mounted: `gcp-visa-disputes-processor-batch-secrets` (RTSI
URLs, license key, scheduler cron, DB password) and `wdp-common-secrets`
(shared WDP settings including active profile and log level).

---

**Error handling and retry**

- **Retry:** All REST calls (both Visa RTSI and internal WDP services)
  use Spring Retry — 3 attempts, 2-second backoff, retries on any
  Exception
- **Item-level isolation:** Processor catches all exceptions and
  returns null, skipping the item. One item failure does not stop the
  batch. With chunk size = 1, writer failure also skips that single item.
- **AOP error logging:** All LoggingException throws trigger an AOP
  aspect that posts error details (platform, network, networkCaseId,
  caseNumber, up to 10,000 chars of stack trace) to gcp-api-log-service
- **Dead letter:** No DLQ or dead letter table. Failed items (null from
  processor) are simply not written. The only error persistence is the
  `status = ERROR` row in chbk_outbox_row for duplicate-check failures
  caused by missing data.

---

**Idempotency**

- **Primary guard:** Writer performs duplicate check against
  `wdp.chbk_outbox_row` on (networkCaseId + networkPhaseId +
  disputeStage) before saving any non-RECALL item. Match → SKIPPED
  status. Missing fields → ERROR status.
- **RECALL queue exception:** RECALL items explicitly bypass the
  duplicate check. No idempotency protection for RECALL items.
- **Pod crash before MarkAsRead:** Items not yet marked as read will
  be re-polled from Visa RTSI on the next run. The chbk_outbox_row
  duplicate check is the guard against double-writing these items.
- **CORE platform items:** Intentionally not marked as read when
  `coreDataNeedToMigrate=false` — consumed by a separate core processor
  in a subsequent run. Not a bug.
- **No Spring Batch JobRepository-level idempotency:** Each run is
  made unique by timestamp parameter — Spring Batch does not deduplicate
  at the infrastructure level.

---

**Key architectural decisions**

- **Pull model for Visa** — WDP polls Visa RTSI rather than Visa
  pushing events. Polling interval controlled by externalized cron.
  No rate limiting is implemented — Visa RTSI quota management relies
  solely on the implicit serialization of sequential HTTP calls.

- **Transactional outbox pattern** — batch writes PENDING rows to
  `wdp.chbk_outbox_row`; a separate Kafka publisher reads and publishes
  to Kafka. This decouples inbound ingestion from Kafka availability.

- **PAN encrypted at ingestion boundary** (DEC-004) — plain account
  number is never persisted. Encryption delegated to
  wdp-encryption-service; batch performs no crypto itself.

- **Sequential processing — deliberate design** — single-threaded
  processing eliminates concurrency complexity at the cost of throughput.
  Volume is bounded by what Visa RTSI returns per queue per run.

- **Kubernetes Deployment not CronJob** — JVM stays warm between cron
  fires, avoiding cold-start latency. Trade-off: replicas > 1 creates
  parallel queue polling risk with no distributed lock.

- **MarkAsRead gated by platform and migration flag** — CORE items
  with `coreDataNeedToMigrate=false` are left unconsumed for a separate
  core processor. PIN items and migrating CORE items are marked as read.

- **DataPower as Visa RTSI gateway** — all Visa API calls go through
  DataPower Gateway using vantiveLicense static key auth, not direct
  Visa endpoint calls.

---

**Risks and constraints**

🔴 HIGH — Replicas > 1 creates parallel Visa queue polling with no
distributed lock. Multiple pods would each fire the cron independently,
poll the same Visa queues simultaneously, and attempt to write the same
events. The chbk_outbox_row duplicate check is the only guard — but it
is per-item and only fires at write time, not at poll time. Duplicate
PENDING rows before the duplicate check runs are not protected against.
Replica count should be 1 until a distributed coordination mechanism
is in place.

🔴 HIGH — No HPA. Scaling is entirely manual via XL Deploy placeholder.
No autoscaling under load or queue backlog.

🟡 MEDIUM — No rate limiting on Visa RTSI API calls. No RateLimiter,
throttle, or sleep between API requests. Quota risk if Visa enforces
API call limits per time window. Only implicit pacing is the sequential
nature of processing.

🟡 MEDIUM — All pages for a queue loaded into memory before processing
begins. Very large queue responses could cause heap pressure given
2GiB memory limit and 256Mi request.

🟡 MEDIUM — RECALL queue has no idempotency protection. If the pod
restarts after polling but before writing, RECALL items will be
re-processed with no duplicate guard.

🟡 MEDIUM — No PodDisruptionBudget defined. Node drain during job
execution terminates the pod mid-batch. Items not yet marked as read
will be re-polled; RECALL items may be duplicated.

🟡 MEDIUM — No CPU limits. Pod runs Burstable QoS — first candidate
for eviction under node memory pressure.

🟡 MEDIUM — Cron expression not auditable from source repository.
Fully externalized via K8s secret. Schedule changes require secret
update, not a config or code change.

🟢 LOW — No circuit breaker on internal service calls. A hung response
from any of wdp-idp-token-service, mdvs-gcp-rules-service,
mdvs-gcp-case-search-service, or wdp-encryption-service blocks the
processing thread for that item until retry exhaustion (3 attempts,
2s apart = ~4 seconds minimum per hung call, ~6s max).

🟢 LOW — `gcp-merchant-transaction-service` URL is configured in all
environments but the call is permanently commented out. Dead config
that could mislead operators.

🟢 LOW — PreArbDetail and PreArbResponseDetail Visa RTSI endpoints are
configured in all environments but never invoked at runtime. Dead
configuration.

---

**Deferred and inactive features**

The following are implemented but not active in the current code path.
They represent deferred or removed features and should be understood
before making changes to this batch:

- **Internal Transaction Lookup (major, removed):** A full alternative
  PIN/CORE routing algorithm using gcp-merchant-transaction-service
  was removed and replaced by Visa Transaction Search. The dead code
  block remains in BatchItemProcessor. The transactionUrl config
  property remains in all profiles. This service call is never made.

- **Pre-Arb Dispute Detail Call (deferred):** getDisputeDetail() is
  fully implemented in VisaRTSIServiceImpl covering both PAB and FAR
  stages, but the call site in the processor is commented out. The
  VisaDisputeDetail field on the processed item is always null at
  runtime. Both prearbDetailUrl and prearbResponseDetailUrl are
  configured but unused.

- **Dead Kafka Infrastructure (deferred):** MSK IAM authentication
  constants interface and Spring Kafka / AWS MSK IAM auth dependencies
  are present but zero Kafka producers or consumers are wired. Kafka
  publishing was either planned and moved to the separate outbox
  publisher, or is deferred for a future version.

- **readSpecificItemFromQueue (active — surgical testing tool):**
  An undocumented feature flag that, when enabled, filters all queue
  polling to only a pre-configured list of specific case numbers.
  Intended for targeted re-processing or manual testing without a
  code change. This is active production configuration — operations
  teams must be aware it exists.

- **Unused enum values:** CardNetwork enum defines 7 network values
  but only VISA is ever used. CaseLiability, TransactionStatus, and
  ChbkEventType.EVIDENCE_ATTACH enums are defined but never referenced
  in active code. Fraud model classes (FraudSightDetailMtr3,
  SignifydDetail, etc.) exist only in the commented-out transaction
  lookup response path.

---

**Planned changes**

- Clean up commented-out transaction lookup code block once confirmed
  the Visa Transaction Search replacement is stable
- Activate or formally remove the Pre-Arb Dispute Detail call —
  current state leaves configured RTSI endpoints permanently unused
- Clean up dead Kafka dependencies (spring-kafka, aws-msk-iam-auth)
  if Kafka publishing responsibility stays with the outbox publisher
- Evaluate distributed locking or replica count enforcement to make
  replicas > 1 safe
- Formally document or remove readSpecificItemFromQueue — currently
  an undocumented surgical tool with no ADR
- Remove dead transactionUrl config property from all profiles
#### 2.2.2 FirstChargebackBatch
**Owner:** Integration Team
**Status:** ✅ Production | ✅ COMPLETE
**Technology:** Spring Boot 3.5.3 / Spring Batch / Java 17
**Repository:** wdp-mcm-first-chargeback-queue-batch

---

**What it does**

FirstChargebackBatch is a continuously-running Spring Batch application
that polls the MasterCard Dispute Management (MCM) queue for unworked
first chargeback claims, enriches each claim with settled transaction
data, encrypts the cardholder PAN, and writes a ready-to-publish event
row into the `wdp.chbk_outbox_row` transactional outbox. It then
acknowledges each processed item back to MCM so it is not re-queued on
the next run.

This batch is the **origin of all MasterCard first chargeback data in
WDP**. If it fails or falls behind, no new MasterCard first chargeback
cases are created or updated downstream.

**What it explicitly does NOT do:**
- Does not publish to Kafka directly — writes to the outbox only;
  a separate outbox publisher reads PENDING rows and publishes
- Does not handle second/subsequent chargebacks, pre-arbitration,
  or arbitration stages — first chargeback only
- Does not process reversed chargebacks or reversal records
- Does not perform case routing, decisioning, or response filing
- Does not write to S3 or any file system
- Does not invoke any WDP case management API to create cases

---

**Boundaries**

Inputs:
- MCM unworked first chargeback queue — polled via DataPower
  Gateway (IBM DataPower acting as API facade to MasterCard)
- MCM claim detail endpoint — fetched per claim via DataPower
- MCM settled transaction endpoint — fetched per claim via
  DataPower for PAN resolution
- PostgreSQL `wdp.chbk_outbox_row` — read for idempotency
  lookups before every write

Outputs:
- PostgreSQL `wdp.chbk_outbox_row` — PENDING or SKIPPED rows
  written for each qualifying first chargeback
- MCM acknowledgement endpoint (PUT via DataPower) — marks
  processed chargebacks as worked so they are not re-queued
- WDP Encryption Service — POST per new claim to encrypt PAN
  before writing to outbox
- IDP Token Service — GET per encryption call to obtain a
  short-lived bearer token for the Encryption Service call

---

**MCM integration via DataPower Gateway**

All MasterCard calls are made through an internal DataPower proxy,
not directly to MasterCard endpoints. DataPower acts as the API
facade. There are four distinct MCM interactions per job run:

1. **Queue poll** — GET unworked first chargeback claim IDs.
   Returns a list of claimIds. If the list is empty, the job
   exits cleanly with zero records processed.

2. **Claim detail fetch** — GET full dispute case detail per
   claim, including all chargeback sub-records, transaction ID,
   PAN, merchant, and network fields.

3. **Settled transaction fetch** — GET the settled clearing
   record per claim to resolve the preferred mapped account
   number. Falls back to the claim-level PAN if the settlement
   record is absent or blank. This call is skipped entirely if
   the settlement transaction ID is not present in the claim.

4. **Batch acknowledgement** — PUT at the end of each chunk
   write, sending all processed chargeback IDs in a single
   request. Marks them as worked in MCM so they are not
   re-queued on the next poll.

**Authentication to MCM:** Static Vantiv/Worldpay license key
passed as a static `Authorization` header — not OAuth. A fresh
correlation ID UUID is generated per request for tracing.

**Authentication to Encryption Service:** Short-lived bearer
token fetched from IDP Token Service on every encryption call.
Token is fetched per-request — there is no in-memory token
caching in the current implementation.

---

**Processing pipeline (architectural flow)**
Schedule fires (every 5 minutes)
→ Fetch all unworked MCM claim IDs (single batch call)
→ Optional filter to specific case IDs (testing flag)
→ For each claim (chunk size = 1):
1. Fetch full claim detail from MCM
2. Filter to qualifying chargebacks:
chargebackType = CHARGEBACK, not reversed, not reversal
3. Idempotency check: does a row already exist for this claim?
[NEW CLAIM PATH]
For each qualifying chargeback:
— Resolve account number (settled transaction preferred,
claim PAN fallback)
— Truncate to 16 chars if necessary
— Masking check: if already masked, skip encryption
— Encrypt PAN → HPAN via Encryption Service
— Map to outbox entity, status = PENDING
— Encryption only on first chargeback per claim;
subsequent chargebacks in same claim skip encryption
[EXISTING CLAIM PATH]
For each qualifying chargeback:
— Lookup existing row by (claimId, chargebackId)
— If not found: status = PENDING
— If found: status = SKIPPED
— No PAN fetch or encryption on update path
4. Write outbox rows to chbk_outbox_row
5. Batch acknowledge all written chargebacks to MCM (PUT)

All processing is **single-threaded**. Spring Batch default
`SyncTaskExecutor` — no thread pool configured.

---

**What is written to the outbox**

Each row written to `wdp.chbk_outbox_row` represents one
qualifying chargeback within a claim. Key fields populated:

- Network identifiers: claimId → `c_ntwk_case_id`,
  chargebackId → `c_ntwk_phase_id`
- Dispute metadata: stage hardcoded `CH1` (First Chargeback),
  card network hardcoded `MASTERCARD`, platform hardcoded `CORE`
- Status: `PENDING` (new) or `SKIPPED` (duplicate)
- Payload: full `CommonEvent` JSON including HPAN (never clear
  PAN), dispute amount, reason code, merchant, scheme references
- `enrichmentFailure` flag is hardcoded `true` on every row —
  signals to downstream processors that transaction enrichment
  is intentionally incomplete at this stage

Fields NOT populated by this batch (set by downstream systems):
`kafka_partition`, `kafka_offset`, `kafka_topic`, `published_at`,
`error_code`, `error_message`, `file_job_id`, `next_retry_at`

---

**PAN handling and security**

- Clear PAN is sourced from the settled transaction preferred
  account number, falling back to the claim-level PAN
- Masking check: if the account number already contains `*`,
  encryption is skipped and the masked value is written as-is
- Encryption is performed only for the first chargeback per
  claim — subsequent chargebacks in the same claim reuse the
  encrypted value without re-calling the Encryption Service
- Encryption is never performed on the update (existing claim)
  path — account number is null on all SKIPPED/update rows
- Clear PAN is never persisted to the database or written to
  logs — excluded from all entity toString output
- If encryption fails, the exception propagates and the entity
  is not saved — the claim remains in the MCM queue for the
  next run

---

**Key architectural decisions**

- **Transactional outbox pattern** — batch writes PENDING rows
  to `wdp.chbk_outbox_row`; a separate Kafka publisher reads
  and publishes to Kafka. This decouples MasterCard ingestion
  from Kafka availability.

- **PAN encrypted at ingestion boundary** (DEC-004) — plain
  account number is never persisted. Encryption delegated to
  wdp-encryption-service; batch performs no crypto itself.

- **DataPower as MCM API gateway** — all MasterCard calls go
  through IBM DataPower using a static Vantiv license key, not
  direct MasterCard endpoint calls. Same pattern used by
  CaseFillingBatch.

- **Kubernetes Deployment, not CronJob** — JVM stays warm
  between cron fires, avoiding cold-start latency. Trade-off:
  replicas > 1 creates parallel queue polling risk with no
  distributed lock.

- **Chunk size 1 — deliberate design** — each claim is its own
  JPA transaction. A failure on one claim does not roll back
  previously written claims. This enables partial-run progress
  at the cost of throughput.

- **`enrichmentFailure` hardcoded true** — every row is
  intentionally marked as needing further downstream enrichment.
  `OriginalTransDetail` is always an empty object. This is a
  deliberate architectural signal to downstream processors,
  not a bug.

- **Sequential, in-memory queue processing** — all claim IDs
  are fetched in a single pre-step call and held in memory.
  Processing then iterates over this in-memory list. No
  streaming or paginated read during processing.

- **Static license key auth for MCM** — no OAuth token
  lifecycle to manage for the MCM integration. Token is a
  long-lived Vantiv proprietary license credential. Trade-off:
  no automatic token rotation.

- **Per-request IDP token fetch for Encryption Service** —
  no token caching. Adds one HTTP call per new claim processed.
  Accepted as low risk given the 5-minute poll cycle and
  typically small MCM queue delta per run.

---

**Retry strategy**

All external calls retry 3 times with a fixed 2-second delay.
No exponential backoff. Applies to all MCM calls (GET and PUT),
Encryption Service calls, and IDP Token Service calls.

---

**Error handling and failure modes**

- **Single claim failure** — does not stop the batch. The
  failed claim is silently skipped. It is not acknowledged to
  MCM and remains in the queue for the next run.

- **Queue poll failure** — silently aborts the run with zero
  records processed. No alert or DLQ entry is created.

- **Acknowledgement failure** — logged and swallowed.
  Already-written outbox rows are NOT rolled back. The claim
  will be re-queued by MCM and re-processed on the next run,
  triggering the idempotency path.

- **No dead letter mechanism** — there is no DLQ, error table,
  or separate error-capture mechanism for failed claims.
  `error_code` and `error_message` columns exist in the outbox
  entity schema but are never populated by this batch.

- **Crash between write and acknowledge** — a pod crash after
  an outbox row is written but before the MCM acknowledgement
  PUT succeeds leaves the claim in the MCM queue. On the next
  run the idempotency check routes the claim to the update path
  and writes a SKIPPED row. The downstream publisher must
  deduplicate on case and chargeback identity.

---

**Idempotency**

Before writing, the processor checks whether the outbox already
contains a row for the given claim. If any row exists for the
claim, all chargebacks within it go through the update path:
each chargeback is individually checked for an existing row,
and any that already exist are written as SKIPPED.

⚠️ **Important limitation:** SKIPPED rows are still written to
the database — the writer does not gate on the skip flag. A
double-trigger or restart will produce a second row with status
SKIPPED for any already-processed chargeback. The downstream
outbox publisher must deduplicate or filter on PENDING status
only.

⚠️ **Stale PENDING rows:** There is no mechanism to retry or
re-evaluate PENDING rows from a prior failed run. The update
path only handles new chargebacks on an existing case.

---

**Schedule and deployment**

- **Trigger:** Spring `@Scheduled` cron — every 5 minutes.
  Expression is injected from Kubernetes secret at deploy time
  and not committed to source.
- **Kubernetes resource type:** Deployment (not CronJob). Pod
  runs continuously; scheduling is handled internally by the
  Spring scheduler.
- **Replica count:** 1 (confirmed production value).
- **Memory:** 204Mi limit / 256Mi request. No CPU limits.
- **HPA:** Not configured — no autoscaling.
- **Rolling update:** maxSurge 1 / maxUnavailable 0 —
  zero-downtime rolling update.
- **Observability:** OpenTelemetry auto-instrumentation via
  pod annotation, Spring Boot Actuator on port 8092, structured
  JSON logging via Logstash encoder.
- **Spring Batch metadata:** Job and step execution state
  persisted to prefixed metadata tables in PostgreSQL. Schema
  must pre-exist — DDL initialisation is disabled.

---

**Risks and constraints**

🔴 HIGH — No dead letter or error-capture mechanism for failed
claims. Claims that consistently fail processing remain silently
in the MCM queue indefinitely. There is no alerting, no error
table, and no visibility into recurring failures.

🟡 MEDIUM — Crash-between-write-and-acknowledge window produces
orphaned outbox rows that will be re-processed and written as
SKIPPED on the next run. Downstream publisher must filter on
PENDING status and deduplicate on network identifiers.

🟡 MEDIUM — Per-request IDP token fetch for every new claim
encryption adds one network call per claim with no caching.
Under abnormal high claim volume this amplifies IDP Token
Service load linearly with claim count. Manageable at normal
5-minute cadence but degrades if the batch falls behind and
processes a large backlog.

🟡 MEDIUM — No rate limiting on MCM API calls. No throttle,
no sleep between claim detail fetches. If MasterCard enforces
API quota limits, high-volume runs risk hitting rate limits
with no back-pressure mechanism.

🟡 MEDIUM — All claim IDs are loaded into memory in a single
pre-step call before processing begins. With a 5-minute poll
cycle the queue delta per run is bounded by what MasterCard
accumulates in that window — likely small under normal
conditions. Risk elevates if the batch falls behind and the
queue builds a large backlog, which would be loaded into
memory in a single call against the 204Mi memory limit.

🟡 MEDIUM — No PodDisruptionBudget. Node drain during a run
terminates the pod mid-batch. Claims not yet acknowledged
return to the MCM queue; the crash-between-write-and-ack risk
applies to any in-flight claim at time of eviction.

🟡 MEDIUM — No CPU limits configured. Pod runs Burstable QoS —
first candidate for eviction under node memory pressure.

🟢 LOW (currently controlled) — Replicas > 1 would create
parallel MCM queue polling with no distributed lock. Multiple
pods would poll the same claim IDs on the same cron tick and
attempt concurrent writes with no poll-time guard. Production
replica count is confirmed as 1, which eliminates this risk
today. Risk becomes 🔴 HIGH immediately if replicas are ever
increased — there is no distributed coordination mechanism
in place and none planned. Any scaling change must explicitly
account for this.

🟢 LOW — No circuit breaker on any external call. A hung
DataPower, IDP, or Encryption Service response blocks the
processing thread for that claim until retry exhaustion
(3 attempts × 2s delay = up to ~6 seconds per hung call).

🟢 LOW — Cron schedule (every 5 minutes) is not auditable from
source repository — fully externalised via Kubernetes secret.
Schedule changes require a secret update, not a config or code
change. Current schedule is confirmed and documented here.

🟢 LOW — `minReadySeconds` is currently placed at an invalid
YAML path in `resources.yaml` (`spec.template.spec` instead of
`spec`). The field has no effect in its current position. Likely
a YAML placement bug requiring correction.

---

**Deferred and inactive features**

The following are implemented or declared but not active in
the current code path:

- **Dead Kafka infrastructure (deferred):** `spring-kafka`,
  `kafka-clients`, and `aws-msk-iam-auth` are declared in
  dependencies but no Kafka producer, consumer, KafkaTemplate,
  or Kafka configuration class is wired. The `kafka_partition`,
  `kafka_offset`, and `kafka_topic` columns in the outbox
  confirm the outbox pattern is in place with Kafka publishing
  delegated to a separate service. Kafka publishing in this
  batch is either deferred or permanently removed.

- **`oauth2-client` declared but unused:** OAuth2 client
  dependency is declared but no OAuth2 configuration is
  present. Authentication is handled via static Vantiv license
  key (MCM) and manual IDP token fetch (Encryption Service).
  Possibly a preparation for future OAuth2 migration.

- **`enrichmentFailure = true` hardcoded:** Every row is
  marked as requiring further enrichment. `OriginalTransDetail`
  is always an empty object. Several `CommonEvent` fields
  (`networkRuling`, `rejectDate`, `rejectReason`,
  `historicalDsptDetail`) are never populated. This is
  intentional — this batch produces a partial event that
  downstream processors must complete.

- **`readSpecificItemFromQueue` flag (active — testing tool):**
  When enabled, filters all MCM queue polling to a configured
  list of specific case IDs. Intended for targeted testing or
  re-processing without a code change. Undocumented in
  production runbooks — operations teams must be aware it
  exists and that it silently suppresses all other claims
  when active.

- **`removeItemFromQueueDisabled` flag (active — testing tool):**
  When enabled, the MCM acknowledgement PUT is never called.
  Claims are processed and written to the outbox but remain
  in the MCM queue. Intended for testing without affecting
  the MCM queue state.

- **Dead model classes:** `CaseEvent`, `WebResponse`,
  `SearchCaseRequest`, `MerchantDetails`, and
  `HistoricalDisputeDetail` exist in source but are never
  instantiated or called in active code paths. Likely remnants
  of earlier payload structures or planned features.

---

**Planned changes**

- Resolve Kafka dependency question: either wire Kafka
  publishing into this batch or formally remove the declared
  dependencies. Current state is ambiguous.
- Implement IDP token caching to avoid per-request fetches
  under higher claim volume
- Populate `error_code` and `error_message` on failed claims
  to provide visibility into recurring failures
- Evaluate dead letter or alerting mechanism for silently
  skipped claims
- Formally document or remove `readSpecificItemFromQueue` —
  currently an undocumented surgical tool with no ADR
- Investigate and resolve `minReadySeconds` YAML placement
  bug in `resources.yaml`
- Remove or activate `oauth2-client` dependency — current
  state adds unused runtime overhead
---

#### 2.2.3 CaseFillingBatch
**Owner:** Integration Team
**Status:** ✅ Production | ✅ COMPLETE
**Repository:** wp-mfd/wdp-mcm-receiver-case-filing-queue-batch
**Maven artifact:** wdp-mcm-prearb-queue-batch
**Technology:** Spring Boot 3.5.3 / Spring Batch / Java 17

---

**What it does**

CaseFillingBatch is the MasterCard inbound batch responsible for
ingesting Pre-Arbitration and Arbitration dispute events into WDP.
It polls the MasterCard Connect Merchant (MCM) ReceiverCaseFiling
queue on a cron schedule, fetches full claim and settlement detail
for each qualifying event, optionally encrypts the cardholder PAN,
constructs a `CommonEvent` payload, and writes a structured event row
to the `wdp.chbk_outbox_row` transactional outbox for downstream
Kafka publishing.

This batch handles dispute lifecycle stages that arise **after** the
merchant has already responded to or ignored a first chargeback. It
is the origin of all MasterCard Pre-Arb and Arb dispute data in WDP.
If this batch fails or falls behind, no new Pre-Arb or Arb cases are
created or updated downstream.

**What it explicitly does NOT do:**
- Does not process First Chargeback events — those are handled by a
  separate sibling batch in its own repository
- Does not process Compliance events (caseType 3 or 4) — explicitly
  skipped
- Does not publish to Kafka directly — writes to outbox only; a
  separate outbox publisher reads PENDING rows and publishes
- Does not call any case-lookup or case-creation APIs — the
  `caseLookupUrl` is configured in all environments but is never
  called at runtime (see Deferred Features)
- Does not write to S3, emit metrics to any external system, or
  produce any file output

---

**Dispute Lifecycle Stages Handled**

This batch exclusively handles five MasterCard dispute stages that
arise after the initial first chargeback:

| Stage Code | Description                                      |
|------------|--------------------------------------------------|
| `PAB`      | Incoming Pre-Arbitration filed by issuer         |
| `ARB`      | Incoming Arbitration or Escalation to Arb        |
| `PRA`      | Withdrawal (of either Pre-Arb or Arb)            |
| `AII`      | Arbitration ruling: issuer favor (FAVOR_SENDER)  |
| `AIM`      | Arbitration ruling: merchant favor (FAVOR_RECEIVER) |

Stage is determined by the combination of `caseType` (1 = PreArb,
2 = Arb), `caseFilingStatus`, `rulingDate`, `rulingStatus`, and
`issuerAction` fields in the MCM claim detail response.

---

**Processing Pipeline**

For each claim read from the MCM ReceiverCaseFiling queue, the
processor executes these steps in sequence:

1. **Claim detail fetch** — calls MCM Claim Detail API to retrieve
   full case filing information for the claim

2. **Claim validation** — skips if `caseFilingInfo` or
   `caseFilingDetails` is null/empty; skips if `caseType` is not
   "1" (PreArb) or "2" (Arb)

3. **Stage and issuer response determination** — resolves dispute
   stage and `issuerResponseType` from the claim detail using a
   priority-ordered branch:
   - Withdrawn status → `IncomingPreArb`, stage PAB
   - Ruling date present + ruling status mapped → `ArbRuling`,
     stage AII or AIM
   - `rebut` issuer action → looks back at previous history entry
     to resolve as `createcase` or `escalate`
   - `createcase` → stage PAB (caseType 1) or ARB (caseType 2)
   - `withdraw` → stage PRA
   - `escalate` → stage ARB

4. **Outbox lookup** — checks `chbk_outbox_row` for an existing row
   by `networkCaseId`. Determines whether this is a new claim or an
   update to an existing claim.

5. **New claim path** — fetches settled transaction detail from MCM
   to obtain the MasterCard Mapping Service account number, extracts
   last 4 digits, encrypts clear PAN via WDP Encryption Service if
   PAN is not already masked, then maps and serializes the
   `CommonEvent` payload

6. **Update claim path** — performs a finer lookup by
   `(networkCaseId, networkPhaseId)`. Skips if a non-SKIPPED row
   already exists for that exact phase. No PAN lookup or encryption
   is performed for updates.

7. **Write to outbox** — saves each non-skipped item to
   `wdp.chbk_outbox_row` with status `PENDING` or `SKIPPED`

---

**Boundaries**

Inputs — MasterCard MCM API (all via DataPowerRestInvoker,
auth via raw Vantive license key):

- **ReceiverCaseFiling Queue** (POST) — polls the MCM queue for all
  open Pre-Arb/Arb claims within the configured date window.
  Paginated — all pages are loaded into memory before processing
  begins (list-based reader, not streaming). Two pagination modes:
  page-count-based (default) and sentinel-value-based
  (`disablePageCount` flag).

- **Claim Detail** (GET) — fetches full case filing detail per
  claim, including `caseType`, `issuerAction`, `rulingStatus`,
  `rulingDate`, `caseFilingRespHistory`, `standardClaims`,
  `primaryAccountNum`, and `transactionId`.

- **Settled Transaction Detail** (GET) — fetches the MasterCard
  Mapping Service account number for PAN enrichment. Called for new
  claims only, and only when `networkSettledTransactionId` is
  non-blank. Falls back to `primaryAccountNum` on error.

Inputs — Internal WDP Services (Bearer token authenticated):

- **wdp-idp-token-service** — IDP token obtained per encryption
  call. No token caching — a fresh token is fetched on every
  invocation. Not called for update claims (no encryption).

- **wdp-encryption-service** — encrypts clear PAN to HPAN before
  persistence. Called for new claims only where PAN does not already
  contain masked characters. Clear PAN is never written to any
  persistent store.

Queue filtering:
- Items where `lastModifiedBy` equals the batch's own user ID
  (`P106040`) are excluded to prevent self-processing loops
- Items outside the configured `[fromDate, toDate]` window are
  excluded
- `readSpecificItemFromQueue` flag (when enabled) limits processing
  to only a pre-configured list of specific claim IDs — active
  debug/support tool

Outputs:

- **wdp.chbk_outbox_row** (PostgreSQL, schema `wdp`) — dispute
  events written with status `PENDING` (ready for downstream Kafka
  publisher) or `SKIPPED` (duplicate already processed). Key fields
  written: `c_ntwk_case_id` (networkCaseId), `c_ntwk_phase_id`
  (networkPhaseId), `c_case_stage`, `c_case_ntwk` (always
  MASTERCARD), `c_acq_platform` (always CORE), `payload`
  (serialized CommonEvent JSON). Kafka topic, partition, and offset
  columns are populated by the downstream outbox publisher, not by
  this batch.

The outbox table is the handoff point to the downstream Kafka
publisher. This batch writes PENDING rows; it does not publish to
Kafka and does not interact with MSK directly.

---

**Trigger and schedule**

Deployed as a Kubernetes **Deployment** (not a CronJob). A
long-running pod runs continuously, with an embedded Spring
`@Scheduled` cron triggering job execution at the configured
interval.

| Parameter       | Value                                               |
|-----------------|-----------------------------------------------------|
| Cron schedule   | Externalized via K8s secret `scheduler_cron`        |
| Kubernetes type | Deployment (not CronJob)                            |
| Job auto-start  | Disabled — driven by internal cron only             |
| Chunk size      | 1 (hardcoded) — every item its own transaction      |
| Run uniqueness  | Timestamp parameter appended to JobParameters       |

The cron expression is not hardcoded in any YAML file — it is
exclusively injected as an environment variable at runtime. Whether
schedules overlap with FirstChargebackBatch in production is not
determinable from this repository.

---

**Scaling and deployment**

| Parameter     | Value                                                 |
|---------------|-------------------------------------------------------|
| Replica count | XL Deploy placeholder — externally injected           |
| HPA           | None — scaling is entirely manual                     |
| Memory limit  | 2048 Mi                                               |
| Memory request| 256 Mi                                                |
| CPU           | No limits or requests defined                         |
| Update type   | RollingUpdate — maxSurge 1, maxUnavailable 0          |
| Observability | OTel Java agent injected; Spring Actuator on port 8082|
| TLS           | Corporate CA cert chain mounted into JRE truststore   |

---

**Error handling**

- **Item-level isolation** — chunk size = 1 means every item is its
  own Spring Batch transaction. One item failure does not stop the
  batch or roll back other items.

- **Retry** — all MCM REST calls (queue poll, claim detail, settled
  transaction) retry 3 times with a fixed 2-second delay on any
  exception. No exponential backoff. IDP token fetch and database
  operations have no retry.

- **Processor failure** — exceptions in the processor are caught at
  the item level and return an empty (skipped) result. The batch
  continues with the next item.

- **Writer failure** — exceptions in the writer are caught and
  logged per item without re-throwing. Failed writes are silently
  skipped.

- **No dead letter mechanism** — there is no DLQ, no error table,
  and no persistent record of which claims failed processing beyond
  application logs.

- **Job-level failure** — if the job itself fails, the next
  scheduled cron execution starts a brand-new job instance (new
  timestamp parameter). There is no Spring Batch checkpoint restart
  — all claims are re-fetched from MCM on each run.

---

**Idempotency**

The batch implements a two-stage soft idempotency check:

1. **By networkCaseId** — checks if any outbox row exists for the
   claim (phase-agnostic). Non-empty result triggers the update
   path rather than the new claim path.

2. **By (networkCaseId + networkPhaseId)** — on the update path,
   checks if a non-SKIPPED row already exists for the exact claim
   and phase combination. If found, the new item is written as
   SKIPPED rather than PENDING, preventing the downstream Kafka
   publisher from processing it twice.

Idempotency gaps:
- Concurrent runs (replicas > 1) — no distributed lock and no
  optimistic locking on insert. Two pods could both read the same
  claim from MCM, both find no outbox row, and both insert a PENDING
  row. Duplicate PENDING rows could reach the Kafka publisher.
- Restart from partial run — on restart, all claims are re-fetched
  from MCM. Claims already in PENDING status from a prior partial
  run are found in the outbox lookup and marked SKIPPED (unless the
  networkPhaseId doesn't match, in which case a new PENDING is
  created).

---

**Key architectural decisions**

- **Pull model for MasterCard** — WDP polls the MCM
  ReceiverCaseFiling queue rather than MCM pushing events. Polling
  window and frequency are externalized. No rate limiting is
  implemented at the batch level.

- **Transactional outbox pattern** — batch writes PENDING rows to
  `wdp.chbk_outbox_row`; a separate outbox publisher reads and
  publishes to Kafka. This decouples Pre-Arb/Arb ingestion from
  Kafka availability.

- **PAN encrypted at ingestion boundary** (DEC-004) — clear
  account number is never persisted. Encryption is delegated to
  wdp-encryption-service via IDP-token-authenticated call. Clear
  PAN exists only transiently in memory during processing.

- **Separate batch from FirstChargebackBatch** — Pre-Arb/Arb events
  are structurally distinct from first chargebacks in MCM. They have
  different claim types, response histories, ruling logic, and PAN
  sourcing paths. Separation keeps each batch's processing logic
  cohesive and independently deployable.

- **Kubernetes Deployment not CronJob** — JVM stays warm between
  cron fires, avoiding cold-start latency on each polling cycle.
  Trade-off: replicas > 1 creates parallel polling risk with no
  distributed lock.

- **No case-lookup or case-creation calls** — unlike the Visa batch,
  this batch does not call a case search service. The downstream
  CaseCreationConsumer handles case create vs update logic after
  reading from Kafka. The `caseLookupUrl` configuration present in
  all environments is a dead config — never called at runtime.

---

**Risks and constraints**

🔴 HIGH — Replicas > 1 creates parallel MCM queue polling with no
distributed lock. Multiple pods would each fire the cron
independently, poll the same ReceiverCaseFiling queue simultaneously,
and attempt to write the same events. The `(networkCaseId,
networkPhaseId)` duplicate check is the only guard — but it only
fires at write time, not at poll time. Duplicate PENDING rows before
the check can run are not protected against. Replica count should be
1 until a distributed coordination mechanism is in place.

🔴 HIGH — No HPA. Scaling is entirely manual via XL Deploy
placeholder. No autoscaling under queue backlog growth.

🟡 MEDIUM — All MCM queue pages are loaded into memory before
processing begins (list-based reader, not streaming). Very large
queue responses could cause significant heap pressure given the
2GiB memory limit and 256Mi request.

🟡 MEDIUM — No dead letter queue or error table. Failed items are
silently logged and skipped. There is no durable record of which
claims failed processing and no automated alerting or reprocessing
mechanism.

🟡 MEDIUM — No rate limiting on MCM API calls. No throttle, sleep,
or circuit breaker between requests. Quota risk if MCM enforces
server-side rate limits.

🟡 MEDIUM — No PodDisruptionBudget. Node drain during job execution
terminates the pod mid-batch. Claims not yet processed will be
re-fetched on the next run; idempotency guards prevent most
duplicates, but the concurrent-insert gap (see Idempotency) remains.

🟡 MEDIUM — No CPU limits or requests. Pod runs at Burstable QoS —
first candidate for eviction under node memory pressure.

🟡 MEDIUM — Cron expression is not auditable from source. Fully
externalized via K8s secret. Schedule changes require a secret
update, not a config or code change.

🟢 LOW — No circuit breaker on internal service calls. A hung
response from wdp-idp-token-service or wdp-encryption-service
blocks that item's processing thread until retry exhaustion
(3 attempts × 2s = ~6s per hung call per item).

🟢 LOW — `caseLookupUrl` is configured in all YAML profiles and
K8s environment variables but is never called. Dead configuration
that could mislead operators or cause confusion during incident
investigation.

---

**Deferred and inactive features**

- **Case Lookup URL (dead config):** `app.disputeService.caseLookupUrl`
  is configured in every environment profile pointing to
  `mdvs-gcp-case-search-service`. No code in this repository ever
  calls this URL. It is either a holdover from a planned enrichment
  feature or was removed from processing logic but not from config.
  This dead config should be formally removed.

- **readSpecificItemFromQueue (active — surgical testing tool):**
  When the `read_specific_item_from_queue` environment flag is
  `true`, the reader filters all queue polling to only claim IDs
  listed in `mcm_case_numbers`. Intended for targeted re-processing
  or manual testing without a code change. This is active production
  configuration — operations teams must be aware it exists and that
  enabling it silently suppresses all other claims.

- **disablePageCount pagination mode:** An alternate pagination
  strategy that switches from page-count-based looping to looping
  until the MCM sentinel error message ("No records found in Queue")
  is returned. Controlled by environment flag `disable_page_count`.

- **Three commented-out CommonEvent fields:** `documentRequired`,
  `networkDocInd`, and `partialIndicator` fields are defined on the
  `CommonEvent` model but are never populated. Code to set them was
  copied from a First Chargeback batch and references methods that
  do not exist on `ClaimDetail`. These fields will always be null in
  the outbox payload until MCM data mapping is completed.

---

**Planned changes**

- Formally remove or activate `caseLookupUrl` configuration — dead
  config in all environments creates operational confusion
- Clean up or document the three commented-out CommonEvent fields —
  either complete the MCM data mapping or remove the dead fields
- Evaluate distributed locking or replica count enforcement to make
  replicas > 1 safe for production
- Formally document or operationally gate `readSpecificItemFromQueue`
  — currently an undocumented surgical tool with no ADR and no audit
  trail when enabled
- Add a durable error record or dead letter mechanism so that failed
  claims are not silently lost
---

### 2.3 File-Based Inbound Path

---

#### 2.3.1 DM Mainframe
**Owner:** WDP Team (on-premise)
**Status:** ✅ Production | 📝 DRAFT

---

**What it does**
On-premise mainframe system owned by
the WDP team. Provides mainframe-to-
mainframe connectivity with external
card networks and merchants. Acts as
the file exchange gateway for all
mainframe-connected sources — both
receiving inbound files and delivering
outbound files. Exchanges files with
Sterling Mailbox in both directions.

**Note:** Sterling Mailbox and ControlM
are enterprise shared services — not
owned by WDP team.

**Boundaries**

Inputs (inbound):
- MI Image Extractor
- IssuerDocumentFile (MAP incoming)
- MasterCard Retrieval/Rejects
- Discover / DiscoverHybrid
- Amex / AmexHybrid
- PIN Networks (NYCE)
- CapitalOne (BJs)
- Walmart (PIN/SIGNATURE)

Outputs (outbound):
- Walmart
- PIN Networks (NYCE)
- CapitalOne (BJs)
- Discover

All via mainframe-to-mainframe protocol
through Sterling Mailbox.

**Key architectural decisions**
- [DEC-PLACEHOLDER: On-premise mainframe
  retained for file-only network
  connectivity]
- [DEC-PLACEHOLDER: Sterling and ControlM
  as enterprise shared services —
  not WDP owned]

**Risks & Constraints**
- On-premise dependency — not in AWS
- Availability and SLA dependent on
  enterprise shared services
  (Sterling, ControlM)
⚠️ DRAFT — availability, SLA, monitoring
to be confirmed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 2.3.2 FileProcessor
**Owner:** Integration Team
**Status:** ✅ Production | ✅ COMPLETE
**Technology:** Spring Boot 4.0.3 / Java 17
**Repository:** wdp-file-processor

---

**What it does**

FileProcessor is the primary inbound file-ingestion component for the
WDP chargeback system. It is the sole component responsible for
converting raw source ZIP files — received from card networks, acquiring
platforms, and Walmart — into structured outbox records that downstream
consumers can act on.

When an inbound ZIP file lands in S3, an SQS event notification
triggers FileProcessor. It identifies the source from the filename,
streams the file, parses every record according to source-specific
format rules, writes dispute events and evidence metadata to three
PostgreSQL outbox tables, stages evidence documents to S3, and
encrypts any PAN data before anything is written to persistent storage.

**What it explicitly does NOT do:**
- Does not execute dispute workflow logic — no state machine, no rules
  engine, no case lookup
- Does not publish to Kafka — the outbox table is the sole output;
  InboundDisputeEventScheduler reads PENDING rows and publishes
- Does not perform case enrichment or create cases directly
- Does not generate ACK file content — it only sets ACK-required flags
  on file_job; a separate processor generates ACK files
- Does not produce outbound network response files
- Does not use Spring Batch — all record iteration is custom per-source
  processing logic

---

**File sources handled**

FileProcessor handles six source acronyms, each with a distinct file
format and dispute lifecycle role:

| Source | Full Name | Format | Role |
|--------|-----------|--------|------|
| DWSG | Walmart Signature Capture | Fixed-width flat text + TIFF images | Evidence only |
| DBLK | Core Bulk Response | Fixed-width flat text + document images | Evidence only |
| DISR | Discover MAP Incoming (Issuer Doc) | Fixed-width flat text + images | Evidence only |
| MFAD | Discover MAP Notice Processor | Quote-delimited flat text + images | Evidence only |
| DCPO | Capital One Disputes Incoming | XML (JAXB/XSD) + document images | Chargeback + Evidence |
| DNWK | Network File Incoming — COBOL copybook | Fixed-width 5700-byte COBOL flat file | Chargeback only |

DNWK is further sub-routed by filename substring into seven network
sub-sources:

| Sub-Source | Network |
|-----------|---------|
| AMEX | AMEX Network Incoming |
| AMEXHYB | AMEX Hybrid Network Incoming |
| AMEXOPTB | AMEX Opt-Blue Network Incoming ⚠️ Not yet implemented — silent no-op |
| DISC | Discover Network Incoming |
| MC_REVREJ | Mastercard Reversal/Rejection ⚠️ Not yet implemented — silent no-op |
| NYCE | NYCE Network Incoming |
| DISCHYB | Discover Hybrid Network Incoming ⚠️ Not yet implemented — silent no-op |

Source identification is entirely filename-based — no metadata in the
SQS message body identifies the source or file type.

---

**Boundaries**

Inputs:
- AWS SQS queue (`wdp-file-arrivals`) — S3 ObjectCreated event
  notifications. One listener, one queue, serving all six sources.
  Strictly single-threaded: maxConcurrentMessages = 1,
  maxMessagesPerPoll = 1. One file processed at a time.
- S3 inbound bucket (`wdp-files`) — ZIP files under path convention
  `<ACRO>/INBOUND/<filename>.zip`. Files are streamed directly from
  S3 — not loaded fully into memory.

Outputs (database):
- `wdp.file_job` — one row per file. Tracks file-level processing
  status from PENDING through to completion or ERROR. Records source,
  ACK requirement, row counts, and error detail.
- `wdp.chbk_outbox_row` — one row per dispute event or evidence
  record. This is the primary handoff to downstream consumers.
  Consumed by InboundDisputeEventScheduler for Kafka publishing.
- `wdp.file_evidence` — one row per evidence document extracted.
  Holds the S3 staging path, case linkage, and attachment status.

Outputs (S3):
- `<ACRO>/STAGING/<YYYY-MM-DD>/<zipName>/<imageName>` — extracted
  evidence document images staged for downstream retrieval.
- `<ACRO>/ARCHIVED/<MMYYYY>/<filename>.zip` — inbound ZIP moved here
  after processing completes or on error.
- `wdp-evidence-failed-files-prod` bucket — ERROR-status evidence
  images moved here for failed records.
- `<ACRO>/PINDISPUTE/<date>/<pinNetwork>/<imageName>` — DWSG PIN
  indicator documents routed separately.

Outputs (external service calls):
- `wdp-encryption-service` — POST per record, DNWK sources only.
  Encrypts full PAN (account number from COBOL record) to HPAN before
  any database write. 3 attempts, 2-second fixed backoff.
- `wdp-idp-token-service` — GET per encryption call. Retrieves a
  short-lived Bearer token used to authenticate the encryption call.
  No token caching — fetched fresh per call.

No other external service calls are made. No Kafka producer exists.
No downstream REST API is called as output.

---

**Processing flow (architectural)**
SQS message received (S3 ObjectCreated event JSON)
|
├─ Visibility timeout extended on SQS message
├─ Idempotency check: file_job exists with non-PENDING status?
│       └─ YES → delete SQS message, stop (duplicate delivery)
|
├─ Insert file_job row (status = PENDING)
├─ Stream ZIP from S3
├─ Route by source acronym (filename-based)
|
├─ DWSG / DBLK / DISR / MFAD → processIndexZipFile
├─ DCPO → processXmlZipFile
└─ DNWK → processZipContainingFlatFile
|
└─ Sub-route by filename substring to NetworkFileService
|
Per-record loop (resume-aware — skips rows already written):
├─ Parse record per source format
├─ Map to outbox entity (RowDetail / CommonEvent)
├─ DNWK only: encrypt PAN via wdp-encryption-service
│           clear PAN never written anywhere
├─ Write chbk_outbox_row (status = LOADING)
└─ Write file_evidence (if evidence record)
|
Bulk status promotion: LOADING → PENDING (all rows for file)
|
Archive ZIP: INBOUND/ → ARCHIVED/<MMYYYY>/
Update file_job to final status

---

**Source-specific outbox behaviour**

The outbox record type and chargeback/evidence split differ by source:

- **DWSG, DBLK, DISR, MFAD** — every record is `EVIDENCE_ATTACH`.
  No chargeback records. These files contain only document images with
  an index file. Evidence rows carry S3 staging path and case linkage.

- **DCPO (Capital One XML)** — each XML message element produces two
  outbox record types: one `CHARGEBACK_PROCESS` row (the chargeback
  event) and N `EVIDENCE_ATTACH` rows (one per image in the message).
  Evidence rows are written with status `BLOCKED` — the downstream
  consumer must not attempt to attach evidence until the parent
  chargeback case is created. Which component unblocks evidence rows
  is outside this service's boundary (handled by a separate consumer).

- **DNWK (Network Files)** — every record is `CHARGEBACK_PROCESS`.
  No evidence rows are created. `isCreateCase` is always false —
  these files never trigger new case creation directly.

---

**PAN handling and encryption**

Only DNWK (Network File) sources contain a full account number (PAN),
extracted from a fixed position in the COBOL record layout. All other
sources carry at most the last four digits of a card number —
no encryption is performed for those sources.

For DNWK records, PAN is encrypted in-flight during field mapping,
before any database write occurs. The encrypted HPAN is stored in
the `payload` JSON column of `chbk_outbox_row`. Clear PAN is never
written to any table, log, or S3 path. If the encryption call fails
after all retries, the record is written to the outbox with ERROR
status — it is not silently dropped.

---

**Idempotency**

File-level idempotency: before processing begins, FileProcessor checks
`file_job` for an existing record with the same filename and S3 key
where status is not PENDING. If found, the SQS message is deleted
immediately and the file is not reprocessed. This protects against
duplicate SQS delivery after successful processing.

Record-level resume: if a pod restarts mid-file, processing resumes
from the last successfully written row rather than restarting from row
zero. This makes processing idempotent at the record level within a
file, provided `file_job` still has PENDING status.

Idempotency gap: if two pods receive the same SQS message
simultaneously (both pass the idempotency check before either writes
`file_job`), duplicate processing can occur. The single-threaded SQS
listener configuration (maxConcurrentMessages = 1) mitigates this in
single-replica deployments but provides no guarantee with multiple
replicas. There is no database unique constraint as a backstop.

---

**Error handling and failure modes**

Record-level isolation: a single record failure does not stop file
processing. Failures are caught per record, the record is written to
the outbox with ERROR status, and processing continues to the next
record. The outer file-level transaction is not rolled back.

File-level failure: if the top-level processing method throws, the
ZIP is archived and `file_job` is set to ERROR. The SQS message is
NOT deleted, so it will reappear after the visibility timeout. The
idempotency check on reappearance detects the non-PENDING `file_job`
and deletes the message — preventing infinite reprocessing.

Retry strategy: PAN encryption REST calls retry 3 times with a
2-second fixed backoff. No retry is applied to S3 reads, S3 writes,
or database operations. No retry on IDP token retrieval.

No dead letter mechanism: there is no DLQ, no error table, and no
S3 error path for file-level failures. Error visibility is through
`file_job.status = ERROR`, `chbk_outbox_row.status = ERROR`,
and `file_evidence.failed_s3_key` for image files that fail staging.

Partial file restart: all AcroService implementations query the last
successfully inserted row on startup. If found, processing skips ahead
to resume from that point. This requires `file_job` to still be in
PENDING status for the resume to be honoured.

---

**ACK file generation — boundary clarification**

FileProcessor sets `ack_required = true` and `ack_status = PENDING`
on `file_job` for sources that require acknowledgement. It does NOT
generate the ACK file content itself. ACK file generation is handled
by a separate component (FileAcknowledgementProcessor). Sources that
require ACK: DWSG (Walmart), DBLK (Core/Walmart).

---

**Scaling and deployment**

Deployed as a Kubernetes Deployment (not CronJob). One SQS listener,
one queue, strictly sequential. With a single replica, one file is
processed at a time. With multiple replicas, each pod processes one
file simultaneously from the same SQS queue — SQS visibility timeout
prevents double-processing of the same message, but the idempotency
gap described above applies.

| Parameter | Value |
|----------|-------|
| Kubernetes type | Deployment |
| Replica count | XL Deploy placeholder — exact prod count not in source |
| Memory limit | 2048Mi |
| Memory request | 1024Mi |
| CPU | Not configured |
| Max concurrent SQS messages | 1 |
| HPA | Not configured |
| PodDisruptionBudget | Not configured |
| Rolling update | maxSurge 1, maxUnavailable 0, minReadySeconds 50 |
| Observability | OTel Java agent injected; Logstash TCP JSON log shipping |
| Feature flags | None — routing is entirely hardcoded by source acronym |

---

**Key architectural decisions**

- **SQS as file processing trigger** — S3 ObjectCreated events routed
  via SQS rather than direct S3 event Lambda or polling. Decouples
  file arrival from processing capacity. SQS visibility timeout
  provides natural re-delivery on pod failure.

- **Strictly sequential single-file processing** — SQS listener
  configured maxConcurrentMessages = 1. One file fully processed
  before the next is dequeued. Simplifies state management and avoids
  concurrent write contention on shared outbox tables. Trade-off:
  throughput is bounded by single-file processing time per pod.

- **Transactional outbox pattern** (DEC-001) — FileProcessor writes
  to `chbk_outbox_row` with PENDING status; InboundDisputeEventScheduler
  reads and publishes to Kafka. File processing is fully decoupled
  from Kafka availability.

- **Three-table outbox model** — `file_job` (file-level ledger),
  `chbk_outbox_row` (dispute events and evidence handoff),
  `file_evidence` (document metadata and S3 paths) separate concerns
  cleanly. file_job enables file-level idempotency and ACK tracking.
  file_evidence enables evidence-to-case linkage independent of the
  chargeback outbox.

- **PAN encrypted at ingestion boundary** (DEC-004) — clear PAN
  is never persisted. Encryption is delegated to wdp-encryption-service
  via IDP-token-authenticated REST call. No crypto performed
  in-process. Encryption failure results in ERROR record — not silent
  data loss.

- **S3 streaming — not full in-memory load** — ZIP files are streamed
  from S3 rather than buffered into memory. Supports large files
  without heap pressure proportional to file size.

- **No Kafka producer** — FileProcessor has zero Kafka dependency.
  The outbox table is the sole output channel. This keeps the
  file-processing boundary clean and makes the component independently
  operable regardless of Kafka health.

- **BLOCKED status for DCPO evidence** — evidence rows from Capital
  One XML files are written BLOCKED to prevent downstream consumers
  from attempting evidence attachment before the parent chargeback
  case is created. Unblocking is the responsibility of a separate
  downstream consumer (outside this service).

- **Filename-only source routing** — no metadata in the SQS message
  identifies the source. All routing decisions are derived from the
  S3 key and filename at processing time. This is simple but means
  any filename convention change breaks routing silently.

---

**Risks and constraints**

🔴 HIGH — Three DNWK sub-sources declared but not implemented:
`AMEXOPTB`, `MC_REVREJ`, and `DISCHYB`. Arrival of files from these
sub-sources results in a silent no-op — the file is consumed from
SQS but no records are written. No error is raised. Operations teams
will not know files were silently dropped unless they monitor `file_job`
for zero-row outcomes.

🔴 HIGH — No HPA configured. File processing throughput scales only
by increasing replica count manually. No autoscaling under queue
backlog growth.

🟡 MEDIUM — Idempotency gap with multiple replicas. Two pods receiving
the same SQS message simultaneously can both pass the file-level
idempotency check and process the same file in parallel. No database
unique constraint protects against this.

🟡 MEDIUM — No retry on S3 reads, S3 writes, or database operations.
A transient S3 or database failure during processing writes the record
as ERROR or fails the file — no automatic recovery.

🟡 MEDIUM — DISR (Discover MAP Incoming) error codes are blank
strings in the constants definition. Records written to ERROR for DISR
sources will have an empty `error_code` column, making error diagnosis
in the outbox table difficult.

🟡 MEDIUM — DBLK production record length is uncertain. Source
contains a comment flagging that the production record length may be
134 bytes but the constant is set to 102. If production files use
134-byte records, DBLK parsing will silently drop or misparse records
beyond byte 102.

🟢 LOW — No dead letter mechanism for file-level failures. Failed
files are visible via `file_job.status = ERROR` but there is no
automated alerting or reprocessing queue. Manual intervention is
required.

---

**Planned changes and known gaps**

- Implement the three missing DNWK sub-source services: AMEXOPTB,
  MC_REVREJ, DISCHYB. Currently declared in the enum routing table
  but have no processing implementation — silent no-op on arrival.

- Confirm DBLK production record length. Source comment indicates
  uncertainty between 102 and 134 bytes. Must be confirmed against
  a live production file before trusting DBLK parse output.

- Activate or formally remove image size validation (3 MB cap) for
  DWSG. The validation was commented out and responsibility moved to
  the downstream consumer — this split of responsibility needs an ADR.

- The raw record detail field is intentionally not populated for DISR
  and MFAD sources — commented-out code exists for both. Confirm
  whether downstream consumers need this field before re-activating
  or removing permanently.

- DNWK flat-file decryption path exists as commented-out code,
  suggesting DNWK files may have previously arrived pre-encrypted
  or that a direct (non-ZIP) read path was planned. Confirm and
  remove dead code or formally document the decision.

- Add a durable error record or operational alert for file-level
  ERROR outcomes. Currently only visible via database query on
  `file_job`.

---

**Downstream dependency**

FileProcessor's output is consumed by:
- **InboundDisputeEventScheduler** — polls `chbk_outbox_row` every
  2 minutes for PENDING rows and publishes to Kafka. FileProcessor
  must complete its bulk LOADING→PENDING promotion before the
  scheduler next polls, otherwise records are invisible to downstream.
- **EvidenceConsumer** — reads `file_evidence` staging paths to
  retrieve documents from S3 and attach them to dispute cases.
- **FileAcknowledgementProcessor** — reads `file_job` ACK flags to
  generate outbound ACK files for sources that require them.

---

#### 2.3.3 InboundDisputeEventScheduler
**Owner:** Integration Team
**Status:** ✅ Production | ✅ COMPLETE
**Technology:** Spring Boot 3.5.7 / Spring Data JPA / Spring Kafka / Java 17
**Repository:** wdp-chargeback-evidence-event-scheduler
**Artifact:** com.wp.gcp:chargeback-evidence-event-schedulerr:1.0.5

---

**What it does**

`wdp-chargeback-evidence-event-scheduler` is the **transactional outbox
relay layer** for the WDP chargeback and evidence domain. It is a
continuously-running Kubernetes `Deployment` — not a `CronJob` — hosting
five independent `@Scheduled` cron jobs that drain multiple outbox and
job tables and relay their contents to AWS MSK (Kafka) topics or to
downstream HTTP services.

It operates entirely **downstream and independent** of FileProcessor.
FileProcessor and other upstream writers (FirstChargebackBatch,
CaseFillingBatch) populate the outbox tables. This service only reads
those tables — it never calls FileProcessor. All coupling is through the
shared PostgreSQL schema.

The most critical sequencing rule owned by this service: `EVIDENCE_ATTACH`
rows in `wdp.chbk_outbox_row` are held in `BLOCKED` status until their
paired `CHARGEBACK_PROCESS` row reaches `SUCCESS`. This prevents
out-of-order evidence publication for a case that does not yet exist
downstream.

**What it explicitly does NOT do:**
- Does not consume from Kafka — no `@KafkaListener`, no consumer group
- Does not parse or validate CSV/file uploads — that is FileProcessor's
  responsibility
- Does not call the Worldpay chargeback API or any card-scheme endpoint
  directly
- Does not perform schema migration — Hibernate DDL auto is `false`
- Does not expose any REST endpoints — no `@RestController`

---

**The Five Schedulers**

All five schedulers use Spring `@Scheduled(cron = ...)` driven by a shared
`ThreadPoolTaskScheduler` with a **pool size of 5** (thread name prefix:
`ChargebackEvidence`). All five can fire and execute concurrently if their
cron windows overlap. All cron expressions are externalized via Kubernetes
Secrets and are not committed to source — actual intervals must be read
from the `wdp-chargeback-evidence-event-scheduler-secrets` K8s secret.

---

**Scheduler1 — ChargebackEvidenceEventScheduler**
Polls: `wdp.chbk_outbox_row`
Query filter: `PENDING`, `FAILED` (retryCount < 3, nextRetryAt elapsed),
`PENDING_DEFERRED` (nextRetryAt elapsed). Sorted by `id` ASC. Page-0
loop — all eligible rows drained per invocation. Page size from K8s
secret `${page_size}`.

This is the highest-criticality scheduler. It:
1. Publishes `CHARGEBACK_PROCESS` rows to the `new-case-events` Kafka
   topic (partition key: `networkCaseId + cardNetwork + platform`)
2. Publishes `EVIDENCE_ATTACH` rows to the `case-evidence-events` Kafka
   topic (partition key: `caseNumber` if present, else `networkCaseId`)
3. After a successful `CHARGEBACK_PROCESS` publish, unblocks dependent
   `EVIDENCE_ATTACH` rows (`BLOCKED` → `PENDING`) for the same case
4. After a failed `CHARGEBACK_PROCESS`, error-marks dependent
   `EVIDENCE_ATTACH` rows to prevent orphaned evidence

---

**Scheduler2 — FileJobStatusCompletionScheduler**
Polls: `wdp.file_job`
Query filter: `status = PROCESSING`. No pagination — full list returned
per run.

Computes file job completion by inspecting `wdp.chbk_outbox_row`
counters. When all outbox rows for a file job reach terminal status, marks
the `file_job` row as `COMPLETED`. Also archives rows from
`wdp.chbk_outbox_row` with `SUCCESS` status that are older than 30 days
into `wdp.chbk_outbox_row_archive`. No retry logic at this level — each
job is reattempted on every scheduler fire until it reaches `COMPLETED`.

---

**Scheduler3 — OutgoingEventScheduler**
Polls: `wdp.outgoing_event_outbox`
Query filter: `FAILED` (retryCount < 3), `PENDING_DEFERRED`.

> ⚠️ **OPEN QUESTION — PENDING absent:** This query does NOT include
> `PENDING` status. The initial `PENDING → PUBLISHED` delivery for rows
> in this table must be handled by a separate component, or `PENDING` rows
> are never processed. This is either an intentional architectural
> contract or a processing gap. Requires architect confirmation.

Routes each event to a Kafka topic resolved dynamically from a
`channelTypeTopicMap` config (injected as a JSON map from K8s secret)
by the `channelType` column value. Known channel-to-topic mappings:
`EXPIRY_EVENTS` → `case-action-events`, `CORE_EVENTS` →
`core-request-events`, `GP_EVENTS` and `SEN_EVENTS` →
`external-request-events`. Partition key: `entity.getCaseNumber()`.

---

**Scheduler4 — BreOrchestrationOutboxScheduler**
Polls: `wdp.bre_orchestration_outbox`
Query filter: `FAILED` (retryCount < 3), `PENDING_DEFERRED`.

> ⚠️ Same `PENDING`-absent issue as Scheduler3. Requires confirmation.

Routes rows to one of two topics based on the target component field:
- `BUSINESS_RULES` → `business-rules` topic (`BREOutboxEvent`)
- `NOTIFICATION_ORCHESTRATOR` → `outgoing-events` topic
  (`NotificationOrchestrationOutboxEvent`)

Partition key: `entity.getCaseNumber()`.

---

**Scheduler5 — EvidenceErrorEmailScheduler**
Polls: `wdp.file_evidence`
Query filter: `attachment_status = ERROR`, updated yesterday only.
Daily lookback — not cumulative. No pagination — flat list.

**Does NOT publish to Kafka.** Generates S3 pre-signed URLs for up to 53
failed evidence documents, renders a CSV report, and POSTs it as a
multipart request to an external email notification REST endpoint
(`${email_notification_url}`). This is an alerting mechanism only — it
does not reattempt failed evidence processing.

---

**Boundaries**

Inputs:
- `wdp.chbk_outbox_row` — polled by Scheduler1, read by Scheduler2
- `wdp.file_job` — polled by Scheduler2
- `wdp.outgoing_event_outbox` — polled by Scheduler3
- `wdp.bre_orchestration_outbox` — polled by Scheduler4
- `wdp.file_evidence` — polled by Scheduler5

Outputs:
- AWS MSK `new-case-events` — ChargebackEvent (Scheduler1)
- AWS MSK `case-evidence-events` — EvidenceEvent (Scheduler1)
- AWS MSK `case-action-events` / `core-request-events` /
  `external-request-events` — OutgoingEvent, channel-type-routed
  (Scheduler3)
- AWS MSK `business-rules` — BREOutboxEvent (Scheduler4)
- AWS MSK `outgoing-events` — NotificationOrchestrationOutboxEvent
  (Scheduler4)
- `wdp.chbk_outbox_row_archive` — SUCCESS rows archived (Scheduler2)
- Email Notification REST Service — CSV error report POST (Scheduler5)
- AWS S3 — pre-signed URL generation for evidence links (Scheduler5,
  read-only access; no file writes)

---

**Error Handling and Retry Strategy**

All four Kafka-publishing schedulers share an identical application-level
retry pattern:

- retryCount incremented on each failure
- If retryCount > 2 → `status = ERROR` (terminal, never requeued by
  any scheduler query)
- Otherwise → `status = FAILED`, `nextRetryAt = now + 1 hour`
  (hardcoded constant: 3,600,000 ms)

This gives **up to 3 total attempts** per row (initial + 2 retries).

The Kafka producer client additionally retries up to 3 times per
`kafkaTemplate.send().get()` call at the Kafka-client level before the
application-level exception propagates.

**Per-row failure isolation:** Each row is processed inside a `for` loop
with a per-row `try/catch`. One failing row does not stop the batch —
processing continues to the next row. An outer `try/catch` ensures even
an unexpected exception does not crash the Spring context.

**Dead letter:** There is no Kafka DLQ and no error archive table for
terminal `ERROR` rows. Scheduler5's daily email report is the only
visibility mechanism and is alerting only — it does not retry or recover
failed rows.

---

**Idempotency and Delivery Guarantee**

Every outbox entity carries an `idempotency_id UUID`. This UUID is
forwarded as a Kafka message header `idempotency-key` on every publish.
Consumer-side deduplication via this header is the stated mechanism —
this service does not enforce it.

The publisher uses a **mark-before-send** pattern:
1. Set `status = PUBLISHED` → save to DB
2. Call `kafkaTemplate.send().get()`

This makes the design **at-most-once** for the scheduler-to-Kafka hop.
If the pod dies between step 1 and step 2, the row is permanently stuck
in `PUBLISHED` (excluded from all eligible-query filters) and is never
retried — the message is silently lost with no recovery path.

The Kafka producer is configured with `enable.idempotence=true` and
`acks=all`, giving exactly-once delivery at the broker level for a single
producer session. Across pod restarts, the producer sequence resets and
idempotency is not maintained.

---

**Scaling and Deployment**

- **Kubernetes resource type:** `Deployment` (not `CronJob`), rolling
  update strategy (`maxSurge: 1`, `maxUnavailable: 0`)
- **Replica count:** Helm/XLD placeholder in `resources.yaml` — actual
  production count not visible in source; must be confirmed from XLD
  Deploy
- **Memory:** 2048Mi limit / 1024Mi request
- **CPU:** No limits or requests configured
- **HPA:** Not configured — no autoscaling
- **Thread pool:** 5 threads (one per scheduler); all five can run
  concurrently
- **Kafka send:** Synchronous blocking call — the next row is not
  processed until the current Kafka write completes or times out.
  Provides implicit natural backpressure. No explicit rate limiting or
  sleep between page iterations.

---

**Key Architectural Decisions**

- **Transactional outbox pattern (DEC-001)** — upstream writers populate
  outbox tables; this service relays to Kafka. Decouples file ingestion
  and MasterCard batch processing from Kafka availability.
- **EVIDENCE_ATTACH sequencing gate** — `BLOCKED` status prevents
  evidence events being published before their parent chargeback case
  is created downstream.
- **Kubernetes Deployment, not CronJob** — JVM stays warm between cron
  fires, avoiding cold-start latency per cycle. Trade-off: replica
  count > 1 creates a concurrency race risk (see Risks).
- **Mark-before-send (at-most-once)** — deliberate design to prevent
  duplicate Kafka publish at the cost of potential silent message loss
  on crash. Consumer deduplication via `idempotency-key` header is the
  downstream recovery contract.
- **No distributed locking** — no ShedLock, no SELECT FOR UPDATE, no
  advisory lock. Race condition mitigation is consumer-side only.
- **Dynamic topic routing via channelTypeTopicMap (Scheduler3)** — topic
  resolved at publish time from a config map rather than hardcoded.
  Allows new channel types to be added via K8s secret without a code
  change.
- **Scheduler5 as alerting-only, not recovery** — error email provides
  operational visibility for failed evidence documents but does not
  constitute a retry or DLQ mechanism.

---

**Risks & Constraints**

🔴 HIGH — **Concurrency race window with replicas > 1.** The outbox
query for all Kafka-publishing schedulers has no `SELECT FOR UPDATE` or
`SKIP LOCKED`. With multiple replicas running the same cron, two pods can
read and begin processing the same row before either commits the
`PUBLISHED` status update, creating a race window for duplicate Kafka
publishing. The only mitigation is consumer-side deduplication via
`idempotency-key`. There is no database-level locking, advisory lock, or
distributed coordination (e.g. ShedLock) in this codebase. Production
replica count is unconfirmed from source — this risk is live if replicas
> 1.

🔴 HIGH — **At-most-once delivery on pod crash.** A pod killed between
the `PUBLISHED` DB write (step 1) and the Kafka send (step 2) leaves the
row permanently stuck in `PUBLISHED` with no requeue mechanism. Message
is silently lost. No dead letter, no alert, no recovery path exists.

🟡 MEDIUM — **PENDING absent from Scheduler3 and Scheduler4 queries.**
Rows written as `PENDING` to `wdp.outgoing_event_outbox` or
`wdp.bre_orchestration_outbox` are never picked up by this service.
If no other component handles initial delivery for these tables, these
rows are silently stranded. Intentional contract or processing gap —
requires architect confirmation before any changes to these tables.

🟡 MEDIUM — **No dead letter queue or error archive for ERROR rows.**
After 3 failed attempts, rows reach terminal `ERROR` and are never
requeued. The only visibility is Scheduler5's daily email report, which
covers `file_evidence` only — not `chbk_outbox_row`, `outgoing_event_outbox`,
or `bre_orchestration_outbox` errors.

🟡 MEDIUM — **Static AWS credentials for S3 (Scheduler5).** Migration
from IAM instance profile (`InstanceProfileCredentialsProvider`) to
static access/secret keys (`StaticCredentialsProvider`) was started but
not completed. Static credentials in production is a security concern;
keys are injected via K8s secrets.

🟡 MEDIUM — **No CPU limits or requests configured.** Pod runs Burstable
QoS — first candidate for eviction under node memory pressure.

🟡 MEDIUM — **No HPA.** Combined with the concurrency race risk, horizontal
scaling is unsafe without a distributed locking solution.

🟡 MEDIUM — **No timeout on RestTemplate for Scheduler5 email endpoint.**
A hung email notification service call blocks the Scheduler5 thread
indefinitely. No circuit breaker or connection timeout configured.

🟢 LOW — **All cron expressions externalized to K8s secrets.** No
schedule values visible in committed source or YAML. Operational
debugging and schedule auditing require access to the K8s secret.

🟢 LOW — **`commons-beanutils` and `json-path` declared in pom.xml with
no visible usage in source.** Potentially unused transitive dependencies.

---

**Planned Changes / Open Items**

- Confirm actual production replica count from XLD Deploy configuration
- Resolve PENDING-absent query behaviour for Scheduler3 and Scheduler4
  — intentional architectural contract or processing gap; ADR required
  if intentional
- Complete IAM instance profile migration for S3 presigner to eliminate
  static credentials in production
- Evaluate ShedLock or SELECT FOR UPDATE to make replicas > 1 safe
- Add durable dead letter or error table for terminal ERROR rows across
  all four Kafka outbox tables
- Configure RestTemplate timeout and circuit breaker for Scheduler5
  email notification call
- Remove or formally document unused pom.xml dependencies

---

#### 2.3.4 FileAcknowledgementProcessor
**Owner:** Integration Team
**Status:** ✅ Production | ✅ COMPLETE
**Technology:** Spring Boot 3.5.11 / Spring Data JPA / AWS SDK v2 / JAXB / Java 17
**Repository:** wdp-evidence-ack-scheduler
**Artifact:** com.wp.wdp.evidence.ack.scheduler:wdp-evidence-ack-scheduler:1.0.3

---

**What it does**

A continuously-running Kubernetes Deployment that
polls the shared PostgreSQL database on a scheduled
cron cadence and generates outbound acknowledgement
files for inbound file jobs that have completed
processing. For each eligible job it reads per-row
detail records, formats them into the
merchant-specific file layout, uploads the result
to AWS S3, and marks the job as acknowledged.

No SFTP, API, or messaging layer is involved —
S3 is the sole delivery target. The upstream
inbound processor (FileProcessor) is responsible
for populating `wdp.file_job` and
`wdp.chbk_outbox_row`. This component only reads
those tables — it does not call FileProcessor and
has no coupling beyond the shared database schema.

**What it explicitly does NOT do:**
- Does not receive or parse inbound files
- Does not deliver ACK files via SFTP, MQ, or API
- Does not process jobs where `ack_required = false`
  — these are silently skipped
- Does not expose any HTTP endpoint

**Boundaries**

Inputs:
- `wdp.file_job` table — polled on every cron fire.
  Eligible rows must have:
  `status IN (COMPLETED, ERROR)`,
  `ack_required = true`,
  `ack_status = PENDING`
- `wdp.chbk_outbox_row` table — read per file job
  to retrieve per-row processing detail.
  `record_detail` column is JSONB.
  No writes to this table.

Outputs — ACK files uploaded to AWS S3
(region `us-east-2`, bucket `wdp-files`):

| Source | Merchant | Format | S3 Key Pattern |
|---|---|---|---|
| `WALMART_SIG_CAP` | Walmart Signature Capture | Fixed-width `.txt` | `DWSG/OUTBOUND/ACK/AUF2_DWSG_ROBCDWL1_WMSIG_CONF_<yyyyMMddHHmmss>.txt` |
| `CORE_BULK_RESP` | Meijer (Core Bulk Response) | Fixed-width `.txt` | `DBLK/OUTBOUND/ACK/DBLK_MEJR_CHRGRESP_CONF_<yyyyMMddHHmmSS>.txt` |
| `CAPONE_CMRTR` | CapitalOne Commercial (CMRTR) | XML-in-ZIP | `DCPO/OUTBOUND/ACK/DCPO_RODMRDOA.WP01.ACK.<yyyyMMddHHmm>.zip` |
| `CAPONE_BJWC` | CapitalOne BJWC | XML-in-ZIP | `DCPO/OUTBOUND/ACK/DCPO_RODMRDOA.WP01.ACK.<yyyyMMddHHmm>.zip` |

Non-production environments use the `wdp-files-dev`
bucket. Any `file_job.source` value not matching
these four constants is logged as
"Unidentified source" and skipped.

After successful upload, writes back to
`wdp.file_job`:
`ack_status = COMPLETED`, `ack_generated_at`,
`updated_at`, `updated_by = WPACKSD`.

**ACK File Formats**

*Walmart SigCap* — fixed-width UTF-8 `.txt`.
Header (`H` + date), one detail line per
`chbk_outbox_row` record (78-char padded body +
2-char return code), trailer (`T` + 9-digit
zero-padded evidence count). Return codes map
processing outcome to standard codes
(00 = success; 30–40 = error variants).
ACK generation is skipped and the job left
`PENDING` if `totalEvidences` is null or
no detail records exist.

*Meijer / Core Bulk Response* — fixed-width
UTF-8 `.txt`. Header (`1` + datetime with
nanosecond-precision `SS`), one detail line per
row (55-char right-padded filename + 3-char code),
trailer (`3` + 9-digit count). Structural error
codes (`901`, `902`, `903`) on the file job cause
synthetic header/trailer defect lines to be
injected into the ACK body.

*CapitalOne CMRTR and BJWC* — JAXB-serialised XML
wrapped in a ZIP archive. XML namespace:
`https://capitalone.com/transactiontrouble/chargeback/wp`.
Schema: `worldpay-cbpackage.xsd` v0.7.0
(stamped Version 1.0.0 – 05/02/2026 — recently
delivered feature). Supports up to 2,000
acknowledgement elements per batch. Only rows with
`event_type = CHARGEBACK_PROCESS` are included;
`EVIDENCE_ATTACH` rows are excluded. Status maps
`chbk_outbox_row.status = SUCCESS` → `Accepted`
and `ERROR` → `Rejected` with a known
`valid-reject` reason where applicable.

**Trigger and Schedule**

Schedule-driven, not event-driven. A Spring
`@Scheduled` cron job fires on a cadence injected
at runtime from the `wdp-evidence-ack-scheduler-secrets`
Kubernetes Secret (`cron_scheduler` key). The actual
production cron value is not visible in source code.
Local development defaults to every minute.

The pod runs continuously as a Kubernetes
**Deployment** (not a CronJob). Scheduling is
handled inside the JVM by Spring `@EnableScheduling`.

**Idempotency**

Two guards are in place:
1. S3 pre-check — `headObject` is called before
   `putObject`. If the key already exists, the
   upload is skipped and `ack_status` is not updated.
2. DB status guard — the query only selects rows
   with `ack_status = PENDING`. Once a job reaches
   `COMPLETED` it is never re-selected.

These guards are effective within a single pod.
Under a rolling deployment with two pods running
concurrently, different file timestamps can be
generated per pod (no cross-JVM uniqueness
guarantee), creating a race condition on the DB
status update.

**Key architectural decisions**
- [DEC-PLACEHOLDER: ACK files written to S3 only
  — no SFTP, MQ, or direct API delivery]
- [DEC-PLACEHOLDER: Schedule-driven polling
  pattern — no event trigger or callback]
- [DEC-010: Immutable Versioned ACK Snapshots
  — verify still applies]
- [DEC-PLACEHOLDER: Kubernetes Deployment not
  CronJob — scheduling owned by Spring inside
  a continuously-running pod]

**Risks & Constraints**

🔴 HIGH — No poison-pill protection. A job with
malformed `record_detail` data will be retried
on every cron fire indefinitely. No maximum-retry
counter or circuit breaker exists.

🔴 HIGH — Two-pod race condition during rolling
updates. Different ACK filenames can be generated
for the same job by two concurrent pods, producing
duplicate S3 objects and a contested DB status
update. No distributed lock or cross-JVM uniqueness
mechanism is in place.

🟡 MEDIUM — No dead letter mechanism. All
failures are log-only. No DLQ, error queue, or
dedicated error table exists.

🟡 MEDIUM — No CPU limits or requests configured.
Unbounded CPU usage is possible; pod is first
candidate for eviction under node memory pressure.

🟡 MEDIUM — No HPA. Combined with the two-pod
race risk, horizontal scaling is unsafe without
a distributed locking solution.

🟡 MEDIUM — No client-side S3 encryption or
signing applied in code. IAM-based access control
via EKS IRSA is the sole access control mechanism.

🟢 LOW — Production cron schedule not visible in
source. Schedule auditing requires access to the
K8s secret `wdp-evidence-ack-scheduler-secrets`.

🟢 LOW — `spring-cloud-aws-starter-s3` declared
as a dependency but auto-configuration is
overridden by a manually constructed `S3Client`
bean. Minor redundancy, not harmful.

**Planned changes**
- Confirm which additional inbound sources require
  ACK files beyond the current four — see
  Section 5.4 PENDING
- Evaluate distributed lock (e.g., ShedLock or
  SELECT FOR UPDATE) to eliminate two-pod race
  condition during rolling deployments
- Add maximum-retry counter or circuit breaker
  to prevent indefinite retry of permanently
  failing jobs
- Add durable error record or operational alert
  for jobs that remain `ack_status = PENDING`
  beyond a threshold

---

## PART 3 — KAFKA EVENT BUS

---

### 3.1 AWS MSK Configuration
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT

---

**What it does**
AWS Managed Streaming for Apache Kafka.
Backbone for all asynchronous
communication between WDP components.

⚠️ DRAFT — broker count, AZ config,
retention settings, storage config
to be confirmed from infrastructure
documentation

**Key architectural decisions**
- [DEC-013: Kafka AWS MSK as event
  streaming platform]
- [DEC-003: Merchant-scoped partitioning]
- [DEC-005: Manual Kafka offset commit]

**Risks & Constraints**
- MSK provisioned storage is permanently
  one-directional — cannot reduce
  after scaling up
- All storage increases are irreversible
  commitments

---

### 3.2 Topic Reference

| Topic | Publisher | Consumer | Partition Key | Status |
|---|---|---|---|---|
| nap-dispute-events | NAPDisputeEventService | NAPDisputeEventProcessor | merchant_id | ✅ |
| new-case-events | InboundDisputeEventScheduler | CaseCreationConsumer | merchant_id | ✅ |
| case-evidence-events | InboundDisputeEventScheduler | EvidenceConsumer | merchant_id | ✅ |
| business-rules | ⚠️ TBC | BusinessRulesProcessor | merchant_id | ✅ |
| outgoing-events | BusinessRulesProcessor | NotificationOrchestrator | merchant_id | ✅ |
| internal-integration-events | AcceptService, ContestService | NAP Outcome Processor, VisaResponseQuestionnaire | merchant_id | ✅ |
| case-action-events (expiry) | NotificationOrchestrator | CaseExpiryUpdateConsumer | ⚠️ TBC | ✅ |
| external-request-events | NotificationOrchestrator | ThirdPartyNotificationConsumer, BEN Consumer, EDIA Consumer | merchant_id | ✅ |
| core-request-events (DB2) | NotificationOrchestrator | CoreNotificationConsumer | merchant_id | ✅ |
| EDIA events | EDIA Consumer | EDIA Platform | TBD | 🔴 |

---

## PART 4 — CORE PROCESSING

---

### 4.1 Event Consumers

---

#### 4.1.1 CaseCreationConsumer
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Primary case creation component for all
non-NAP dispute events. Consumes from
new-case-events topic. Enriches dispute
events with merchant and transaction data
from acquiring platform APIs before
creating dispute cases in WDP Core.
Handles PAN decryption transiently for
enrichment API calls — clear PAN never
persisted.

**Boundaries**

Inputs:
- Consumes from: new-case-events topic

Outputs:
- Creates dispute cases in WDP Core
  (via CaseManagementService)
- Calls MerchantTransactionService
  (CORE disputes enrichment)
- Calls LATAM platform API directly
  (LATAM disputes enrichment)
- Calls VAP platform API directly
  (VAP disputes enrichment)
- Calls EncryptionService to decrypt
  EPAN transiently for API calls
- Stores HPAN in case table — never
  stores clear PAN

**Enrichment by acquiring platform:**

| Platform | Enrichment Approach |
|---|---|
| CORE | Calls MerchantTransactionService |
| LATAM | Direct API call |
| VAP | Direct API call |
| NAP | Not handled here — NAPDisputeEventProcessor |

**Key architectural decisions**
- [DEC-007: Two-Token PAN Strategy]
- [DEC-004: Encrypt PAN at ingestion]
- [DEC-PLACEHOLDER: CaseCreationConsumer
  as single case creation path for all
  non-NAP platforms]

**Risks & Constraints**
- Does not currently handle NAP disputes
- LATAM and VAP enrichment APIs not
  yet in production
⚠️ DRAFT — consumer group, concurrency,
error handling, outbox table details
to be completed from Copilot CLI analysis

**Planned changes**
- NAP disputes to be migrated to this
  consumer once NAPDisputeEventProcessor
  writes to chbk_outbox_row

---

#### 4.1.2 EvidenceConsumer
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Consumes from case-evidence-events topic
and attaches evidence documents to
existing dispute cases. Retrieves
documents directly from S3 /staging
using path reference in the event.
Calls DocumentManagementService to
store document in S3 and metadata
in DynamoDB.

**Boundaries**

Inputs:
- Consumes from: case-evidence-events
- Retrieves documents from S3 /staging

Outputs:
- Calls DocumentManagementService
  (stores document in S3, metadata
  in DynamoDB)

**Key architectural decisions**
- [DEC-PLACEHOLDER: Evidence attached
  via separate consumer — not inline
  with case creation]

**Risks & Constraints**
⚠️ DRAFT — consumer group, error
handling, retry to be completed
from Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.1.3 BusinessRulesProcessor
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Kafka consumer that applies configured
business rules to dispute events.
Execution engine for business rules —
distinct from BusinessRulesService
which manages rule definitions via UI.
Makes direct DB calls to fetch rules —
does not call BusinessRulesService.
Publishes processed events to
outgoing-events topic.

**Boundaries**

Inputs:
- Consumes from: business-rules topic
- Reads rules directly from
  dispute rules DB (Aurora PostgreSQL)

Outputs:
- Publishes to: outgoing-events topic
- Updates dispute cases in WDP Core

**Key architectural decisions**
- [DEC-PLACEHOLDER: BRE as async Kafka
  consumer rather than synchronous
  in-process execution]
- [DEC-PLACEHOLDER: Direct DB call for
  rules — not via BusinessRulesService]
- [DEC-011: BRE Crash Recovery via Step
  Checkpointing — verify still applies]

**Risks & Constraints**
⚠️ Publisher of business-rules topic
to be confirmed
⚠️ DRAFT — step checkpointing details,
consumer group, error handling to be
completed from Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.1.4 CaseExpiryUpdateConsumer
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Consumes from case-action-events (expiry)
topic and updates case expiry status in
WDP Core. Updates case expiry status only
— no further downstream processing
triggered.

**Boundaries**

Inputs:
- Consumes from:
  case-action-events (expiry) topic

Outputs:
- Updates case expiry status in
  WDP Core (Aurora PostgreSQL)

**Key architectural decisions**
- [DEC-PLACEHOLDER: Expiry handled as
  async event rather than scheduled job]

**Risks & Constraints**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.1.5 NotificationOrchestrator
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Central outbound routing component.
Consumes from outgoing-events topic
and routes each event to the appropriate
outbound Kafka topic based on business
logic defined in code. No merchant
configuration table — routing is
code-defined. Also writes to
file_notifications DB table for
file-based outbound generation.

**Boundaries**

Inputs:
- Consumes from: outgoing-events topic

Outputs:
- Publishes to: external-request-events
- Publishes to: core-request-events (DB2)
- Publishes to: case-action-events (expiry)
- Writes to: file_notifications DB table

Does NOT publish to:
internal-integration-events
(that is published by AcceptService
and ContestService directly)

**Key architectural decisions**
- [DEC-PLACEHOLDER: Routing logic
  in code — no config table]
- [DEC-PLACEHOLDER: file_notifications
  DB table as bridge between Kafka
  and file generation components]

**Risks & Constraints**
⚠️ DRAFT — routing logic details,
consumer group, error handling to be
completed from Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

### 4.2 Case Action Services

---

#### 4.2.1 AcceptService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Processes merchant decisions to accept
a dispute. Makes direct synchronous API
calls to Visa and MasterCard to notify
the network of acceptance. For NAP
disputes across all networks, publishes
to internal-integration-events so NAP
Outcome Processor can notify NAP-DPS.

**Boundaries**

Inputs:
- REST API calls from API Gateway
  (merchant or ops portal actions)

Outputs:
- Direct synchronous API call to Visa
- Direct synchronous API call to MasterCard
- Publishes to internal-integration-events
  (NAP disputes only — all networks)

**Key architectural decisions**
- [DEC-PLACEHOLDER: Direct synchronous
  API for Visa/MasterCard accept —
  not async]
- [DEC-PLACEHOLDER: NAP outcome via
  internal-integration-events]

**Risks & Constraints**
⚠️ DRAFT — timeout handling, retry
strategy, error handling for network
API failures to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.2.2 ContestService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Processes merchant decisions to contest
a dispute. Makes direct synchronous API
calls to Visa and MasterCard. After
successful contest publishes two event
categories to internal-integration-events:
NAP dispute events (all networks) for
NAP-DPS notification, and Visa dispute
events (all acquiring platforms) for
Visa questionnaire retrieval.

**Boundaries**

Inputs:
- REST API calls from API Gateway

Outputs:
- Direct synchronous API call to Visa
- Direct synchronous API call to MasterCard
- Publishes to internal-integration-events:
  NAP dispute contest events (all networks)
  Visa dispute contest events
  (all acquiring platforms)

**Publishing rules on
internal-integration-events:**

| Event | Condition |
|---|---|
| NAP contest | NAP disputes — all networks |
| Visa contest | Visa disputes — all acquiring platforms |

**Key architectural decisions**
- [DEC-PLACEHOLDER: Two separate event
  categories on same topic for different
  consumers]
- [DEC-PLACEHOLDER: ContestService
  triggers Visa questionnaire retrieval
  via event]

**Risks & Constraints**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.2.3 ChargebackService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
The only WDP Core service accessible
programmatically to external merchants
via APIGEE and Akamai. Supports both
read and action operations. Also serves
as the callback endpoint for third-party
systems (SignifyD, JustAI) and merchant
notification systems (BEN) after they
receive dispute notifications.

**Boundaries**

Inputs:
- External merchant system API calls
  via Akamai → APIGEE → API Gateway
- Callback calls from SignifyD
- Callback calls from JustAI
- Callback calls from BEN merchants

Outputs:
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: Single externally
  exposed service — all external
  merchants use ChargebackService only]
- [DEC-PLACEHOLDER: ChargebackService
  as callback endpoint for third-party
  notification systems]

**Risks & Constraints**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.2.4 DisputeService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Manages the core dispute lifecycle —
state transitions, dispute data retrieval,
and dispute-level operations.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.2.5 CaseManagementService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Owns the dispute case record. Responsible
for case creation, case updates, and
maintaining the integrity of the case
state machine. Authoritative source for
all case data in WDP.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: Single service owns
  case record — authoritative source]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.2.6 CaseActionService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Handles specific case-level actions taken
by operations teams — routing a case to
a queue, writing off a case, splitting
a case, or advancing a case through the
dispute lifecycle.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.2.7 NotesService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Allows operations teams and merchants
to add notes to a dispute case. Notes
are retained as part of the immutable
case audit trail.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.2.8 QuestionnaireService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Manages the response questionnaire
merchants complete before submitting
a contest. Captures structured evidence
and reasoning submitted to card network
as part of the contest response.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

### 4.3 Search & Display Services

---

#### 4.3.1 CaseSearchService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Provides dispute search capability
across the platform. Supports extensive
filter criteria. Serves both portal UIs
and programmatic search requests.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.3.2 DisplayCodeService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Manages mapping between internal WDP
codes and human-readable display values
shown in UI — reason codes, network
codes, status codes, action codes.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

### 4.4 Queue & Workflow Services

---

#### 4.4.1 FaxQueueService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Handles fax communications sent by
merchants to WDP. Operations teams
act on merchant faxes through this
service. Available to Ops Portal
users only.

⚠️ Detailed fax functionality to be
documented in dedicated section —
see Section 7.4 Queues Section

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.4.2 UserQueueSkillService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Manages the relationship between users,
queues, and skills. Determines which
queues a user can access and which cases
within those queues are eligible based
on assigned skills.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

### 4.5 Rules & Configuration Services

---

#### 4.5.1 BusinessRulesService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Provides CRUD management of business
rules used in dispute processing.
Called synchronously by portal UIs
to add, modify, retrieve, and delete
rules. Manages rule definitions only —
does NOT execute rules. Execution is
handled by BusinessRulesProcessor.

**Boundaries**

Inputs:
- REST API calls from portal UIs
  via API Gateway

Outputs:
- Reads/writes dispute rules DB
  (Aurora PostgreSQL)

**Key architectural decisions**
- [DEC-PLACEHOLDER: Separation of rule
  management (BusinessRulesService) from
  rule execution (BusinessRulesProcessor)]

**Risks & Constraints**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.5.2 RulesService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Manages additional configuration rules
for dispute routing, queue assignment,
and case handling behaviour. Works
alongside BusinessRulesService to
provide full rules configuration
capability.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

### 4.6 Organisation & User Services

---

#### 4.6.1 OrgManagementService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Manages organisational hierarchies
within WDP. Handles merchant account
configuration, org-level settings,
and routing rule configuration at
the organisation level.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD from
  Copilot analysis]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.6.2 MerchantTransactionService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Provides merchant and transaction data
enrichment for CORE acquiring platform
disputes. Called by CaseCreationConsumer
during dispute case creation to fetch
merchant and transaction details from
the CORE platform (WDP owned).
LATAM and VAP enrichment is handled
by CaseCreationConsumer calling those
platform APIs directly.

**Boundaries**

Inputs:
- REST API calls from
  CaseCreationConsumer

Outputs:
- Calls CORE platform API for merchant
  and transaction details
- Returns enriched data to
  CaseCreationConsumer

**Key architectural decisions**
- [DEC-PLACEHOLDER: Centralised CORE
  enrichment in dedicated service —
  LATAM/VAP called directly]

**Risks & Constraints**
- CORE platform dependency —
  if CORE API unavailable, CORE
  dispute enrichment fails
⚠️ DRAFT — timeout, retry, error
handling to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

### 4.7 Supporting Services

---

#### 4.7.1 EncryptionService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Sole component authorised to handle
plaintext PAN data in WDP. Encrypts
PANs at point of ingestion and provides
decryption only to authorised callers.
No other service stores or processes
raw cardholder data. Implements
two-token PAN strategy — HPAN for
deterministic lookups, EPAN for
reversible storage.

**Boundaries**

Inputs:
- Encryption calls from all inbound
  processors (FileProcessor,
  VisaDisputeBatch, MC Batch jobs)
- Decryption calls from
  CaseCreationConsumer (transient only)

Outputs:
- HPAN — stored in case table
- EPAN — stored in EPAN→HPAN
  mapping table
- DEK managed via AWS KMS
  (6-hour cache)

**PAN security rules:**

| Rule | Detail |
|---|---|
| Plaintext PAN | Only EncryptionService handles it |
| PAN at rest | Never stored in plaintext |
| PAN in transit | Never travels in plaintext |
| PAN in logs | Never appears in logs |
| NAP exception | No PAN in NAP disputes |

**Key architectural decisions**
- [DEC-004: Encrypt PAN at ingestion]
- [DEC-007: Two-Token PAN Strategy]
- [DEC-008: AWS KMS Key Management]

**Risks & Constraints**
- Global dependency — all inbound
  processing depends on this service
- DEK cache 6-hour window — if KMS
  unavailable, service continues for
  up to 6 hours then fails
- If EncryptionService is down, all
  inbound processing stops
⚠️ DRAFT — availability NFR, scaling,
key rotation handling to be completed
from Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.7.2 TokenService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Centralised JWT token management service.
All WDP consumers and batch jobs call
TokenService to obtain JWT tokens for
making API calls. Caches tokens in AWS
ElastiCache and refreshes from IDP only
when expired. Eliminates need for every
component to implement its own IDP
integration.

**Note:** TokenService has NO relation
to PAN tokenisation — it is purely
for JWT token management.

**Boundaries**

Inputs:
- JWT token requests from all WDP
  consumers and batch jobs

Outputs:
- Returns cached JWT from ElastiCache
  if valid
- Calls IDP to get new JWT if expired
- Updates ElastiCache with new token

**Key architectural decisions**
- [DEC-PLACEHOLDER: Centralised JWT
  management — single service rather
  than per-component IDP integration]
- [DEC-PLACEHOLDER: ElastiCache for
  JWT caching]

**Risks & Constraints**
- If TokenService is down, all
  components making external API
  calls cannot authenticate
- ElastiCache availability dependency
⚠️ DRAFT — cache TTL, refresh strategy,
error handling to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.7.3 DocumentManagementService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Handles evidence document storage and
retrieval. Documents stored in S3 with
metadata in DynamoDB. Serves inbound
evidence from file processing and
merchant-submitted evidence from
dispute response. Called by
EvidenceConsumer, VisaResponseQuestionnaire,
and NAPDisputeEventService (119 docs).

**Boundaries**

Inputs:
- Document store calls from
  EvidenceConsumer
- Document store calls from
  VisaResponseQuestionnaire
- Document attach calls from
  NAPDisputeEventService (119 docs)
- Document store calls from
  merchant UI submissions

Outputs:
- Stores documents in S3 Documents
- Stores metadata in DynamoDB
- Returns document reference

**Key architectural decisions**
- [DEC-PLACEHOLDER: DynamoDB for
  document metadata — not Aurora]
- [DEC-PLACEHOLDER: S3 for document
  storage — not DB blob]
- [DEC-PLACEHOLDER: Single service
  owns all document operations]

**Risks & Constraints**
- DynamoDB is exclusive to this service
- S3 document retention policy needed
⚠️ DRAFT — scaling, error handling,
S3 path structure to be completed
from Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 4.7.4 APILogService
**Owner:** Core WDP Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Provides API-level audit logging across
the platform. Captures all inbound API
calls for audit, compliance, and
operational troubleshooting.

**Boundaries**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Key architectural decisions**
- [DEC-PLACEHOLDER: Centralised API
  audit logging service]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ DRAFT — to be confirmed

---

## PART 5 — OUTBOUND PROCESSING

---

### 5.1 Card Network Response

---

#### 5.1.1 NAP Outcome Processor
**Owner:** Integration Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Consumes from internal-integration-events
and delivers dispute outcomes to NAP-DPS
for NAP acquiring platform money movement.
Processes NAP accept events (from
AcceptService) and NAP contest events
(from ContestService) for all networks.

**Boundaries**

Inputs:
- Consumes from:
  internal-integration-events topic
  NAP accept events (all networks)
  NAP contest events (all networks)

Outputs:
- Direct API call to NAP-DPS
  with dispute outcome

**Key architectural decisions**
- [DEC-PLACEHOLDER: Direct API to
  NAP-DPS — temporary pattern
  pending EDIA migration]

**Risks & Constraints**
- NAP-DPS API dependency —
  if NAP-DPS unavailable, outcome
  delivery fails
⚠️ DRAFT — retry strategy, error
handling to be completed from
Copilot CLI analysis

**Planned changes**
- Migrate from direct NAP-DPS API
  to EDIA route
- Timeline: TBD

---

#### 5.1.2 VisaResponseQuestionnaire
**Owner:** Integration Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Consumes from internal-integration-events
and retrieves the Visa questionnaire
submitted as part of a merchant contest.
Attaches questionnaire as a document to
the dispute case via DocumentManagementService.
Triggered for all acquiring platforms
whenever a merchant contests a Visa dispute.

**Boundaries**

Inputs:
- Consumes from:
  internal-integration-events topic
  Visa contest events
  (all acquiring platforms)

Outputs:
- Calls Visa API to get questionnaire
- Calls DocumentManagementService to
  store questionnaire (S3 + DynamoDB)

**Key architectural decisions**
- [DEC-PLACEHOLDER: Questionnaire
  retrieval async via Kafka event
  — not inline with contest call]

**Risks & Constraints**
- Visa API dependency
⚠️ DRAFT — retry strategy, error
handling to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

### 5.2 Notification Consumers

---

#### 5.2.1 ThirdPartyNotificationConsumer
**Owner:** Integration Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Consumes from external-request-events
and delivers dispute events to third-party
fraud and intelligence systems via REST API.
After receiving notification, SignifyD and
JustAI call back ChargebackService to get
dispute details and act on disputes.

**Boundaries**

Inputs:
- Consumes from:
  external-request-events topic

Outputs:
- REST API call to SignifyD
- REST API call to JustAI

**Key architectural decisions**
- [DEC-PLACEHOLDER: Third-party systems
  receive notification then pull details
  via ChargebackService callback]

**Risks & Constraints**
- SignifyD and JustAI API dependencies
⚠️ DRAFT — retry strategy, error
handling to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 5.2.2 BEN Consumer
**Owner:** Integration Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Consumes from external-request-events
and delivers dispute lifecycle
notifications to BEN merchant notification
platform via webhook. Merchants receive
notifications via BEN and call back
ChargebackService to get dispute details
and act on disputes.

**Boundaries**

Inputs:
- Consumes from:
  external-request-events topic

Outputs:
- Webhook call to BEN platform

**Key architectural decisions**
- [DEC-PLACEHOLDER: Webhook delivery
  to BEN — not REST API]
- [DEC-PLACEHOLDER: Merchants pull
  details via ChargebackService after
  BEN notification]

**Risks & Constraints**
- BEN platform webhook dependency
⚠️ DRAFT — retry strategy, error
handling to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 5.2.3 CoreNotificationConsumer
**Owner:** Integration Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Consumes from core-request-events (DB2)
and delivers dispute outcome notifications
to CORE acquiring platform via DB2.
CORE is WDP-owned so has direct
connection bypassing EDIA.

**Boundaries**

Inputs:
- Consumes from:
  core-request-events (DB2) topic

Outputs:
- Delivers to CORE platform via DB2

**Key architectural decisions**
- [DEC-PLACEHOLDER: Direct DB2
  connection for CORE — not via EDIA]
- [DEC-PLACEHOLDER: WDP-owned platform
  gets direct connection]

**Risks & Constraints**
- DB2 dependency for CORE platform
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Planned changes**
- Potential future migration to
  EDIA route as EDIA adoption matures

---

#### 5.2.4 EDIA Consumer
**Owner:** Integration Team
**Status:** 🔴 Planned
**Repository:** ⚠️ To be confirmed

---

**What it does**
WDP-owned consumer that will consume
from external-request-events and convert
WDP internal dispute events into
enterprise-defined EDIA format before
publishing to EDIA platform Kafka topic.
Acts as anti-corruption layer — WDP
internal format decoupled from EDIA
enterprise format.

**Boundaries**

Inputs:
- Will consume from:
  external-request-events topic

Outputs:
- Will publish to:
  EDIA events topic (EDIA platform)
  in EDIA enterprise format

**Acquiring platforms via EDIA:**
- NAP 🔴 Planned
- LATAM 🔴 Planned
- VAP 🔴 Planned

**Key architectural decisions**
- [DEC-PLACEHOLDER: EDIA as strategic
  direction for all acquiring platform
  outbound integration]
- [DEC-PLACEHOLDER: WDP owns format
  translation — EDIA consumer owned
  by WDP not EDIA team]

**Risks & Constraints**
- EDIA platform dependency
- EDIA enterprise format defined
  externally — WDP must keep in sync

**Planned changes**
- Full design and implementation
  pending
- Timeline: TBD

---

### 5.3 Outbound File Generation

---

#### 5.3.1 CapitalOne Response file processor
**Owner:** Integration Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Reads from file_notifications DB table
and generates CapitalOne response file.
Places file in S3 /outbound/capitalOne/
for ControlM to transfer to Sterling
Mailbox for delivery to CapitalOne
via DM Mainframe.

**Boundaries**

Inputs:
- Reads from file_notifications table
  (Aurora PostgreSQL)

Outputs:
- CapitalOne response file →
  S3 /outbound/capitalOne/

**Key architectural decisions**
- [DEC-PLACEHOLDER: file_notifications
  DB table as trigger for file generation]
- [DEC-PLACEHOLDER: S3 folder-per-target
  pattern]

**Risks & Constraints**
⚠️ DRAFT — file format, generation
schedule, error handling to be
completed from Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 5.3.2 FileAcknowledgementProcessor (Outbound)
**Owner:** Integration Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Generates acknowledgement files for
inbound sources that require confirmation
of file receipt and processing outcome.
Places ACK files in target-specific
S3 /outbound folders.

**Boundaries**

Inputs:
⚠️ DRAFT — trigger mechanism to be
confirmed from Copilot CLI analysis

Outputs:
- Meijer ACK → S3 /outbound/meijer/
- Walmart ACK → S3 /outbound/walmart/
- CapitalOne ACK → S3 /outbound/capitalOne/

**Key architectural decisions**
- [DEC-010: Immutable Versioned ACK
  Snapshots — verify still applies]

**Risks & Constraints**
⚠️ DRAFT — to be completed

**Planned changes**
⚠️ Which other sources need ACK files
to be confirmed — see Section 5.4 PENDING
in WDP-ARCHITECTURE.md

---

#### 5.3.3 NetworkResponseFileProcessor
**Owner:** Integration Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Reads from file_notifications DB table
and generates four network response files.
Each file placed in its own dedicated
S3 /outbound folder.

**Boundaries**

Inputs:
- Reads from file_notifications table

Outputs:
- Amex file → S3 /outbound/amex/
- AmexHybrid file → S3 /outbound/amexHybrid/
- Discover file → S3 /outbound/discover/
- DiscoverHybrid file →
  S3 /outbound/discoverHybrid/

**Note on DiscoverHybrid:**
DiscoverHybrid has a special outbound
flow — on-premise File Transfer Batch
(WDP owned) pulls via SFTP from
S3 /outbound/discoverHybrid/ and
deposits to NAS on RMO servers.
All other files go via ControlM →
Sterling → DM Mainframe or SFTP.

**Key architectural decisions**
- [DEC-PLACEHOLDER: Single processor
  generates all four network files]
- [DEC-PLACEHOLDER: DiscoverHybrid
  special pull flow vs standard
  ControlM push flow]

**Risks & Constraints**
⚠️ DRAFT — file formats, generation
schedule, Discover vs DiscoverHybrid
differences to be documented
(see open discussion point)

**Planned changes**
⚠️ Amex vs AmexHybrid differences
to be documented — see Section 5.4
PENDING in WDP-ARCHITECTURE.md

---

#### 5.3.4 Dialogu Issuer document Processor
**Owner:** Integration Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Reads from file_notifications DB table
and generates a ZIP file containing all
issuer documents received by WDP.
Places ZIP in S3 /outbound/dialogu/
for delivery to merchants via Sterling
→ SFTP → Dialogue.

**Boundaries**

Inputs:
- Reads from file_notifications table

Outputs:
- ZIP of issuer documents →
  S3 /outbound/dialogu/

**Key architectural decisions**
- [DEC-PLACEHOLDER: ZIP packaging
  for issuer document delivery]

**Risks & Constraints**
⚠️ DRAFT — ZIP contents, generation
schedule to be completed from
Copilot CLI analysis

**Planned changes**
⚠️ DRAFT — to be confirmed

---

#### 5.3.5 NYCE File Generation Processor
**Owner:** Integration Team
**Status:** 🔴 Planned
**Repository:** ⚠️ Not yet created

---

**What it does**
Will generate outbound files for PIN
Networks (NYCE) and place them in
S3 /outbound/pinNetworks/ for ControlM
to transfer to Sterling → DM Mainframe
→ PIN Networks.

**Boundaries**

Inputs:
- Will read from file_notifications table

Outputs:
- PIN Networks file →
  S3 /outbound/pinNetworks/

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD during design]

**Risks & Constraints**
- Not yet designed or implemented

**Planned changes**
- Full design and implementation
  pending
- Timeline: TBD

---

## PART 6 — ACQUIRING PLATFORM INTEGRATION

---

### 6.1 NAP Platform Integration
**Status:** ✅ Production (current path)
           ⚠️ Migration planned

---

**Integration summary**

| Aspect | Current | Planned |
|---|---|---|
| Inbound | API Push via WPG/DPS | Common chbk_outbox_row path |
| Enrichment | Pre-enriched by NAP-DPS | No change needed |
| Outbound | NAP Outcome Processor → direct API | EDIA route |
| PAN | No full PAN in NAP data | No change |

**CB911 migration context:**
WDP is migrating NAP merchants from
CB911 portal to WDP. During migration:
- srv116 — migrated merchants
- srv117 — non-migrated merchants
  (data consistency)
- vin-loss — non-migrated merchants
  (data consistency)

**Key architectural decisions**
- [DEC-PLACEHOLDER: NAP as API Push
  model — only acquiring platform
  that pushes to WDP]
- [DEC-PLACEHOLDER: NAP migration to
  common path — planned]

**Risks & Constraints**
- Most complex integration — special
  cases in both inbound and outbound
- CB911 migration timeline affects
  srv117 and vin-loss processing

**Planned changes**
- Inbound: migrate to chbk_outbox_row
  → CaseCreationConsumer
- Outbound: migrate to EDIA route
- Timeline: TBD

---

### 6.2 CORE Platform Integration
**Status:** ✅ Production

---

**Integration summary**

| Aspect | Approach |
|---|---|
| Inbound | Visa/MasterCard batch + file-only networks |
| Enrichment | MerchantTransactionService |
| Outbound | CoreNotificationConsumer → DB2 |
| PAN | Standard HPAN/EPAN strategy |

**Key architectural decisions**
- [DEC-PLACEHOLDER: Direct DB2
  connection for CORE — WDP owned]
- [DEC-PLACEHOLDER: MerchantTransaction
  Service for CORE enrichment]

**Planned changes**
- Potential EDIA migration
  (future consideration)

---

### 6.3 LATAM Platform Integration
**Status:** 🔄 In Progress

---

**Integration summary**

| Aspect | Approach |
|---|---|
| Inbound | Visa/MasterCard batch + LATAM regional files via SFTP |
| Enrichment | CaseCreationConsumer → LATAM API direct |
| Outbound | EDIA Consumer → EDIA platform 🔴 |
| PAN | Standard HPAN/EPAN strategy |

**Key architectural decisions**
- [DEC-PLACEHOLDER: LATAM regional
  files via SFTP in addition to
  standard Visa/MasterCard batch]

**Risks & Constraints**
- Not yet in production
- EDIA outbound not yet built

**Planned changes**
- Full integration in progress
- Timeline: TBD

---

### 6.4 VAP Platform Integration
**Status:** 🔄 In Progress

---

**Integration summary**

| Aspect | Approach |
|---|---|
| Inbound | Visa/MasterCard batch + file-only networks |
| Enrichment | CaseCreationConsumer → VAP API direct |
| Outbound | EDIA Consumer → EDIA platform 🔴 |
| PAN | Standard HPAN/EPAN strategy |

**Key architectural decisions**
- [DEC-PLACEHOLDER: TBD during design]

**Risks & Constraints**
- Not yet in production
- EDIA outbound not yet built

**Planned changes**
- Full integration in progress
- Timeline: TBD

---

### 6.5 EDIA Platform
**Status:** 🔴 Planned

---

**What it is**
Enterprise-level Kafka streaming platform
provided across all teams. Strategic
direction for all system-to-system
communication from WDP to external
acquiring platforms. Replaces REST calls
with event-driven integration.

**Not owned by WDP team.**
WDP owns EDIA Consumer (the translator)
but not the EDIA platform itself.

**Integration pattern:**
```
NotificationOrchestrator
→ external-request-events (WDP Kafka)
→ EDIA Consumer (WDP owned translator)
→ EDIA events topic (EDIA platform)
→ NAP / LATAM / VAP (consuming platforms)
```

**Key architectural decisions**
- [DEC-PLACEHOLDER: EDIA as strategic
  enterprise integration standard]
- [DEC-PLACEHOLDER: WDP owns format
  translation via EDIA Consumer]

**Planned changes**
- EDIA Consumer build: TBD
- NAP migration to EDIA: TBD
- LATAM via EDIA: TBD
- VAP via EDIA: TBD

---

## PART 7 — UI LAYER

---

### 7.1 WDP Merchant Portal
**Owner:** UI Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Merchant-facing UI. Allows merchants to
view disputes, take actions, submit
evidence, and manage their organisation
and users. Traffic routes through Akamai
for CDN and edge security.

**Boundaries**

Inputs:
- Merchant user interactions
- Via Akamai → API Gateway

Outputs:
- API calls to WDP Core services
  via API Gateway

**UI sections available:**
- Disputes ✅
- User Management ✅
- Org Management ✅
- Dashboard 🔴 Planned

**Key architectural decisions**
- [DEC-PLACEHOLDER: Merchant Portal
  routes through Akamai — Ops Portal
  does not]

**Risks & Constraints**
⚠️ DRAFT — technology stack, browser
support, mobile support to be
completed from Copilot CLI analysis

**Planned changes**
- Dashboard Section 🔴 Planned

---

### 7.2 WDP Ops Portal
**Owner:** UI Team
**Status:** ✅ Production | 📝 DRAFT
**Repository:** ⚠️ To be confirmed

---

**What it does**
Internal operations-facing UI. Allows
WDP operations teams to manage disputes,
configure queues, manage users and
organisations. Connects directly to
API Gateway — does not route through
Akamai.

**Boundaries**

Inputs:
- Ops user interactions
- Direct → API Gateway
  (no Akamai)

Outputs:
- API calls to WDP Core services
  via API Gateway

**UI sections available:**
- Disputes ✅
- Queues ✅ (Ops only)
- User Management ✅
- Org Management ✅
- Dashboard 🔴 Planned

**Key architectural decisions**
- [DEC-PLACEHOLDER: Ops Portal direct
  to API Gateway — no Akamai needed
  for internal traffic]

**Risks & Constraints**
⚠️ DRAFT — to be completed from
Copilot CLI analysis

**Planned changes**
- Dashboard Section 🔴 Planned

---

### 7.3 Disputes Section
**Owner:** UI Team
**Status:** ✅ Production | 📝 DRAFT

---

**What it does**
Primary working area for both merchants
and operations teams. Provides dispute
search, dispute detail view, and all
dispute actions including accept, contest,
evidence submission, and notes.

**Key services called:**
- CaseSearchService (search)
- DisputeService (dispute lifecycle)
- AcceptService (accept action)
- ContestService (contest action)
- NotesService (notes)
- QuestionnaireService (questionnaire)
- DocumentManagementService (evidence)

⚠️ DRAFT — detailed UI flows to be
completed from Copilot CLI analysis

---

### 7.4 Queues Section
**Owner:** UI Team
**Status:** ✅ Production | 📝 DRAFT
**Available to:** Ops Portal only

---

**What it does**
Queue-based workload management for
operations teams. Cases routed to queues
based on configurable skill-based routing
rules. Operators work through disputes
assigned to their queues.

Includes Fax Queue functionality for
ops teams to act on merchant faxes.

⚠️ Detailed fax functionality to be
documented as a dedicated sub-section

**Key services called:**
- FaxQueueService
- UserQueueSkillService
- CaseActionService

⚠️ DRAFT — to be completed from
Copilot CLI analysis

---

### 7.5 User Management Section
**Owner:** UI Team
**Status:** ✅ Production | 📝 DRAFT

---

**What it does**
Allows administrators to manage users
within their organisation — creating,
modifying, deactivating users, managing
roles and permissions.

**Key services called:**
- UserAccessManagementService
- OrgManagementService

⚠️ DRAFT — to be completed from
Copilot CLI analysis

---

### 7.6 Org Management Section
**Owner:** UI Team
**Status:** ✅ Production | 📝 DRAFT

---

**What it does**
Allows administrators to manage
organisational hierarchies, configure
merchant accounts, and manage org-level
settings and routing rules.

**Key services called:**
- OrgManagementService
- CoreHierarchyAuthorizationService
- RulesService

⚠️ DRAFT — to be completed from
Copilot CLI analysis

---

### 7.7 Dashboard Section
**Owner:** UI Team
**Status:** 🔴 Planned

---

**What it does**
Will provide dispute analytics and
insights to merchants and operations
teams. Not yet developed.

**Key services needed:**
⚠️ To be defined during design

**Planned changes**
- Full design and implementation
  pending
- Timeline: TBD

---

## Appendix — Copilot CLI Question Sets

This appendix contains the standard
question sets to ask GitHub Copilot CLI
for each component type. Use these
when completing DRAFT sections above.

---

### For REST Microservices

```
1. "What is the primary responsibility
   of this service? Summarise in
   2-3 sentences."

2. "What REST endpoints does this
   service expose? List path, method,
   and purpose for each."

3. "What database tables does this
   service read from and write to?"

4. "What external services or APIs
   does this service call? List each
   dependency and purpose."

5. "How does this service handle
   failures — retries, fallbacks,
   error responses?"

6. "What are the key configuration
   parameters — timeouts, pool sizes,
   cache TTLs?"

7. "How many pods does this service
   run and how does it scale?"
```

---

### For Kafka Consumers

```
1. "What Kafka topic(s) does this
   consumer listen to and what is
   the consumer group ID?"

2. "What is the primary responsibility
   of this consumer? Summarise in
   2-3 sentences."

3. "What does this consumer produce
   or write as output — Kafka topics,
   database tables, API calls?"

4. "How does this consumer handle
   processing failures — retry,
   dead letter, outbox table?"

5. "What is the offset commit strategy
   — auto or manual? When is the
   offset committed?"

6. "What is the concurrency
   configuration — thread count,
   max poll records, partition
   assignment?"

7. "Does this consumer maintain
   an outbox table for idempotency?
   If so what table and what columns?"
```

---

### For Batch Jobs

```
1. "What is the schedule or trigger
   for this batch job?"

2. "What is the primary responsibility
   of this job? Summarise in
   2-3 sentences."

3. "What external APIs or systems
   does this job call?"

4. "What does this job write as output
   — database tables, files, events?"

5. "How does this job handle failures
   — retry, alerting, partial failure?"

6. "What is the expected volume —
   records per run, frequency,
   peak load?"

7. "What are the key configuration
   parameters — batch size, timeout,
   thread pool?"
```

---

### For File Processing Components

```
1. "What triggers this component —
   SQS event, schedule, DB polling?"

2. "What file formats does this
   component read or generate?"

3. "What S3 paths does this component
   read from and write to?"

4. "What database tables does this
   component read and write?"

5. "How does this component handle
   file processing failures — retry,
   error tracking, alerting?"

6. "What is the expected file volume
   and size range?"

7. "Does this component have any
   idempotency mechanism to handle
   duplicate file delivery?"
```

---

### For UI Components

```
1. "What is the primary purpose of
   this UI section or portal?"

2. "What backend services does this
   UI call and for what purpose?"

3. "What is the technology stack —
   framework, state management,
   build tool?"

4. "How does authentication work —
   token storage, refresh, expiry?"

5. "What are the key user flows
   and what services do they invoke?"

6. "Are there any significant
   performance constraints or
   known bottlenecks?"

7. "What browser and device support
   is required?"
```

---

*This document is built iteratively.
Sections marked 📝 DRAFT are pending
Copilot CLI analysis.
Sections marked ✅ COMPLETE have been
confirmed by the architecture team.*

*Last updated: April 2026*
*Next component to document: 1.2 UserAccessManagementService*
