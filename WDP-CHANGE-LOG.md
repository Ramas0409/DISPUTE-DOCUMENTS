# WDP-CHANGE-LOG.md
**Worldpay Dispute Platform — Platform-Level Change Log**
*Started: April 2026*

---

## Purpose

This log captures the **platform-level impact** of every component update. It is
written by per-component chats at the end of Phase 2 and consumed by
**reconciliation sessions** that rebuild derivative documents (WDP-DB.md,
WDP-KAFKA.md, WDP-HANDOVER.md, and others) from the current state of the
component files.

### What belongs here
- Which derivative documents are affected by a component change
- Which rows, sections, or entries inside those documents need to change
- Deviation flag status for each component update
- Candidate new ADRs surfaced by the component update

### What does NOT belong here
- Full component file contents (those live in `WDP-COMP-NN-*.md`)
- Rewritten derivative documents inline (those are produced in reconciliation
  sessions, not here)
- Implementation details, code, or config

---

## How to use this file

### In a per-component chat (Phase 2)
At the end of Phase 2, Claude appends a new block under **Pending Entries**
using the template at the bottom of this file. Nothing above the Pending
Entries header is modified.

### In a reconciliation session
1. Read this file and note the current **Reconciliation Marker**
2. Consume all entries under **Pending Entries**
3. Rebuild the affected derivatives
4. Move the consumed entries from **Pending Entries** to **Reconciled**
5. Update the **Reconciliation Marker** with the new date and the derivatives
   that were rebuilt

---

## Reconciliation Markers

*Each marker records one reconciliation session — what was rebuilt, when, and
against which set of Pending Entries.*

| Date | Derivatives rebuilt | Entries consumed | Notes |
|------|---------------------|------------------|-------|
| *(none yet)* | — | — | Awaiting first reconciliation session |

---

## Pending Entries

*New entries are appended here by per-component chats. Reconciliation sessions
consume entries from this section and move them to **Reconciled** below.*

### 2026-04-23 — COMP-20 ContestService · v1.0 DRAFT → v2.0 DRAFT

**Source:** `gcp-disputes-contest-service` (dispute-contest-service v1.6.6) —
source-verified by Claude Code 2026-04-23. Architect confirmation pending.

**Nature of change:** Correction pass against source plus headline new
content — full Visa RTSI submission contracts documented per contest stage
(five submission methods, stage→questionnaire mapping, field-level payload
tables) and full MCM submission contract documented (CH1/RE2 chargeback and
PAB updateCase paths). Six corrections to v1.0 DRAFT, eight new
production-risk findings, one headline split-state risk on the CRMR
pre-publish pattern surfaced during audit.

#### Corrections to v1.0 DRAFT
1. **Max Kafka publishes per request is 2, not 3.** CRMR branches are
   `if / else if` — mutually exclusive.
2. **Card-network failures return HTTP 400, not HTTP 500.** All MC and
   Visa exceptions are wrapped in `BadRequestException` by the outer
   catch. Only Kafka publish exhaustion and internal RuntimeException
   paths return HTTP 500.
3. **Liveness path is `/livez`, not `/lives`.**
4. **ErrorLogService commented-out call sites: 8, not 6.**
5. **CAD/USD commented-out currency blocks: 5, not 1.** Across all five
   Visa submission methods.
6. **DEC-014 reclassified.** Absence-of-library fact, not a deviation —
   consistent with platform-voided DEC-014.

#### Headline new content
- **Visa RTSI submission contracts** fully documented per responseType
  (`allocprearb` · `allocarb` · `representment` · `prearbresp` ·
  `precomresp`) with stage→questionnaire mapping table and field-level
  payload tables for each of the five submission methods.
- **MasterCard MCM submission contract** fully documented for
  `createSecondPresentmentChargeback` (CH1/RE2) and
  `retrieveClaim + updateCase` (PAB).

#### Platform-level impacts

**WDP-DB.md**
- No change. ContestService owns no tables and reads none — confirmed
  unchanged from v1.0.

**WDP-KAFKA.md · Section 3 (Topic Registry) — `internal-integration-events`**
- Update **Notes** column for `internal-integration-events` row:
  correct "Up to 3 publishes per ContestService request when CRMR action
  code present" → **"Up to 2 publishes per ContestService request when
  CRMR action code present (CRMR pre-publish + main; CRMR branches are
  mutually exclusive)"**.
- Confirm existing row otherwise accurate: Publishers `COMP-19 + COMP-20`,
  Consumers `COMP-39 + COMP-40`, Key `merchantId`, No DLQ, No outbox.

**WDP-KAFKA.md · Section 4 (Producer/Consumer Map) — COMP-20 row**
- Correct "Up to 3 events per request" → **"Up to 2 events per request
  (CRMR pre-publish + main)"** in Notes column.
- Add: **"⚠️ Runtime topic resolution depends on env var `KAFKA_TOPICID`
  (code reads `${kafka.topicId}` but YAML default is `kafka.topic` —
  startup fails without the env var)."**

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: "COMP-20 ContestService emits **at most 2** Kafka publishes per
  request (CRMR pre-publish + main; CRMR branches mutually exclusive)."
- Add: "COMP-20 card-network failures return **HTTP 400**, not 500.
  Only Kafka publish exhaustion returns HTTP 500."
- Add: "COMP-20 calls five distinct Visa RTSI submission methods —
  `createDisputePreCompResponse` (APC), `createDisputePreArb`
  (CH1+VISA_ALLOCATION), `createDisputeResponse` (CH1+VISA_COLLABORATION),
  `submitDisputeFilingRequest` (PAB+VISA_ALLOCATION),
  `createDisputePreArbResponse` (PAB+VISA_COLLABORATION). These are
  distinct from COMP-40's five retrieval methods."
- Add: "COMP-20 MCM routes: CH1/RE2 → `createSecondPresentmentChargeback`;
  PAB → `retrieveClaim + updateCase(action=REJECT)`; other stageCode →
  HTTP 400."
- Resolved open questions: (none from v1.0 — v1.0 gaps on
  known-callers and replica count remain open, environment-config
  category).
- New open questions:
    - **OQ: Does CaseManagement `POST insertActions` always place the
      CRMR action sequence at `actionSequenceList[0]` when CRMR is the
      second action code?** COMP-20 removes index 0 unconditionally in
      both CRMR branches. Follow-up Claude Code question against the
      COMP-23 repo: *"In `mdvs-gcp-case-management-service`, search the
      `insertActions` endpoint handler. When the rules engine supplies
      a CRMR action in any slot of the insert request, does the
      response `actionSequenceList` place the CRMR sequence at index 0?
      Report the ordering logic with file:lines."*
    - **OQ: Is the Kafka topic property-path mismatch
      (`${kafka.topicId}` in code vs `kafka.topic` in YAML)
      intentional env-var-only override, or a wiring bug?** Team
      confirmation.
    - **OQ: Is the Visa PIN dispatch anomaly a bug?** URL selector
      keyed on `platform != NAP`, invoker selector keyed on
      `platform == PIN`. For CORE/VAP/LATAM the URL points at Visa
      RTSI DataPower but the invoker attaches a Bearer IDP token.
      Architect decision + team confirmation.
    - **OQ: Should MCM and Visa retry count / delay be externalised
      to environment variables, matching the Kafka retry pattern?**
      Architect decision.
    - **OQ: Should the Kafka `@Recover` blank-message-swallowing
      behaviour be corrected?** Latent reliability gap — architect
      decision.

**WDP-DECISIONS.md · Candidate new ADRs**
- **Candidate ADR — Direct Kafka Publish Pattern for Case-Level REST
  Services.** Pattern confirmed on COMP-19, COMP-20, COMP-23, COMP-37.
  Either document as an accepted platform position with a defined
  recovery procedure or record as a cross-component DEC-001 remediation
  item with a target release.
- **Candidate ADR — CRMR Pre-Publish Split-State Risk.** Two-message
  pattern on `internal-integration-events` (CRMR pre-publish +
  subsequent main publish) can produce a permanently split broker
  state if the main publish fails after the CRMR has been committed.
  Requires either an outbox remediation or documented downstream
  compensation procedure.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- Candidate new RISK rows (pending architect decision):
    - **"COMP-20 CRMR pre-publish / main publish split-state — CRMR
      event commits to broker, main publish fails after retry
      exhaustion. Downstream consumers see unpaired CRMR event. No
      retraction mechanism."** 🔴 HIGH.
    - **"COMP-20 `@Recover` swallows blank-message exceptions — Kafka
      publish failure with empty exception message returns HTTP 200
      silently with no event delivered. Latent reliability gap."**
      🟡 MEDIUM.
    - **"COMP-20 no idempotency — replayed POST re-calls card-network
      API (duplicate Visa / MC submission), inserts additional
      actions, emits duplicate Kafka events. Only guard is
      `actionStatus == CLOSED` which does not fire on mid-flight
      retry."** 🔴 HIGH (DEC-020 deviation — consider covering under
      a platform-wide risk since pattern repeats across case-level
      REST services).
    - **"COMP-20 Visa PIN dispatch anomaly — URL/invoker selector
      mismatch for CORE/VAP/LATAM. Potential auth failure on those
      platforms."** 🟡 MEDIUM.
    - **"COMP-20 MCM/Visa retry counts hardcoded — not
      env-configurable. Runtime retry tuning requires a build."**
      🟡 MEDIUM.
    - **"COMP-20 `v-correlation-id` not propagated on DataPower paths
      (MCM non-NAP and Visa RTSI PIN). Log correlation broken on those
      paths."** 🟡 MEDIUM.
    - **"COMP-20 Kafka topic property-path mismatch
      (`${kafka.topicId}` vs `kafka.topic`). Startup failure mode if
      env var not supplied by K8s secret."** 🟡 MEDIUM.
    - **"COMP-20 bare `RestTemplate` with no connection or read
      timeouts — hung downstream blocks request thread indefinitely.
      Thread-pool exhaustion risk."** 🟡 MEDIUM (may already be
      covered by an existing platform-wide RISK — suggest consolidation
      with similar findings in COMP-19 / COMP-23 / COMP-37).

**WDP-INTEGRATIONS.md · Section 3.1 (Visa RTSI API)**
- Confirm entry accurate. Add cross-reference to COMP-20 for the five
  submission methods (currently only retrieval methods via COMP-40
  are cross-referenced). Note that COMP-20 uses the submission set
  (`createDispute*`, `submitDisputeFilingRequest`) and COMP-40 uses
  the retrieval set (`getDispute*Details`).

**WDP-INTEGRATIONS.md · Section 3.2 (Mastercard MCM API)**
- Confirm existing entry accurate. Add detail: COMP-20 MCM path is
  stage-dependent — CH1/RE2 uses `createSecondPresentmentChargeback`
  (POST); PAB uses `retrieveClaim + updateCase(action=REJECT)` (GET
  then PUT). The DataPower non-NAP path uses the same VANTIV license
  header as Visa DataPower (shared `licenseKey` secret).

#### Deviation flags for COMP-20

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⛔ DEVIATES | 🔴 HIGH |
| DEC-003 Kafka Partition Key = merchantId | ✅ COMPLIES | 🟢 |
| DEC-004 PAN Encryption Before Persistence | ✅ NOT APPLICABLE | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE (producer only) | — |
| DEC-014 Resilience4j Circuit Breakers | ℹ️ ABSENT (voided platform-wide) | — |
| DEC-019 No Clear PAN in Persistent Store | ✅ NOT APPLICABLE | — |
| DEC-020 Full At-Least-Once Idempotency | ⛔ DEVIATES | 🔴 HIGH |
| DEC-023 Replica = 1 Hard Constraint | ✅ NOT APPLICABLE (stateless REST) | — |

**DEC-001 DEVIATES detail:** Direct publish inside HTTP request handler.
CaseManagement `insertActions` commit is permanent regardless of Kafka
outcome. Retry-exhaustion path writes an SNOTE but does not recover the
event. Matches platform pattern on COMP-19 / COMP-23 / COMP-37 — candidate
for platform-level ADR.

**DEC-020 DEVIATES detail:** No idempotency-key read, no `(caseNumber +
actionSequence)` dedup store. `actionStatus == CLOSED` guard does not
fire on replay of mid-flight failures (input action set to ERROR, not
CLOSED). Replay re-calls card-network API → duplicate Visa/MC submission,
additional action inserts, duplicate Kafka events. Kafka idempotent
producer guards within a single session only.

#### Doc status after this change
- `WDP-COMP-20-CONTEST-SERVICE.md` → `v2.0 DRAFT` — source-verified
  2026-04-23 · architect confirmation pending
---
---
### 2026-04-23 — COMP-19 AcceptService · v1.0 DRAFT → v2.0 DRAFT

**Source:** `mdva-gcp-disputes-accept-service` v1.4.7 — source-verified by
Claude Code 2026-04-23. Architect confirmation pending.

**Nature of change:** Correction pass against source. No functional change in
production; six corrections to the v1.0 DRAFT plus three previously-undocumented
production-risk findings surfaced during the audit. All three new findings rise
to 🔴 HIGH severity. Closes 16 of 16 enumerated gaps; introduces 10 open
questions (one inherited from v1.0 OCR limitations, three new architect
decisions required, six environment / team confirmations).

#### Corrections to v1.0 DRAFT

1. Stage code is **CHI** (letter I) throughout all three validators — not
   `CH1` as previously documented. Affects VISA, MASTERCARD/MAESTRO, and
   OTHER rows of the stage/actionCode matrix.
2. VISA PAB permits **WDNL** (in addition to IDCL/IPAB/CHGM). VISA APC uses
   **PCMP** (not POMP).
3. `expiryFlow=true` EACP override applies to the first action of the
   outgoing `AddActionRequest` **for all card networks** (not Visa-only as
   v1.0 stated). It mutates the AddActionRequest only — it does **not**
   affect the Kafka publish gate, which reads actionCode from
   `CaseLookupResponse`.
4. MC CHI silent no-op applies to **both NAP and PIN** — not PIN only.
   `MasterCardServiceImpl.accept` wraps the entire NAP+PIN flow in one
   outer `if (PAB || ARB)`. Non-PAB/ARB returns void with no log.
5. `errorLogService.saveErrorLog` — **2 of 9 call sites are ACTIVE**
   (`CaseServiceImpl` Step 6 case-action-add; `MasterCardServiceImpl`
   Step 5d MC PIN claim lookup). v1.0 incorrectly claimed all sites
   were commented. `${errorlog.save}` config is reached on those two paths.
6. Liveness/Readiness probe paths end in **`livez` / `readyz`** (with `z`),
   not `liver` / `ready` as v1.0 stated. Spring Actuator health groups
   `liveness` and `readiness` expose additional-paths matching.

#### Newly-confirmed HIGH-severity findings

1. **MC CHI on NAP can publish `AcceptEvent` without MC network
   notification** (🔴 HIGH split-brain). `MasterCardServiceImpl.accept`
   silently returns for CHI. The Step 8 Kafka gate still fires if
   inbound actionCode is `FCHG/IPAB/IARB/IDCL`. Result: NAPOutcomeProcessor
   delivers the acceptance to NAP-DPS while Mastercard was never asked.
2. **AMEX/DISCOVER on NAP can publish `AcceptEvent` without network
   notification** (🔴 HIGH split-brain). `cardNetwork` switch defaults to
   `log.warn` for AMEX/DISCOVER. Case action is added; on NAP with
   eligible actionCode, AcceptEvent is published. NAP-DPS receives an
   outcome for a network that was never notified.
3. **Compensation has no inner try/catch** (🟠 HIGH operability /
   observability). `updateAction(ERROR)` and `addErrorNotes` are called
   without defensive handling in every Step 5/6/4 catch block. A secondary
   failure in compensation propagates and **replaces** the original
   business exception in the HTTP response. Operators see a misleading
   error; the original cause is lost.

#### Other facts now pinned from source

- **Add-case-action runs AFTER the network call** (not before).
  Compensation `updateAction(ERROR)` targets the **original** action
  sequence, not a newly-added row.
- **Kafka gate actionCode** is read from `CaseLookupResponse`, not from
  the rules response and not from `AddActionResponse`. EACP override does
  not change the gate.
- All outbound REST calls use single-attempt — no `@Retryable` on any
  outbound REST. Only Kafka publish has Spring Retry.
- No correlation-id propagation: `HttpInterceptor` puts `correlation-id`
  on MDC, but no `ClientHttpRequestInterceptor`, no `RestTemplateCustomizer`,
  no `ProducerInterceptor`, no Kafka record headers. End-to-end tracing is
  broken at this hop.
- No idempotency-key generated or propagated anywhere.
- Plain `new RestTemplate()` with default `SimpleClientHttpRequestFactory` —
  no pool, no keep-alive management, no connect/read timeout.
- Kafka producer security: SASL_SSL + AWS_MSK_IAM via `IAMLoginModule` and
  `IAMClientCallbackHandler`. Key serialiser `StringSerializer`; value
  serialiser Spring `JsonSerializer`. `acks`, `linger.ms`, `batch.size`,
  `compression.type`, `retries` all left untuned.
- `validateDisputeAmount` defined but unused — `AmountCalculationUtil.amountValidation`
  commented out is what severs the call chain.
- Controller `correlationId` `@RequestHeader` parameter declared but never
  read inside the method body.
- `HttpRequestMethodNotSupportedException` annotation says 400 but runtime
  returns 405 — flagged with `// TODO` in source.
- `ACCEPT_RESPONSE_TYPE` is declared **once**, not duplicated as v1.0 claimed.

#### Platform-level impacts

**WDP-DB.md**
- No change. COMP-19 owns no database state and reads no tables.

**WDP-KAFKA.md**
- Section 3 Topic Registry, `internal-integration-events` row — no
  publisher list change (COMP-19 + COMP-20 already listed). Confirm note
  that COMP-19 partition key is `caseNumber` from `AddActionResponse`.
- Section 2 Confirmed DEC-003 Deviations — no change (COMP-19 row already
  reads `caseNumber` confirmed from source).
- Section 2 Confirmed DEC-001 Deviations — no change (COMP-19 row already
  reads "Direct synchronous publish — no outbox").
- **New context to add at the topic-level note for `internal-integration-events`:**
  AcceptEvent can be published in two split-brain scenarios where no
  card-network notification actually occurred: (a) MC CHI on NAP, (b)
  AMEX/DISCOVER on NAP with eligible inbound actionCode. NAPOutcomeProcessor
  consumers should not assume that an AcceptEvent implies the network was
  successfully notified.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: COMP-19 confirms (a) add-case-action ordering is AFTER the network
  call, (b) Kafka gate actionCode is sourced from CaseLookupResponse not
  from rules/AddActionResponse, (c) two new split-brain Kafka publish
  paths (MC CHI on NAP, AMEX/DISCOVER on NAP), (d) compensation has no
  inner try/catch — confirmed pattern likely shared with sibling
  COMP-20 ContestService.
- Resolved open questions: previous OQ-4 partition key open question
  remains — confirmed deviation, awaiting architect decision.
- New open questions: OQ-9 (architect decision on MC CHI / AMEX / DISCOVER
  silent-publish behaviour); OQ-10 (re-fetch source for files missing from
  OCR pass — `AcceptRequest`, `RestInvoker`, `NotFoundException`,
  `EnumName`, `DisputeAmount`, plus the `getCaseActionDetais` typo).

**WDP-DECISIONS.md · Candidate new ADRs**
- **ADR candidate (HIGH priority):** Document the AcceptService split-brain
  publish behaviour formally — either (a) accept the risk and document a
  manual recovery procedure for MC CHI / AMEX / DISCOVER NAP cases, or
  (b) remediate by fail-closing those branches before they reach the
  Kafka publish gate. This is the same severity class as DEC-019 and
  DEC-020 risk-accepted ADRs.
- DEC-001 deviation map for COMP-19 already present (no change). Severity
  reaffirmed as 🔴 HIGH.
- DEC-003 deviation map for COMP-19 already present (no change).
- DEC-014 — voided platform-wide. COMP-19 is consistent with the void.

**WDP-ARCHITECTURE.md**
- No topology change. Section 8.1 "Card Network Direct API Calls"
  diagram remains accurate at the system-context level. Worth noting in
  prose that AMEX/DISCOVER show as targets in the diagram but in fact
  fall through to no-op for AcceptService.

**WDP-NFRS.md · Section 6 Risk Register**
- Add **RISK-NEW-1 (🔴 HIGH):** AcceptService NAP split-brain on MC CHI
  and AMEX/DISCOVER — `AcceptEvent` published to NAP-DPS without any
  network notification. Affected components: COMP-19. ADR pending.
- Add **RISK-NEW-2 (🟠 HIGH):** AcceptService compensation has no inner
  try/catch — secondary failure replaces the original business exception
  in HTTP response, masking root cause. Affected component: COMP-19.
  Likely shared with COMP-20 ContestService — confirm in next pass.
- Reaffirm RISK-001 (no circuit breakers) and RISK-002 (no REST
  timeouts) for COMP-19 — 14 outbound REST calls, none with timeout
  or breaker.

**WDP-INTEGRATIONS.md**
- Section 3.1 (Visa RTSI) and Section 3.2 (Mastercard MCM) — no contract
  change. Update the "MC PIN PAB/ARB special case" note in 3.2 to read
  "MC CHI silent no-op applies to **both NAP and PIN** — not PIN only.
  Only PAB and ARB stages invoke an MCM network call on either platform."
- Section 3.1 — the existing "AMEX and DISCOVER gap in AcceptService"
  note can now be sharpened with the NAP-publish split-brain consequence.

#### Deviation flags for COMP-19

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 | ⛔ DEVIATES | 🔴 |
| DEC-003 | ⛔ DEVIATES | 🟡 |
| DEC-004 | ✅ NOT APPLICABLE | — |
| DEC-005 | ✅ NOT APPLICABLE | — |
| DEC-019 | ✅ NOT APPLICABLE | — |
| DEC-020 | ⚠️ PARTIAL | 🟡 |

DEC-001 deviation confirmed — direct synchronous Kafka publish after
case-action commit. HTTP 500 informs the caller but does not undo the
case action; NAPOutcomeProcessor is not notified. State permanently
inconsistent on Kafka final failure.

DEC-003 deviation confirmed — partition key is `caseNumber` from
`AddActionResponse`. `merchantId` exists on `CaseLookupResponse` but is
never mapped into `AcceptEvent`. No documented reason in source.

DEC-020 partial — no duplicate-detection on (caseNumber + actionSequence).
Partial guard via Step 3 case/action status membership check only.
Repeats that survive the membership guard re-execute the full flow,
including a second network call and a second Kafka publish.

#### Remaining gaps

- **OQ-COMP-19-1** Literal value of `${kafka_nap_topic}` per environment
  — environment config / K8s Secret. Confirmed `internal-integration-events`
  in WDP-KAFKA.md but not in source.
- **OQ-COMP-19-2** Production replica count and `deploymentApiVersion` —
  XL Deploy / Deploy.it manifests. Not in repo.
- **OQ-COMP-19-3** Architect confirmation of inferred logical-component
  mappings (case-lookup → COMP-23/27, rules → COMP-32, action add/update
  → COMP-24, notes → COMP-25, error log → COMP-38) — architect decision.
- **OQ-COMP-19-4** Why `caseNumber` is used as Kafka partition key
  instead of `merchantId` (DEC-003 deviation) — architect decision; ADR
  candidate.
- **OQ-COMP-19-5** Why 7 of 9 `saveErrorLog` calls are commented out —
  team confirmation. Intentional rollback, deferred refactor, or stuck
  mid-migration?
- **OQ-COMP-19-6** AMEX / DISCOVER routing absence — intentional scope
  boundary or production gap? Compounded by the new finding that NAP +
  AMEX/DISCOVER + eligible inbound actionCode publishes `AcceptEvent` to
  NAP-DPS — team / architect decision.
- **OQ-COMP-19-7** Production exclusion of `spring-boot-devtools` — relies
  on `spring-boot-maven-plugin` repackage being the deployed artefact.
  CI/CD pipeline confirmation required.
- **OQ-COMP-19-8** Confirmed list of callers (Merchant Portal, Ops Portal,
  API Gateway, automated workflows) — caller-repo analysis required.
- **OQ-COMP-19-9** Architect decision required on MC CHI silent no-op on
  NAP and AMEX/DISCOVER on NAP — should these branches fail loud, write
  a SNOTE, or block the Kafka publish? Blocks closing the two new HIGH
  split-brain risks.
- **OQ-COMP-19-10** Re-fetch source for files missing from OCR pass —
  `AcceptRequest.java` (entire `model/request/` package), `RestInvoker.java`
  (referenced from 7 service classes — its retry/header behaviour is
  inferred), `NotFoundException.java`, `EnumName.java`, `DisputeAmount.java`,
  plus the `getCaseActionDetais` (missing 'l') typo. Developer confirmation
  on source state.

#### Doc status after this change
- `WDP-COMP-19-ACCEPT-SERVICE.md` → `v2.0 DRAFT` — source-verified
  2026-04-23 · architect confirmation pending
---
### 2026-04-23 — COMP-37 DocumentManagementService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `gcp-document-management-service` (artifact `document-management-service` v2.2.8) — source-verified by Claude Code 2026-04-23. Architect confirmation pending.

**Nature of change:** Correction pass. Source audit closed the "is this service Kafka-connected?" knowledge-base discrepancy, reframed DEC-003 from "inconsistent per code path" to "uniformly non-compliant with legacy dead code", corrected the primary-flow step order (Kafka BEFORE action-update, not after), corrected the Endpoint 11 HTTP verb (POST, not PUT) and transactional semantics (stronger than DRAFT claimed), and closed multiple DynamoDB attribute-name and payload-field inaccuracies.

#### Platform-level impacts

**WDP-DB.md**

- **Section 1 (Data store inventory) — Amazon DynamoDB row:** no ownership change. Clarify attribute set.
- **Section 2 (Schema ownership map) — add rows:**
  - `QuestionnaireEntity` (WDP/US datasource) — OWNED by COMP-37 DocumentManagementService — caseNumber + actionSequence PK. No unique constraint beyond PK.
