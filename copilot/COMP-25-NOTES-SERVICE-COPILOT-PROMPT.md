# Copilot CLI Prompt — COMP-25 NotesService Source-Verification Pass

You are auditing the WDP component **NotesService** in the repository **mdvs-gcp-notes-service**. A prior pass already produced a DRAFT architecture document. Your job is to CONFIRM OR CORRECT the items listed under "ALREADY DOCUMENTED" from source, and to CLOSE the items listed under "GAPS TO FILL" by reading the code and config directly.

Produce a single structured research report organised under the section headings below. For every claim, cite the file path and line range you read it from. Where source does not show the answer, state that explicitly and label it "not determinable from source". Do NOT guess. Do NOT summarise without evidence. Do NOT use the existing document as a substitute for source — treat it as a hypothesis to verify.

---

## REFERENCE DOCUMENT

The existing component file `WDP-COMP-25-NOTES-SERVICE.md` is provided alongside this prompt. **Treat it as HYPOTHESES to verify, not ground truth.**

For each documented claim:
- Confirm with `file:Lstart-Lend`, OR
- Write a CORRECTION block in this exact form:
  - **What doc says:** `<verbatim claim>`
  - **What source shows:** `<corrected fact + file:Lstart-Lend>`
  - **Section in doc to update:** `<which heading>`

At the end of your report, in Section 10, list every documented claim that could NOT be verified from source.

---

## ALREADY DOCUMENTED — confirm or correct, do NOT re-document

For each item below, you must either:
- (a) write `Still accurate — <file>:<lines>`, OR
- (b) write a CORRECTION block as above with the actual value cited to file:lines.

Do **NOT** re-document any item in this list from scratch.

### Identity & framing
- Repository name: `mdvs-gcp-notes-service`
- Build artifact: `notes-service`, group `com.wp.gcp`, version `1.4.5`
- Runtime: Java 17 / Spring Boot 3.x
- Server context path: `/merchant/gcp/notes`, injected via `SERVER_SERVLET_CONTEXT_PATH`
- Component type: REST API + Kafka Producer — has NO Kafka consumer, NO scheduler, NO batch job

### Endpoints
- Two REST endpoints exist, both rooted at `/{platform}/case/{caseNumber}`:
  - `POST /{platform}/case/{caseNumber}` — add notes (request body is JSON array of `AddNotesRequest`)
  - `GET /{platform}/case/{caseNumber}` — search notes (query params: optional `noteType`, optional `actionSequence`)
- Probe endpoints: `/merchant/gcp/notes/live`, `/merchant/gcp/notes/ready`, `/merchant/gcp/notes/actuator/health` — all unauthenticated
- Swagger UI / OpenAPI JSON suppressed when `ENV_NAME = "prod"`

### Authentication & authorisation
- All non-probe endpoints require Bearer JWT
- Multi-issuer validation via `JwtIssuerAuthenticationManagerResolver`, trusted issuers from `${jwt_trusted_issuer_urls}`
- No `@PreAuthorize` / `@Secured` / role-based gating at the service or controller layer
- `displayUserId` is derived from the JWT issuer claim via `UserIdUtil` — used for column population only, not for access control

### Validation rules (POST)
- `platform` path variable must match enum: `NAP`, `VAP`, `LATAM`, `CORE`, `PIN` (case-insensitive)
- `actionSequence` body field: `@Pattern("[0-9]*$")`, `@Range(min=01, max=99)`, AND single-digit values 1–9 are additionally rejected by `RequestValidator.validateActionSequence`
- `noteType` must match `NoteType` enum (codes listed in v1.0 file)
- `text` is `@NotBlank`, `@Size(max=1000)`
- `userId` is `@NotBlank`

