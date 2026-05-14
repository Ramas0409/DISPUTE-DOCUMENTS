# WDP-FLOW-INDEX.md
**Worldpay Dispute Platform — Workflow Document Index**
*Version: 1.1*

---

## Purpose of This Document

This is the navigation index for all WDP workflow documents.
Workflows describe cross-component choreography — how multiple
components collaborate to deliver a business outcome end to end.

**The distinction between component files and workflow files:**
- WDP-COMP-[NN]-*.md answers: *what does this component do?*
- WDP-FLOW-*.md answers: *how do multiple components work together
  to deliver a business outcome?*

Workflows are never documented inside component files. A workflow
spans multiple components — putting it in any one component file
forces duplication and causes divergence over time.

---

## Naming Convention

```
WDP-FLOW-[SHORTNAME].md
```

Short names use uppercase hyphenated descriptors.
Examples:
```
WDP-FLOW-DISPUTE-INBOUND-NAP.md
WDP-FLOW-DISPUTE-LIFECYCLE.md
WDP-FLOW-MERCHANT-CONTEST.md
```

---

## What a Workflow File Contains

Each workflow file follows this structure:

```
1. Business Context
   What business event triggers this flow.
   What the end-to-end outcome is.
   Which card networks and acquiring platforms this applies to.

2. End-to-End Sequence Diagram (Mermaid)
   All components in order.
   Every message or call between them.
   Decision points and branches.
   Failure paths.

3. Step-by-Step Narrative
   Each step explained: who triggers it, what happens,
   what payload moves, what the next trigger is.

4. Components Involved
   Cross-references to WDP-COMP-[NN]-*.md files.

5. Timing & SLA Constraints
   Any network deadlines, expiry windows, SLA targets.

6. Failure & Retry Behaviour
   What happens at each failure point.
   Recovery paths. Dead-letter handling.

7. Known Variants
   NAP vs PIN vs LATAM vs VAP differences in this flow.
   Card network differences (Visa vs Mastercard vs Amex vs Discover).
```

---

## Workflow Registry

### Core Dispute Flows

| ID | Workflow | Trigger | Acquiring Platforms | Networks | Doc Status | File |
|----|----------|---------|--------------------|---------|----|------|
| FLOW-01 | Dispute Inbound — NAP | NAP-DPS pushes SRV-116 event via REST | NAP | All | ⬜ NOT STARTED | WDP-FLOW-DISPUTE-INBOUND-NAP.md |
| FLOW-02 | Dispute Inbound — Visa Batch | VisaDisputeBatch cron polls Visa RTSI | CORE, PIN | Visa | ⬜ NOT STARTED | WDP-FLOW-DISPUTE-INBOUND-VISA-BATCH.md |
| FLOW-03 | Dispute Inbound — Mastercard Batch | FirstChargebackBatch / CaseFillingBatch cron polls MCM | CORE | Mastercard | ⬜ NOT STARTED | WDP-FLOW-DISPUTE-INBOUND-MC-BATCH.md |
| FLOW-04 | Dispute Inbound — File-Based | ZIP file lands in S3, SQS triggers FileProcessor | CORE, NAP (Walmart/CapOne) | Amex, Discover, NYCE, Walmart, CapitalOne | ⬜ NOT STARTED | WDP-FLOW-DISPUTE-INBOUND-FILE.md |
| FLOW-05 | Dispute Lifecycle | Case created → merchant notified → merchant acts → outcome delivered | All | All | ⬜ NOT STARTED | WDP-FLOW-DISPUTE-LIFECYCLE.md |

### Merchant Action Flows

| ID | Workflow | Trigger | Acquiring Platforms | Networks | Doc Status | File |
|----|----------|---------|--------------------|---------|----|------|
| FLOW-06 | Merchant Accept | Merchant accepts dispute via portal or API | All | Visa, Mastercard | ⬜ NOT STARTED | WDP-FLOW-MERCHANT-ACCEPT.md |
| FLOW-07 | Merchant Contest | Merchant submits representment evidence and contests | All | Visa, Mastercard | ⬜ NOT STARTED | WDP-FLOW-MERCHANT-CONTEST.md |