- **Section 2 — add shared-write flag (⚠️ shared table risk):**
  - `USCaseEntity` and `UKCaseEntity` — COMP-37 is a column-level UPDATE co-writer on the desk-blanking branch. Likely owned (INSERT/DELETE) by COMP-22 DisputeService. Cross-component ownership review needed.
- **Section on DynamoDB GSIs (new detail):** `WDP_PIN_DISPUTE_DOCUMENTS` and `NAP_DISPUTE_DOCUMENTS` declare five GSIs (`C_STAGE_CODE`, `I_ACTION_SEQ`, `C_DOC_TYPE`, `N_DOC_NAME`, `Z_UPDT`) — none queried from Java. Projection type not determinable from source — flag as external-config dependency.
- **Retention table row for Evidence documents:** no change (still TBC pending AWS console review).

**WDP-KAFKA.md**

- **Section 3 — Topic Registry — `business-rules` row:** Add COMP-37 DocumentManagementService to the Publisher list with note: "caseNumber key — DEC-003 deviation — publishes on upload (Endpoints 1, 9, 10) and questionnaire (Endpoint 11) paths, gated by `notifyBRQueue` + `startRuleGroup`; Endpoint 8 NAP base64 path never publishes (controller override). DEC-001 deviation on upload path (direct sync publish post-DDB); questionnaire path wraps publish in `@Transactional(rollbackOn=Exception.class)` — stronger atomicity than other publishers on this topic."
- **Section 2 — Confirmed DEC-001 Deviations table:** Add row for COMP-37 — topic `business-rules` — "Direct synchronous publish after DynamoDB write on primary upload path; no outbox. Questionnaire path is @Transactional(rollbackOn=Exception.class) — partial mitigation but still no outbox."
- **Section 2 — Confirmed DEC-003 Deviations table:** Add row for COMP-37 — topic `business-rules` — "caseNumber on every reachable publish (legacy merchantId path is dead code, 0 callers)."
- **Section 6 — Components Confirmed Kafka-Free:** ⚠️ **REMOVE** the COMP-37 row ("REST API. No Kafka dependency confirmed from source."). This row is factually incorrect and contradicts the component file, COMP-INDEX, and now-verified source.
- **Section 2 — DEC-005 Deviations table:** no change (consumer-only decision — COMP-37 has no consumer).

**WDP-HANDOVER.md · Confirmed Architectural Facts**

- **Add:**
  - COMP-37 DocumentManagementService is the fifth (sixth if COMP-14 is later confirmed) publisher of `business-rules` — caseNumber key, uniform DEC-003 deviation. Resolves the WDP-KAFKA.md discrepancy that had listed this component as Kafka-free.
  - COMP-37 primary upload step order is `S3 → DDB → desk update → Kafka publish → action-indicator update`. Action-indicator update is the LAST step — making action-indicator staleness the terminal failure mode if Kafka succeeds but the final REST call fails.
  - COMP-37 Endpoint 11 (POST questionnaire) is the only publish path in the platform that wraps the Kafka send inside `@Transactional(rollbackOn=Exception.class)`. Kafka failure rolls back the PostgreSQL save — stronger atomicity than the other five business-rules publishers.
  - COMP-37 `USCaseEntity` and `UKCaseEntity` are column-level UPDATE co-writers only (desk blanking). INSERT/DELETE ownership likely belongs to COMP-22 DisputeService — requires cross-component confirmation.
  - COMP-37 declares five DynamoDB GSIs but queries none of them from Java — either external consumer dependency or over-provisioning.
- **Resolved open questions:** the WDP-KAFKA.md vs WDP-COMP-INDEX.md / WDP-COMP-37 discrepancy about Kafka dependency is now definitively resolved. COMP-37 IS a producer.
- **New open questions (add to Open Questions table):**
  - COMP-37 Endpoint 6 (`GET /download`) is missing `@PathVariable("platform")` — source defect or OCR-drop in snapshot? Runtime-impact verification needed.
  - COMP-37 `BusinessRulesData` payload — `merchantId` field is declared in DTO but never populated; emitted as null on the wire. Does COMP-16 BusinessRulesProcessor handle `merchantId=null` safely?
  - COMP-37 RESPDOC→source divergence: primary upload maps to `BRRSUP`; questionnaire path maps to `BRMRUP`. Intended behaviour or bug? Requires COMP-16 verification.
  - COMP-37 five DynamoDB GSIs declared but unused from Java — which (if any) external consumer depends on them? Projection type on each?
  - COMP-37 `user-access-management-service` + `core-hierarchy-authorization-service-url` URLs configured in prod YAML but zero Java references — confirm dead config for removal.

**WDP-DECISIONS.md · Candidate new ADRs**

- **Candidate ADR:** Standardise Kafka partition key across `business-rules` producers. Every current publisher (COMP-15, COMP-23, COMP-24, COMP-25, COMP-37 upload/questionnaire, COMP-12 Scheduler4) uniformly deviates from DEC-003 by using `caseNumber`. The decision is no longer "whether to deviate" — it is whether to formally update DEC-003 to recognise case-scoped ordering on this topic, or whether to undertake a platform-wide migration to `merchantId`. COMP-37 data point strengthens the case for updating DEC-003.
- **Candidate ADR (related):** Formalise the `@Transactional(rollbackOn=Exception.class)` pattern used by COMP-37 Endpoint 11 as a stopgap atomicity pattern for components that cannot yet adopt an outbox. COMP-25 NotesService and COMP-15 EvidenceConsumer have similar shapes; platform could align on the pattern pending full outbox rollout.
- **Candidate ADR:** DEC-020 idempotency — formalise that `idempotency-key` header is accepted platform-wide but not used for dedup. COMP-37 is consistent with this. Decide whether to make this an accepted risk ADR or to mandate header-driven dedup on all write endpoints.

**WDP-ARCHITECTURE.md**

- No topology change.
- Minor clarification opportunity: COMP-37 is the only WDP component writing to AWS S3 and DynamoDB as primary data stores. Existing architecture diagram is accurate; add a note to the Data Storage section clarifying S3 + DynamoDB + dual PostgreSQL datasources are unique to this service.

**WDP-NFRS.md · Section 6 Risk Register**

- **Add RISK:** "COMP-37 primary upload path has a 5-step non-atomic write chain (S3 → DDB → desk → Kafka → action-indicator) with no compensating action at any boundary. Failure at any step after the first leaves partial state; failure after Kafka is unrecoverable without manual BR re-drive or action-indicator correction." Severity 🔴 HIGH. Link to DEC-001.
- **Add RISK:** "COMP-37 RestTemplate has no connect/read timeout on any outbound REST call (13+ call sites across 10 logical targets, plus a second inline `new RestTemplate()` in `RestInvoker.postData()`). Any hanging downstream blocks handler threads indefinitely." Severity 🔴 HIGH. Platform-wide RestTemplate timeout pattern should be revisited.
- **Add RISK:** "COMP-37 DynamoDB duplicate-check race — two concurrent requests with identical `(caseNumber, actionSequence, documentName)` both pass the non-atomic application-level check; later `putItem` silently overwrites earlier row (including insert-audit fields); Kafka publishes both events." Severity 🟡 MEDIUM.
- **Add RISK:** "COMP-37 `idempotency-key` header accepted and propagated as an outbound Kafka record header, but not used for deduplication at any write site. Client retries produce duplicate DDB rows, duplicate S3 writes, duplicate Kafka events." Severity 🟡 MEDIUM. Link to DEC-020.
- **Update RISK (if existing):** "topology spread `matchLabels` uses `${BRANCH_NAME_PLACEHOLDER}`" — confirm this pattern also affects other components and is not COMP-37-specific before generalising.

**WDP-INTEGRATIONS.md**

- No external integration contract change. All AWS S3 / DynamoDB / MSK usage was already documented at platform level.
- Minor clarification opportunity: if WDP-INTEGRATIONS.md has an AWS MSK section, note that COMP-37 shares the same MSK cluster as the other `business-rules` publishers — no separate cluster.

#### Deviation flags for COMP-37

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 | ⛔ DEVIATES (primary upload path); ⚠️ PARTIAL (questionnaire path — @Transactional wraps Kafka publish) | 🔴 HIGH |
| DEC-003 | ⛔ DEVIATES (uniform — `caseNumber` on every active path; legacy `merchantId` path is dead code) | 🔴 HIGH |
| DEC-004 | ✅ COMPLIES | — |
| DEC-005 | ✅ NOT APPLICABLE (producer only, no consumer) | — |
| DEC-019 | ✅ COMPLIES | — |
| DEC-020 | ⛔ PARTIAL DEVIATION (non-atomic dup check; idempotency-key accepted but unused; Endpoint 11 repeat-PUT overwrites state) | 🔴 HIGH |

**DEC-001 narrative:** Primary upload path performs direct synchronous Kafka publish after DynamoDB write and desk-number update — no outbox table, no spanning transaction. Broker unavailability after DDB commit = permanent BR event loss. Questionnaire path (Endpoint 11) is materially stronger than DRAFT v1.0 claimed: the publish is inside `@Transactional(rollbackOn=Exception.class)` alongside the PostgreSQL save, so a Kafka exhaustion rolls back the save. Remaining leak: broker ACK followed by a later commit-time exception can emit a published-but-unpersisted event. Remediation: adopt outbox pattern (platform-wide DEC-001 enforcement) or formalise the `@Transactional`-wrap pattern as a stopgap.

**DEC-003 narrative:** DRAFT v1.0 described the key as "inconsistent per code path" because two publish methods (legacy `sendBusinessRules(merchantId)` and newer `sendBusinessRulesToKafka(caseNumber)`) both appeared active. Source audit confirms zero callers of the legacy method. The runtime inconsistency does not exist — every message on `business-rules` from this service is keyed on `caseNumber`. The deviation from DEC-003 is uniform. Remediation options: (a) delete dead `sendBusinessRules()` and accept `caseNumber` as the de-facto standard across the five-plus producers on this topic; (b) migrate this service and the four other publishers to `merchantId`.

**DEC-020 narrative:** Three distinct gaps: (1) DynamoDB duplicate check is application-level query + in-memory compare, non-atomic — concurrent identical uploads both land; (2) `idempotency-key` header accepted, logged, MDC-tagged, echoed, and propagated as an outbound Kafka record header but not used for deduplication at any write site; (3) Endpoint 11 `POST ...action/{actionSeq}` treats repeat submissions as last-write-wins on `QuestionnaireEntity` and republishes Kafka each time — no idempotency guard beyond the primary key. Accepted-risk register candidate alongside COMP-23 and COMP-26.

#### Doc status after this change
- `WDP-COMP-37-DOCUMENT-MANAGEMENT-SERVICE.md` → `v1.1 DRAFT` — source-verified 2026-04-23 · architect confirmation pending
---
### 2026-04-23 — COMP-24 CaseActionService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `gcp-cas-actions-service` (artifact `case-actions-service`) — source-verified by Claude Code 2026-04-23. Architect confirmation pending.

**Nature of change:** Correction pass against source. Thirteen corrections to the v1.0 DRAFT — most material: (1) BRE Kafka publish is INSIDE @Transactional (not post-commit split-brain as v1.0 claimed); (2) FULL_CTM action code is CHGM not CHMR; (3) NAP conditional outbox writes to `nap.DISPUTE_EVENT_CONSUMER_ERROR`, not `wdp.chbk_outbox_row`; (4) `wdp.dispute_event_change_log` does not exist in this repo — dormant-table row removed. Several new risks surfaced: open-action constraint race window, last-write-wins on shared case/action tables, NAP vs US EP 5 asymmetry on chbkOutbox handling, EP 9 three-transaction no-compensation sequence.

#### Platform-level impacts

**WDP-DB.md**

- 🟡 `wdp.dispute_event_change_log` row — **REMOVE** the COMP-24 reference (and the row itself if COMP-24 was the only listed writer/reader). Source grep confirms zero matches for `EventChangeLog`, `dispute_event_change_log`, or any ChangeLog repository in the `gcp-cas-actions-service` repo.
- 🔴 **NEW ROW REQUIRED** in Shared Table Risk Register: `nap.DISPUTE_EVENT_CONSUMER_ERROR`. COMP-24 writes this table on NAP insert-path when `chbkOutbox` block present (same transaction as `nap.case` write). Not currently in WDP-DB.md. Follow-up needed on other writers — likely NAP-side consumers (COMP-04 / COMP-05). Suggested initial severity: 🟡 MEDIUM pending full writer confirmation. Note the US-vs-NAP asymmetry: NAP EP 5 does not touch this table even when `chbkOutbox` is present, so cross-component orchestration contracts that assume symmetric outbox acknowledgement are not upheld.
- 🟢 `wdp.notes` row — clarify COMP-24's scope: insert-path endpoints (EP 1 / 2 / 8 / 9) only, US/PIN/CORE/VAP/LATAM only. EP 5 on the US path does not insert notes. NAP path writes `nap.notes` (already listed correctly — confirm scope is insert-path only).
- 🟢 `wdp.case` and `wdp.action` rows — add note that COMP-24 has zero `@Version` annotations and zero pessimistic/optimistic locking. Last-write-wins semantics confirmed. Race window on the in-memory open-action constraint newly documented — supports existing 🔴 HIGH severity on `wdp.action` and 🟠 MEDIUM-HIGH on `wdp.case`.
- 🟢 `nap.case` and `nap.action` rows — same note (zero locking) applies to the NAP path.
- 🟢 `wdp.chbk_outbox_row` row — no change; COMP-24 remains a status-UPDATE-only writer, US/PIN/CORE/VAP/LATAM path only, conditional on `chbkOutbox` in request.

**WDP-KAFKA.md**

- **Section 2 — DEC-001 publisher deviation table**: correct COMP-24 row. v2.0 currently states *"Synchronous publish post-DB-commit — split-brain risk confirmed"*. Source shows the `BusinessRuleEvent` publish is inside the `@Transactional` service method — failure rolls back the DB. The split-brain applies only to the `ActionEvent` publish on EP 2 / 8 / 9. Revised text: *"BRE synchronous publish inside @Transactional — DB rolls back on send failure. ActionEvent post-commit publish — failures swallowed, genuine split-brain surface."*
- **Section 2 — DEC-003 deviation table**: no change — COMP-24 row already correct (`business-rules, ActionEvent topic | caseNumber`).
- **Section 3 — Topic Registry**: no change to topic rows. Both topics (`${kafka.business-rule-topic}` and `${kafka.topic}`) remain unresolved to literal names — resolution requires the K8s Secret values.
- **Topic consumer for `${kafka.topic}` (ActionEvent)**: still TBC. Suggested consumer remains COMP-39 NAPOutcomeProcessor — unverified. Follow-up Claude Code question recorded below.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Add / correct:
- COMP-24 BusinessRuleEvent Kafka publish is inside the `@Transactional` service method. A Kafka send failure rolls back the domain write before commit. Corrects the v1.0 claim that it was post-commit.
- COMP-24 ActionEvent Kafka publish (EP 2 / 8 / 9, conditional on `napUpdateEvent=true`) IS post-commit; failures are swallowed. This is a genuine split-brain surface.
- COMP-24 action code for FULL_CTM / CTM is `CHGM` (not `CHMR` as v1.0 DRAFT said).
- COMP-24 `recordTypeIndicator` default for FULL_CTM is `"1"` (not `"T"`).
- COMP-24 historicalData migration path is US/PIN only. NAP always emits BRE on insert. `napCaseActionDao.insertHistoricalAction` exists but has no service-layer caller.
- COMP-24 NAP conditional outbox writes `nap.DISPUTE_EVENT_CONSUMER_ERROR`, not `wdp.chbk_outbox_row`.
- COMP-24 NAP EP 5 (`UKCaseActionDaoImpl.updateAction`) does NOT process `chbkOutbox` on any branch — functional asymmetry vs US DAO.
- COMP-24 EP 5 reopen path writes `caseLiability = ApplicationConstants.SPACE` (single-space string), not null.
- COMP-24 EP 5 close-case path does NOT update `I_CASE_ACTION_MAX_SEQ` at the call site (only insert-path `updateCaseEntity` mutates max-seq).
- COMP-24 EP 9 runs three sequential independent transactions (insert + Document Service POST + indicator update + optional post-commit ActionEvent). No compensation between stages.
- COMP-24 has no `@Version`, no `@Lock`, no `SELECT FOR UPDATE` — last-write-wins on all owned tables.
- COMP-24 open-action constraint (≤ 1 non-REQ non-CLOSED action per case) is an in-memory check — race window exists across concurrent POSTs.
- COMP-24 memory limit is `4096Mi` (v1.0 DRAFT OCR-read "409EM" was incorrect).
- COMP-24 has Kubernetes liveness and readiness probes; startup probe absent; PodDisruptionBudget absent.
- COMP-24 repository name is `gcp-cas-actions-service` (v1.0 DRAFT used `mdvs-gcp-case-actions-service`; the `mdvs-` prefix appears in K8s app-label only).
- COMP-24 has no custom Micrometer meters — only a single `application` tag.
- COMP-24 dead config keys `${auth_url}` and `${pin_auth_url}` wire to the unused `RestInvoker.authorizeUser` path — formal RBAC ADR candidate.
- `wdp.dispute_event_change_log` / `EventChangeLogRepository` is not present in the `gcp-cas-actions-service` repository. Remove from platform-level references.

Resolved open questions (remove from HANDOVER):
- COMP-24 memory limit — now 4096Mi.
- COMP-24 liveness / readiness probe configuration — now documented.
- COMP-24 historicalData scope — US/PIN only.
- COMP-24 BRE publish transactional coupling — inside @Transactional.
- COMP-24 RestInvoker.authorizeUser caller confirmation — zero callers.
- COMP-24 MDC / correlation ID propagation — documented.
- COMP-24 Kafka producer config — documented (idempotent producer, retries=${kafka.retry-count}, all other timing/batch/compression configs default).

New open questions (add to HANDOVER):
- **OQ-COMP-24-1 Literal topic name for `${kafka.business-rule-topic}`** — env config. Confirm from K8s Secret `gcp-case-actions-service-secrets` or MSK configmap.
- **OQ-COMP-24-2 Literal topic name for `${kafka.topic}` (ActionEvent)** — env config. Same.
- **OQ-COMP-24-3 Consumer of `${kafka.topic}` (ActionEvent)** — follow-up Claude Code question: *"In the NAPOutcomeProcessor repository (COMP-39), search for any `@KafkaListener` on a topic resolved from `${kafka.topic}` or an env-var alias, consuming payloads with fields caseNumber, actionSequences, platform, currentActionSequence, networkCaseId. Report the topic property key, consumer group ID, and @KafkaListener file:line."*
- **OQ-COMP-24-4 Effective production IDP token URI** — env config. `application.yml` line 42 holds a hardcoded UAT `fiscloudservices.com` URI; only client-id and client-secret are externalised. Confirm whether any production env-var override exists.
- **OQ-COMP-24-5 Production replica count for COMP-24** — XL Deploy placeholder `{{replicas-gcp-case-actions-service}}`. Team confirmation.
- **OQ-COMP-24-6 DB unique constraint on `wdp.action` / `nap.action`** — DBA confirmation. Determines severity of the open-action constraint race window. No DDL in this repo.
- **OQ-COMP-24-7 Other writers of `nap.DISPUTE_EVENT_CONSUMER_ERROR`** — follow-up Claude Code pass on COMP-04 NAPDisputeEventService and COMP-05 NAPDisputeEventProcessor repos: *"Search for any JPA repository, entity, or native SQL that writes to `nap.DISPUTE_EVENT_CONSUMER_ERROR`. Report the entity class, repository, call sites, and transaction boundary."*
- **OQ-COMP-24-8 UKCaseActionDaoImpl `updatePreviousNapActionEntity`** — follow-up Claude Code pass on this repo: *"Read the body of `updatePreviousNapActionEntity` in `UKCaseActionDaoImpl` and related service helper. Report every field written on the previous-action entity, including any `revrsl-ind` / `revrnlInd` style field, and the conditions under which each write fires."*
- **OQ-COMP-24-9 NAP EP 5 chbkOutbox asymmetry** — architect decision. Is `UKCaseActionDaoImpl.updateAction` missing the `chbkOutbox` processing deliberately (NAP callers never signal via this field on update) or is it a gap?
- **OQ-COMP-24-10 Aurora HikariCP pool sizes** — env config. `application.yml` does not set pool sizes; Spring Boot defaults apply but production sizing should be confirmed with the platform team.

**WDP-DECISIONS.md · Candidate new ADRs**

- 🟠 **HIGH candidate — "RBAC enforcement absent in CaseActionService"**: `RestInvoker.authorizeUser` defined with zero callers; no `@PreAuthorize` / `@RolesAllowed` / `@Secured` anywhere; dead config keys `${auth_url}` and `${pin_auth_url}` wire to this unused path. Formally record at next WDP-DECISIONS rebuild window. Links to RISK-005 already in WDP-NFRS.md Section 6.
- 🟠 **MEDIUM-HIGH candidate — "Open-action constraint enforced in-memory only; no DB-level guard"**: Race window across concurrent POSTs on same `caseNumber`. Applies to COMP-24 insert endpoints and EP 5 update. Pairs with absence of `@Version` / `@Lock` on case/action entities.
- 🟠 **MEDIUM-HIGH candidate — "ActionEvent post-commit publish with swallowed failures"**: Distinct from DEC-001 deviation map. Specifically names the ActionEvent topic surface on EP 2 / 8 / 9 as a genuine split-brain; BRE publish is NOT a split-brain in this component (already covered by DEC-001 deviation map).
- 🟡 **MEDIUM candidate — "EP 9 retrieval-respond three-transaction sequence without compensation"**: Pattern-level risk — document-registration flow has no saga, no reconciliation job, no compensating delete/rollback. Applies to COMP-24 EP 9.
- 🟡 **MEDIUM candidate — "NAP vs US functional asymmetry on EP 5 `chbkOutbox` processing"**: `UKCaseActionDaoImpl.updateAction` silently ignores `chbkOutbox` on all branches. If NAP callers are sending this field expecting outbox acknowledgement, signals are lost.

**WDP-ARCHITECTURE.md**

- No change. Topology and principles unchanged. COMP-24 remains an action-management REST + Kafka Producer on the Core Processing band.

**WDP-NFRS.md · Section 6 Risk Register**

- RISK-005 (RBAC not enforced in CaseActionService) — already recorded. Consider adding the two dead config keys (`${auth_url}`, `${pin_auth_url}`) as additional evidence.
- **New row candidate** — "COMP-24 open-action constraint in-memory only; race window on concurrent POSTs for same caseNumber". Severity 🟠 MEDIUM-HIGH. No mitigation in place — no DB unique constraint visible.
- **New row candidate** — "COMP-24 ActionEvent post-commit split-brain on EP 2 / 8 / 9". Severity 🔴 HIGH. Distinct from RISK-003 (generic Kafka-failure event loss) because this is a component-specific, endpoint-specific deliberate swallow path.
- **New row candidate** — "COMP-24 EP 9 three-transaction sequence — no compensation". Severity 🟠 MEDIUM-HIGH. RRSP action commits before Document Service POST and indicator update; any subsequent failure leaves the DB ahead of the business outcome.
- **New row candidate** — "COMP-24 last-write-wins on `wdp.case` / `wdp.action` / `nap.case` / `nap.action` — no `@Version`, no pessimistic lock, no advisory lock". Severity 🟠 MEDIUM-HIGH. Cross-component concurrent update risk with COMP-23 CaseManagementService and COMP-15 EvidenceConsumer.

**WDP-INTEGRATIONS.md**

- No change. External integrations for COMP-24 are all internal WDP-to-WDP (Error Log, Notes, Document Service, IDP Token Service). No external-system contract affected.

#### Deviation flags for COMP-24

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⚠️ PARTIAL | 🟠 MEDIUM-HIGH (ActionEvent post-commit split-brain; BRE inside @Transactional — corrected) |
| DEC-003 Kafka Partition Key = merchantId | ⛔ DEVIATES | 🟡 MEDIUM |
| DEC-004 PAN Encryption Before Persistence | ✅ NOT APPLICABLE | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⛔ DEVIATES | 🟠 MEDIUM-HIGH |

**DEC-001 PARTIAL detail:** `wdp.chbk_outbox_row` / `nap.DISPUTE_EVENT_CONSUMER_ERROR` inbound-outbox UPDATEs are inside the same `@Transactional` as the domain write — compliant as inbound-outbox acknowledgement. BRE publish is also inside `@Transactional`, so send failure rolls back the DB — domain coupling honoured. ActionEvent publish is outside `@Transactional` on EP 2 / 8 / 9 — genuine post-commit split-brain when `napUpdateEvent=true`.

**DEC-003 DEVIATES detail:** Both topics use `caseNumber` key. Consistent with platform-wide deviation pattern across all five confirmed business-rules publishers (COMP-12, COMP-15, COMP-23, COMP-24, COMP-25).

**DEC-020 DEVIATES detail:** `idempotency-key` captured by HttpInterceptor and forwarded on Kafka outbound but never validated server-side. No seen-key store, no Redis cache, no DB duplicate check. Duplicate POSTs insert duplicate action rows. Under the current at-most-once platform delivery model (DEC-005) this is not triggered by Kafka redelivery; the risk is limited to concurrent HTTP caller retries.

#### Remaining gaps

- **OQ-COMP-24-1** `${kafka.business-rule-topic}` literal name — env config / team confirmation. Not in repo.
- **OQ-COMP-24-2** `${kafka.topic}` literal name — env config / team confirmation. Not in repo.
- **OQ-COMP-24-3** ActionEvent consumer identity — follow-up Claude Code pass on COMP-39 repo (question recorded above).
- **OQ-COMP-24-4** Effective production IDP token URI — env config. UAT URL in source; no prod override property visible.
- **OQ-COMP-24-5** Production replica count — XL Deploy placeholder.
- **OQ-COMP-24-6** DB unique constraint on `(I_CASE, I_ACTION_SEQ)` — DBA confirmation. Schema DDL outside this repo.
- **OQ-COMP-24-7** Other writers of `nap.DISPUTE_EVENT_CONSUMER_ERROR` — follow-up Claude Code pass on COMP-04 / COMP-05 (question recorded above).
- **OQ-COMP-24-8** `updatePreviousNapActionEntity` body — follow-up Claude Code pass on this repo (question recorded above).
- **OQ-COMP-24-9** NAP EP 5 chbkOutbox asymmetry — architect decision (intent or gap?).
- **OQ-COMP-24-10** Aurora HikariCP pool sizes — env config / team confirmation.