### Processing flow — POST
- Step sequence: HTTP interceptor → JWT validation → Controller entry → Action Service GET → platform branch (NAP vs WDP) → entity build → @Transactional DB write → Kafka publish (non-SNOTE only) → 200 OK
- DB write and Kafka publish share the same `@Transactional` boundary
- Action Service is called BEFORE the DB write, to validate that the case and action sequence exist
- `actionSequence` resolution: from request OR computed from max action sequence in the action summary
- The Kafka publish loop iterates per-note; only notes whose `noteType` does not start with `SNOTE` produce a publish

### Processing flow — GET
- Step sequence: HTTP interceptor → JWT validation → Controller entry → DAO query against the platform-matching schema → if non-empty result, Display-Code Service POST to enrich `noteType` → response mapping → 200 OK
- 404 returned when result list is empty
- Display-Code Service call is `@Cacheable("displayCodes")` — in-memory, no TTL, cache key is the full request object, lifetime is the JVM

### Kafka producer
- Framework: Spring Kafka `KafkaTemplate`
- Idempotent producer enabled (`ENABLE_IDEMPOTENCE_CONFIG = true`)
- Acks: `all`
- Max in-flight requests: 5
- Auth: SASL_SSL with AWS MSK IAM (`aws-msk-iam-auth` v2.3.2)
- Sync publish via `kafkaService.retryKafkaCallWithRecovery`
- Topic: `${kafka_business_event_topic}` — literal name not in source
- Message key: `caseNumber` (DEC-003 deviation)
- `SNOTE`-prefixed notes are written to DB but never trigger a Kafka publish

### Database
- Two schemas, two HikariCP pools, two transaction managers: `napTransactionManager` (NAP) and `wdpTransactionManager` (WDP for VAP/LATAM/CORE/PIN)
- Tables written: `nap.NOTES` (NAP path), `wdp.NOTES` (WDP path)
- Tables read: same two tables on the GET path
- No other tables touched by this service
- DDL is not in this repo (sequence `wdp.NOTES_I_NOTE_ID_SEQUENCE` referenced)

### Error handling
- Bespoke error body: `StandardErrorResponse` wrapping `List<StandardDisplayError>` with `errorMessage` and `target` fields
- Action Service `RestClientException` → 500
- Display-Code Service `RestClientException` → 500 (cached after first success per JVM)
- IDP token failure → 500
- Kafka send failure → `isErrorOccurred=true` → `InternalServerError` thrown inside `@Transactional` → DB rollback → 500

### Outbound dependencies
- Action Service: `${gcp_env_url}/case-actions/{platform}/case/{caseNumber}/actions` — REST GET, Bearer JWT via `TokenServiceImpl`
- Display-Code Service: `${gcp_display_code_env_url}` — REST POST, Bearer JWT via `TokenServiceImpl`
- IDP / OAuth2: `${idp_token_url}` — OAuth2 client credentials
- AWS MSK Kafka: `${kafka_business_event_topic}`

### Resilience posture
- No `resilience4j` dependency in `pom.xml`
- No `@Retryable` / `@CircuitBreaker` / `@Bulkhead` / `@TimeLimiter` anywhere
- No connection or read timeout configured on the `RestTemplate` bean
- Kafka producer-level retries via `${kafka_retry_count}` env var; no application-level Spring Retry around the Kafka call

### Deployment & observability
- Kubernetes resource type: `Deployment`
- Replica count: `{{ replicas-mdvs-gcp-notes-service }}` — XL Deploy / Helm placeholder, exact value not in repo
- Memory request 1024Mi, Memory limit 2048Mi
- CPU request and CPU limit: NOT configured
- HPA: absent
- PodDisruptionBudget: absent
- Rolling update: `maxSurge: 1, maxUnavailable: 0, minReadySeconds: 30`
- Topology spread: `ScheduleAnyway`, `topologyKey: kubernetes.io/hostname`, label `app: mdvs-gcp-notes-service${BRANCH_NAME_PLACEHOLDER}` — no label mismatch
- OTel agent: pod annotation `instrumentation.opentelemetry.io/inject-java`
- Actuator: `info`, `health`, `prometheus` exposed
- Logstash: `LogstashTcpSocketAppender` → `${logstash_server_host_port}`, JSON via `LogstashEncoder`