### Notification & Outbound Flows

| ID | Workflow | Trigger | Acquiring Platforms | Networks | Doc Status | File |
|----|----------|---------|--------------------|---------|----|------|
| FLOW-08 | Network Response Submission | Merchant contest approved → response submitted to card network | All | Visa (VROL), Mastercard (MasterCom) | ⬜ NOT STARTED | WDP-FLOW-NETWORK-RESPONSE.md |
| FLOW-09 | Notification Delivery | Dispute lifecycle event → BEN / ThirdParty / CORE notification | All | All | ⬜ NOT STARTED | WDP-FLOW-NOTIFICATION-DELIVERY.md |
| FLOW-10 | ACK File Generation | File processing completes → ACK file delivered to merchant | CORE (Walmart, Meijer, CapitalOne) | N/A — file-based | ⬜ NOT STARTED | WDP-FLOW-ACK-GENERATION.md |

### Operational Flows

| ID | Workflow | Trigger | Acquiring Platforms | Networks | Doc Status | File |
|----|----------|---------|--------------------|---------|----|------|
| FLOW-11 | Case Expiry — Subsystem Lifecycle | COMP-18 publishes to `case-action-events` when an action's expiry data changes; COMP-17 maintains `wdp.case_expiry`; COMP-51 scans and acts on past-due deadlines | All | All | ⬜ NOT STARTED | WDP-FLOW-CASE-EXPIRY.md |

---

## Per-Flow Authoring Notes

*Notes accumulated from component-level audits that should be captured
when each flow document is authored. These are NOT a replacement for the
flow document — they are inputs for the eventual author.*

### FLOW-07 Merchant Contest — Visa contest end-to-end variant

When the Visa contest end-to-end flow document is authored, capture the
following partial-success behaviour explicitly:

- **Path:** COMP-21 ChargebackService (merchant API) → COMP-20 ContestService
  (publishes to `internal-integration-events`) → COMP-40 VisaResponseQuestionnaire
  (consumes events with non-null `visaResponseIds`; calls Visa RTSI to retrieve
  questionnaire) → COMP-37 DocumentManagementService (stores questionnaire in
  S3 + DynamoDB).
- **Partial-success on `allocarb` loops:** when the Visa response contains
  multiple `visaResponseIds`, COMP-40 iterates per ID. Failure on any one ID
  partially completes the loop — the consumer pre-ACKs (DEC-005 deviation),
  so retry is **not automatic**. Failure semantics on partial loops must be
  documented when the flow is authored.
- **Silent-discard behaviour:** COMP-40 silently discards events without
  `visaResponseIds` (e.g. AcceptService events that share the same topic).
  This filtering must be diagrammed at the consumer boundary.

### FLOW-11 Case Expiry — Subsystem Lifecycle

When this flow is authored, capture the following confirmed participant
chain and shared-state risks:

- **Trigger chain:** dispute action expiry data updated → COMP-18
  NotificationOrchestrator publishes to `case-action-events` Kafka topic →
  COMP-17 CaseExpiryUpdateConsumer consumes and upserts/deletes
  `wdp.case_expiry` rows → COMP-51 CaseExpiryProcessor scheduled batch reads
  `wdp.case_expiry`, calls Case Action / Case Management / Expiry Rules APIs,
  and acts via one of three terminal APIs (Update Action / Accept / Add
  Action).
- **Failure path:** COMP-51 retry-exhausted failures land in
  `wdp.outgoing_event_outbox` with `channel_type=EXPIRY_BATCH` and
  `created_by=WCSEEXPB`, status=ERROR direct (no PUBLISHED→FAILED transition).