#### Doc status after this change
- `WDP-COMP-24-CASE-ACTION-SERVICE.md` → `v1.1 DRAFT` — source-verified 2026-04-23 · architect confirmation pending
---
### 2026-04-23 — COMP-23 CaseManagementService · v1.0 DRAFT → v1.1 DRAFT

**Source:** `mdws-gcp-case-management-service` — source-verified by Claude Code
2026-04-23. Architect confirmation pending.

**Nature of change:** Correction pass against source. Multiple material
corrections to v1.0 DRAFT (Kafka/DB ordering, non-existent table,
enrichment scope, column names, validation scope), plus five new
source-level defects and risks surfaced that were not visible in v1.0.

#### Platform-level impacts

**WDP-DB.md**
- 🔴 **CORRECTION REQUIRED — remove COMP-23 as a writer of
  `wdp.dispute_event_change_log`.** Source grep confirms zero references
  to that table in the `mdws-gcp-case-management-service` repository. The
  v1.0 DRAFT claim of a cross-datasource audit write from the NAP create
  path via `wdpTransactionManager` is not supported by source and is
  withdrawn. If the table is still active in production, its writers live
  outside this component.
- Owned-table count for COMP-23 drops from 7 to 6:
  `nap.case`, `nap.ACTION`, `nap.NOTES`, `WDP.CASE`, `wdp.ACTION`, `wdp.NOTES`.
- Correct the `nap.case` row: the PAN column COMP-23 writes clear on
  standard create is `I_ACCT_CDH` (consistent across both schemas). v1.0
  DRAFT had `I_ACCI_CDH` on the NAP side — typo in source-to-doc carry.
- Add to `wdp.NOTES` row notes: "COMP-23 US create path contains a
  duplicate-insert source defect — two identical `USNotesEntity` save
  blocks execute when `notesRequest != null`, producing two rows per
  create. New finding 2026-04-23. Defect not fixed in current working tree."
- Add to `NAP.DISPUTE_EVENT_CONSUMER_ERROR` row notes: "COMP-23 NAP create
  path calls `.save()` on a freshly-constructed `ConsumerErrorEntity` with
  three fields populated, without a prior `findById` guard. If the id does
  not already exist in the table, JPA merge INSERTs a sparse row. Cross-
  component write into a COMP-05-owned table with no owner check. New
  finding 2026-04-23."
- Add to `wdp.chbk_outbox_row` row notes: "COMP-23 update path uses
  `findById(...).isPresent()` guard (no-op if missing). No terminal-status
  guard — a PROCESSED row can still be overwritten. Same `@Transactional`
  as the case save."
- Shared Table Risk Register: append note to `wdp.NOTES` risk entry —
  "COMP-23 duplicate-insert defect inflates row count per case create.
  Investigation needed on whether downstream readers deduplicate."
- Shared Table Risk Register: append note to `NAP.DISPUTE_EVENT_CONSUMER_
  ERROR` risk entry — "COMP-23 blind-merge pattern can produce sparse
  rows in a table it does not own. Ownership / write-contract clarification
  required between COMP-05 and COMP-23."
- Add note: "No Flyway / Liquibase / schema.sql / data.sql in the COMP-23
  repository. DDL is managed elsewhere. Unique-constraint enforcement on
  COMP-23-owned tables cannot be determined from this repo."
- Case-number sequence ownership row: add COMP-23 as owner of
  `nap.case_i_case_sequence` (NAP) and `wdp.pin_case_i_case_sequence`
  (shared by PIN, CORE, VAP, LATAM). Both called via native `nextval(...)`
  OUTSIDE any `@Transactional` — sequence values are consumed on rollback
  (standard Postgres behaviour).

**WDP-KAFKA.md**
- 🔴 **CORRECTION REQUIRED in Section 2 DEC-001 deviation list.** COMP-23
  row currently reads "Synchronous publish after DB transaction commits —
  Kafka failure post-commit is unrecoverable". Replace with: "Synchronous
  publish INSIDE `@Transactional` before commit — Kafka failure rolls back
  DB consistently; narrow orphan-Kafka window is Kafka success followed
  by DB commit failure (constraint at flush, connection drop at commit).
  `kafka-before-commit` pattern — not an outbox, but not the `kafka-after-
  commit` pattern either."
- Section 2 DEC-003 deviation list: COMP-23 entry unchanged
  (`caseNumber` as partition key — confirmed intentional for case-level
  ordering).
- Section 3 Topic Registry row for `${kafka_business_event_topic}`:
  confirm COMP-23 as sole publisher; confirm COMP-16 as sole consumer;
  no change required if already present.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: COMP-23 Kafka publish is INSIDE `@Transactional` before commit —
  NOT after commit. v1.0 DRAFT was incorrect on this point. Kafka failure
  rolls back DB consistently. Orphan-Kafka window is narrow but non-zero.
- Add: COMP-23 owns 6 tables (not 7). `wdp.dispute_event_change_log` is
  NOT written by COMP-23 — zero references in source.
- Add: COMP-23 enrichment endpoint accepts four card networks
  (VISA/MC/AMEX/DISCOVER) × four platforms (PIN/CORE/VAP/LATAM), plus a
  second `noMid=true` sub-path that runs Step E only. v1.0 DRAFT described
  it as VISA-only.
- Add: COMP-23 Step C skip constant is `RDR`, not `RDF`. v1.0 DRAFT was
  wrong on this.
- Add: COMP-23 Step B method is always invoked on the standard enrichment
  path; the HTTP call is suppressed for non-CORE platforms and the method
  returns empty. Flow continues. v1.0 DRAFT treated this as a hard skip.
- Add: COMP-23 NAP historical blocking is implicit — a US-only whitelist
  in `createHistoricalCase`; any non-US platform (including NAP) returns
  400. v1.0 DRAFT implied an explicit NAP check.
- Add: COMP-23 Merchant Details failure swallow applies ONLY on the
  non-noMid path. The `noMid=true` path throws `InternalServerError` on
  Step E failure, rolling back the `@Transactional`.
- Add: COMP-23 PAN column is `I_ACCT_CDH` on both schemas (was mis-written
  as `I_ACCI_CDH` on the NAP side in v1.0).
- Add: COMP-23 liveness probe path is `/livez` (v1.0 had `/lives`).
- Add: COMP-23 contains a TODO at `GlobalExceptionHandler.java:169` on
  `METHOD_NOT_ALLOWED` — single TODO in the entire source tree.
- Add: COMP-23 PIN cardNumber-required rule applies only to PIN, not to
  CORE/VAP/LATAM. v1.0 DRAFT phrasing implied broader scope.
- Add: COMP-23 single shared `RestTemplate` bean constructed bare (no
  `ClientHttpRequestFactory`, no interceptors, no timeouts, no pool, no
  retries). IDP token is framework-cached via `OAuth2AuthorizedClient-
  Manager` (not per-call fetch).
- Add: COMP-23 no `@Scheduled`, no `@EnableScheduling`, no ShedLock, no
  Spring Batch, no Kafka consumer, no `OncePerRequestFilter`, no
  `@PreAuthorize` / `@Secured` anywhere. Confirms DEC-023 Not Applicable.
- Add: COMP-23 case-number generation runs OUTSIDE any `@Transactional`;
  sequence values are consumed on rollback. Sequence shared across PIN/
  CORE/VAP/LATAM. NAP has its own sequence.
- Add: COMP-23 `C_DUPLICATE_IND` — US side has field, `@Column`, and
  setter ALL commented out (not persisted). UK side is fully wired and
  setter invoked. v1.0 DRAFT's "silent null persist" claim applies to
  neither schema.
- Add: COMP-23 new source-level defects surfaced by audit (all present
  in current working tree, none fixed): (a) duplicate `wdp.NOTES` insert
  on US create, (b) `NAP.DISPUTE_EVENT_CONSUMER_ERROR` blind-merge,
  (c) `RequestCorrelation` ThreadLocal leak, (d) case-number NPE risk at
  12+ total length, (e) `spring-boot-devtools` shipping in prod image.
- Resolved open questions:
    - Kafka vs DB commit ordering — now documented as
      `kafka-before-commit`.
    - `wdp.dispute_event_change_log` cross-datasource write — does not
      exist.
    - PinActionDao — interface-only, no impl, not injected anywhere.
    - BusinessRulesConsumerErrorRepository — declared, `.save()` never
      called, not injected.
    - IDP token acquisition pattern — framework-cached, not per-call.
    - Production active profile — `${gcp_env}`; no profile-specific YAML
      in repo.
- New open questions:
    - Known callers of `POST /{platform}/transactions/enrich` — still
      not determinable from this repo.
    - Full field inventory of `UpdateCaseRequest` nested objects —
      `CaseServiceUpdateUtil` 1100+ lines not walked end-to-end.
    - `${BRANCH_NAME_PLACEHOLDER}` substitution mechanism — no XL Deploy
      dictionary, Helm values, Kustomize overlay, or Jenkinsfile in repo.
    - Production replica count, `${kafka_retry_count}`, and all other
      env-secret values.
    - Logback layout / structured-log JSON format — no `logback-spring.xml`
      in repo.
    - Unique-constraint enforcement at DB schema level — no DDL in repo.
    - Whether the duplicate-NOTES-insert and blind-save-into-consumer-
      error defects are known to the team or fresh findings.
    - Is the dual-schema asymmetry on `C_DUPLICATE_IND` (NAP wired, US
      disabled) intentional?

**WDP-DECISIONS.md · Candidate new ADRs**
- **HIGH candidate: "Kafka-before-commit synchronous publish pattern"** —
  name the pattern, distinguish from DEC-001 outbox and from the
  "kafka-after-commit" anti-pattern. Affects COMP-23 confirmed; likely
  affects COMP-24 CaseActionService, COMP-25 NotesService, COMP-19
  AcceptService, COMP-20 ContestService (all flagged in Kafka deviation
  list as "synchronous publish inside/around @Transactional"). Recommend
  explicit ADR on the intended pattern across case-write services at next
  DECISIONS rebuild window — currently these services are lumped under
  DEC-001 non-compliance without distinguishing the two failure modes.
- **MEDIUM candidate: "Blind JPA merge into cross-component-owned tables"**
  — COMP-23 writes into `NAP.DISPUTE_EVENT_CONSUMER_ERROR` (COMP-05-owned)
  without `findById` guard. This is a cross-component write-contract
  violation that neither DEC-001 nor DEC-020 currently cover. Recommend
  explicit ADR on cross-component table write contracts.
- **MEDIUM candidate: "No timeouts on any outbound `RestTemplate` call"**
  — confirmed on COMP-23; the pattern is already confirmed on multiple
  components. DEC-014 was voided platform-wide; a positive ADR stating
  "accepted operating condition: bare RestTemplate, no timeouts, no pool,
  no retries, no circuit breakers" may be warranted to close the gap
  between DEC-014 VOID and current platform reality.
- **LOW candidate: "Feature-flag hardcoded behaviour (productDefender,
  skipFraudSwitchAPI)"** — enrichment flow hardcodes product-entitlement
  result to FALSE and skips Fraud Switch HTTP for non-CORE. Both governed
  by comments, not a feature-flag framework. Consider whether the
  platform should adopt a feature-flag standard — affects COMP-23 today;
  pattern likely repeats.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- Candidate new RISK (HIGH): "Clear PAN persisted on standard case
  creation at COMP-23 (DEC-019 accepted risk) — remediation timeline
  still TBC. PCI-DSS compliance gap." — already implicit via DEC-019 but
  may warrant an explicit RISK row.
- Candidate new RISK (MEDIUM): "Orphan Kafka event window at COMP-23
  (and peer case-write services) — Kafka success + DB commit failure
  produces an event with no persisted record. Narrow but non-zero. No
  reconciliation path." — new RISK row recommended; differs from DEC-001
  generic non-compliance in that it describes the specific failure mode
  of the `kafka-before-commit` pattern.
- Candidate new RISK (MEDIUM): "COMP-23 duplicate `wdp.NOTES` insert
  defect — two rows per case create when `notesRequest` present.
  Investigation on downstream reader dedup needed."
- Candidate new RISK (MEDIUM): "COMP-23 `RequestCorrelation` ThreadLocal
  leak — latent cross-request contamination on pooled Tomcat worker
  threads. Impact: inaccurate correlation IDs in logs and Kafka headers
  on unusual code paths."
- Candidate new RISK (LOW): "COMP-23 case-number NPE risk — `getRandomDigits`
  returns null once sequence length + prefix + random alpha reaches 12.
  Deterministic once sequence grows; DBA confirmation on current high-
  water mark needed."
- Candidate new RISK (LOW): "`spring-boot-devtools` shipping in COMP-23
  prod image — dev-time class-path scanning and auto-restart behaviour
  may execute in production."

**WDP-INTEGRATIONS.md**
- No change. COMP-23's external dependencies (Settlement Details, Fraud
  Switch, Product Entitlement, Encryption, Merchant Details, Display
  Code, IDP) are all internal WDP services or WDP-facing integrations
  already documented. Kafka is WDP-owned MSK. No external integration
  contract changes.