### Known dead / commented code
- Dead code: `validateCaseNumber`, `createErrorResponseEntity`, `createDuplicateResponseEntity`
- Commented-out Logstash destinations in `logback-spring.xml` — legacy dev/test config

### DEC deviations & compliance posture
- DEC-001 (transactional outbox): DEVIATES — direct sync publish, no outbox
- DEC-003 (Kafka partition key = merchantId): DEVIATES — uses `caseNumber`
- DEC-004 (PAN encryption): COMPLIANT — no PAN data path exists
- DEC-005 (manual offset commit after processing): NOT APPLICABLE — no consumer
- DEC-014 (Resilience4j): DEVIATES — not present
- DEC-019 (no clear PAN persisted): COMPLIANT — no PAN handling
- DEC-020 (full at-least-once): DEVIATES — `idempotency-key` forwarded to Kafka header but never checked against any store; duplicate POSTs create duplicate rows

---

## GAPS TO FILL — answer from source

For each item below, answer the sub-questions with file:line citations.

### G1 — Action Service caller chain detail
- What is the exact HTTP method and full URL template for the Action Service call?
- What is the response model class and which fields drive `actionSequence` validation?
- For each non-200 response shape (4xx, 404, 5xx, empty body, malformed body) — what is the precise behaviour? Is each lumped into "RestClientException → 500" or are some handled distinctly?
- Is there ANY conditional that skips the Action Service call (e.g. specific noteType)?

### G2 — Idempotency mechanism explicit confirmation
- Confirm there is no DB-level unique constraint visible in JPA entity annotations on `nap.NOTES` or `wdp.NOTES` (search for `@UniqueConstraint`, `@Index(unique=true)`).
- Confirm there is no application-side dedup (search for any use of the `idempotency-key` header beyond forwarding it to the Kafka message header).
- Confirm there is no `@Version` / optimistic lock / pessimistic lock on either notes entity.
- State explicitly: "Replaying the same POST with the same `idempotency-key` produces N additional rows" — confirm or correct.

### G3 — Multi-note batch atomicity
- The POST body is a JSON array. When N notes are submitted and one of them is `SNOTE` and others are non-`SNOTE`:
  - What is the precise commit order? Is `saveAll` one INSERT batch or N separate INSERTs within the transaction?
  - If Kafka publish fails on the 3rd of 4 non-SNOTE notes, does the entire batch roll back, or only that note? Cite the loop and the exception handling.
  - Are SNOTE rows committed even if a later non-SNOTE Kafka publish fails?

### G4 — Whole-service entry-path negative confirmation
- Confirm the service exposes EXACTLY: 2 REST endpoints + 3 actuator/probe endpoints + 2 non-prod swagger endpoints, and NOTHING else.
- Specifically confirm: no `@KafkaListener`, no `@Scheduled`, no `@JmsListener`, no SQS listener, no webhook receiver, no admin/internal POST endpoint, no second controller class with extra paths.
- File:line for each negative — show the package scan covers controllers and that no other `@RestController` / `@Component`-with-listener exists.

### G5 — RestTemplate & timeout configuration
- Is there a single shared `RestTemplate` bean or multiple? File:line for the bean definition.
- Confirm absence of `setConnectTimeout`, `setReadTimeout`, custom `ClientHttpRequestFactory`, custom error handler, custom interceptor.
- If a `RestTemplateBuilder` is used, list every method called on it. Confirm timeouts are NOT injected via builder.

### G6 — Kafka producer error semantics
- For each of these failure modes, what is the application's behaviour?
  - AWS MSK IAM auth failure (token invalid / expired)
  - Broker unavailable / connection refused
  - Serialiser exception (payload malformed)
  - Send timeout exceeded