- **🔴 Shared-state risk:** COMP-17 and COMP-51 both write to
  `wdp.case_expiry` with **no row-level lock, no version column, no
  SELECT FOR UPDATE**. Race possible on `i_retry_count` overwrite and
  mid-flight upsert during chunk processing. The flow document must
  diagram this coordination gap explicitly.
- **🔴 Outbox terminal-write-only gap:** EXPIRY_BATCH rows have **no
  identified downstream consumer** — COMP-12 Scheduler3 reads only FAILED /
  PENDING_DEFERRED, so COMP-51's status=ERROR rows are not picked up.
  The flow document must mark this as an unresolved architect decision.
- **Replicas=1 OPERATIONAL ONLY:** no `@SchedulerLock`, no advisory lock —
  concurrency safety relies entirely on K8s `replicas: 1` (DEC-023 pattern,
  same posture as COMP-07 / COMP-08 / COMP-09 polling batches).

---

## Workflow Status Key

- ✅ COMPLETE — sequence diagram, narrative, and variants confirmed
- 📝 DRAFT — initial diagram written, narrative in progress
- ⬜ NOT STARTED — no content yet

---

## Workflow Priority Order

Document flows in this order — highest architectural value first:

1. **FLOW-05 Dispute Lifecycle** — the master end-to-end flow.
   All other flows are either inputs to or outputs from this one.
   Document this first so every other flow has a parent to reference.

2. **FLOW-01 Dispute Inbound NAP** — most complex inbound path.
   Most components involved. Most deviations from the standard path.

3. **FLOW-07 Merchant Contest** — highest merchant value action.
   Triggers the most downstream components across the platform.
   *(Visa contest end-to-end variant — see authoring notes above.)*

4. **FLOW-02 Dispute Inbound Visa Batch** — origin of all Visa
   disputes. Foundational for CORE and PIN platforms.

5. **FLOW-03 Dispute Inbound Mastercard Batch** — origin of all
   Mastercard disputes.

6. **FLOW-09 Notification Delivery** — spans most outbound
   components. Documents the full notification fan-out.

7. **FLOW-06 Merchant Accept** — simpler than contest but high volume.

8. **FLOW-08 Network Response Submission** — Visa VROL and
   MasterCom response paths documented together.

9. **FLOW-04 Dispute Inbound File** — file-based inbound path.
   Documents all six source types and the outbox-to-Kafka bridge.

10. **FLOW-10 ACK Generation** — outbound ACK file generation
    for all four merchant types.

11. **FLOW-11 Case Expiry — Subsystem Lifecycle** — operational
    flow. Now has confirmed three-component participant chain
    (COMP-18 → COMP-17 → COMP-51) and two 🔴 HIGH risks worth
    surfacing in the flow document — see authoring notes above.

---

## Variant Flows (Add as Platforms Come Online)

When LATAM and VAP integrations go to production, add:

| ID | Workflow | Trigger | File |
|----|----------|---------|------|
| FLOW-12 | Dispute Inbound — LATAM | ⚠️ Pending LATAM integration design | WDP-FLOW-DISPUTE-INBOUND-LATAM.md |
| FLOW-13 | Dispute Inbound — VAP | ⚠️ Pending VAP integration design | WDP-FLOW-DISPUTE-INBOUND-VAP.md |
| FLOW-14 | EDIA Outbound Delivery | ⚠️ Pending EDIA Consumer build | WDP-FLOW-EDIA-OUTBOUND.md |

---

## Adding New Workflow Documents (Protocol)

1. Assign the next FLOW-ID number (current last: 11; reserved 12–14 for
   variant flows above)
2. Add a row to the registry table above
3. Create the file using the structure described in this document
4. Cross-reference the relevant WDP-COMP-[NN]-*.md files in step 4
   of the workflow document
5. Update this index with the new doc status once complete
6. Append a Pending Entry to WDP-CHANGE-LOG.md if the new flow surfaces
   any new platform-level facts (deviation flags, integration contracts)

---

*Current workflow count: 11 core flows identified, 0 documented*
*Next available FLOW ID: 15 (12–14 reserved for variant flows)*