#### Deviation flags for COMP-23

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⛔ NON-COMPLIANT (kafka-before-commit pattern) | 🔴 HIGH |
| DEC-003 Kafka Partition Key = merchantId | ⛔ DEVIATES (key is caseNumber) | 🟡 MEDIUM |
| DEC-004 PAN Encryption Before Persistence | ⛔ VIOLATION (standard create path) | 🔴 HIGH |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE (no consumer) | — |
| DEC-014 Resilience4j Circuit Breakers | ⛔ NON-COMPLIANT (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ⛔ CONFIRMED VIOLATION (accepted risk) | 🔴 HIGH |
| DEC-020 Full At-Least-Once Idempotency | ⛔ CONFIRMED GAP (accepted risk) | 🔴 HIGH |
| DEC-023 Replica = 1 Hard Constraint | ✅ NOT APPLICABLE (stateless REST) | — |

**DEC-001 NON-COMPLIANT detail:** Kafka publish is INSIDE `@Transactional`
before commit ("kafka-before-commit" pattern). Kafka failure rolls back
DB consistently. The narrow unrecoverable window is Kafka success
followed by DB commit failure (constraint at flush, connection drop at
commit) — produces an orphan Kafka event with no reconciliation path.
v1.0 DRAFT characterised this as "kafka-after-commit" which would have
widened the failure window significantly — the correction materially
changes the risk profile for the better, but DEC-001 non-compliance
remains.

**DEC-003 DEVIATES detail:** Partition key = `caseNumber`. Intentional
for per-case ordering across the BRE consumer (COMP-16). Platform
standard is `merchantId`. Pattern matches COMP-24 / COMP-25 / COMP-19 /
COMP-20 — all case-level services key by `caseNumber`. May warrant a
platform-standard amendment rather than treating each as a deviation.

**DEC-004 / DEC-019 VIOLATION detail:** Clear `cardNumber` written to
`nap.case.I_ACCT_CDH` and `WDP.CASE.I_ACCT_CDH` on standard create via
`CaseServiceSupport.mapNapCaseDetails` and `.mapPinCaseDetails`. PAN
encryption via EncryptionService (COMP-35) occurs ONLY on the enrichment
retry flow's Step D, which writes HPAN back to `WDP.CASE`. Cases created
but never enriched retain clear PAN indefinitely. PCI-DSS compliance gap.

**DEC-014 NON-COMPLIANT detail:** Single shared bare `RestTemplate` bean
on all 7 outbound call sites. No `ClientHttpRequestFactory`, no
interceptors, no connection pool, no timeouts (connection or read), no
retries, no circuit breakers. No Resilience4j / Spring Retry / Apache
HttpClient / OkHttp dependency. A slow dependency blocks the Tomcat
request thread indefinitely; thread-pool exhaustion cascades into
request-queue buildup.

**DEC-020 CONFIRMED GAP detail:** No SELECT-before-INSERT on any create
path. No DB unique constraint relied upon. `idempotency-key` header is
propagated from inbound request → MDC → Kafka message header but NEVER
consulted for duplicate detection. Two concurrent identical requests
produce two separate case records with different case numbers. Client-
side retry storms produce N records per request. Accepted risk per
WDP-DECISIONS.md DEC-020; no remediation timeline.

#### Doc status after this change
- `WDP-COMP-23-CASE-MANAGEMENT-SERVICE.md` → `v1.1 DRAFT` — source-
  verified 2026-04-23 · architect confirmation pending

### 2026-04-18 · COMP-18 NotificationOrchestrator · v2.0 DRAFT

Source-verified audit of `wp-mfd/wdp-outgoing-consumer` on master branch.
v1.0 DRAFT confirmed substantively accurate; seven factual corrections,
one new risk, and one confirmed cross-document error (WDP-DB.md).

#### Platform-level impacts
---
**WDP-DB.md**
- 🔴 **CORRECTION REQUIRED — remove COMP-18 as a writer of
  `wdp.outgoing_event_outbox`.** Source grep confirms zero references to
  that table in the `wdp-outgoing-consumer` repository. The row for
  `wdp.outgoing_event_outbox` must list only COMP-17 CaseExpiryUpdateConsumer
  (channel_type=EXPIRY_EVENTS) and COMP-43 CoreNotificationConsumer
  (channel_type=CORE_EVENTS). COMP-18's writes land exclusively on
  `wdp.bre_orchestration_outbox` (component=NOTIFICATION_ORCHESTRATOR) and
  `wdp.file_generation_event`.
- Row for `wdp.bre_orchestration_outbox`: confirm COMP-18 write detail —
  `created_by / updated_by = WDPOUCU`; columns written include
  `idempotency_id, component, event_timestamp, status, i_case, i_action_seq,
  target_action, published_action, retry_count, error_code, error_message,
  next_retry_at, original_event`. No `@Transactional` on any write. No
  `SELECT FOR UPDATE`. Four distinct write points (Step 3a ERROR, Step 3d
  PUBLISHED, Step 6 ERROR/PENDING_DEFERRED, Step 7e SUCCESS/FAILED).
- Row for `wdp.file_generation_event`: confirm COMP-18 as sole writer in
  this repo. `created_by / updated_by = WDPOUCU`. INSERT-only from COMP-18.
  Status hardcoded `STAGED` at entity field initializer. Columns left NULL
  on INSERT: `batch_id, next_retry_at, error_code, error_message`. No
  `@Transactional`. One row per document name in `documentNameList`.
- Add DB-level concern under "Shared-write risk" for `wdp.bre_orchestration_outbox`:
  previous-event guard SQL returns ALL rows for a caseNumber (not filtered
  in SQL); Java applies `id<eventId`, `component`, `status NOT IN
  (SUCCESS,SKIPPED)` predicates in-memory. Memory and latency scale with
  case history length.

**WDP-KAFKA.md**
- No structural change. Confirmations only:
  - Consumer topic `outgoing-events`, group `outgoing-events-group`,
    `AckMode.MANUAL_IMMEDIATE`, `syncCommits=true`, concurrency=1 default,
    ErrorHandlingDeserializer wrapping JsonDeserializer.
  - Outbox-table-to-Kafka-relay mapping for `wdp.bre_orchestration_outbox`
    remains: COMP-12 Scheduler4 reads FAILED and PENDING_DEFERRED rows;
    PUBLISHED rows are NOT re-driven. COMP-12 relay target for
    NOTIFICATION_ORCHESTRATOR rows is `outgoing-events`.
  - Producer configuration: `enable.idempotence=true`, `acks=all`,
    `max.in.flight.requests.per.connection=5`. `retries`,
    `delivery.timeout.ms`, `request.timeout.ms`, `linger.ms`,
    `batch.size` NOT set — Kafka client defaults apply.
  - Message key on all three outbound topics (`case-action-events`,
    `core-request-events`, `external-request-events`) is pass-through
    of inbound `KafkaHeaders.RECEIVED_KEY`. No `merchantId` reference
    anywhere in source.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add / correct:
    - COMP-18 does NOT write to `wdp.outgoing_event_outbox`. WDP-DB.md
      is incorrect on this point.
    - COMP-18 Step 6 previous-event guard is two-stage: SQL `WHERE
      caseId=?` returns all rows; `id<eventId`, `component`, and
      `status NOT IN (SUCCESS, SKIPPED)` filters applied in-memory in
      Java. Scales poorly for long-lived cases.
    - COMP-18 Filter 4 Sub-condition B stage+action set is
      `{RE2+REPR, PAB+MDCL, REQ+RRSP, ARB+MDCL}` — REPR not REFR.
    - `documentNameList` IS published on `PublishedNotificationEvent`
      when set. v1.0 DRAFT incorrectly listed it as absent from the
      published payload.
    - COMP-18 has `readinessProbe` and `livenessProbe` configured at
      `/merchant/gcp/outgoing-event/actuator/health` port 8082,
      initialDelaySeconds=120. NO `startupProbe`. v1.0 DRAFT wrongly
      implied all probes absent.
    - COMP-18 default Actuator exposure only —
      `management.endpoints.web.exposure.include` is not set.
    - COMP-18 K8s Secret name is `wdp-outgoing-consumer-secrets`.
    - COMP-18 outbound REST headers on DMS call include
      `v-correlation-id` and `idempotency-key` (both set to inbound
      `idempotencyId`). IDP Token Service call does NOT carry a
      correlation header.
    - COMP-18 outbound Kafka publishes forward `event-timestamp` and
      `idempotency-key` headers only. No correlation header on Kafka.
    - COMP-18 MDC correlation is effectively not wired —
      `HttpInterceptor` puts MDC but is not registered into any
      `WebMvcConfigurer`, and `RequestCorrelation` ThreadLocal is set
      but never read. Log lines do not carry per-event correlation
      context.
    - COMP-18 feature flags `coreMigration` and `disputesAPIMigration`
      have NO Java default and NO YAML default — startup fails if env
      vars `core_migration` / `disputes_api_migration` are absent.
      Flags read per-event via instance field access in filter
      methods.
    - COMP-18 no DDL / Flyway / Liquibase / schema.sql exists in the
      repository. UNIQUE constraints on outbox tables cannot be
      verified from this repo.
    - COMP-18 duplicate PUBLISHED idempotency match still flows
      through Step 7e and issues a SUCCESS UPDATE on the existing
      outbox row — overwrites `updated_at`, `retry_count`,
      `error_code`. Safe at-most-once (no re-publish) but
      audit-trail impact.
    - COMP-18 `retry_count` is incremented only when Step 7e writes
      FAILED; SUCCESS writes do not touch it. Escalation to ERROR
      only fires when the incoming status being written is FAILED
      and the new `retry_count > 2`.
    - COMP-18 DMS empty-list response is treated as an error path
      (errorOccured=true, errorReason set, INSERT skipped) — NOT a
      zero-row INSERT.

- Resolved open questions (remove from HANDOVER):
    - COMP-18 writership of `wdp.outgoing_event_outbox` — now
      resolved as NOT a writer.
    - COMP-18 K8s probes — now confirmed (liveness + readiness
      present; startup absent).
    - COMP-18 Actuator exposure configuration — now confirmed
      (default only).
    - COMP-18 Kafka producer config detail (retries,
      delivery.timeout.ms, etc.) — now confirmed not set.
    - COMP-18 DMS `@Retryable` attributes and `@Recover` presence —
      now documented: 3 × 2000ms fixed, `retryFor=Exception`,
      `exclude=RestTemplateCustomException`, NO `@Recover`.
    - COMP-18 IDP Token Service caching, retry, timeout — now
      documented as none.

- New open questions (add to HANDOVER):
    - Production values of `coreMigration` and `disputesAPIMigration`
      — K8s secrets, not auditable from repo. Routing table changes
      significantly. Team confirmation required.
    - Production replica count for COMP-18 — XL Deploy placeholder.
      Any value > 1 activates the SELECT-then-INSERT race window on
      idempotency across replicas.
    - Production values of `max_poll_records`, `max_poll_interval`,
      `session_timeout_ms`, `heartbeat_interval_ms` — env-var
      passthroughs with no defaults, not visible in source.
    - UNIQUE constraint on `wdp.bre_orchestration_outbox
      (idempotency_id, component, event_timestamp)` — DBA
      confirmation required; no DDL in this repo.
    - Upstream COMP-16 publish key on `outgoing-events` — determines
      whether DEC-003 deviation is `caseNumber`-scoped or otherwise.
      Cross-repo verification against COMP-16 at next audit.
    - Schema annotation inconsistency `WDP` vs `wdp` on entity
      annotations — depends on how schemas were created on the
      cluster. DBA confirmation required.
    - Owner of manual re-drive runbook for PUBLISHED-status orphan
      rows in `wdp.bre_orchestration_outbox` — RISK-015 captured at
      platform level; runbook not identified.

**WDP-DECISIONS.md · Candidate new ADRs**
- **MEDIUM candidate:** "Previous-event guard via full-case-history
  fetch + in-memory filter." COMP-18's Step 6 fetches every outbox
  row for the case and applies `id<eventId`, `component`, and status
  predicates in Java. Memory and latency grow with case history
  length. Pattern may recur in other consumers — worth checking
  COMP-17, COMP-43 at next review. Recommend raising at next
  DECISIONS rebuild if the pattern is confirmed elsewhere.
- **LOW candidate:** "No `@Transactional` on any service method in
  NotificationOrchestrator." Four independent outbox writes per
  event (3a / 3d / 6 / 7e) with no transactional grouping. Partial
  state between writes is the normal terminal state, not an
  exception. Already implicitly captured by DEC-001 PARTIAL, but
  explicit recognition at component-pattern level may be useful.

**WDP-ARCHITECTURE.md**
- §7.3 NotificationOrchestrator — confirm DB write target is
  `wdp.file_generation_event`, not `file_notifications`. Correction
  already applied in v1.0 DRAFT's table-name-correction callout;
  v2.0 confirms the correction from source.
- §7.3 — clarify that `external-request-events` NAP route (Filter 3
  `platform=NAP`, `migrationStatus=Y`) is already coded and live in
  this component — it is not a future-only path. The downstream
  COMP-44 EDIAConsumer is the planned consumer.
- No other topology or principle change.

**WDP-NFRS.md · Section 6 Risk Register**
- RISK-015 (bre_orchestration_outbox PUBLISHED orphan — no
  auto-redrive) — no change. Continues to cover this component.
- **New RISK candidate (MEDIUM):** "COMP-18 previous-event guard —
  unbounded in-memory filter on case history." Scales poorly for
  long-lived cases. Related to DEC-001 but a distinct operational
  risk.
- **New RISK candidate (HIGH):** "COMP-18 IDP Token Service call
  has no timeout configured — unbounded thread-block on IDP
  latency." With concurrency=1, one hung IDP call stalls all
  `outgoing-events` processing for the replica. Pattern matches
  RISK-already-covered on COMP-12 email relay, but merits separate
  entry because IDP is a cross-platform dependency used by many
  components and is on the hot path of Step 7d.
- **New RISK candidate (MEDIUM-HIGH):** "COMP-18 no-op
  `CommonErrorHandler` combined with `ErrorHandlingDeserializer` —
  malformed payloads silently dropped with no audit row, no DLT,
  no alerting." Pattern identical to COMP-14 RISK-COMP-14-C;
  consider generalising to a platform-wide risk covering all
  consumers using this deserialiser + no-op handler pattern.

**WDP-INTEGRATIONS.md**
- No change. COMP-18 has no external integrations — all targets
  (case-action-events, core-request-events, external-request-events,
  wdp.file_generation_event, wdp.bre_orchestration_outbox, IDP Token
  Service, Document Management Service) are WDP-internal.

#### Deviation flags for COMP-18

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-003 Kafka Partition Key = merchantId | ⛔ DEVIATES | 🟡 MEDIUM |
| DEC-004 PAN Encryption Before Persistence | ✅ NOT APPLICABLE | — |
| DEC-005 Manual Kafka Offset Commit | ⛔ DEVIATES (pre-publish ACK) | 🔴 HIGH |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ NOT APPLICABLE | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL — three gaps | 🔴 HIGH |
| DEC-023 Replica = 1 Hard Constraint | ✅ NOT APPLICABLE (Kafka consumer, not polling batch) | — |

**DEC-001 PARTIAL detail:** `wdp.bre_orchestration_outbox` is used
as the outbox, but zero `@Transactional` annotations exist in the
codebase. Four distinct write points (3a ERROR, 3d PUBLISHED, 6
ERROR/PENDING_DEFERRED, 7e SUCCESS/FAILED) are independent
auto-commits. Outbox INSERT at Step 3d is not atomic with the
Kafka offset commit at Step 4, nor with the downstream publishes
at Step 7. No SELECT FOR UPDATE on the idempotency lookup.

**DEC-003 DEVIATES detail:** No `merchantId` / `merchant_id` /
`MERCHANT_ID` string appears anywhere in `src/`. Kafka message key
on all three outbound topics is pass-through of inbound
`RECEIVED_KEY`. Actual partition-key identity depends on what
COMP-16 BusinessRulesProcessor sets when publishing to
`outgoing-events` — not determinable from this repo alone.
Cross-component deviation pattern — already recorded on COMP-12,
COMP-14, COMP-15, COMP-22, COMP-25, COMP-41, COMP-42.

**DEC-005 DEVIATES detail (🔴 HIGH):** `acknowledgment.acknowledge()`
is at a single call site after Step 3d outbox INSERT and before all
Step 7 writes. No code path defers ACK until after Step 7. Any
crash between ACK and Step 7e leaves an outbox row at `PUBLISHED`
with empty `publishedAction`. COMP-12 Scheduler4 reads only FAILED
and PENDING_DEFERRED rows — PUBLISHED orphans have no automatic
re-drive. RISK-015 already covers this platform-wide.

**DEC-014 DEVIATES detail:** No Resilience4j on classpath
(`pom.xml` lines 26-168 verified). No `@CircuitBreaker`,
`@TimeLimiter`, `@Bulkhead` annotations anywhere. Plain
`RestTemplate` with `SimpleClientHttpRequestFactory` — no pool, no
connect timeout, no read timeout. IDP Token Service call and DMS
call both use this bare template. DMS has `@Retryable` 3 × 2000ms;
IDP has no retry.

**DEC-020 PARTIAL detail (🔴 HIGH):** Three concurrent gaps:
(a) SELECT-then-INSERT race window at Steps 3b→3d — no
`@Transactional`, no `SELECT FOR UPDATE`, no advisory lock. Two
replicas processing the same idempotencyId in the same poll window
could both INSERT a PUBLISHED row. Mitigation depends entirely on
Kafka consumer group partition assignment keeping same-key events
on the same replica — which depends on upstream COMP-16 publish
key.
(b) Post-ACK crash gap between Step 4 and Step 7e — PUBLISHED
orphan rows with empty `publishedAction` are invisible to COMP-12
Scheduler4 (which queries FAILED/PENDING_DEFERRED only). Manual
runbook required; runbook owner not identified.
(c) Deserialisation silent loss via `ErrorHandlingDeserializer`
null-payload + no-op `CommonErrorHandler` — no audit row, no DLT,
no alerting. Exact broker-side offset commit sequencing is
Spring-Kafka-internal and not determinable from source alone.

#### Remaining gaps

- **OQ: Production values of `coreMigration` and
  `disputesAPIMigration`** — K8s secret, not in repo. Team
  confirmation. **Routing table changes materially depending on
  these values — blocker for DEC-003 deviation severity
  calibration.**
- **OQ: Production replica count** — XL Deploy placeholder. Any
  value > 1 activates the DEC-020 race window and is 🔴 HIGH.
- **OQ: Production values of `max_poll_records`,
  `max_poll_interval`, `session_timeout_ms`,
  `heartbeat_interval_ms`** — all env-var passthroughs with no
  defaults. Team / XLD config confirmation.
- **OQ: UNIQUE constraint on `wdp.bre_orchestration_outbox
  (idempotency_id, component, event_timestamp)`** — DBA team
  confirmation via actual DDL. No DDL in this repo.
- **OQ: Upstream COMP-16 publish key on `outgoing-events`** —
  follow-up Claude Code question against COMP-16 repo at its next
  audit: *"Search `wdp-business-rules-processor` (COMP-16) for
  every call to `KafkaTemplate.send` with destination
  `outgoing-events`. Report the exact message key source field
  and any key-derivation logic. Confirm whether `merchantId` or
  another field is used."*
- **OQ: Schema annotation `WDP` vs `wdp` resolution on actual
  cluster** — DBA confirmation via `\dn` on the target database.
- **OQ: Owner of manual re-drive runbook for PUBLISHED-status
  orphan rows** — operations / ops-runbook confirmation.
- **OQ: Spring Kafka container behaviour for null-payload +
  no-op `CommonErrorHandler`** — Spring Kafka documentation /
  framework-level verification required to determine exact
  offset-commit sequencing on deserialisation failure.

#### Doc status after this change
- `WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md` → `v2.0 DRAFT` —
  source-verified 2026-04-18 · architect confirmation pending
---

### 2026-04-18 — COMP-16 BusinessRulesProcessor (source audit confirmed)

**Component:** 16 — BusinessRulesProcessor
**Doc version:** v1.0 DRAFT → v1.1 (architect confirmation still PENDING)
**Trigger:** Direct source audit of `gcp-business-rules-processor` on
2026-04-18. Closed configuration/scaling gaps left open in v1.0 DRAFT;
corrected a material mis-label (FCMG → FCHG); surfaced two new US-only
behaviours and one asymmetry between UK and US paths.

---

**1. Corrections to facts previously in COMP-16 DRAFT**

| Topic | v1.0 DRAFT said | Source shows |
|---|---|---|
| DEC-005 pre-ACK line numbers | line 37 ack / line 40 process | **line 38 ack / line 41 process** (`KafkaConsumer.java:38,41`) |
| FCMG guard | `actionType == FCMG AND prior FCMG without RCAL → set queue to WEXQUE` | **FCHG / RCAL** pairing guard — checks FirstChargeback (`FCHG`) action types paired with `RCAL`. On guard failure, UK code calls `updateCase(...)` at `NAPRulesProcessServiceImpl.java:207`. The "queue to WEXQUE" behaviour is not supported by source and has been removed |
| US DB bean name | `wdpDataSource` | **`wdpdataSource`** (lowercase `d`, `USDataSourceConfig.java:29`) |
| UK transaction manager | Implied `napTransactionManager` | **`ukTransactionManager`** (`UKDataSourceConfig.java:43`) |
| Token cleanup | "in finally block" | **Asymmetric** — UK `tokenService.clear()` INSIDE finally (`NAPRulesProcessServiceImpl.java:157`); US `tokenService.clear()` OUTSIDE finally (`USRulesProcessServiceImpl.java:144`) |
| REST base path | Inferred from artefact naming | **Confirmed** `/merchant/gcp/business-rules-processor` from `SERVER_SERVLET_CONTEXT_PATH` in `resources.yml:62-63` |
| `ErrorLogService` | "commented out" | **Class does not exist in source tree** — autowire and call sites commented out in `KafkaServiceImpl.java:31-32, 66-67` |

**2. New facts added**

- **US-only `DOCUMENT_ATTACHED_TO_OPEN_CASE` recursive fallback** —
  `USRulesProcessServiceImpl.java:170-185`. When issuer-doc check returns
  `issuerDocAddedToCase=true` and the matched rule's `applyRuleGroup` is
  blank, the US path recursively evaluates rules under
  `DOCUMENT_ATTACHED_TO_OPEN_CASE`. UK has no equivalent.
- **US-only action types** — `US_OUTGOING_PRE_ARB` and `MERCHANT_ACCEPT`
  dispatched only on the US path.
- **US-only double `updateCaseAction` call** — US path calls
  `caseService.updateCaseAction()` twice per rule action (pre with
  `isActionUpdate=true` at line 600; post with `isActionUpdate=false` at
  line 739). UK path calls it once.
- **`v-correlation-id` header always null in production** —
  `RestClientInvoker.getHeaders()` reads from `MDC.get(CORRELATION_ID)`,
  but no MDC-populating filter or interceptor exists anywhere in the
  component. All outbound correlation-id propagation is effectively
  broken.
- **`outgoing-events` publish and `internal-integration-events` publish
  can both fire for the same message** — DMT001 publish does not suppress
  the finally-block outgoing publish.
- **`isErrorOccured` flag from outgoing publish is ignored by caller** —
  publish failures surface only as SNOTE via REST; the caller does not
  react to the flag.
- **Audit-log sequences** — `nap.br_case_audit_log_id_seq`,
  `wdp.br_case_audit_log_id_seq`, both `allocationSize=1`. No
  `@Transactional` on `saveAuditLog` — implicit Spring-Data-JPA
  transaction per `saveAll`. No unique constraint on either audit table.

**3. Configuration and scaling gaps closed**

| Parameter | Value | Source |
|---|---|---|
| Memory request / limit | 2048 Mi / 4096 Mi | `resources.yml:58,60` |
| CPU request / limit | Not set (Burstable QoS) | `resources.yml` |
| Rolling update | maxSurge=1, maxUnavailable=0, minReadySeconds=30 | `resources.yml:11-14,25` |
| Topology spread | maxSkew=1, topologyKey=`kubernetes.io/hostname`, whenUnsatisfiable=ScheduleAnyway | `resources.yml:26-32` |
| HPA | None | No `HorizontalPodAutoscaler` in manifest |
| PDB | None | No `PodDisruptionBudget` in manifest |
| Liveness | `GET /livez`, initialDelay 40s, period 10s, timeout 5s, failureThreshold 3 | `resources.yml:45-52` |
| Readiness | `GET /readyz`, initialDelay 30s, period 10s, timeout 5s, failureThreshold 3 | `resources.yml:36-44` |
| Actuator endpoints | `info`, `health`, `prometheus` on port 8082 | `application.yml:35-39` |
| Container port | 8082 | `resources.yml:55` |
| Security | OAuth2 Resource Server (JWT), CSRF disabled, whitelist `/actuator/health`, `/livez`, `/readyz` | `SecurityConfig.java:29-37` |

---

**4. WDP-KAFKA.md updates required**

**Section 2 — DEC-005 Deviations table, row for COMP-16:**
Change "line 37 of KafkaConsumer.java before processRulesEvent()" to
**"line 38 of KafkaConsumer.java before processRulesEvent() on line 41"**.

**Section 2 — DEC-001 Deviations table, row for COMP-16:** No change
(still covers both `outgoing-events` and `internal-integration-events`).

**Section 2 — DEC-003 Deviations table, row for COMP-16:** No change
(caseNumber on both producer topics).

**Section 3 — Topic Registry:**
- `outgoing-events` row: no content change; reference updated to v1.1.
- `internal-integration-events` row: reword COMP-16 note to clarify
  DMT001-only publish is **independent of** the finally-block
  `outgoing-events` publish (both can fire for the same message).
- `business-rules` row: no change.

**Section 4 — Consumer Group Registry, row for
`business-rules-processor-group`:** Update wording to
**"Pre-ACK — MANUAL_IMMEDIATE committed on line 38 of KafkaConsumer.java,
before processRulesEvent() on line 41 (DEC-005 deviation). Concurrency: 1
(Spring default)."**

**Section 5 — Components without Kafka involvement:** No change (COMP-16
is not in this list).

---

**5. WDP-DB.md updates required**

**Section 2 — Tables, `wdp.br_case_audit_log` row:** Append to Notes
**"No unique constraint declared — duplicate rows possible if DEC-005
pre-ACK is ever remediated without adding dedup. Sequence
`wdp.br_case_audit_log_id_seq`, allocationSize=1. No `@Transactional`
on save — implicit per-`saveAll` JPA transaction."**

**Section 2 — Tables, `nap.br_case_audit_log` row:** Same appended note
with UK sequence name.

**Section 2 — `wdp.CASE`, `wdp.ACTION`, `nap.case`, `nap.ACTION`,
`nap.rules`, `nap.rule_criterion`, `nap.rule_action`, `nap.rule_group`,
`wdp.rules`, `wdp.rule_criterion`, `wdp.rule_action`, `wdp.rule_group`
rows:** No ownership change. Confirm COMP-16 listed as reader.

**No new shared table risk introduced** by this revision. The existing
shared-audit-log note (COMP-31 reads these tables for its GET
`/audit-log/case/{caseNumber}` endpoint) remains accurate.

---

**6. Deviation flags — COMP-16 confirmed**

| DEC | Requirement | Actual | Severity |
|---|---|---|---|
| DEC-001 | Transactional outbox for Kafka publish | Not implemented on either producer topic — direct synchronous `.send().get()` | 🔴 HIGH |
| DEC-003 | Partition key = `merchantId` | `caseNumber` on both producer topics (consumer inbound is `merchantId` — upstream-compliant) | 🟡 MEDIUM |
| DEC-004 | PAN encryption before persistence | NOT APPLICABLE — no PAN handled by this component | 🟢 LOW (n/a) |
| DEC-005 | Manual offset commit AFTER processing | Pre-ACK: line 38 ack precedes line 41 process in `KafkaConsumer.java` | 🔴 HIGH |
| DEC-011 | BRE step checkpointing | ⛔ VOID — never implemented; formally voided April 2026 | 🟢 LOW (informational) |
| DEC-014 | Resilience4j circuit breakers | ⛔ VOID — absent platform-wide; formally voided April 2026 | 🟡 MEDIUM (accepted) |
| DEC-017 | BRP reads rules directly from DB | COMPLIES — confirmed | 🟢 LOW (compliant) |
| DEC-019 | Clear PAN on persistent store | NOT APPLICABLE — no PAN handled | 🟢 LOW (n/a) |
| DEC-020 | Full at-least-once idempotency | DEVIATES — `idempotency-key` pass-through only, no dedup store; compounded by DEC-005 pre-ACK | 🔴 HIGH |

---

**7. Remaining gaps — need follow-up**

| Gap | What type of follow-up |
|---|---|
| COMP-14 CaseCreationConsumer — does it publish to `business-rules` topic after case creation? | **Claude Code follow-up** on COMP-14 repo: *"Does `CaseCreationConsumer` publish to the `business-rules` Kafka topic after case creation? If so, include the call site (file:line), the Kafka template bean, and the payload type."* |
| `v-correlation-id` gap — fix or accept? | **Architect decision** — defect (add MDC filter populated from inbound Kafka headers) or accepted limitation? |
| US-path token cleanup outside finally (line 144) — defect or intentional? | **Architect decision** / team confirmation — candidate for symmetry fix with UK path |
| FCHG guard failure → `updateCase(...)` semantic — is this the intended final behaviour, or is there a missing downstream queue-transfer step that was removed? | **Team confirmation** — owning team to validate |
| Admin `PUT /event` endpoint — runbook and production posture | **Team confirmation** — remain enabled in prod, or gate by profile? |
| Exact production values for `max_poll_records`, `session_timeout_ms`, `heartbeat_interval_ms`, `max_poll_interval` env vars | **Environment config / team confirmation** — values live in K8s secrets outside repo |
| Replica count `{{ replicas-gcp-business-rules-processor }}` XL-Deploy substitution | **Environment config / team confirmation** — value outside repo |
| Kafka producer `retries`, `linger.ms`, `batch.size`, `compression.type`, `delivery.timeout.ms`, `request.timeout.ms` — not set, relying on client defaults | **Architect decision** — accept defaults or set explicit values for prod? |
| `rule_group` owning writer for both schemas | **Claude Code follow-up** on COMP-32 RulesService repo (already open question in WDP-HANDOVER) |
---

### 2026-04-18 — COMP-15 EvidenceConsumer · v1.0 DRAFT → v1.1 DRAFT

**Source:** `wdp-evidence-consumer` @ v1.1.2 — source-verified by
Claude Code 2026-04-18. Architect confirmation still pending.

**Nature of change:** Correction pass against source. No functional change
in production; 17 corrections and additions to the documented behaviour of
an existing component. Two findings rise to new 🔴 HIGH risk severity
(V3 silent-ATTACHED data-integrity defect, V3 PATCH ghost-upload). Mermaid
flow rebuilt to show two distinct transaction boundaries and correct
WDP-path-only gating of IDP / field-check / multi-doc.

#### Platform-level impacts

**WDP-DB.md**
- No change to the shared-writer row for `wdp.chbk_outbox_row`,
  `wdp.CASE`, `wdp.ACTION`, `wdp.disputes_questionnaire`, or `wdp.file_evidence` —
  COMP-15 write attribution is unchanged.
- Clarification candidates (not corrections):
    - `wdp.file_evidence` row for COMP-15: add note that
      `appended_file_name` may be `null` on V3 path for
      MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC due to confirmed data-integrity
      defect — downstream readers should not assume this column is
      populated on every ATTACHED row.
    - `wdp.ACTION` shared-writer row: note confirmed — no `@Version`,
      no `@UniqueConstraint`, no `SELECT FOR UPDATE` on any WDP entity
      accessed by COMP-15. Existing 🔴 HIGH severity retained.
    - `wdp.disputes_questionnaire` shared-writer row: confirmed —
      COMP-15 writes only on WDP path RESPDOC. Existing 🔴 HIGH severity
      (COMP-26 POST idempotency gap) unchanged.

**WDP-KAFKA.md**
- Section 2 (Topic Registry) — `case-evidence-events` row: add clarification
  that inbound `RECEIVED_KEY` (caseNumber) is `@Nullable`; ordering
  guarantee by `caseNumber` holds only when upstream sets a non-null key.
- Section 2 (Topic Registry) — `business-rules` row: the COMP-15 producer
  entry is still accurate, but:
    - Add: RESPDOC WDP path emits **two** events per success
      (`ACTION_UPDATED` then `DOCUMENT_UPLOADED`, sorted alphabetically).
    - Add: DSNOTDOC WDP path publishes `source=""` (builder has no
      DSNOTDOC branch — downstream-routing defect).
    - Add: Platform string is uppercased on publish by COMP-15.
    - Add: Compensating BR events emitted by `updateFailedStep` when
      `retry_count > 2` escalates to ERROR and gate conditions met
      (WDP path AND `!isMultiDocPending` AND `documentType≠DSNOTDOC`
      AND `caseLookupResponse` present).
- Section 3 Confirmed DEC-005 Deviations — COMP-15 entry still accurate.
- Section 4 Confirmed DEC-001 Deviations — COMP-15 entry still accurate;
  add note that the publish is `kafkaTemplate.send(...).get()` blocking
  on the future inside `@Transactional`.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: On COMP-15 V3 path, MISCDOC / DRFTDOC / RESPQDOC / ISSRQDOC
  document types silently mark `file_evidence.attachment_status=ATTACHED`
  without any upload occurring — data-integrity defect, not merely an
  "unfinished state" as previously documented.
- Add: On COMP-15 V3 RESPDOC path, a failed ownership-transfer PATCH
  leaves the document uploaded to V3 Core but the WDP DB unmarked —
  ghost-upload failure mode. No automatic reconciliation.
- Add: COMP-15 publishes two BR events per successful RESPDOC on the
  WDP path (`ACTION_UPDATED` then `DOCUMENT_UPLOADED`), sent synchronously
  inside `TXN_WDP` via `kafkaTemplate.send(...).get()`.
- Add: COMP-15 DSNOTDOC WDP events publish with `source=""` — the
  builder has no DSNOTDOC branch. This is a suspected downstream-routing
  bug that requires COMP-16 consumer-side investigation.
- Add: COMP-15 V3 Update Action PATCH mutates the shared `RestTemplate`
  bean's request factory — after the first V3 PATCH call, every
  subsequent REST call on the pod inherits 30s/30s timeouts. Order-
  dependent global state.
- Add: COMP-15 inbound Kafka key and `idempotency-key` header are both
  `@Nullable`. Nulls pass through to DMS upload and to the outbound
  `business-rules` message.
- Add: COMP-15 WDP-path gating is narrower than prior documentation
  implied — IDP fetch, required-field validation, and multi-doc sibling
  scan are all skipped on the V3 legacy path.
- Add: COMP-15 `coreMigrationFlag` is sourced from env var
  `${core_migration_flag}` via K8s secrets / configmap. Fixed at
  container start; no runtime toggle.
- Resolved open questions: COMP-15 Step 6 `@Transactional` boundary
  (closed — `wdpTransactionManager`, `rollbackFor=Exception.class`,
  default propagation, two distinct transaction methods for WDP vs V3);
  COMP-15 compensating-BR behaviour (closed — lives inside
  `updateFailedStep`, gated as above); `isMultiDocPending` scan
  semantics (closed — negation against `NOT_VALID_STATUS = {ATTACHED,
  SKIPPED, ERROR}`); PENDING_DEFERRED re-driver (closed at COMP-15 scope
  — no re-driver in this repo, cross-component follow-up needed).
- New open questions to add to the platform list:
    - OQ: COMP-15 V3-path MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC silently
      mark `file_evidence=ATTACHED` — is the fix an upload path for
      these types, or an explicit skip/reject path? Architect decision.
    - OQ: COMP-15 DSNOTDOC WDP events publish with `source=""` — is
      COMP-16 tolerant of this, or does it misroute? Follow-up Claude
      Code question against `wdp-business-rules-processor`: *"How does
      BusinessRulesProcessor (COMP-16) handle inbound events on the
      `business-rules` topic where the `source` field is the empty
      string?"*
    - OQ: PENDING_DEFERRED re-driver ownership — which WDP component
      polls or re-drives `chbk_outbox_row.status=PENDING_DEFERRED`?
      Follow-up Claude Code scan across all components. If no re-driver
      exists, this is a latent stuck-record source.
    - OQ: COMP-15 `vantiveLicense` YAML/Java key case mismatch — is the
      production path relying on env-var override of `${vantive_license}`
      from K8s secrets? Team confirmation.
    - OQ: COMP-15 has no K8s liveness/readiness/startup probes — is this
      intentional for this workload class, or a deployment template
      gap? Operational confirmation.
    - OQ: COMP-15 `{{ replicas-wdp-evidence-consumer }}` runtime value —
      environment config. Confirm via XL Deploy / Helm.

**WDP-DECISIONS.md · Candidate new ADRs**
- No new ADRs. The DEC-001 deviation map already captures COMP-15.
  Candidate clarifications:
    - DEC-001 COMP-15 entry: add note that the publish is a blocking
      `future.get()` inside `@Transactional` — the ghost-event window
      is narrow but non-zero (Kafka success → later DB write failure in
      the same transaction).
    - DEC-020 deviation map: add COMP-15 — four distinct at-least-once
      gaps (pre-ACK silent loss window; deserialization silent drop;
      no DB-level unique constraint on `file_evidence`; compensating
      BR events require `caseLookupResponse` to be present, so errors
      before case lookup emit no compensating event). Severity 🔴 HIGH.
    - DEC-003 deviation map: COMP-15 entry unchanged — `caseNumber` on
      both inbound and outbound topics.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- Candidate new RISK rows (pending architect decision):
    - "V3-path document-type dispatcher silently marks
      `file_evidence=ATTACHED` without upload for unhandled document
      types (MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC) — silent data-integrity
      defect on CORE legacy path." 🔴 HIGH.
    - "V3 Update Action PATCH failure leaves V3-side document uploaded
      but WDP DB unmarked — ghost-upload requiring manual
      reconciliation." 🔴 HIGH.
    - "Shared-`RestTemplate` mutation in COMP-15: V3 PATCH permanently
      reconfigures the shared bean's request factory, making timeout
      behaviour order-dependent across REST calls on the same pod."
      🟠 MEDIUM-HIGH (latent).
    - "COMP-15 has no Kubernetes liveness/readiness/startup probes —
      hung pods are not evicted by kubelet." 🔴 HIGH (operational).
- Existing RISK-013 (replica constraint has no automated enforcement) —
  not applicable to COMP-15 (no replica=1 constraint; this is a standard
  scaled consumer).

**WDP-INTEGRATIONS.md**
- No change. COMP-15 inbound and outbound contracts are component-
  internal (COMP-11 DB handoff, COMP-12 Kafka handoff, COMP-16 Kafka
  handoff, COMP-37 DMS call). No new external integration surface.

#### Deviation flags for COMP-15

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⛔ DEVIATES | 🔴 HIGH |
| DEC-003 Kafka Partition Key = merchantId | ⛔ DEVIATES | 🟡 MEDIUM |
| DEC-004 PAN Encryption Before Persistence | ✅ NOT APPLICABLE | — |
| DEC-005 Manual Kafka Offset Commit | ⛔ DEVIATES (pre-ACK) | 🔴 HIGH |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL — four gaps | 🔴 HIGH |

**DEC-001 DEVIATES detail:** Business Rules publish is
`kafkaTemplate.send(message).get()` called blocking inside
`TXN_WDP (@Transactional(wdpTransactionManager))`. Failure modes: (a)
Kafka send failure → JPA rollback — recoverable via updateFailedStep.
(b) Kafka success followed by a DB failure at a subsequent write in the
same transaction → ghost BR event in `business-rules` topic with no
corresponding WDP DB state — unrecoverable without manual intervention.

**DEC-003 DEVIATES detail:** Inbound `case-evidence-events` and
outbound `business-rules` both key on `caseNumber`. Consistent with
platform-wide deviation pattern across business-rules publishers.

**DEC-005 DEVIATES detail:** Pre-ACK at Step 1 — offset committed
immediately on receipt, before any DB read, S3 download, case lookup,
upload, or DB write. Any pod death after Step 1 causes permanent
evidence loss for that message.

**DEC-020 PARTIAL detail (severity 🔴 HIGH):** Four concurrent gaps.
(a) Pre-ACK silent loss window — any crash between Step 1 and Step 6
loses the message permanently. (b) Deserialization silent drop — no
audit record anywhere for malformed payloads. (c) No DB-level UNIQUE
constraint on `file_evidence (chbk_outbox_row_id)` — rolling deploy
replica overlap can produce duplicate upload. (d) Compensating BR
events require `caseLookupResponse` to be present, so errors occurring
before Step 4 (IDP, S3 download, field validation) emit no compensating
BR event on escalation.

#### Remaining gaps

- **OQ-COMP-15-1 V3 unhandled document-type defect** — architect
  decision. Do MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC have a legitimate V3
  upload path that is missing, or should they be explicitly rejected?
- **OQ-COMP-15-2 DSNOTDOC BR `source=""`** — follow-up Claude Code
  question against `wdp-business-rules-processor`: *"How does
  BusinessRulesProcessor (COMP-16) handle inbound events on the
  `business-rules` topic where the `source` field is the empty
  string? Is there a default rule group, a silent drop, or an
  error path?"*
- **OQ-COMP-15-3 PENDING_DEFERRED re-driver ownership** — follow-up
  Claude Code scan across all components: *"Which component reads or
  updates `wdp.chbk_outbox_row` rows where status=PENDING_DEFERRED?
  If none, this status is a terminal stuck state."*
- **OQ-COMP-15-4 `vantiveLicense` resolution path** — team
  confirmation. Is `${vantive_license}` env var injected in production
  via K8s Secret, making the YAML/Java key case mismatch harmless,
  or is the YAML path actually dead?
- **OQ-COMP-15-5 Absent K8s health probes** — operational confirmation.
  Is the absence of livenessProbe/readinessProbe/startupProbe intentional
  for this workload class, or a deployment template gap?
- **OQ-COMP-15-6 Production replica count** — environment config
  (`{{ replicas-wdp-evidence-consumer }}`). Not in repo.
- **OQ-COMP-15-7 V3 `${v3_upload_doc_url}` / `${v3_upload_action_url}`
  fully-qualified URLs** — environment config. Not in repo.
- **OQ-COMP-15-8 FileProcessor (COMP-11) write pattern on
  `wdp.file_evidence`** — confirmed from prior COMP-11 audit (see
  WDP-CHANGE-LOG.md 2026-04-18 COMP-11 entry). Cross-reference only.

#### Doc status after this change
- `WDP-COMP-15-EVIDENCE-CONSUMER.md` → `v1.1 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending
---

### 2026-04-18 — COMP-14 CaseCreationConsumer · v1.0 DRAFT → v2.0 DRAFT

**Source:** `gcp-case-creation-consumer` v1.3.7 — source-verified by Claude Code
2026-04-18. Architect confirmation pending.

**Nature of change:** Correction pass against source. No functional change in
production; 11 corrections to the v1.0 DRAFT plus four previously-undocumented
production-risk findings surfaced during the audit. Three findings rise to
🔴 HIGH severity. Closes 20 of 21 enumerated gaps (one gap remains not
determinable from source).

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- Clarify COMP-14's writer scope: status transitions ONLY (`FAILED`, `ERROR`,
  `SUCCESS`, `PENDING_DEFERRED`, `SKIPPED`, and `PENDING` on EVIDENCE_ATTACH
  children). This component does NOT insert new rows.
- Add: "COMP-14 writes are all **auto-commit** — zero `@Transactional`
  annotations anywhere in `gcp-case-creation-consumer` source. Parent SUCCESS
  save and EVIDENCE_ATTACH child `saveAll` are two separate auto-commits. A
  crash between them leaves the outbox in a non-atomic, non-self-healing state
  that the Kafka offset (pre-ACK) will not cause to be redelivered."
- Add: "COMP-14 `retry_count > 2 → ERROR` promotion is Java-side, not a DB
  trigger or CHECK constraint. Any direct-DB status manipulation bypasses the
  promotion logic."

**WDP-DB.md · Section 2 · `wdp.case` row**
- Add COMP-14 to read-only accessor list. Purpose: Capone REQ stage duplicate
  check via JPA method-name derivation over (level1Entity, tr, acctCdhLst,
  purchaseAmt, originalDisputeAmount). Read-only; no `@Query`; no lock.

**WDP-DB.md · Section 4 · Shared Table Risk Register**
- Append to `wdp.chbk_outbox_row` risk entry: "COMP-14 source-verification
  2026-04-18 confirms the Layer-1 prior-chargeback dedup is a 2-column DB
  query (`networkCaseId + cardNetwork`) plus an in-memory Java stream filter
  for status/event-type. No DB lock, no `SELECT ... FOR UPDATE`. Concurrent
  processing of the same `(networkCaseId, cardNetwork)` by two replicas — for
  example during rolling-update overlap or any concurrency-setting increase —
  can produce duplicate case creation. Current safety is entirely operational:
  concurrency = 1 and Kafka consumer-group partition assignment."

**WDP-KAFKA.md · Section 3 Topic Registry · `new-case-events` row**
- Add / confirm: Consumer groups `new-case-events-group` (prod) and
  `new-case-events-group-cert` (cert). The earlier documented `-dev` name is
  incorrect — the cert profile uses `-cert`. Confirm whether a separate dev
  profile and group exists (not in source for this audit pass).
- Add partition-key annotation: compound key logged as
  `keyNetworkCaseCardNetworkId` — consumer side. DEC-003 deviation confirmed
  both sides (producer COMP-12 + consumer COMP-14).

**WDP-KAFKA.md · Section 4 Consumer Group Registry · `new-case-events` row**
- Confirm: AckMode = `MANUAL_IMMEDIATE`, syncCommits = true, pre-ACK before
  `processKafkaNotificationEvent()`. DEC-005 deviation.
- Correct: `auto.offset.reset = latest` (not `earliest` — the v1.0 DRAFT and
  the KAFKA.md entry should be cross-checked for any earlier-propagated
  `earliest` claim).
- Add: Empty anonymous `CommonErrorHandler` — deserialization exceptions
  silently swallowed, no DLT, no log, no counter. Add this as a row-level
  note alongside the existing pre-ACK annotation.
- Concurrency = 1 (default — `setConcurrency()` never called). Max poll
  records, max poll interval, session timeout, heartbeat interval are all
  env-injected with no YAML defaults — values not determinable from source.

**WDP-KAFKA.md · Section 6 Components Confirmed Kafka-Free**
- No change. COMP-14 is a consumer, not producer-free. However, add an
  explicit note to the CaseCreationConsumer row in Section 3/4 stating
  "no Kafka producer in this component — no `KafkaTemplate`, no
  `ProducerFactory`, no reference to `business-rules` topic. This closes
  Observability-doc OQ-02 (COMP-14 does not publish to business-rules)."

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: COMP-14 has zero `@Transactional` annotations — all DB writes
  auto-commit. Parent SUCCESS and EVIDENCE_ATTACH child writes are not
  atomic. Implication is that DEC-001 deviation in COMP-14 is not just
  "outbox-not-transactional" but "no transactions at all on the DB path".
- Add: COMP-14 `auto.offset.reset = latest` — cold starts with no committed
  offset SKIP messages rather than replay.
- Add: COMP-14 registers an empty `CommonErrorHandler`; combined with
  `ErrorHandlingDeserializer` and pre-ACK, deserialization exceptions are a
  distinct silent-loss class from the pre-ACK window.
- Add: COMP-14 has no liveness probe and no startup probe — only readiness.
  A stuck listener thread will not restart the pod.
- Add: COMP-14 Layer-1 prior-chargeback dedup is in-memory stream filter
  over a 2-column query — safe only by concurrency=1 operational constraint.
- Resolved open questions:
    - OQ-02 (observability doc): COMP-14 does not publish to `business-rules`.
      Confirmed — no producer in source.
    - DEC-003 consumer-side verification: compound key
      `keyNetworkCaseCardNetworkId` logged on receipt; not used for routing.
      Deviation confirmed both sides.
    - Kafka `auto.offset.reset` value: `latest` (not `earliest`).
    - ACK line: KafkaConsumer.java:38 (not :36).
    - Cert group: `new-case-events-group-cert` (not `-dev`).
    - Workflow hardcode trigger: `QueueName.COMP / PRECOMP` (not
      `schemeRef.category`).
    - NAP guard: confirmed NAP is not blocked — flows through lookups.
    - OQ-03 (observability doc): `wdp.chbk_outbox_row.status = SUCCESS` writer
      is COMP-14 via `chbkOutboxRepository.save()` at ChbkOutboxServiceImpl.
      EVIDENCE_ATTACH children set to PENDING on the NEW path, also by
      COMP-14, via a separate auto-commit `saveAll`.
- New open questions:
    - OQ-14.1: Mastercard `RE2` / `reversal=Y` handling in COMP-14.
      **Follow-up Claude Code question** against the COMP-14 repo:
      *"Search `gcp-case-creation-consumer` for any branch, switch, or
      conditional that evaluates `cardNetwork == MASTERCARD` together with
      either `disputeStage == RE2` or `reversal == Y`. Cite file:line. If no
      such branch exists, search for any reference to `MASTERCARD` that
      invokes a distinct action filter. Report whether the logic is inside
      `firstActionRuleLookup` generically or is absent entirely."*
    - OQ-14.2: `preActionStatusRule` lookup `@Recover` behaviour.
      **Follow-up Claude Code question:** *"In `EventProcessingServiceImpl`
      (gcp-case-creation-consumer), locate the `@Recover` method associated
      with the pre-action-status-rule lookup. Report its return value
      (null / FAILED write / other) and whether its null return is null-
      checked by the caller."*
    - OQ-14.3: Production values for `dsCaseLookup`, `skipStageList`,
      `creditMerchantActionCode`, `complianceRecordType`,
      `precomplianceRecordType`, `apcCaseLookupDisputeStages`,
      `acfCaseLookupDisputeStages` — **environment config / team
      confirmation.** None have YAML defaults; all env-var only.
    - OQ-14.4: Actual production replica count — **environment config /
      team confirmation.** Helm/Jinja placeholder in repo.
    - OQ-14.5: Env-injected Kafka client tunings (`max_poll_records`,
      `max_poll_interval`, `session_timeout_ms`, `heartbeat_interval_ms`)
      — **environment config / team confirmation.**

**WDP-DECISIONS.md · Candidate new ADRs**
- **Candidate ADR — "CaseCreationConsumer Operates Without @Transactional"
  🔴 HIGH.** Currently covered partially under the DEC-001 deviation map, but
  this component is a more extreme case than "no outbox atomicity" — it has
  no transactions at all on the Kafka processing path. Consider whether this
  warrants a dedicated risk-accepted ADR (analogous to DEC-019 and DEC-020)
  to make the condition explicit and to record operational compensations
  (e.g. monitoring for EVIDENCE_ATTACH drift).
- **Candidate addition to DEC-005 deviation map:** COMP-14's pre-ACK combined
  with the empty `CommonErrorHandler` and absence of a liveness probe
  compounds the at-most-once behaviour — deserialization errors on cold
  start with `auto.offset.reset = latest` produce silent skip; thread stalls
  produce silent block with a Ready pod. Add this as a "compound risk
  pattern" note.

**WDP-ARCHITECTURE.md**
- Clarify the CaseCreationConsumer section: enrichment paths for PIN share
  the same `gcp-merchant-transaction-service` URL as CORE (confirmed). The
  open question "does PIN follow MerchantTransactionService path" is
  resolved — yes, with additional per-platform validation fields
  (`termSequence`, `fromAcro`, `toAcro`).
- Clarify NAP guard: no code-level guard exists. Protection is operational
  (COMP-04 does not publish NAP events to this topic). If this assumption
  ever changes, COMP-14 will process NAP events and route them through
  non-NAP URL templates.

**WDP-NFRS.md · Section 6 Risk Register**
- Add RISK-COMP-14-A 🔴 HIGH: "CaseCreationConsumer has zero `@Transactional`
  annotations — all DB writes auto-commit. Parent SUCCESS and EVIDENCE_ATTACH
  child writes are two independent operations with no atomicity guarantee.
  Combined with DEC-005 pre-ACK, a crash between them produces silent
  inconsistency that is not self-healing."
- Add RISK-COMP-14-B 🔴 HIGH: "CaseCreationConsumer `RestTemplate` is bare-
  constructed — no connect timeout, no read timeout, no pool. Concurrency = 1
  means any hung downstream stalls the entire consumer thread. No liveness
  probe exists — pod remains Ready indefinitely. Compound with the 16+
  downstream REST dependencies."
- Add RISK-COMP-14-C 🔴 HIGH: "CaseCreationConsumer registers an empty
  `CommonErrorHandler`. Combined with `ErrorHandlingDeserializer` and
  MANUAL_IMMEDIATE pre-ACK, deserialization exceptions and any unhandled
  application exception are silently swallowed. No log line, no DLT, no
  counter. This is a silent-loss class distinct from the pre-ACK offset
  window."
- Add RISK-COMP-14-D 🟡 MEDIUM: "Four `@Recover` methods on the enrichment
  chain (`transactionLookup`, `productEntitleMentLookup`, `fraudSwitch`,
  `displayCodeDescription`) return `null` silently with a
  `// Error handling -- todo` comment. Processing continues with null fields
  and may reach SUCCESS on an under-enriched case."

**WDP-INTEGRATIONS.md**
- No change. COMP-14's external integration (Fraud Switch, AID user-detail)
  already listed. Internal dependencies are intra-platform.

#### Deviation flags for COMP-14

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⛔ DEVIATES | 🔴 HIGH |
| DEC-003 Kafka Partition Key = merchantId | ⛔ DEVIATES | 🟡 MEDIUM |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES | — |
| DEC-005 Manual Kafka Offset Commit | ⛔ DEVIATES | 🔴 HIGH |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⛔ DEVIATES | 🔴 HIGH |

**DEC-001 DEVIATES detail:** Zero `@Transactional` annotations anywhere in
`gcp-case-creation-consumer` source. Parent `chbkOutboxRepository.save()`
(SUCCESS write) and EVIDENCE_ATTACH `chbkOutboxRepository.saveAll()` (children
→ PENDING) are independent auto-commit operations. The upstream case-creation
REST call (`mdvs-gcp-case-management-service`) is a third independent
operation with no atomicity relative to either DB write. The `ChkbOutbox`
block on the case-management request body partially delegates outbox-closure
responsibility to COMP-23, making data-integrity ownership ambiguous. Retry
count promotion (`retry_count > 2 → ERROR`) is Java-side logic — any direct-DB
manipulation of the row bypasses it.

**DEC-003 DEVIATES detail:** Producer-side (COMP-12) already records the
deviation — compound key `networkCaseId + cardNetwork + platform`. Consumer-
side confirmed 2026-04-18: key received as `keyNetworkCaseCardNetworkId` and
logged only; not used for routing inside the consumer. Per-partition ordering
is scoped to the compound key, not to `merchantId`.

**DEC-005 DEVIATES detail:** `acknowledgment.acknowledge()` is called at
KafkaConsumer.java:38, immediately before `processKafkaNotificationEvent(...)`
at line 43. `AckMode.MANUAL_IMMEDIATE` with `syncCommits = true`. At-most-once
from the moment the listener enters. Combined with the empty
`CommonErrorHandler` and `auto.offset.reset = latest`, this produces three
distinct silent-loss classes: (1) deserialization errors swallowed by
`CommonErrorHandler`; (2) crashes between ACK and any outbox write; (3) cold
starts skipping any uncommitted offset.

**DEC-014 DEVIATES detail:** No Resilience4j dependency on classpath. No
`@CircuitBreaker`, `@Retry` (in the Resilience4j sense), `@RateLimiter`. The
only resilience is Spring-Retry `@Retryable` + `@Recover` at the individual
REST-method level — 3 of 7 `@Recover` methods write FAILED/ERROR; 4 absorb
failures by returning null with no outbox write. No timeouts on
`RestTemplate` — a hung downstream blocks the consumer thread indefinitely.
Platform-wide VOID already recorded.

**DEC-020 DEVIATES detail:** Multiple compounding violations of at-least-once
idempotency — pre-ACK (W2 crash-window loss), non-atomic parent-child writes
(W4/W5 inconsistency), Layer-1 dedup done in application memory over a
2-column query with no DB lock, silent-null `@Recover` stubs on enrichment,
empty `CommonErrorHandler` on deserialization, and reliance on downstream
services' un-verified idempotency via `idempotency-key` HTTP header. Local
dedup for concurrent-partition-assignment safety depends entirely on
operational concurrency = 1.

#### Doc status after this change
- `WDP-COMP-14-CASE-CREATION-CONSUMER.md` → `v2.0 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending

#### Remaining gaps

- **OQ-14.1 Mastercard RE2 / reversal=Y handling** — follow-up Claude Code
  question against the COMP-14 repo (exact wording above under New open
  questions).
- **OQ-14.2 `preActionStatusRule` @Recover wiring** — follow-up Claude Code
  question (exact wording above).
- **OQ-14.3 Production feature-flag values** — environment config / team
  confirmation. Seven env-only flags with no YAML defaults.
- **OQ-14.4 Production replica count** — environment config / team
  confirmation. Helm/Jinja placeholder only.
- **OQ-14.5 Env-injected Kafka client tunings** — environment config / team
  confirmation. Four tunings injected, no YAML defaults.
---

### 2026-04-18 — COMP-13 FileAcknowledgementProcessor · v1.0 DRAFT → v2.0 DRAFT

**Source:** `wdp-evidence-ack-scheduler` — source-verified by Claude Code 2026-04-18.
Architect confirmation pending.

**Nature of change:** Correction pass against source. No functional change in
production; the v1.0 DRAFT (GitHub Copilot CLI extraction) contained multiple
transcription errors in the Identity block and S3 key patterns, a referenced
profile file that does not exist, and missed three latent runtime bugs and one
third `headObject` failure branch. Three findings rise to HIGH severity.

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.file_job` row**
- No change to the writer list. Clarify under COMP-13's column scope: this
  component writes exactly four columns (`ack_status`, `ack_generated_at`,
  `updated_at`, `updated_by`), enforced by `@DynamicUpdate` on the `FileJob`
  entity. The broader "never-touches" column list previously implied by the
  DRAFT is NOT verifiable from this repo — the entity class declares only a
  narrow column set (`id`, `status`, `source`, `ack_required`, `ack_status`,
  `ack_generated_at`, `total_rows`, `total_evidences`, `error_code`,
  `updated_at`, `updated_by`). Columns that may exist in the live schema but
  are not bound on this entity (e.g. `file_name`, `s3_key`, `successful_rows`,
  `completed_at`, `error_message`, `created_by`) cannot be confirmed from
  this repo.
- Add cross-component note (already present for the COMP-11 attribution but
  reinforced here): COMP-11 never writes `status = COMPLETED`; COMP-12
  Scheduler2 owns that transition per WDP-DB.md. COMP-13 filters on
  `status IN (COMPLETED, ERROR)` so the ACK pipeline depends entirely on
  COMP-12 actually flipping the status. Contract is undocumented in all
  three repos.

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- No change to writer list. Reinforce read-only consumer attribution for
  COMP-13 (already present).
- Add note: schema ownership remains outside this repo (no DDL / Flyway /
  Liquibase / schema.sql). `ddl-auto = false`. `@Id` on `id` is the only
  entity-level constraint. Unique constraints and indexes remain not
  determinable from source — same open question as COMP-07, COMP-08, COMP-11.

**WDP-KAFKA.md · Sections 3 and 4**
- No change. COMP-13 remains in Section 4's Kafka-free components list. Audit
  re-confirms zero `spring-kafka`, `kafka-clients`, `aws-msk-iam-auth` in
  `pom.xml` and zero `@KafkaListener` / `KafkaTemplate` in source.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Corrections to DRAFT Identity block (all five values wrong):
- Artifact version: `1.0.3` → **`1.0.5`**
- Spring Boot: `3.5.11` → **`4.0.3`** (parent)
- Spring Cloud AWS: `3.1.0` → **`4.0.0`** (only `spring-cloud-aws-starter-s3`
  declared)
- JAXB: `4.0.6` → not pinned; `jaxb2-maven-plugin` = `4.0.0`, `jaxb-runtime`
  resolved via Spring Boot BOM
- logstash-logback-encoder: `9.0` → **`8.1`**

Corrections to S3 key patterns (digit-zero rendered as letter-O, digit-one as
letter-L, and CMRTR/BJWC outer prefixes previously shown as identical when
they differ by one character in source):
- Walmart filename: `AUF2_DWSG_ROBCDWL1_WMSIG_CONF_` →
  **`AUE2_DWSG_R0BCDWL1_WMSIG_CONF_`**
- CMRTR outer zip: `DCPO_RODMRDOA.WP01.ACK.` → **`DCPO_R0DMRDDA.WP01.ACK.`**
- CMRTR inner XML: `WP.CL.CMRTRTROUBLE.ACK.PROD.` →
  **`WP.C1.COMMTRANTROUBLE.ACK.PROD.`**
- BJWC outer zip: was shown identical to CMRTR →
  **`DCPO_R0DMRBDA.WP01.ACK.`** (distinct — differs from CMRTR by one char:
  `DD` vs `DB`)
- BJWC inner XML: `WP.BJW.CL.TRANTROUBLE.ACK.PROD.` →
  **`WP_BJWC.C1.TRANTROUBLE.ACK.PROD.`**

Corrections to scheduler / profile facts:
- `application-local.yml` **does not exist** in repo. Only `application.yml`,
  `application-cert.yml`, `application-prod.yml`. The hardcoded local-dev
  cron `0 */1 * * * *` previously attributed to `application-local.yml` is
  not present anywhere in source. The prod/cert value is env-injected via
  `${cron_scheduler}`; which of two `envFrom` secrets
  (`wdp-evidence-ack-scheduler-secrets` or `wdp-common-secrets`) supplies
  it is not determinable from source.

Corrections to flow / skip semantics:
- **DBLK has NO explicit skip branch.** v1.0 DRAFT implied one. Source shows
  `BulkResponseAckServiceImpl.generateAck` calls `uploadAckFileToS3`
  unconditionally with whatever `buildFile(...)` returned; on `IOException`
  the formatter returns `null` and the service does not guard, so `null`
  reaches the S3 upload path and is swallowed there.
- **`headObject` has THREE outcomes, not two.** v1.0 DRAFT flowchart
  showed "exists / not-found" binary. Source shows: (a) 200 exists → no
  upload; (b) `NoSuchKeyException` → proceed to `putObject`; (c) any other
  Exception → generic catch at `CommonAckServiceImpl`, returns `false`,
  no upload, `ack_status` stays `PENDING`. Branch (c) is a distinct
  stuck-job-adjacent path.
- `performTask` has an **outer catch** that aborts the entire cron fire on
  any uncaught exception from the poll or loop wrapper. A single bad job or
  a poll failure starves the whole batch until the next cron.

Add: three new latent bugs not surfaced in v1.0:
- **DBLK `yyyyMMddHHmmSS` pattern bug.** `BulkResponseConstants` timestamp
  pattern uses capital `SS` (fraction-of-second in `DateTimeFormatter`), not
  seconds. Formatting a `LocalDateTime.now()` throws
  `UnsupportedTemporalTypeException` at runtime. Every DBLK ACK attempt may
  currently be failing at the formatter stage, with the exception swallowed
  by the service-level generic catch. Severity 🔴 HIGH. Requires prod log
  evidence to confirm production-active impact — no test coverage because
  `src/test/` does not exist.
- **`uploadAckFileToS3` guard bug — `||` should be `&&`.** Guard
  `if (!key.isEmpty() || content != null)` lets null content or empty key
  through; downstream S3 SDK throws, caught by generic catch. Severity
  🟡 MEDIUM. Obscures root cause of any stuck-by-null-content case.
- **CapitalOne minute-resolution stuck-job.** CapitalOne S3 key timestamp
  truncates to `yyyyMMddHHmm`. Two CapitalOne ACK jobs processed within the
  same calendar minute in the same JVM produce the **same S3 key**. The
  second job's `headObject` returns 200, skips upload, never reaches the
  DB write — stuck on first encounter, not just after a failure.
  Deterministic failure mode under throughput > 1 CapitalOne ACK / minute.
  Severity 🔴 HIGH.

Add: `UniqueTimestampGenerator` behaviour.
- JVM-local `AtomicReference<LocalDateTime>` initialised to `LocalDateTime.MIN`.
  `generate()` is `synchronized`; returns `max(now(), last.plusSeconds(1))`.
- Minimum spacing 1 second inside one JVM. Not pinned to any input.
- Pod restart resets the counter.
- Service callers immediately re-format with a per-source
  `DateTimeFormatter` — this is what collapses the minimum-1-second
  guarantee down to seconds / minute resolution per source.

Add: `@DynamicUpdate` on `FileJob` entity — emits only dirty columns in
UPDATE. The write set is constrained by the method, but shared-column
collision protection relies on this annotation.

Add: TaskScheduler pool size = 1 (Spring Boot default, not explicitly set).
Intra-JVM overlapping cron fires are **queued**, not concurrent.

Add: dead config / hygiene findings.
- `spring.cloud.aws.region.static: "us-east-2"` is present in cert and prod
  profiles but not read; `S3ClientConfig` hardcodes `Region.US_EAST_2`.
- Redundant `jackson-core` re-declaration in `pom.xml` (already resolved via
  Spring Boot BOM).
- `spring-cloud-aws-starter-s3` autoconfig is shadowed by a manual
  `S3Client` bean in `config/S3ClientConfig.java`. Candidate for replacing
  with just the raw SDK module.
- Stale hardcoded Logstash destination (`10.43.145.125:5044`) commented out
  in `logback-spring.xml:17-18`.
- `@Configuration` annotation on both `FileJob` and `ChbkOutboxRow`
  entities — unintended, no observed downstream impact.
- No `src/test/` — zero tests of any kind.

Resolved open questions (from v1.0 DRAFT):
- "CapitalOne 2000-element cap behaviour" — RESOLVED. 2000 is XSD-only
  (`maxOccurs="2000"`). Java has no slicing, no truncation, no guard.
  Marshaller not configured with `setSchema`, so no runtime validation.
  Full list marshalled regardless; downstream receives XSD-invalid file
  if count > 2000.
- "CapitalOne MessageType four-way branch" — RESOLVED. `claim-request` and
  `outcome-chargeback` handled correctly; `claim-response` and
  `outcome-no-chargeback` fall through catch-all `else` to
  `status=REJECTED, validReject=false`. No tests cover either.
- "Does `headObject` throw anything other than `NoSuchKeyException` get
  handled?" — RESOLVED. Third branch exists (generic catch), returns
  `false`, no upload, ack_status stays PENDING.
- "Is @Transactional meaningful for a single save?" — RESOLVED. Propagation
  REQUIRED, default isolation (effectively PG `READ COMMITTED`). Rolls back
  the single save on throw, which is semantically inert because there is
  nothing else to roll back.
- "TaskScheduler behaviour on overlapping fires" — RESOLVED. Default pool
  size 1; overlapping fires are queued.

New open questions to add to the platform list:
- OQ: Is the DBLK `yyyyMMddHHmmSS` pattern actually failing in production,
  or has a pre-prod path been patched that never propagated to source?
  Requires prod log evidence from Logstash / ELK.
- OQ: Has the CapitalOne minute-resolution stuck-job been observed in
  production? Requires confirmation of current CapitalOne ACK volume vs
  1-per-minute threshold from Integration Team.
- OQ: Do `claim-response` and `outcome-no-chargeback` MessageType values
  ever appear in production CapitalOne data? Requires confirmation from
  Integration Team.
- OQ: Production replica count (XL Deploy placeholder). If > 1, HIGH
  concurrency-race risk is live.
- OQ: Production cron expression value.
- OQ: Which of `wdp-evidence-ack-scheduler-secrets` or `wdp-common-secrets`
  supplies `cron_scheduler`.
- OQ: Whether upstream (COMP-11 Walmart path) can write unmasked PAN into
  `chbk_outbox_row.record_detail.walmartSigCap.recordData`. DEC-004 /
  DEC-019 verdict at component egress depends on this. Requires upstream
  data analysis.
- OQ: DDL-level unique constraints on `wdp.file_job` and
  `wdp.chbk_outbox_row` (same open question as COMP-07 / COMP-08 / COMP-11).
- OQ: Cross-component ownership of `file_job.status = COMPLETED`
  transition. WDP-DB.md attributes to COMP-12 Scheduler2. Contract is
  undocumented in any of COMP-11, COMP-12, COMP-13 repos.

**WDP-DECISIONS.md · Candidate new ADRs**

- **Candidate new RISK (🔴 HIGH):** "Silent runtime bug via unreviewed
  `DateTimeFormatter` pattern in DBLK ACK generation — zero test coverage
  across the ACK component allowed a capital-`SS` typo to potentially reach
  production." Applies specifically to COMP-13 but surfaces a platform-wide
  concern: no test directory exists in this repo at all. Raise alongside
  any test-strategy ADR.
- **Candidate new RISK (🔴 HIGH):** "Minute-resolution S3 key timestamp
  collapse — CapitalOne ACK generation for two jobs in the same calendar
  minute produces the same S3 key, silently stranding the second job."
  Specific to COMP-13 CapitalOne path.
- **DEC-010 (Immutable Versioned ACK Snapshots) — candidate review.** The
  timestamp-in-S3-key pattern does not reliably produce unique versions at
  the precision the formatters actually use (seconds / fraction-buggy /
  minute). Needs restated, clarified, or voided. Raise as a dedicated ADR
  update.
- **DEC-023 scope extension:** extend the covered-component list to
  include COMP-13. Pattern matches COMP-07, COMP-08, COMP-09 — no code-level
  guard, replica=1 is operational policy only.
- **DEC-001 deviation map:** add COMP-13 — S3 write + DB write pair with
  no outbox, no saga, no compensating path. Severity 🔴 HIGH due to the
  stuck-job scenario.
- **DEC-020 deviation map:** add COMP-13 — three distinct at-least-once
  gaps (stuck-job, cross-replica race, CapitalOne minute collapse).
  Severity 🔴 HIGH.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- No new rows required from the RISK-013 / RISK-014 platform-wide coverage
  perspective.
- Minor clarification on RISK-013 (replica constraint has no automated
  enforcement): extend covered-components list to include COMP-13
  explicitly — pattern matches COMP-07, COMP-08, COMP-09.
- Candidate new RISK row pending architect decision on the CapitalOne
  minute-resolution failure mode: "Throughput-dependent silent ACK stuck-job
  on CapitalOne path — activates when > 1 CapitalOne ACK per calendar
  minute." HIGH.
- Candidate new RISK row pending architect decision on the DBLK
  `SS` pattern: "Unreviewed `DateTimeFormatter` pattern in DBLK ACK
  generation throws `UnsupportedTemporalTypeException` at runtime — no test
  coverage catches it." HIGH.

**WDP-INTEGRATIONS.md**
- No change. COMP-13 has no external integration contract (S3 is internal
  WDP infrastructure; ControlM / Sterling Mailbox / DM Mainframe handoff is
  out of this component's scope and already documented under COMP-10
  placeholder).

#### Deviation flags for COMP-13

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⛔ DEVIATES — not using outbox | 🔴 HIGH |
| DEC-003 Kafka Partition Key = merchantId | ✅ N/A | — |
| DEC-004 PAN Encryption Before Persistence | ⚠️ UNVERIFIABLE (Walmart pass-through) | 🟡 MEDIUM |
| DEC-005 Manual Kafka Offset Commit | ✅ N/A | — |
| DEC-014 Resilience4j Circuit Breaker | ⛔ ABSENT (void platform-wide) | 🟡 MEDIUM |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES at component scope | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-022 `removeItemFromQueueDisabled` Safety Switch | ✅ N/A | — |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY | 🟡 MEDIUM |

**DEC-001 detail:** S3 `putObject` and DB `updateAckStatus` are two
independent operations separated by a `@Transactional` boundary. No outbox
table, no mark-before-send, no saga, no compensating path. `headObject` gate
partially prevents duplicates but actively blocks recovery in the stuck-job
scenario. No source-level comment acknowledges this as deliberate.

**DEC-004 UNVERIFIABLE detail:** Walmart `recordData` is an opaque
pass-through from `chbk_outbox_row.record_detail`. Component does not
inspect, mask, or encrypt. Three non-Walmart formats carry no card-number
fields. PAN presence at ACK egress is an upstream data question and cannot
be resolved from this repo alone.

**DEC-020 PARTIAL detail — escalated to HIGH for COMP-13.** Three
independent gaps combine:
(1) stuck-job when `putObject` succeeds and DB write fails — no automatic
    recovery, `headObject=200` on next cron blocks retry;
(2) cross-replica race — if replicas > 1, two pods can both pass the
    `headObject` 404 window before either completes `putObject`;
(3) CapitalOne minute-resolution collapse — two CapitalOne ACKs in the same
    calendar minute produce the same S3 key; second is stuck on first
    encounter, not just after a failure.

**DEC-023 OPERATIONAL ONLY detail:** Same pattern as COMP-07, COMP-08,
COMP-09. No `@SchedulerLock`, no ShedLock dependency, no advisory lock, no
synchronized guard, no `@Version`. Replicas value is an XL Deploy
placeholder with no static `replicas: 1` assertion in the manifest.
Replica=1 is policy, not code.

#### Remaining gaps

- **OQ-Prod-Replicas:** Environment config — XL Deploy / deployit value for
  `{{ replicas-wdp-evidence-ack-scheduler }}` in each environment. Not in
  repo.
- **OQ-Prod-Cron:** Environment config — actual cron expression bound to
  `${cron_scheduler}` in production and cert. Not in repo.
- **OQ-Secret-Source:** Environment config — which of
  `wdp-evidence-ack-scheduler-secrets` or `wdp-common-secrets` supplies
  `cron_scheduler`. Not in repo.
- **OQ-DBLK-SS-Pattern:** Follow-up Claude Code or team confirmation —
  *"Does production Logstash / ELK show any
  `UnsupportedTemporalTypeException` or formatter-related failure coming
  from `wdp-evidence-ack-scheduler` BulkResponse path over the last 30
  days? If zero, is there a runtime-reachable branch that bypasses the
  `yyyyMMddHHmmSS` pattern? Cite file:line."*
- **OQ-CapitalOne-MinuteRace:** Team confirmation — is CapitalOne ACK
  volume ever > 1 per calendar minute in production? If yes, the
  minute-resolution stuck-job is active.
- **OQ-CapitalOne-MessageTypes:** Integration Team — do
  `claim-response` or `outcome-no-chargeback` MessageType values ever appear
  in production CapitalOne RTSI data? If yes, the catch-all REJECTED branch
  is a confirmed bug, not theoretical.
- **OQ-Walmart-RecordData-PAN:** Upstream data analysis — can
  `chbk_outbox_row.record_detail.walmartSigCap.recordData` contain
  unmasked PAN in production? Requires COMP-11 source + upstream data
  inspection. Not resolvable from this repo.
- **OQ-DDL-Constraints:** Schema-owner / DBA confirmation — unique
  constraints and indexes on `wdp.file_job` and `wdp.chbk_outbox_row`. Not
  in any component repo. Same open question as COMP-07 / COMP-08 / COMP-11.
- **OQ-FileJob-COMPLETED-Owner:** Architect decision — cross-component
  contract for `file_job.status = COMPLETED`. WDP-DB.md attributes to
  COMP-12 Scheduler2. Follow-up Claude Code question on COMP-12 source:
  *"Does `InboundDisputeEventScheduler` (COMP-12) update
  `wdp.file_job.status` to COMPLETED after successfully publishing all
  `chbk_outbox_row` rows for a given `file_job_id`, and does that
  transition trigger under both COMP-11 source profiles (DCPO/DNWK/DWSG/DBLK)
  uniformly? Cite file:line. If not, which component owns this transition
  and does every ACK-required source reach it?"*
- **OQ-DEC-010-Review:** Architect decision — is DEC-010 Immutable
  Versioned ACK Snapshots still in force given per-source timestamp
  precision (seconds / fraction-buggy / minute)?
- **OQ-Test-Strategy:** Architect decision — zero tests of any kind
  in `wdp-evidence-ack-scheduler`. Is this acceptable for a
  production component?

#### Doc status after this change
- `WDP-COMP-13-FILE-ACK-PROCESSOR.md` → `v2.0 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending. Supersedes v1.0 DRAFT.
---

### 2026-04-18 — COMP-12 InboundDisputeEventScheduler · v1.0 DRAFT → v1.1 DRAFT

**Source:** `wdp-chargeback-evidence-event-scheduler` — source-verified by
Claude Code 2026-04-18. Architect confirmation pending.

**Nature of change:** Correction pass — audit re-characterised the
transaction semantics of the Kafka relay (at-least-once, not at-most-once),
corrected one channel-map value, and confirmed several previously-open
items (K8s probes absent, bare `RestTemplate` everywhere, asymmetric
Kafka-metadata write-back across the five topics).

#### Platform-level impacts

**WDP-DB.md**
- `wdp.chbk_outbox_row` row: no change to COMP-12 ownership scope. Add
  note: `idempotencyId` is typed as `UUID` on this table's entity.
- `wdp.outgoing_event_outbox` row: add note — `idempotencyId` is typed
  as `String` on this table's entity. **Cross-table type inconsistency
  with `chbk_outbox_row` (UUID) — flag for downstream consumers that do
  shared dedup logic.**
- `wdp.chbk_outbox_row_archive` row: add note — archive INSERT SQL
  references column `c_ntwrk_phase_id` that the `ChbkOutboxArchiveEntity`
  does not map. Either dead column reference or incomplete entity.
  **DDL verification required** (schema lives outside this repo).
- Shared Table Risk Register row for `wdp.chbk_outbox_row`: no change —
  writers and mitigations remain as documented.

**WDP-KAFKA.md**
- Section 3 (Topic Registry) row for `case-action-events` /
  `core-request-events` / `external-request-events`: correct the
  `channelTypeTopicMap` non-prod default mapping from `SEN_EVENTS` to
  **`BEN_EVENTS`**. Prior v1.0 DRAFT listed `SEN_EVENTS` — this was a
  transcription error. Source: `application-local.yml:14`.
- Section 3 rows for the five topics COMP-12 publishes to: add a note
  on Kafka-metadata write-back asymmetry — `new-case-events` and
  `case-evidence-events` have `kafka_offset` / `kafka_partition` /
  `kafka_topic` persisted back to the outbox row; the other three paths
  (Scheduler3 and Scheduler4) pass a NULL entity and **do not persist
  Kafka metadata** to `outgoing_event_outbox` or
  `bre_orchestration_outbox`.
- Section 4 (Consumer Groups): no change — COMP-12 has no consumer.
- Section 5 (Outbox tables that feed Kafka): existing rows for
  `chbk_outbox_row`, `outgoing_event_outbox`, `bre_orchestration_outbox`
  remain accurate.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add / correct:
    - COMP-12 uses mark-and-send **within a single `@Transactional`** —
      broker ack precedes TX commit. This is **at-least-once with
      duplicate-possible crash window**, NOT at-most-once with
      silent-loss window as v1.0 DRAFT stated. Consumer-side
      `idempotency-key` dedup is the intended mitigation.
    - COMP-12 has **no Kubernetes probes** (liveness, readiness,
      startup all absent in `resources.yml`). Previously listed as
      an open question — now concrete.
    - COMP-12 uses a **bare `RestTemplate`** in `config/CommonConfig.java`
      for the Scheduler5 email POST — no connect timeout, no read
      timeout, no connection pool. Stalled peer blocks a scheduler
      thread indefinitely.
    - COMP-12 has a **shared `ThreadPoolTaskScheduler` with poolSize=5**,
      across five schedulers. Sibling-starvation risk under load or
      slow peer.
    - COMP-12 `channelTypeTopicMap` non-prod default keys are
      `EXPIRY_EVENTS`, `CORE_EVENTS`, `GP_EVENTS`, `BEN_EVENTS`
      (NOT `SEN_EVENTS`).
    - COMP-12 has **asymmetric Kafka-metadata write-back**: Chargeback
      and Evidence paths persist offset/partition/topic back to the
      outbox row within the same TX; Outgoing and BRE paths pass a
      NULL entity — operators cannot trace outgoing/BRE Kafka messages
      back to their outbox rows via offset.
    - COMP-12 retry policy is **fixed 1-hour backoff, 3 attempts** —
      `ApplicationConstants.NEXT_RETRY = 3600000L`. Not exponential.
    - COMP-12 has **no Spring Batch** (no `@EnableBatchProcessing`,
      no `BATCH_*` metadata tables, no `Job` / `Step` / `ItemReader` /
      `ItemWriter`), and no `fixedDelay` / `fixedRate` — all five
      triggers are cron expressions.
    - COMP-12 `ChbkOutboxEntity.idempotencyId` is `UUID` while
      `OutgoingEventOutboxEntity.idempotencyId` is `String` —
      cross-table type inconsistency.
    - COMP-12 has **no application-level observability** — OTel agent
      is injected by K8s annotation but no code-level MDC, no
      Micrometer registry, no custom meters.
    - COMP-12 S3 credentials use `StaticCredentialsProvider` from K8s
      secrets (migration away from `InstanceProfileCredentialsProvider`
      was completed — old IAM code retained commented).
    - COMP-12 has `Region.US_EAST_2` hardcoded in
      `S3PresignerConfiguration` — multi-region blocker.
    - COMP-12 has two `@Value` / yml path mismatches:
      (1) `S3PresignerConfiguration` reads `app.aws.accesskey` /
          `app.aws.secretkey`; yml defines `app.s3.aws.*`.
      (2) `EvidenceErrorEmailServiceImpl` reads
          `app.presignedUrlExpiryTime`; yml defines
          `app.s3.presignedUrlExpiryTime`.
      Both would fail Spring startup unless env-overrides resolve them.
- Resolved open questions (remove from HANDOVER):
    - COMP-12 transaction semantics (mark-before-send vs in-TX-with-send)
      — now documented; in-TX-with-send confirmed.
    - COMP-12 K8s probes presence — now documented; confirmed absent.
    - COMP-12 `RestTemplate` timeout config — now documented;
      confirmed bare.
    - COMP-12 retry backoff shape — now documented; fixed 1h confirmed.
    - COMP-12 Spring Batch presence — now documented; confirmed absent.
    - COMP-12 thread model — now documented; `ThreadPoolTaskScheduler`
      poolSize=5 confirmed.
- New open questions (add to HANDOVER):
    - Production replica count for COMP-12 — XLD placeholder
      `{{ replicas-wdp-chargeback-evidence-event-scheduler }}` not
      visible in source. **Any value > 1 is an unmitigated 🔴 HIGH
      concurrency race** — no ShedLock, no `SELECT FOR UPDATE`, no
      advisory lock, no synchronized guard.
    - Production cron values for all five schedulers — all externalised
      to K8s secrets (`chargeback_evidence_scheduler_cron`,
      `file_completion_scheduler_cron`, `outgoing_scheduler_cron`,
      `bre_outbox_event_scheduler_cron`, `evidence_email_scheduler_cron`).
    - Production `channelTypeTopicMap` complete mapping — K8s secret
      `${channel_topic_map}`; non-prod shows four channels.
    - `PUBLISHED → SUCCESS` transition owner for `chbk_outbox_row`
      CHARGEBACK_PROCESS rows — likely COMP-14 CaseCreationConsumer,
      but not verified from either repo. The Scheduler1 Phase 2
      unblock query depends on this transition.
    - Initial `PENDING` writer(s) for `wdp.outgoing_event_outbox` and
      `wdp.bre_orchestration_outbox` — COMP-17 / COMP-18 / COMP-43
      candidates identified, not verified against source of those
      components.
    - Archive-table column `c_ntwrk_phase_id` — present in
      `ChbkOutboxArchiveRepository` native INSERT, absent from the
      entity. DBA / DDL verification required.
    - Two `@Value` / yml path mismatches — are env-var overrides or
      profile-specific properties resolving them in deployed
      environments? Dev team confirmation required.
    - Actuator endpoint exposure on port 8082 —
      `management.endpoints.web.exposure.include` value not captured
      this pass; needed for probe wiring plan.

**WDP-DECISIONS.md · Candidate new ADRs**
- **MEDIUM candidate (cross-component):** "Outbox relays that mark-and-
  send within a single `@Transactional` accept a duplicate-possible
  crash window between broker-ack and TX-commit; consumer-side
  `idempotency-key` dedup is the contracted mitigation." Explicitly
  recognising this as an intentional design pattern (vs DEC-001's
  strict relay model) across COMP-12 and any future outbox relays
  would be useful. Recommend raising at next DECISIONS rebuild.
- **MEDIUM candidate:** "No Kubernetes probes on continuously-running
  Deployment-hosted schedulers" — confirmed on COMP-12. Pattern likely
  matches other scheduler components (COMP-13 at minimum). Recommend
  cross-component re-audit before formalising an ADR.
- **MEDIUM candidate:** "`idempotencyId` column typing is inconsistent
  across outbox tables (`UUID` on `chbk_outbox_row`, `String` on
  `outgoing_event_outbox`)." Either normalise to one type or document
  the divergence as intentional per table domain.
- DEC-001 deviation map: add COMP-12 as a **compliant variant** (not
  a deviation) — mark-and-send-within-TX is a valid outbox-relay
  implementation. Remove any prior "PARTIAL deviation / silent-loss"
  framing if present.
- DEC-003 deviation map: add COMP-12 entry — five topics published,
  all with non-`merchantId` keys (networkCaseId+cardNetwork+platform,
  caseNumber, caseNumber/networkCaseId).
- DEC-020 deviation map: add COMP-12 partial — no replica lock;
  terminal ERROR rows have no DLQ; `PUBLISHED` rows with failed
  metadata UPDATE are stranded (no scheduler re-queries `PUBLISHED`).

**WDP-ARCHITECTURE.md**
- No change to platform topology. COMP-12 remains the sole Kafka
  producer for the inbound relay chain and a retry relay for the
  outgoing / BRE outbox tables.
- Section 6 (Kafka Event Bus) — minor: when describing the
  `channelTypeTopicMap`, use `BEN_EVENTS` not `SEN_EVENTS` in any
  example mapping.

**WDP-NFRS.md · Section 6 Risk Register**
- **RISK-015** (`bre_orchestration_outbox` PUBLISHED orphan rows —
  no auto-redrive) — already present; covers COMP-12 / COMP-18. Extend
  the note to mention the asymmetric Kafka-metadata write-back (NULL
  entity for Scheduler4 means PUBLISHED-status orphans carry no offset
  breadcrumb — makes orphan identification harder).
- **New RISK candidate:** "No Kubernetes health probes on COMP-12 —
  continuously-running Deployment with five cron schedulers; a
  deadlocked JVM or broken DB connection is invisible to Kubernetes."
  🔴 HIGH. Likely applies to other scheduler components — cross-audit
  before formalising.
- **New RISK candidate:** "Bare `RestTemplate` (no timeouts, no pool)
  on COMP-12's Scheduler5 email POST — sibling-starvation risk on
  the shared 5-thread pool if the email relay is slow." 🔴 HIGH in
  interaction with the shared pool.
- **New RISK candidate:** "COMP-12 has no replica guard (no ShedLock,
  no `SELECT FOR UPDATE`, no advisory lock). Replica=1 is operational
  policy, not code-enforced." Extend RISK-013 to explicitly cover
  COMP-12 (current coverage: COMP-07, COMP-08) — or create a new
  parallel RISK if DEC-023 is considered scoped to Spring-Batch
  polling components only.
- **New RISK candidate (MEDIUM):** "COMP-12 asymmetric Kafka-metadata
  write-back — Scheduler3 and Scheduler4 paths pass NULL entity, so
  `outgoing_event_outbox` and `bre_orchestration_outbox` rows never
  record Kafka offset/partition/topic. Incident correlation degraded."
- **New RISK candidate (MEDIUM):** "COMP-12 `idempotencyId` type
  inconsistency across outbox tables — UUID vs String. Shared dedup
  logic on consumer side must handle both."
- **New RISK candidate (MEDIUM):** "COMP-12 archive-table SQL
  references unmapped column `c_ntwrk_phase_id` — either DDL column
  exists and entity is incomplete, or column is missing and archive
  fails at runtime. DBA verification required."

**WDP-INTEGRATIONS.md**
- No change. COMP-12 has no external integrations — all Kafka
  targets are WDP-owned MSK, the email relay is an internal WDP
  endpoint, and S3 is internal. No Visa / MasterCard / CORE / NAP
  integration contract changes.

#### Deviation flags for COMP-12

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ COMPLIES (compliant variant — mark-and-send within single `@Transactional`) | — |
| DEC-003 Kafka Partition Key = merchantId | ⛔ DEVIATES (all five topics) | 🟡 MEDIUM |
| DEC-004 PAN Encryption Before Persistence | ✅ NOT APPLICABLE (no PAN) | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE (producer only, no consumer) | — |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ✅ NOT APPLICABLE (no PAN) | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY (no code-level guard) | 🔴 HIGH |

**DEC-001 compliant-variant detail:** Upstream writers (COMP-07, 08, 09,
11 for `chbk_outbox_row`; COMP-17, 18, 43 for `outgoing_event_outbox`;
COMP-18 for `bre_orchestration_outbox`) perform the strict outbox write
atomically with their business write. COMP-12 is the relay layer. The
mark-and-send within a single `@Transactional` method (broker ack before
TX commit) is a valid outbox-relay implementation — at-least-once with a
duplicate-possible crash window between broker-ack and TX-commit, handled
by consumer-side `idempotency-key` deduplication. v1.0 DRAFT's "PARTIAL /
silent-loss" characterisation was incorrect and has been reversed.

**DEC-003 DEVIATES detail:** All five topics use non-`merchantId` keys.
`new-case-events` uses `networkCaseId + cardNetwork + platform`.
`case-evidence-events` uses `caseNumber` with fallback to `networkCaseId`.
`case-action-events` / `core-request-events` / `external-request-events`
(dynamic via `channelTypeTopicMap`) use `caseNumber`. `business-rules`
uses `caseNumber`. `outgoing-events` uses `caseNumber`. All consumers
downstream receive case-scoped ordering, not merchant-scoped.

**DEC-014 DEVIATES detail:** No Resilience4j on classpath. Bare
`RestTemplate` with no connect/read timeouts, no pool. No circuit
breaker anywhere. Only resilience is Kafka-client `retries=3` on send.

**DEC-020 PARTIAL detail:** Four concurrent gaps: (a) no replica lock,
so replicas > 1 produce cross-replica duplicates (producer idempotence
dedupes within a session only); (b) Kafka metadata UPDATE failure at
Scheduler1 Step 4 leaves the row at PUBLISHED with null metadata and
no scheduler re-selects PUBLISHED rows; (c) terminal ERROR rows on
any of the four outbox tables have no DLQ or re-drive path; (d)
Scheduler3 and Scheduler4 do not persist Kafka metadata (NULL entity
passed), degrading incident correlation on the outgoing/BRE paths.

**DEC-023 OPERATIONAL ONLY detail:** No `@SchedulerLock`, no ShedLock
dependency, no advisory lock, no synchronized guard, no `SELECT FOR
UPDATE`. The shared `ThreadPoolTaskScheduler` with poolSize=5 does not
provide cross-pod mutual exclusion. Horizontal scaling is unsafe as
built — replica=1 is operational policy, not enforced by code. Suggest
extending DEC-023's covered-components list to include COMP-12, or
create a parallel ADR if DEC-023 is considered scoped to Spring Batch
polling components only.

#### Remaining gaps

- **OQ: Production replica count** — XLD placeholder not auditable
  from repo. Team / XLD config confirmation. **Any value > 1 is 🔴
  HIGH and active.**
- **OQ: Production cron values** (five) — K8s secrets, not auditable
  from repo. Team confirmation.
- **OQ: Production `channelTypeTopicMap`** — K8s secret, non-prod
  shows four channels. Team confirmation.
- **OQ: `PUBLISHED → SUCCESS` transition owner** on `chbk_outbox_row`
  CHARGEBACK_PROCESS rows. Follow-up Claude Code question against
  the COMP-14 repo: *"Search `wdp-case-creation-consumer` (or the
  COMP-14 repo) for any write of `chbk_outbox_row.status='SUCCESS'`.
  Report the exact condition, transaction boundary, and timing
  relative to the downstream case creation."*
- **OQ: Initial PENDING writer** for `outgoing_event_outbox` and
  `bre_orchestration_outbox`. Follow-up Claude Code question for
  each candidate: *"In `<repo>`, search for any INSERT / `save()`
  that creates a fresh `OutgoingEventOutboxEntity` or
  `BreOrchestrationOutboxEntity` with `status=PENDING`. Report the
  exact condition, transaction boundary, and discriminator value
  (channel_type / component)."* Candidates: COMP-17, COMP-18,
  COMP-43 for outgoing; COMP-18 for BRE.
- **OQ: Archive column `c_ntwrk_phase_id`** — DBA / schema-owner
  confirmation whether the column exists in DDL. If yes, add mapping
  to `ChbkOutboxArchiveEntity`; if no, remove from archive SQL.
- **OQ: `@Value` / yml path mismatches in production** — do env-var
  overrides or profile-specific properties resolve
  `app.aws.accesskey` / `app.aws.secretkey` and
  `app.presignedUrlExpiryTime` in deployed environments? Dev team
  confirmation.
- **OQ: Actuator endpoints exposed on port 8082** — not captured this
  pass. Follow-up Claude Code question: *"Read
  `application*.yml` and any `management` config class; report
  `management.endpoints.web.exposure.include` value and any
  endpoint-specific enable / disable settings."*

#### Doc status after this change
- `WDP-COMP-12-INBOUND-EVENT-SCHEDULER.md` → `v1.1 DRAFT` —
  source-verified 2026-04-18 · architect confirmation pending
---

### 2026-04-18 — COMP-11 FileProcessor · v1.0 DRAFT → v1.1 DRAFT

**Source:** `wdp-file-processor` — source-verified by Claude Code 2026-04-18.
Architect confirmation pending.

**Nature of change:** Correction pass — audit corrected several facts in the
prior DRAFT (created_by attribution, MC_REVREJ routing status, CHARGEBACK_PROCESS
status-flow model, DCPO empty-catch orphan risk, DNWK silent-row-loss pattern,
DEC-004 non-numeric acctNum branch) and closed several previously-open items.

#### Platform-level impacts

**WDP-DB.md**
- `wdp.file_job` row: correct `created_by` from `"WPFLEPR"` to `"FILE_PROCESSOR"`.
  Correct `updated_by` likewise. Add note: this component owns only the columns
  `s3_key, file_name, s3_bucket, file_size_bytes, source, status, total_rows,
  total_evidences, ack_required, ack_status, error_code, error_message,
  created_by, updated_by`. It does not write `completed_at, successful_rows,
  failed_rows, error_rows, attached_evidences, failed_evidences` — those
  belong to downstream components sharing the table.
- `wdp.file_evidence` row: correct `created_by` from `"WPFLEPR"` to
  `"FILE_PROCESSOR"`.
- `wdp.chbk_outbox_row` row: correct `created_by` from `"WPFLEPR"` to
  `"FILE_PROCESSOR"` in the COMP-11 attribution. No change to writer list.
  Clarify: entity in this repo does not declare `kafka_topic / kafka_partition /
  kafka_offset / idempotency_id / retry_count / next_retry_at / published_at` —
  consistent with COMP-12 being sole setter. Initial CHARGEBACK_PROCESS status
  is PENDING (PrePersist); LOADING is transient during evidence phase; BLOCKED
  applies only to DCPO evidence.
- Shared-write risk table row for `wdp.chbk_outbox_row`: no change to severity.
  Note the new per-record-loss gap in DNWK handlers under "Mitigations".
- Shared-write risk table row for `wdp.file_job`: no change to severity.
  Note COMP-11 never writes COMPLETED → cross-component break with COMP-13
  already captured; unchanged.

**WDP-KAFKA.md**
- No change. COMP-11 remains in Section 4 Kafka-free components list with the
  note "SQS-triggered — uses Spring Cloud AWS SQS listener, not Kafka. Zero
  Kafka infrastructure. Confirmed from source." Audit re-confirms zero
  `spring-kafka`, `kafka-clients`, `spring-cloud-stream` in pom.xml and zero
  `@KafkaListener` / `KafkaTemplate` in source.

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add: COMP-11 `created_by` constant is `"FILE_PROCESSOR"` on all three
  owned/shared tables (not `"WPFLEPR"` as v1.0 DRAFT stated).
- Add: DNWK silent-file-loss paths are **two** — `DISCHYB` (no bean
  registered for `DISCHYB_NETWORK` qualifier) and `AMEXOPTB` (no `FileAcroEnum`
  prefix entry). `MC_REVREJ` **is** registered (`McRevRejNetworkFileServiceImpl`)
  — v1.0 DRAFT incorrectly listed it as silent-loss.
- Add: COMP-11 initial CHARGEBACK_PROCESS status is PENDING via PrePersist.
  LOADING appears only transiently during the evidence phase (parent flipped
  by `insertEvidenceOutBox` @Transactional). DCPO evidence rows are explicitly
  BLOCKED; DCPO parent stays PENDING. DNWK writes CHARGEBACK_PROCESS at PENDING
  directly with no evidence phase.
- Add: Only two `@Transactional` sites exist in the whole repo —
  `insertEvidenceOutBox` and `updateStatusToPendingByFileJobIdAndStatusLoading`.
  Everything else is auto-commit Hibernate.
- Add: DCPO CHARGEBACK_PROCESS save is auto-commit AND the per-image evidence
  loop is wrapped in an empty catch block — orphan-parent risk on any image
  failure, no observability signal.
- Add: DNWK per-record failure path is a one-retry-then-silent-row-loss
  pattern — on second save exception, `rowCount` has already advanced and
  resume logic skips the row.
- Add: `RestTemplate` is constructed with no customization — no connect
  timeout, no read timeout, no pooling. `IdpRestInvoker` has no `@Retryable`.
- Add: DEC-004 has a non-numeric-acctNum edge case — `NetworkFileSupport`
  encrypts only when `acctNum` matches `\d+`; otherwise passes raw into
  payload.account_number.
- Add: Container binds port 8082 (Tomcat starts because `spring-boot-starter-web`
  is on classpath even though no REST routes are registered). AWS region
  `us-east-2` is hardcoded in `S3ClientConfiguration`.
- Add: `S3ServiceImpl` silently swallows get/move/put failures and returns null
  — not a throw path.
- Resolved open questions: none from the platform-level list — the resolved
  items were COMP-11-internal (DCPO mapping confirmed, DNWK handler registry
  confirmed, @Transactional site count confirmed).
- New open questions to add to the platform list:
    - OQ: DB-level UNIQUE constraints on `wdp.file_job (file_name, s3_key)`
      and `wdp.chbk_outbox_row (file_job_id, event_type, row_number)` — not
      in JPA entities, no DDL in this repo. Schema owned outside COMP-11.
    - OQ: DCPO `file_job.source` value and CMRTR vs BJWC sub-type detection
      (follow-up Claude Code question).
    - OQ: DCPO evidence BLOCKED → PENDING promotion — which component owns
      the gate? Not COMP-11, not COMP-12 (which publishes PENDING rows).
    - OQ: DEC-004 non-numeric `acctNum` branch — can production DNWK ever
      produce a non-numeric PAN-bearing value?
- Keep open: SQS visibility timeout numeric value (OQ-1); SQS DLQ config
  (OQ-2); production replica count (OQ-6); `file_job.status = COMPLETED`
  transition ownership (WDP-DB.md attributes it to COMP-12; COMP-11 source
  neither writes nor observes it — confirm via COMP-12 source).

**WDP-DECISIONS.md · Candidate new ADRs**
- No new ADRs. DEC-004 deviation map should add a note that COMP-11 DNWK
  handler encrypts only when `acctNum` matches `\d+` — the non-numeric branch
  is a latent risk but not a confirmed production-active violation. Treat as
  a clarification to the existing DEC-004 entry, not a new ADR.
- DEC-001 deviation map already records COMP-11 as a deviation; audit
  reinforces the detail (three separate transaction boundaries, plus DCPO
  empty-catch and DNWK silent-row-loss) — consider enriching the existing
  entry at next DEC reconciliation.
- DEC-020 deviation map already records COMP-11 idempotency gap; audit
  reinforces the cross-replica race and the DCPO/DNWK row-loss paths —
  consider enriching at next DEC reconciliation.

**WDP-ARCHITECTURE.md**
- No change to platform topology. COMP-11 remains the single inbound
  file-ingestion component between the Sterling → ControlM → S3 pipeline
  and the `chbk_outbox_row` outbox consumed by COMP-12.

**WDP-NFRS.md · Section 6 Risk Register**
- Add (or confirm if already present) RISK rows:
    - COMP-11 DCPO empty-catch evidence loop — orphaned CHARGEBACK_PROCESS
      rows published without evidence. 🟡 MEDIUM.
    - COMP-11 DNWK per-record one-retry-then-silent-loss — row counter
      advances even if both save attempts fail. 🟡 MEDIUM.
    - COMP-11 DEC-004 non-numeric acctNum branch — raw PAN-shaped value
      written to outbox payload. 🟡 MEDIUM (contingent on OQ confirming
      whether production can ever produce non-numeric acctNum).
    - COMP-11 `S3ServiceImpl` silently returns null on all S3 failures —
      transient S3 errors produce no alert. 🟡 MEDIUM.
    - COMP-11 no K8s liveness / readiness / startup probes; Actuator
      absent from classpath. 🔴 HIGH.
    - COMP-11 no HPA — throughput scales only by manual replica increase.
      🔴 HIGH.
    - COMP-11 DISCHYB + AMEXOPTB silent-file-loss paths. 🔴 HIGH.
      Correction: v1.0 DRAFT additionally listed MC_REVREJ — that entry
      should be removed; MC_REVREJ handler is registered.

**WDP-INTEGRATIONS.md**
- No change. COMP-11 inbound integration contract (Sterling → ControlM →
  S3 → SQS `wdp-file-arrivals`) unchanged. DNWK sub-source list unchanged
  at the integration level (registration status is a component-level
  concern).

#### Deviation flags for COMP-11

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-014 Resilience4j Circuit Breakers | ⛔ DEVIATES (platform VOID) | 🔴 HIGH |
| DEC-019 No Clear PAN in Persistent Store | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🟡 MEDIUM |

**DEC-001 PARTIAL detail:** `file_job`, `chbk_outbox_row`, and `file_evidence`
are written in separate transactions. Only `insertEvidenceOutBox`
(parent-flip-to-LOADING + EVIDENCE_ATTACH insert + file_evidence insert) and
the bulk LOADING→PENDING UPDATE carry `@Transactional`. DCPO CHARGEBACK_PROCESS
is auto-commit with an empty-catch evidence loop — orphan-parent risk on any
image failure. DNWK per-record saves are auto-commit with a
one-retry-then-silent-row-loss pattern.

**DEC-004 PARTIAL detail:** DNWK encryption runs only when `acctNum` matches
`\d+`; non-numeric values pass raw into `payload.account_number`. Non-DNWK
sources carry no PAN fields. Open question OQ-5: can production ever produce
a non-numeric PAN-bearing `acctNum`?

**DEC-014 DEVIATES detail:** Platform-wide void already recorded. COMP-11
confirms: no Resilience4j on classpath, no circuit breakers, no `RestTemplate`
connect/read timeouts, no `IdpRestInvoker` retry, no connection pooling. Only
resilience is `@Retryable` 3×/2s on PAN encryption call.

**DEC-019 PARTIAL detail:** Contingent on the same DEC-004 non-numeric branch.
Raw PAN-shaped value could reach `chbk_outbox_row.payload` persistent store
via this branch.

**DEC-020 PARTIAL detail:** File-level idempotency is application-side only;
no DB UNIQUE visible on `(file_name, s3_key)`. Cross-replica race window.
`file_job.status ≠ PENDING` short-circuit means redelivery after a mid-archive
crash is treated as duplicate (SQS message deleted, no reprocessing). DCPO
empty-catch orphan and DNWK silent-row-loss paths are additional at-least-once
violations.

#### Remaining gaps

- **OQ-1 SQS visibility timeout numeric value** — environment config or team
  confirmation. Not in repo.
- **OQ-2 SQS DLQ (RedrivePolicy / maxReceiveCount)** — team confirmation via
  AWS console / Terraform / CDK. Not in repo.
- **OQ-3 DB UNIQUE constraints** on `file_job (file_name, s3_key)` and
  `chbk_outbox_row (file_job_id, event_type, row_number)` — team confirmation
  or schema-owner repo inspection. Schema owned outside COMP-11.
- **OQ-4 `file_job.status = COMPLETED` transition** — architect decision.
  WDP-DB.md attributes it to COMP-12; COMP-11 source neither writes nor
  observes it. Follow-up Claude Code question on COMP-12 source: *"Does
  InboundDisputeEventScheduler (COMP-12) update `wdp.file_job.status` to
  COMPLETED after successfully publishing all chbk_outbox_row rows for a
  given `file_job_id`? Cite file:line. If not, which component does?"*
- **OQ-5 DEC-004 non-numeric acctNum branch** — team confirmation from
  Integration team: can production DNWK flat files ever produce a
  non-numeric `acctNum` that should still be treated as PAN-sensitive?
- **OQ-6 production replica count** — environment config. Not in repo.
- **OQ-7 DCPO `file_job.source` value and sub-type detection** —
  follow-up Claude Code question: *"In `CaponeDisputesIncomingServiceImpl`,
  what exact string is written to `file_job.source`? Is CMRTR vs BJWC sub-type
  determined (filename pattern, XML content, constant)? What is `ack_required`
  set to on the final `updateFileJobTable` call, and under which condition?"*
- **OQ-8 DCPO evidence BLOCKED → PENDING promotion** — architect decision.
  Not in this repo, not in COMP-12 (which publishes PENDING). Follow-up
  needed with the evidence-consumer team.
- **OQ-9 DWSG / DBLK / DISR / MFAD column-level outbox mapping** — follow-up
  Claude Code pass if needed. Audit did not re-read `EventChargebackSupport`,
  `CoreBulkSupport`, `DiscoverMapSupport` line-by-line.
- **Spring Boot 4.0.3 `EntityManagerFactoryBuilder` import path** — developer
  confirmation, not an architecture matter. Flagged in audit, noted here for
  completeness.

#### Doc status after this change
- `WDP-COMP-11-FILE-PROCESSOR.md` → `v1.1 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending
---

### 2026-04-18 — COMP-09 CaseFillingBatch · v1.0 DRAFT → v2.0 DRAFT

**Source:** `wdp-mcm-receiver-case-filing-queue-batch` — source-verified by
Claude Code 2026-04-18. Architect confirmation still pending.

**Nature of change:** Correction pass against source. No functional change in
production; 14 corrections and clarifications to the documented behaviour of
an existing component. Four findings rise to HIGH risk severity.

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- Add `created_by` / `updated_by` constant for COMP-09: `"WMPAPB"`. This is
  distinct from COMP-07 (`"WVDPB"`), COMP-08 (`"WMFDPB"`), and COMP-11
  (`"WPFLEPR"`). Four batch writers, four different user constants.
- Add `c_case_stage` concatenation convention: COMP-09 writes the base stage
  (`PAB`, `ARB`, `PRA`, `AII`, `AIM`) **except** when `issuerCaseStatus =
  withdraw`, in which case the column is written as `{stage}_withdraw`
  (e.g. `PAB_withdraw`, `PRA_withdraw`). Downstream consumers that key on
  `c_case_stage` must tolerate this concatenation.
- Add `network_notes` to COMP-09's written columns — conditional, only when
  `schemeRef.notes` is non-empty. No other batch writer populates this
  column.
- Clarify `c_ntwk_case_id` semantics for COMP-09: value is `standardClaimId`,
  which equals `claimDetail.standardClaims` when `claimType = "CaseFiling"`,
  else `claimId`. The DRAFT labelled this simply `networkCaseId`.
- Add to the Shared Table Risk Register notes: **COMP-09 update path always
  INSERTs a new row**. Existing rows are never UPDATEd by COMP-09. A claim
  that progresses through its phase lifecycle produces multiple rows, one
  per phase observation, distinguished by status (PENDING / SKIPPED). Table
  growth is proportional to phase count × rerun frequency.
- Add to risk notes: **COMP-09 writer swallows all save exceptions**. A DB
  save failure produces no error row, no retry, and no Spring Batch skip
  counter increment — the chunk reports success. Same pattern observed on
  COMP-07's `BatchItemWriter`. This is a cross-batch platform pattern.
- Add to risk notes: **COMP-09 skip paths write no row at all**. Claims that
  fail null-field validation, stage-determination, IDP token, or encryption
  leave zero database trace. `error_code` / `error_message` columns exist
  on the entity but are never populated by COMP-09.
- Confirm "no DB unique constraint" for COMP-09: no DDL, Flyway, Liquibase,
  or `schema.sql` in the COMP-09 repo; `ddl-auto=false`;
  `initialize-schema=never`; no entity-level `uniqueConstraints`.

**WDP-KAFKA.md**
- No change. COMP-09 has no active Kafka producer or consumer. Kafka
  artifacts in `pom.xml` (`spring-kafka`, `kafka-clients`,
  `aws-msk-iam-auth`) are declared but not wired. An accidental import of
  `org.apache.kafka.shaded.com.google.protobuf.ServiceException` is the
  only active Kafka-package reference.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Add / correct:
- COMP-09 writes `wdp.chbk_outbox_row` with writer user `"WMPAPB"`. Shared
  table; four writer components, four distinct user constants.
- COMP-09 update path always INSERTs a new outbox row; existing rows are
  never UPDATEd by this batch.
- COMP-09 skip paths write no database row — null-validation, unmatched
  stage, IDP failure, and Encryption failure all silently drop the claim.
- COMP-09 writer catches and swallows all save exceptions — DB save
  failures are invisible to Spring Batch.
- COMP-09 MCM acknowledgement is not implemented. `DataPowerRestInvoker.put()`
  is fully implemented with `@Retryable` but never called. `McmService`
  exposes 3 methods, no acknowledge method.
- COMP-09 has no liveness, readiness, or startup probes in its Kubernetes
  manifest.
- COMP-09 has no MDC-per-claim correlation scope. Per-call
  `v-correlation-id` headers are added by DataPowerRestInvoker and
  RestInvoker; **IdpRestInvoker does not propagate any correlation id**.
- COMP-09 `enrichmentFailure=true` on payload is set unconditionally on
  every row, for both new and update paths. It does not indicate a failure
  and downstream consumers should not infer meaning from it.
- COMP-09 Spring Batch metadata tables run on the `@Primary wdpDataSource`;
  no dedicated batch datasource; `spring.batch.job.enabled=false`;
  `isolation-level-for-create=REPEATABLE_READ`.
- COMP-09 `JobLauncher` uses Spring Boot auto-configured
  `TaskExecutorJobLauncher` + `SyncTaskExecutor` — synchronous, single
  thread. No `@SchedulerLock`, no ShedLock, no advisory lock, no
  synchronized block. Replica=1 is operational policy, not code-enforced.

Resolved open questions:
- COMP-09 writer failure semantics — now documented.
- COMP-09 update-path behaviour — now documented (always INSERT).
- COMP-09 skip-path row semantics — now documented (no row written).
- COMP-09 outbox column-by-column field mapping — now documented.
- COMP-09 concurrent execution guard — now documented (none; SyncTaskExecutor).
- COMP-09 DB unique constraints — now documented (none in this repo).
- COMP-09 RestTemplate pooling — now documented (bare, no pool).
- COMP-09 health probes — now documented (none).
- COMP-09 observability and correlation — now documented.
- COMP-09 feature flag defaults — now documented (all K8s secrets, no fallback).
- COMP-09 caseType meanings — now documented (1=PreArb, 2=Arb, 3/4 dead).

New open questions (add to HANDOVER):
- Production replica count for COMP-09 — XL Deploy placeholder
  `{{ replicas-wdp-mcm-receiver-case-filing-queue-batch }}`; any value > 1
  is a DEC-023 violation and creates duplicate-INSERT risk.
- Production cron value for COMP-09 — K8s secret `scheduler_cron`, not
  auditable from repo. Application fails to start if absent.
- COMP-09 Spring Batch `table_prefix` K8s secret value — schema name not
  derivable from source. DRAFT conjecture of `WDP.BATCH_` is unverified.
- Downstream MCM queue lifecycle for un-ACKed processed claims — requires
  MasterCard MCM team input. WDP never sends `PUT` acknowledgement; MCM's
  re-queue / TTL / back-pressure behaviour is unknown from our side.
- COMP-12 Scheduler1 filter behaviour on COMP-09 output rows — whether
  Scheduler1 applies any `event_type`, `c_case_ntwk`, or `c_acq_platform`
  filter that would exclude COMP-09's rows. Filter lives in COMP-12 repo.
- `c_case_stage` concatenation (`PAB_withdraw`, `ARB_withdraw`,
  `PRA_withdraw`) — do downstream consumers (COMP-14 CaseCreationConsumer)
  parse this concatenation, or do they match on the base stage only?
  Architect decision required.
- `enrichmentFailure=true` unconditional semantics — protocol-level
  permanent flag or a misconfiguration? No downstream consumer currently
  treats it as a signal, but the name implies it should.

**WDP-DECISIONS.md · Candidate new ADRs**

- **HIGH candidate (cross-component):** "Silent exception swallow in batch
  writers" — confirmed at COMP-07 and now COMP-09. Likely applies to
  COMP-08 by pattern. Recommend raising at next DECISIONS rebuild window.
  Proposed wording: "WDP polling batches catch and swallow all writer
  exceptions, causing DB save failures to appear as successful chunks with
  no audit trail." Severity: HIGH — produces invisible data loss.
- **HIGH candidate:** "MCM acknowledgement not implemented on COMP-09."
  Fully built PUT method and `@Retryable` exist; never called; no interface
  method declared. Open architectural question around MCM queue lifecycle.
  Recommend raising as formal ADR — either implement, formally remove, or
  document the accepted risk with MCM team confirmation on re-queue
  behaviour.
- **HIGH candidate:** "Skip paths in polling batches write no audit row."
  Confirmed at COMP-09 — null-validation, unmatched stage, IDP failure,
  encryption failure all result in no database row. Combined with the
  writer exception swallow, this means there is no operational view of
  which claims were dropped or why, beyond unstructured INFO logs. Likely
  applies to COMP-07 null-return paths too. Severity: HIGH — no
  reconciliation path, no DLQ.
- **MEDIUM candidate:** "Outbox rows are INSERT-only — never UPDATED by
  polling batches." Confirmed design pattern at COMP-09 update path (and
  consistent with COMP-07, COMP-08). Worth recording as an explicit
  architectural principle so future components do not inadvertently mutate
  outbox rows. Severity: MEDIUM — design-clarity, not a defect.