- Confirm the path through `kafkaService.retryKafkaCallWithRecovery` — does it differentiate, or treat all as the same `Exception`?
- Cite the `@Recover` method (if any) and what it returns.

### G7 — `AddNotesBREvent` payload schema
- Locate the event POJO. List every field with type and whether nullable.
- Cite the entity-to-event mapping code that populates each field.
- Confirm the `idempotencyKey` field is on the message body OR on the message header, not both. Cite the producer call site that adds Kafka headers.

### G8 — JWT issuer-based `displayUserId` derivation
- Locate `UserIdUtil` (or equivalent).
- List every JWT issuer URL the code matches against, and the resulting display user ID prefix/string for each.
- Confirm no JWT scope, role, or audience claim is consulted.

### G9 — `actionSequence` resolution rule
- In `NotesServiceSupport.saveFinDetails` (or the equivalent), trace the resolution logic:
  - When `actionSequence` IS present in the request — is it matched against the action summary? What happens on mismatch?
  - When `actionSequence` IS NOT present — is it computed from `Math.max` over the action summary? What happens when the action summary is empty? When it has only one entry?
- Cite each branch.

### G10 — Cross-write awareness
- Confirm there is NOTHING in this repo that signals awareness of other services writing to `nap.NOTES` or `wdp.NOTES`. Specifically:
  - No DB-level unique constraint declared in JPA
  - No advisory lock acquired
  - No `@Version` / optimistic lock
  - No co-ordination annotation, README note, or comment referencing COMP-23 / COMP-24 / "case-management" / "case-actions"
- This is a NEGATIVE confirmation pass — the answer is expected to be "no awareness present, file:lines confirm absence."

### G11 — Probe configuration completeness
- Cite the actuator group config for `live` and `ready`.
- Confirm the Spring Security config explicitly permits the actuator paths unauthenticated.
- Confirm whether the `prometheus` actuator endpoint is exposed AND unauthenticated, or exposed but auth-required.

### G12 — `spring-boot-devtools` presence
- Search `pom.xml` for `spring-boot-devtools`.
- If present, what is the `<scope>` (or absence thereof)?
- If present and shipped to prod, flag this as the same pattern audit found in COMP-23.

### G13 — K8s manifest scope in repo
- List every K8s manifest, Helm chart, values file, Kustomize overlay, or `resources.yaml` present in the repo.
- For each, summarise its content scope (Deployment? Service? Ingress? HPA? PDB?).
- If no K8s manifest is in repo, state that and identify what IS in the repo (Dockerfile, application.yaml, etc.).

### G14 — Kafka producer config completeness
- Cite the `ProducerConfig` map / `KafkaProducerFactory` configuration.
- Confirm: `acks`, `enable.idempotence`, `max.in.flight.requests.per.connection`, `retries`, `delivery.timeout.ms`, `request.timeout.ms`, `linger.ms`, `batch.size`, key/value serialisers.
- Confirm whether the `retries` count is hardcoded, env-injected (`${kafka_retry_count}`), or both — and whether the env value has a default if unset.

### G15 — Downstream-consumer references
- Search the repo (README, docs/, comments) for any reference to consumers of `${kafka_business_event_topic}` or `AddNotesBREvent`. Likely "not determinable from source" — confirm and state explicitly.

---

## REPORT STRUCTURE — use these exact section headings, in order

### Section 1 — PROCESSING FLOW

#### a) STEPS IN ORDER
For each of the two endpoints (POST add, GET search), list every processing step from trigger to final output. For each step: what happens, which service / table is called, what the output of that step is. Cite file:lines.

#### b) DECISION POINTS
For each endpoint, every branch in the flow — the condition evaluated and every possible outcome with the path it leads to. Cite file:lines.