- **MEDIUM candidate (cross-component):** "No health probes on polling
  batches." Confirmed at COMP-07 and COMP-09. Batches have no liveness,
  readiness, or startup probe, so a deadlocked or slow-starting JVM is
  invisible to Kubernetes. Recommend raising at next DECISIONS rebuild.
- **MEDIUM candidate (cross-component):** "No code-level concurrency guard
  on polling batches — DEC-023 operational-only enforcement." Reconfirmed
  at COMP-09. Replica = 1 is policy, not enforced by code anywhere.
- **LOW candidate:** "Inconsistent correlation-id propagation across REST
  invokers in polling batches." COMP-09 confirmed: DataPowerRestInvoker
  and RestInvoker both add `v-correlation-id`; IdpRestInvoker omits it.
  Partial trace continuity per claim.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- Existing RISK-013 (DEC-023 operational-only) — confirmed applies to
  COMP-09. Same clarification as COMP-07 — no code-level guard, only
  operational enforcement.
- **New RISK candidate:** "Batch writer exception swallow — invisible DB
  save failures across polling batches." COMP-07 and COMP-09 confirmed;
  likely COMP-08. Severity HIGH. Raise alongside the corresponding ADR.
- **New RISK candidate:** "Skip paths write no audit row in polling
  batches." Severity HIGH. Applies to all four null-return / silent-skip
  exits on COMP-09; likely same pattern on COMP-07 and COMP-08. Raise
  alongside the corresponding ADR.
- **New RISK candidate (MEDIUM):** "No health probes on polling batches —
  JVM deadlock invisible to Kubernetes." Confirmed COMP-07 and COMP-09.

**WDP-INTEGRATIONS.md**
- No change. COMP-09 integration to MasterCard MCM via DataPower was
  already documented under Section 3.2 / 3.3. No new integration contracts;
  corrections are internal to COMP-09.

#### Deviation flags for COMP-09

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ COMPLIES | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES (with logging finding) | 🟡 MEDIUM (HPAN in logs) |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-014 Resilience4j Circuit Breaker | ⛔ DEVIATES (void platform-wide) | 🟡 MEDIUM |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-022 `removeItemFromQueueDisabled` Safety Switch | ✅ NOT APPLICABLE | — |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY | 🟡 MEDIUM |

**DEC-004 logging finding detail:** HPAN is logged at INFO level post-encryption
in `DisputeServiceImpl`, and the full `CommonEvent` (including HPAN via
`originalTransIdentifier.accountNumber`) is logged via Lombok `@Data` toString
in `ProcessorUtil`. Logs ship to Logstash/ELK via `LogstashTcpSocketAppender`.
`CheckmarxUtil.sanitizeString()` (corrected spelling — DRAFT had `CheckmarkUtil`)
is defined but never called. No `@ToString.Exclude` on HPAN-bearing fields.
This constitutes effective persistence of encrypted cardholder data in logs.
PCI scoping review required.