#### c) SKIP AND BYPASS CONDITIONS
- The `SNOTE` Kafka skip rule
- Any other condition that short-circuits processing (early return, exception caught and turned into a no-op)

#### d) FAILURE PATH PER STEP
For each step that calls an external dependency (Action Service, Display-Code Service, IDP, Kafka, DB), state per step:
- Exception type caught
- Number of retries (application-level vs Kafka-producer-level)
- Post-exhaustion behaviour
Do NOT generalise across steps.

#### e) MULTIPLE ENTRY PATHS
Confirm exactly two REST entry paths (POST, GET). State explicitly:
- No `@KafkaListener`
- No `@Scheduled`
- No SQS listener
- No webhook receiver
- No admin / internal endpoint beyond the two documented

State whether the two endpoints share any service-layer code path or are fully independent.

#### f) STATE AT EACH STEP
For the POST path, describe what is persisted to DB vs Kafka at each step, and what is in-memory only.

### Section 2 — FUNCTIONAL BEHAVIOUR

#### a) CLASSIFICATION AND ROUTING LOGIC
- The platform path-variable branch (NAP vs WDP)
- The `noteType` `SNOTE` skip branch
- The JWT-issuer-based `displayUserId` derivation

#### b) BUSINESS RULES APPLIED
- `actionSequence` validation against action summary
- `actionSequence` zero-padded two-digit single-digit rejection
- `noteType` enum membership
- `text` size and blank checks

#### c) DATA TRANSFORMATIONS
- Inbound `AddNotesRequest` → `NAPNotesEntity` / `USNotesEntity` mapping
- Outbound entity → `AddNotesBREvent` mapping
- Outbound entity → `GetNotesResponse` mapping (GET path)

#### d) IDEMPOTENCY MECHANISM
- Detection key fields (none — the `idempotency-key` header is forwarded but not checked)
- Duplicate behaviour (duplicate rows + duplicate Kafka events)
- Gaps — explicit list: no DB unique constraint, no application dedup, no replica-race protection, JVM-crash window between DB commit and Kafka send produces lost event

### Section 3 — CONTRACTS

#### REST API
For each of the two endpoints:
- Method, path, request fields with validation, response fields
- Every HTTP status code and what triggers it
- Auth model
- Known callers from any source code reference (likely Merchant Portal + Ops Portal — but ONLY if a comment, README, or config shows it; otherwise state "not determinable from source")
- Error body structure (`StandardErrorResponse` schema)

#### Kafka Producer
- Topic name (config key), message key, sync/async, idempotence setting
- Publish trigger (when does the call happen)
- Failure behaviour
- Full message payload schema for `AddNotesBREvent`

State explicitly: "Not present — no Kafka consumer", "Not present — no batch / scheduler".

### Section 4 — DEPENDENCIES

For every outbound call: Action Service, Display-Code Service, IDP, Kafka, `nap.NOTES`, `wdp.NOTES`:
- Logical target name
- Protocol and auth method
- Purpose — at which step in the flow
- Connection timeout (state "not configured" if absent)
- Read timeout (state "not configured" if absent)
- Retry: count, delay, backoff type
- Circuit breaker: Resilience4j configured? If yes — failure rate threshold, wait duration, half-open permitted calls. If no — state explicitly "not configured".
- Behaviour if unavailable: halt / skip / degrade / item lost

### Section 5 — DATABASE

For `nap.NOTES` and `wdp.NOTES`:
- Schema and exact table name
- Read or write — state clearly per endpoint
- Purpose — at which step in the flow
- Key columns used in queries or writes
- Whether writes are in the same transaction as Kafka publish
- Transaction manager bean used (`napTransactionManager` vs `wdpTransactionManager`)
- Whether `SELECT FOR UPDATE` / row locks used

Confirm explicitly: any tables NOT touched that the existing doc might have implied — e.g. no `nap.case`, no `wdp.case`, no `nap.ACTION`, no `wdp.ACTION`, no `wdp.outgoing_event_outbox`, no `wdp.chbk_outbox_row`, no error tables.