**DEC-020 PARTIAL detail — escalated to HIGH vs COMP-07 MEDIUM.** Four
independent gaps combine:
(1) no DB unique constraint;
(2) **skip paths write no audit row** — null-validation, unmatched stage,
    IDP failure, and Encryption failure all silently drop the claim;
(3) **writer exception swallow** — DB save failures report success, leaving
    no trace;
(4) crash window between MCM poll and DB write. Unlike COMP-07 — which at
    least writes an ERROR row on missing key data and a SKIPPED row on
    duplicate detection — COMP-09 has **no mechanism whatsoever** for
    observing dropped claims beyond unstructured INFO logs.

**DEC-023 OPERATIONAL ONLY detail:** No `@SchedulerLock`, no ShedLock
dependency, no advisory lock, no synchronized guard. `SyncTaskExecutor`
JobLauncher by Spring Boot auto-config. Replica=1 is policy, not enforced
by code.

#### Remaining gaps

- **Needs follow-up Claude Code question (in a different repo):** "In the
  COMP-12 InboundDisputeEventScheduler Scheduler1 poll query, does any
  WHERE clause filter on `event_type`, `c_case_ntwk`, or `c_acq_platform`
  that would exclude rows written by COMP-09 (event_type=CHARGEBACK_PROCESS,
  c_case_ntwk=MASTERCARD, c_acq_platform=CORE)? Report the exact predicate."
- **Needs architect decision:** Are `PAB_withdraw`, `ARB_withdraw`,
  `PRA_withdraw` concatenated `c_case_stage` values intentional, and is
  any downstream consumer coded to parse them? If not, COMP-14
  CaseCreationConsumer may be silently treating them as unrecognised
  stages.
- **Needs architect decision:** Is the unconditional `enrichmentFailure=true`
  on the payload intentional or legacy? If intentional, consider removing
  the field name that implies meaning it does not carry.
- **Needs architect decision:** Formal ADR for the missing MCM
  acknowledgement — implement, formally remove, or accept with a team
  confirmation on MCM queue behaviour for un-ACKed claims.
- **Needs environment / team confirmation:** Production XL Deploy replica
  count value for COMP-09. If > 1, duplicate-INSERT risk is live.
- **Needs environment / team confirmation:** K8s secret `table_prefix`
  value — Spring Batch metadata schema. Prior DRAFT conjecture `WDP.BATCH_`
  is unverified.
- **Needs environment / team confirmation:** Production cron value for
  COMP-09 (`scheduler_cron` secret).
- **Needs MasterCard team confirmation:** What does MCM do with a claim
  that WDP has processed but never acknowledged? Re-queue? TTL? This is
  the blocker for resolving the MCM ACK ADR.

#### Doc status after this change
- `WDP-COMP-09-CASE-FILLING-BATCH.md` → `v2.0 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending

---
### 2026-04-18 — COMP-08 FirstChargebackBatch · v1.0 DRAFT → v2.0 DRAFT

**Source:** `wdp-mcm-first-chargeback-queue-batch` — source-verified by Claude
Code 2026-04-18. Architect confirmation still pending.

**Nature of change:** Correction pass against source, with three new
architectural findings surfaced that were not visible in the v1.0 DRAFT.
No functional change in production; corrections plus new findings for the
documented behaviour of an existing component.

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- Add the `created_by` distinction: `"WMFDPB"` for COMP-08 (distinct from
  COMP-07 `"WVDPB"` and COMP-11 `"WPFLEPR"`).
- Add "COMP-08 duplicate-check path writes SKIPPED marker rows rather than
  suppressing the insert — `skipCase` flag on `ProcessedItem` is computed
  but never consumed by its Writer. One SKIPPED row accumulates per
  re-polled known chargeback per COMP-08 scheduler run. Growth boundedness
  depends on whether COMP-12 Scheduler2 archives SKIPPED rows — not
  currently confirmed." to the notes.
- Add "COMP-08 update-path writes status=PENDING with accountNumber=null on
  newly-arrived chargebacks on an existing claim — `processUpdatedClaims`
  never sets `isAccountNumberRequired=true`. Suspected defect." to the
  notes.
- Add "COMP-08 writer-ACK hazard — mid-chunk JPA save failure does not
  prevent the subsequent ACK PUT to MCM; ACK PUT exceptions are swallowed
  silently with no log, no metric, no outbox marker. Items acknowledged
  off MCM with no corresponding outbox row are unrecoverable." to the
  notes.
- Reinforce "no DB unique constraint confirmed — no DDL / Flyway /
  Liquibase / entity-level `uniqueConstraints` in COMP-08 repo;
  `ddl-auto: false`; `initialize-schema: never`." (same as COMP-07 —
  pattern repeats.)

**WDP-DB.md · Section 2 · Spring Batch Metadata Tables section**
- Add "COMP-08 sets `spring.batch.jdbc.initialize-schema = never` and
  `spring.jpa.hibernate.ddl-auto = false` — Spring Batch metadata tables
  must pre-exist. DDL is not managed by COMP-08." (same pattern as
  COMP-07 — applies to both.)

**WDP-DB.md · Section 4 · Shared Table Risk Register**
- Append clarifying note to the `wdp.chbk_outbox_row` risk entry: "COMP-08
  source verification 2026-04-18 also confirms no DB unique constraint
  visible in repo. Whether a unique index exists in the live schema must
  be confirmed with the DBA team — same open question as COMP-07."
- Append new risk note: "SKIPPED-row accumulation is a shared-table growth
  concern introduced by COMP-08's insert-a-marker idempotency pattern.
  Whether COMP-07 and COMP-09 share the same pattern needs re-audit."

**WDP-KAFKA.md · Sections 3 and 4**
- No change. COMP-08 has no active Kafka involvement (staged POM deps
  only). Explicit confirmation for next reconciliation session: grep
  against source for `@EnableKafka`, `KafkaTemplate`, `ProducerFactory`,
  `@KafkaListener` returns zero hits.

**WDP-KAFKA.md · Section 6 · Components Confirmed Kafka-Free**
- Minor enrichment to COMP-08 row: note that the entity declares `kafka_*`
  columns but they are never written by this component (owned by COMP-12
  downstream), and that `DisputeServiceImpl` carries a commented-out
  Kafka-shaded Protobuf import — only classpath-visible Kafka reference
  in the codebase.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Add or correct:
- COMP-08 duplicate-check path writes SKIPPED marker rows rather than
  suppressing the insert. The `skipCase` flag on `ProcessedItem` is
  computed but never consumed by the Writer. First confirmed
  "insert-a-marker" idempotency pattern in WDP (distinct from
  "suppress-insert").
- COMP-08 update-path `processUpdatedClaims` never sets
  `isAccountNumberRequired=true`. Newly-arrived chargebacks on an
  existing claim write `status=PENDING` with `accountNumber=null` — no
  HPAN in the payload. Suspected defect.
- COMP-08 writer-ACK hazard: mid-chunk JPA save failure aborts the inner
  save loop but control still reaches the ACK PUT with whatever pairs
  already reached `acknowledgeList`. ACK PUT exception swallowed silently
  in the MCM client — no log of failed pairs, no outbox marker, no
  metric. Potential silent data loss.
- COMP-08 uses a single shared `RestTemplate` bean for all six outbound
  integrations with no connect timeout, no read timeout, no connection
  pool, no custom `ClientHttpRequestFactory`.
- COMP-08 IDP Token GET is unauthenticated (no Authorization header, no
  API key, no mTLS on the application side).
- COMP-08 DataPower auth uses static Vantiv licence as the raw
  `Authorization` header value (no `Bearer` / `Basic` prefix) — Vantiv
  scheme, not a defect.
- COMP-08 `correlationId` is a per-event UUID, not per-job or per-item.
  No MDC correlation across a cron fire.
- COMP-08 `enrichmentFailure=true` inside `OriginalTransIdentifier` is
  set unconditionally on every outbox payload — flag name misleading.
- COMP-08 currency exponents are hardcoded (`USD_EXPONENT=2`,
  `CAD_EXPONENT=2`); actual currency field is never consulted.
- COMP-08 masked-PAN last-4 extraction takes the last 4 characters of
  the masked string — correct only if upstream mask preserves last-4
  positions.
- COMP-08 `created_by` is `"WMFDPB"` (distinct from COMP-07 `"WVDPB"`
  and COMP-11 `"WPFLEPR"`).
- COMP-08 Spring Batch job uniqueness is per-millisecond via a single
  timestamp `JobParameter` (`yyyyMMdd_HHmmss.SSS`). Two launches within
  the same ms collide; different-ms parallel launches proceed.
- COMP-08 has no liveness, readiness, or startup probes wired. Actuator
  is on classpath but Kubernetes is not configured to use it.
- COMP-08 `minReadySeconds: 30` is placed at `spec.template.spec` level
  in `resources.yml` — effectively ignored by Kubernetes for a
  Deployment.
- COMP-08 JobLauncher is default `SyncTaskExecutor` — no custom
  `TaskExecutor`.
- COMP-08 has no `@SchedulerLock`, no advisory lock, no `synchronized`
  guard, no `SELECT ... FOR UPDATE`. Replica=1 is operational policy,
  not code-enforced.
- COMP-08 has no `@Recover` method anywhere. Retry-exhaustion is either
  caught in the processor outer catch (claim skipped, no ACK, re-queued)
  or swallowed in the MCM client (for ACK PUT) — never propagated.
- COMP-08 `ddl-auto=false`, `initialize-schema=never` — both application
  and Spring Batch metadata tables must pre-exist.

Resolved open questions (remove from HANDOVER):
- Writer failure semantics for COMP-08 — now documented.
- ACK failure semantics for COMP-08 — now documented.
- COMP-08 masked-PAN branch semantics — confirmed acceptable under
  DEC-019.
- COMP-08 Spring Batch metadata schema provisioning — now documented
  (same pattern as COMP-07: no default, must pre-exist).

New open questions (add to HANDOVER):
- Production replica count for COMP-08 — XL Deploy placeholder value
  needs confirmation; any value > 1 is a DEC-023 violation.
- Production cron value for COMP-08 — K8s Secret, not auditable from
  repo.
- DB unique constraint on `wdp.chbk_outbox_row (c_ntwk_case_id,
  c_ntwk_phase_id)` — DBA team confirmation needed (same open question
  as COMP-07).
- Does COMP-12 Scheduler2 archive SKIPPED rows, or only SUCCESS rows?
  Determines whether COMP-08 SKIPPED-row accumulation is bounded or
  unbounded.
- Does MCM deliver chargebacks in non-two-decimal-exponent currencies
  (e.g. JPY, KWD) on the `AcquirerFirstCBUnworked` queue? Relevant to
  the hardcoded `USD_EXPONENT=2` / `CAD_EXPONENT=2` risk.
- Under what upstream conditions does MCM return `primaryAccountNum`
  containing `*`? Is the mask guaranteed to preserve the last 4
  positions?
- COMP-08 update-path PENDING-without-HPAN — defect or deliberate
  design? Architect decision required.
- COMP-08 writer-ACK hazard — accepted risk or bug? Architect decision
  required on whether to gate the ACK on per-item save success.
- Do COMP-07 VisaDisputeBatch and COMP-09 CaseFillingBatch exhibit the
  same writer-ACK hazard pattern? Needs targeted re-audit.
- Do COMP-07 and COMP-09 use the same insert-a-marker idempotency
  pattern that COMP-08 does? Needs targeted re-audit.

**WDP-DECISIONS.md · Candidate new ADRs**
- **HIGH candidate (new):** "Idempotency pattern classification —
  suppress-insert vs insert-a-marker on shared outbox tables." COMP-08
  is the first confirmed instance of the insert-a-marker approach on
  `wdp.chbk_outbox_row`. Platform-level ADR required to document which
  pattern is acceptable, because insert-a-marker has table-growth
  implications for a shared outbox.
- **HIGH candidate (new):** "Writer-ACK hazard on MCM-style outbox
  ingest — DB save failure does not prevent upstream ACK." If the
  pattern is confirmed in COMP-07 and COMP-09 on re-audit, this needs a
  platform-level ADR covering all three ingestion batches.
- **HIGH candidate (extends COMP-07 raise):** "Silent exception swallow
  in batch writer and ACK paths" — COMP-08 now also confirmed.
  Strengthens the case for a platform-level ADR covering COMP-07,
  COMP-08, and likely COMP-09.
- **MEDIUM candidate (extends COMP-07 raise):** "No health probes on
  polling batches" — COMP-08 also confirmed absent.
- **MEDIUM candidate (extends COMP-07 raise):** "No code-level
  concurrency guard on polling batches — DEC-023 operational-only
  enforcement" — COMP-08 also confirmed.
- **MEDIUM candidate (new):** "`enrichmentFailure=true` misleading flag
  name — unconditionally true on every COMP-08 outbox payload." Needs
  developer intent confirmation before becoming an ADR or being removed
  from the payload.
- **MEDIUM candidate (new):** "Currency exponent hardcoding
  (`USD_EXPONENT=2`, `CAD_EXPONENT=2`) in COMP-08 — actual currency
  never consulted." Needs confirmation of whether MCM delivers
  non-two-decimal currencies on this queue before disposition.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- No new rows required. RISK-013 (replica constraint has no automated
  enforcement) and RISK-014 (`removeItemFromQueueDisabled` has no
  automated state check) already cover the HIGH risks for COMP-08 as
  well — same as COMP-07.
- Minor clarification opportunity on RISK-013: extend the covered-
  components list to include COMP-08 explicitly (pattern matches
  COMP-07).
- Candidate new RISK row pending architect decision on writer-ACK
  hazard: "Silent ACK-after-DB-failure data loss on MCM-style ingest —
  COMP-08 confirmed; COMP-07 and COMP-09 re-audit pending."

**WDP-INTEGRATIONS.md**
- No change. COMP-08 integration to MCM via IBM DataPower is already
  documented and unchanged.

#### Deviation flags for COMP-08

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ COMPLIES | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES (conditional) | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🔴 HIGH |
| DEC-022 `removeItemFromQueueDisabled` Safety Switch | ✅ PRESENT | — |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY | 🟡 MEDIUM |

**DEC-004 CONDITIONAL detail:** Compliant on the new-claim first-chargeback
path — clear PAN encrypted by COMP-35 before the outbox save. Masked PANs
arriving masked from MCM are written verbatim without encryption —
acceptable under DEC-019 (no clear PAN reaches storage) but noted as
conditional compliance rather than unconditional. Non-first chargebacks on
a new claim and all chargebacks on the existing-claim path write
`accountNumber=null` — no PAN is persisted on these paths, so DEC-004 does
not apply to them.

**DEC-020 PARTIAL detail (severity elevated to HIGH vs COMP-07's MEDIUM):**
Four concurrent gaps: (a) no DB unique constraint visible in repo,
(b) Spring Batch JobInstance uniqueness is per-millisecond only (no
distributed lock), (c) writer-ACK hazard can ACK items off MCM without a
corresponding outbox row (silent data loss path), and (d) update-path
writes PENDING rows with `accountNumber=null` on newly-arrived chargebacks
on existing claims — idempotency semantically completes but delivers
incomplete data. The severity elevation vs COMP-07 is driven by (c) and
(d), which are specific to COMP-08.

**DEC-023 OPERATIONAL ONLY detail:** Identical pattern to COMP-07. No
`@SchedulerLock`, no advisory lock, no synchronized guard. Default
`SyncTaskExecutor` JobLauncher. Rollout with `maxSurge=1
maxUnavailable=0` can temporarily run replicas+1 pods. Replica=1 is
policy, not code.

#### Doc status after this change
- `WDP-COMP-08-FIRST-CHARGEBACK-BATCH.md` → `v2.0 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending

---
---

### 2026-04-18 — COMP-07 VisaDisputeBatch · v1.0 DRAFT → v1.1 DRAFT

**Source:** `gcp-visa-disputes-processor-batch` — source-verified by Claude
Code 2026-04-18. Architect confirmation still pending.

**Nature of change:** Correction pass against source. No functional change in
production; corrections to the documented behaviour of an existing component.

#### Platform-level impacts

**WDP-DB.md · Section 2 · `wdp.chbk_outbox_row` row**
- Add "COMP-07 also writes SKIPPED on duplicate and ERROR on missing key data;
  `skipCase` items write no row" to the writer list.
- Add the `created_by` distinction: `"WVDPB"` for COMP-07 vs `"WPFLEPR"` for
  COMP-11.
- Add "no DB unique constraint confirmed — no DDL / Flyway / Liquibase /
  entity-level `uniqueConstraints` in COMP-07 repo; `ddl-auto: false`;
  `initialize-schema: never`" to the notes.

**WDP-DB.md · Section 4 · Shared Table Risk Register**
- Append clarifying note to the `wdp.chbk_outbox_row` risk entry: "COMP-07
  source verification 2026-04-18 confirms no DB unique constraint visible in
  repo. Whether a unique index exists in the live schema must be confirmed
  with the DBA team."

**WDP-KAFKA.md · Sections 3 and 4**
- No change. COMP-07 has no active Kafka involvement (staged POM deps only).
  Explicit confirmation for next reconciliation session: grep against source
  for `@EnableKafka`, `KafkaTemplate`, `ProducerFactory`, `@KafkaListener`
  returns zero hits.

**WDP-HANDOVER.md · Confirmed Architectural Facts**

Add or correct:
- COMP-07 `skipCase` path writes no outbox row at all (previously implied
  SKIPPED was written).
- COMP-07 writer save failures and MarkAsRead retry-exhaustion are
  caught-and-logged at `BatchItemWriter.java:139-142`, never propagated.
  Spring Batch sees chunk completion as successful.
- COMP-07 has no `@Transactional` anywhere. Chunk transaction is the only
  transactional boundary.
- COMP-07 has no liveness, readiness, or startup probes wired.
- COMP-07 feature flags (`coreDataNeedToMigrate`,
  `removeItemFromQueueDisabled`, `readSpecificItemFromQueue`) have no
  defaults — startup fails if K8s Secret absent.
- COMP-07 JobLauncher is default `SyncTaskExecutor` — no custom TaskExecutor.
- COMP-07 correlation ID is a fresh UUID `v-correlation-id` per outbound call,
  not per-item or per-run, and not added by `IdpRestInvoker`. No MDC.

Resolved open questions (remove from HANDOVER):
- Writer failure semantics — now documented.
- MarkAsRead retry-exhaustion behaviour — now documented.
- `skipCase` path end-to-end — now documented.
- Spring Batch metadata schema behaviour — now documented (no default,
  startup-fails-if-absent).
- Feature flag defaults — now documented.

New open questions (add to HANDOVER):
- Production replica count for COMP-07 — XL Deploy placeholder value needs
  confirmation; any value > 1 is a DEC-023 violation.
- Production cron value for COMP-07 — K8s Secret, not auditable from repo.
- DB unique constraint on `wdp.chbk_outbox_row (networkCaseId,
  networkPhaseId, disputeStage)` — DBA team confirmation needed.
- `migrationStatus = "Y"` hardcoded — permanent design or migration-era
  workaround? Architect decision required.

**WDP-DECISIONS.md · Candidate new ADRs**
- **HIGH candidate:** "Silent exception swallow in batch writer and
  MarkAsRead paths" — affects COMP-07 confirmed, likely affects COMP-08 and
  COMP-09 by pattern. Recommend raising at next DECISIONS rebuild window.
- **MEDIUM candidate:** "No health probes on polling batches" — affects
  COMP-07 confirmed, likely COMP-08 and COMP-09.
- **MEDIUM candidate:** "No code-level concurrency guard on polling
  batches — DEC-023 operational-only enforcement" — explicit recognition
  that replica=1 is policy, not code.

**WDP-ARCHITECTURE.md**
- No change. Topology and principles unchanged.

**WDP-NFRS.md · Section 6 Risk Register**
- No new rows required. RISK-013 (replica constraint has no automated
  enforcement) and RISK-014 (`removeItemFromQueueDisabled` has no
  automated state check) already cover the HIGH risks for this component.
- Minor clarification opportunity on RISK-013: add note that there is also
  no code-level guard, only operational enforcement.

**WDP-INTEGRATIONS.md**
- No change. COMP-07 integration to Visa RTSI via DataPower is already
  documented and unchanged.

#### Deviation flags for COMP-07

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 Transactional Outbox | ✅ COMPLIES | — |
| DEC-003 Kafka Partition Key = merchantId | ✅ NOT APPLICABLE | — |
| DEC-004 PAN Encryption Before Persistence | ✅ COMPLIES | — |
| DEC-005 Manual Kafka Offset Commit | ✅ NOT APPLICABLE | — |
| DEC-019 No Clear PAN in Persistent Store | ✅ COMPLIES | — |
| DEC-020 Full At-Least-Once Idempotency | ⚠️ PARTIAL | 🟡 MEDIUM |
| DEC-022 `removeItemFromQueueDisabled` Safety Switch | ✅ PRESENT | — |
| DEC-023 Replica = 1 Hard Constraint | ⚠️ OPERATIONAL ONLY | 🟡 MEDIUM |

**DEC-020 PARTIAL detail:** RECALL bypasses duplicate check; no DB unique
constraint; null-return paths write no audit; writer and MarkAsRead
exceptions caught-and-logged; crash window exists between Visa poll and
DB save.

**DEC-023 OPERATIONAL ONLY detail:** No `@SchedulerLock`, no advisory lock,
no synchronized guard. `SyncTaskExecutor` JobLauncher. Replica=1 is policy,
not enforced by code.

#### Doc status after this change
- `WDP-COMP-07-VISA-DISPUTE-BATCH.md` → `v1.1 DRAFT` — source-verified
  2026-04-18 · architect confirmation pending

---

## Reconciled

*Entries moved here once a reconciliation session has absorbed them into
the derivative documents. Kept for audit.*

*(none yet)*

---

## Template for new entries

*Per-component chats copy this template and fill it out. Paste the filled
block immediately above the most recent Pending Entry so Pending Entries
remain reverse-chronological.*

```
### YYYY-MM-DD — COMP-NN ComponentName · vX.Y DRAFT → vX.Y+1 DRAFT

**Source:** `<repo-name>` — source-verified by Claude Code YYYY-MM-DD.
Architect confirmation [pending / confirmed].

**Nature of change:** [Correction pass / New component / Feature addition /
Decommission / Other — one sentence].

#### Platform-level impacts

**WDP-DB.md**
- [Row / section affected + what changes, or "No change."]

**WDP-KAFKA.md**
- [Topic / consumer group / section affected + what changes, or "No change."]

**WDP-HANDOVER.md · Confirmed Architectural Facts**
- Add / correct: [facts to add or correct]
- Resolved open questions: [list items to remove]
- New open questions: [list items to add]

**WDP-DECISIONS.md · Candidate new ADRs**
- [New ADR candidates with severity, or "No candidates."]

**WDP-ARCHITECTURE.md**
- [Topology / principle change, or "No change."]

**WDP-NFRS.md · Section 6 Risk Register**
- [New RISK rows or clarifications, or "No change."]

**WDP-INTEGRATIONS.md**
- [Integration contract change, or "No change."]

#### Deviation flags for COMP-NN

| DEC | Status | Severity |
|-----|--------|----------|
| DEC-001 | [✅ COMPLIES / ⚠️ PARTIAL / ⛔ DEVIATES / ✅ NOT APPLICABLE] | [— / 🟢 / 🟡 / 🔴] |
| DEC-003 | ... | ... |
| DEC-004 | ... | ... |
| DEC-005 | ... | ... |
| DEC-019 | ... | ... |
| DEC-020 | ... | ... |

[Add narrative paragraphs for any ⚠️ PARTIAL or ⛔ DEVIATES entries.]

#### Doc status after this change
- `WDP-COMP-NN-NAME.md` → `vX.Y DRAFT` — source-verified YYYY-MM-DD ·
  architect confirmation [pending / confirmed]
```

---

*End of WDP-CHANGE-LOG.md*