### Section 6 — PLATFORM STANDARD DEVIATIONS

For each, output `COMPLIES` / `DEVIATES` / `NOT APPLICABLE` with evidence cited to file:lines.

- **DEC-001:** Transactional outbox — used? Which table? Outbox write in same transaction as business write?
- **DEC-003:** Kafka partition key = `merchantId`? If not, what is it and why?
- **DEC-004:** PAN encryption — encrypted before any persistent write? Clear PAN ever written? Which service handles encryption?
- **DEC-005:** Kafka offset — committed manually after all processing? Or auto / pre-ACK? Exact timing relative to each write.
- **DEC-019:** Clear PAN — does this component write clear PAN to any persistent store?
- **DEC-020:** Idempotency — full at-least-once implemented? If not, what gap exists?

### Section 7 — SCALING AND DEPLOYMENT

- Kubernetes resource type
- Replica count — exact value or placeholder, state which and which variable
- Memory and CPU limits and requests
- HPA — present or absent (cite the manifest)
- Rolling update strategy
- PodDisruptionBudget — present or absent
- Topology spread — functional or label mismatch
- Liveness / readiness / startup probes
- Ports exposed
- Observability — OTel agent, Actuator, Logstash, Prometheus
- `spring-boot-devtools` presence in `pom.xml` (and shipping to prod risk)

If the K8s manifest, Helm chart, or values.yaml is NOT in the repo, state "not in repo" for this entire section and explicitly list what IS in the repo (Dockerfile, application.yaml, etc.) so I know what is missing vs absent.

### Section 8 — PLANNED AND INCOMPLETE WORK

- Commented-out code — what it did and why (`validateCaseNumber`, `createErrorResponseEntity`, `createDuplicateResponseEntity`, commented Logstash destinations — confirm and look for any others)
- Unused dependencies in `pom.xml`
- Feature flags active in production — exact flag names, behaviour, default value if env var unset
- TODO / FIXME references — file:line and verbatim text
- Stub implementations returning fixed responses
- Properties configured in `application.yaml` but never read by `@Value` or `@ConfigurationProperties`

### Section 9 — CONFIDENCE ASSESSMENT

For each of sections 1 through 8 above, state:
- Confidence: High / Medium / Low
- Basis: direct source reading / inferred from pattern / not determinable from source alone

### Section 10 — ITEMS NOT VERIFIABLE FROM SOURCE

List every claim from the existing component document that you could NOT cross-check against code or config — neither confirmed nor contradicted. Format:
- **Claim:** `<verbatim from doc>`
- **Reason not verifiable:** `<runtime-only / env-config / cross-repo / DBA / etc.>`

Also list any file paths referenced in the existing doc that you could NOT locate in the repo, under a sub-heading **REFERENCED BUT NOT FOUND IN REPO**.

---

## OUTPUT RULES

- Cite file paths and line ranges for every concrete claim. Format: `<relative/path>:Lstart-Lend`
- State "not determinable from source" for any item the code does not reveal — do not guess.
- Use plain Markdown. No code blocks except when quoting a short annotation, config value, or DDL fragment.
- No narrative preamble. Start at Section 1.
- Target correctness over completeness. If a section confirms existing documentation, a one-paragraph "still accurate — file:lines" is acceptable.
- If you would restate something the reference document already states verbatim, replace that prose with `Document accurate — <file>:<lines>`. Do not restate.
- Do not invent file paths. If a file is referenced in the existing doc but you cannot locate it, list it under "REFERENCED BUT NOT FOUND IN REPO" at the end of Section 10.
- For items in the ALREADY DOCUMENTED block, do NOT re-document from scratch. Confirm with `Still accurate — <file>:<lines>` or write a CORRECTION block.
- For items in the GAPS TO FILL block, give a complete answer to every sub-question.
