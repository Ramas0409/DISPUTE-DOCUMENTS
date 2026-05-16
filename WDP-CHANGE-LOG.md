# WDP-CHANGE-LOG.md
**Worldpay Dispute Platform — Platform-Level Change Log**
*Baseline: v2.2 — established 2026-04-30*

---

## Baseline Declaration

This log tracks platform-level architectural change **from v2.2 baseline forward**. All component-level findings discovered during the documentation-of-existing-state phase (April 2026 and earlier) have been absorbed into the v2.2 baseline derivative documents (`WDP-ARCHITECTURE.md`, `WDP-DECISIONS.md`, `WDP-INTEGRATIONS.md`, `WDP-NFRS.md`, `WDP-DB.md`, `WDP-KAFKA.md`, `WDP-COMP-INDEX.md`, `WDP-FLOW-INDEX.md`, `WDP-HANDOVER.md`) and per-component files (`WDP-COMP-NN-*.md`). They are no longer tracked individually in this log.

If you need the audit trail of how WDP arrived at v2.2 baseline, the source-verified component files (`WDP-COMP-NN-*.md`) hold the per-component detail; the v2.2 derivative documents hold the platform-level synthesis. There is no need to retain a separate change-log archive of that work.

---

## Purpose

This log captures the **platform-level impact** of every component update made after the v2.2 baseline. It is written by per-component chats at the end of Phase 2 and consumed by **reconciliation sessions** that rebuild derivative documents from the current state of the component files.

### What belongs here
- Which derivative documents are affected by a component change
- Which rows, sections, or entries inside those documents need to change
- Deviation flag status for each component update
- Candidate new ADRs surfaced by the component update

### What does NOT belong here
- Full component file contents (those live in `WDP-COMP-NN-*.md`)
- Rewritten derivative documents inline (those are produced in reconciliation sessions, not here)
- Implementation details, code, or config

---

## How to use this file

### In a per-component chat (Phase 2)
At the end of Phase 2, the chat appends a new block under **Pending Entries** using the template at the bottom of this file. Nothing above the Pending Entries header is modified.

### In a reconciliation session
1. Read this file and note the current Reconciliation Marker
2. Consume all entries under **Pending Entries**
3. Rebuild the affected derivatives
4. Move the consumed entries to the **Reconciled Archive** (compressed form per the policy below)
5. Add a row to the **Reconciliation Markers** table
6. Reset **Pending Entries** to empty

### Compression policy
After a reconciliation session, full Pending Entry content is **not retained verbatim** in this log. Each consumed entry is compressed to a single short paragraph capturing:
- Component + version transition
- 2–4 sentence summary of platform-level findings
- Cross-references to RISK IDs (in `WDP-NFRS.md`) and ADR-CAND IDs (in `WDP-DECISIONS.md`) where the architectural detail now lives
- Pointer to the component file (`WDP-COMP-NN-*.md`) where the full audit detail lives

The full source-verified detail lives in three places after reconciliation:
1. **`WDP-COMP-NN-*.md`** — component file (full audit)
2. **`WDP-NFRS.md`** — RISK rows (risk-class detail)
3. **`WDP-DECISIONS.md`** — DEC updates and ADR-CAND entries (architectural decisions)

This log is the audit trail of *what was reconciled when*, not a duplicate store of the findings.

---

## Reconciliation Markers

*Each marker records one reconciliation session — what was rebuilt, when, and against which set of Pending Entries.*

| Date | Derivatives rebuilt | Entries consumed | Notes |
|------|---------------------|------------------|-------|
| 2026-05-06 | WDP-ARCHITECTURE, WDP-COMP-INDEX, WDP-NFRS, WDP-DECISIONS, WDP-HANDOVER | COMP-42 (2026-04-30); COMP-49 + COMP-50 (2026-05-06) | First reconciliation session post-baseline. 6 RISK rows added (RISK-195–200); 6 ADR-CAND raised (ADR-CAND-060–065); DEC-018 amplified by COMP-49 OPS UI super-user bypass; DEC-019 (4) UI input-layer PARTIAL exception added. Architect open questions on COMP-49+50 captured but unresolved — top priority: ADR-CAND-060 OPS bypass remediation strategy. |
| 2026-05-15 | WDP-NFRS (v2.3), WDP-DECISIONS (v2.3), WDP-KAFKA (v2.3), WDP-DB (v2.3), WDP-COMP-INDEX (v2.3), WDP-HANDOVER (v3.3) | COMP-16 v1.3 (2026-05-15); COMP-18 v2.1 (2026-05-15) | Second reconciliation session post-baseline. **19 new RISK rows** (RISK-201–218 from COMP-16; RISK-219 from COMP-18); **7 new ADR-CANDs** (ADR-CAND-066–072); **DEC-001 flipped from ⛔ DEVIATES to ⚠️ PARTIAL for COMP-16** (event-level outbox `WDP.bre_orchestration_outbox` now exists, Kafka publish still direct synchronous, no relay); **DEC-014 VOID risk amplitude amplified** by COMP-18 DMS `@Retryable` exclude removal; **DEC-020 materially mitigated for COMP-16** (idempotencyId+eventTimestamp dedup + previous-event ordering guard). RISK-015 / RISK-076 / RISK-083 extended with COMP-16. OQ-3 RESOLVED (COMP-18 consumer timing env-var passthrough). **Highest-priority architect items:** ADR-CAND-066 (platform-wide REST `@Retryable` posture); ADR-CAND-070 (cross-datasource non-atomic UK write window on COMP-16); ADR-CAND-071 (US migration guard intent). ARCHITECTURE and INTEGRATIONS not touched — both entries declared "no change" at platform-topology level. |

---

## Pending Entries

*Each per-component chat appends a new block here at the end of Phase 2.*

*(none — last reconciliation session 2026-05-15; see Reconciliation Markers above and Reconciled Archive below)*

---

## Reconciled Archive

*Compressed entries from completed reconciliation sessions, in reverse-chronological order.*

### 2026-05-15 · COMP-16 BusinessRulesProcessor · v1.1 DRAFT → v1.3 DRAFT (forensic re-audit)

User-requested forensic behavioural re-audit prompted by material repository changes since 2026-04-18. `WDP.bre_orchestration_outbox` is now actively read+written as an event-level status register on the Kafka consumer path (statuses: PUBLISHED, SUCCESS, FAILED, ERROR, PENDING_DEFERRED, SKIPPED) — flips DEC-001 from ⛔ VOID to ⚠️ PARTIAL for COMP-16, but Kafka publishes remain direct synchronous (no relay). Two-branch Kafka consumer confirmed — Branch A (eventId non-null) is pre-ACK; Branch B (eventId null) is persist→ACK→process with new application-level dedup on `idempotencyId + eventTimestamp` plus previous-event ordering guard (DEC-020 materially mitigated). Five HIGH-severity new risks: `applyRuleGroup` recursion has no cycle/depth guard (RISK-201); singleton service instance-field state (RISK-202); cross-datasource non-atomic UK write window between `nap.br_case_audit_log` and `WDP.bre_orchestration_outbox` (RISK-203); US migration guard commented out, FCHG path runs unconditionally (RISK-204); UK `getIssuerDoc` missing `attachedIssueDoc` disjunct (RISK-205). 13 additional MEDIUM/LOW risks logged including `/actuator/prometheus` JWT requirement, US/UK dispatch asymmetries, dead code, and `INTEGER BETW` inverted bounds.

**Platform record:** RISK-201 through RISK-218 (`WDP-NFRS.md` §6.1.B COMP-16 sub-section); ADR-CAND-068 through ADR-CAND-072 (`WDP-DECISIONS.md` Phase 3 candidates table); DEC-001 PARTIAL row in deviation map updated; DEC-020 row reworded to "DEVIATES, materially mitigated"; DEC-005 row reworded to "two-branch pre-ACK"; DEC-011 VOID re-confirmed (new outbox is event-state, not step-checkpointing); DEC-017 re-confirmed; RISK-015 (PUBLISHED orphan), RISK-076 (spring-boot-devtools in prod), RISK-083 (minReadySeconds misplaced) all extended with COMP-16. **Highest-priority architect items raised:** ADR-CAND-070 (cross-datasource non-atomic UK write — accept, consolidate datasource, or XA/saga); ADR-CAND-071 (US migration guard intent — cleanup or regression?). **Full audit detail:** `WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md`.

---

### 2026-05-15 · COMP-18 NotificationOrchestrator · v2.0 DRAFT → v2.1 DRAFT (delta-audit)

Delta-audit of `wp-mfd/wdp-outgoing-consumer` against the 2026-04-18 baseline. Only two source files changed in the working copy (`NotificationServiceImpl.java`, `RestInvoker.java`); architecture topology unchanged. Git history not recoverable on the audited clone — dating bounded by file mtime only. **One HIGH-severity finding:** DMS `@Retryable` lost `exclude=RestTemplateCustomException` — HTTP 4xx/5xx responses are now retried 3 × 2000ms instead of fast-failing; at consumer concurrency=1 this turns transient DMS error events into ~6 s of single-thread block per event on `outgoing-events`, amplifying DEC-014 VOID circuit-breaker gap (eventual flow outcome unchanged — message still lands at FAILED in outbox once retries exhausted; only throughput affected). Three lower-severity findings: correlationId UUID now propagated onto outbound Kafka payload BODIES via ModelMapper but not into log MDC (partial wiring — `RequestCorrelation.getId()` has zero callers); Filter 4 fileType resolution has six outcomes not four (AMEX, AMEX_HYBRID, DISCOVER, DISCOVER_RMO, BJS_PLCC, ISSUER_DOCS) with latent DISCOVER + hybridMerchant overwrite defect; three K8s `secretRef`s mounted (`{{ ingressTLSsecretName }}`, `wdp-outgoing-consumer-secrets`, `wdp-common-secrets`) not one as previously documented. Platform-wide implication: the `wdp-common-secrets` mount warrants cross-component scan at next reconciliation.

**Platform record:** RISK-219 (`WDP-NFRS.md` §6.1.B COMP-18 sub-section); ADR-CAND-066 — platform-wide REST `@Retryable` exclude posture, triggered by COMP-18 (`WDP-DECISIONS.md`); ADR-CAND-067 — correlation ID write-side-without-read-side pattern, COMP-18 trigger with survey candidates COMP-14/15/17/43; DEC-014 VOID note amended for risk-amplitude amplification; DEC-001 / DEC-005 / DEC-020 postures unchanged for COMP-18. **Resolved open question:** OQ-3 (consumer timing parameters) — confirmed env-var passthrough only, production values still external. **Full audit detail:** `WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md`.

---

### 2026-05-06 · COMP-49 + COMP-50 WDP Portal · v1.0 → v3.0 (joint, structural consolidation)

WDP Portal documentation consolidated from two separate component files into one canonical file (`WDP-COMP-49-WDP-PORTAL.md`), with `WDP-COMP-50-OPS-PORTAL.md` retained as a stub pointer per the permanent-numbering convention. Source-verified findings: COMP-49 + COMP-50 are a single Angular 19 SPA running in Merchant and Ops modes (not two applications); three user types confirmed (`MERCHANT_USER`, `OPS_USER`, `PB_USER`); OPS-mode UI bypass at `WdpUtilService.canUserPerformDisputeAction()` amplifies DEC-018 (top-priority architect resolution); LFT export threshold corrected to > 200 records (not 5,001–25,000); five previously undocumented More Actions surfaced; Defend has three contest modes; Administration is NAP-only; Fax Queue is two distinct sections (Fax Matching + Fax Analytics).

**Platform record:** RISK-195 through RISK-200 (`WDP-NFRS.md` §6.1.B); ADR-CAND-060 through ADR-CAND-065 (`WDP-DECISIONS.md` §"Phase 3 — post-baseline candidates"); DEC-018 amplification + DEC-019 (4) UI input-layer PARTIAL exception. **Architect open questions captured but unresolved:** OPS-bypass remediation (intentional vs remediate); production `console.*` stripping; mid-session permission refresh strategy; per-action audit of 5 new More Actions; org-specific UI customisation pattern (Macy's); ratification of canonical-file-with-stub documentation convention. **Full audit detail:** `WDP-COMP-49-WDP-PORTAL.md`.

---

### 2026-04-30 · COMP-42 BENConsumer · documentation correction (post-baseline)

BEN delivery mechanism was mislabelled in `WDP-ARCHITECTURE.md` §8.4 as "Webhook" — corrected to Kafka publish to BEN-owned MSK cluster, consistent with §7.4 prose and §12 inventory. Mermaid node label and Notification Targets table row both updated. **No new RISK or ADR-CAND raised; documentation correction only.** **Full audit detail:** `WDP-COMP-42-BEN-CONSUMER.md`.

---

## Template for a New Pending Entry

Copy the block below and fill it in at the end of a per-component chat. Append the new block above this template (i.e., under the **Pending Entries** header), and do not modify anything else in this file.

```markdown
### YYYY-MM-DD · COMP-NN ComponentName · vX.Y → vA.B

**Component file:** `WDP-COMP-NN-COMPONENT-NAME.md`

**Source verification:** [Claude Code | Copilot CLI | manual] against [repo path / commit]

**Platform-level findings:**
- [2–4 short bullets capturing what changed at platform level — not the component-internal detail]
- [Each bullet should be a single sentence describing the architectural impact]

**Affected derivatives:**
| Document | Section | Change required |
|----------|---------|-----------------|
| WDP-ARCHITECTURE.md | §X.Y | [row/paragraph-level description] |
| WDP-NFRS.md | RISK-NNN | [new / refined / withdrawn] |
| WDP-DECISIONS.md | DEC-NNN or ADR-CAND-NNN | [new candidate / deviation map update / VOID] |
| WDP-INTEGRATIONS.md | §X.Y | [contract change description] |
| WDP-DB.md | Section/row | [new table / column / writer / consumer] |
| WDP-KAFKA.md | Topic registry row | [new publisher / consumer / partition key] |
| WDP-COMP-INDEX.md | Component row | [status change / source-verification mark] |
| WDP-FLOW-INDEX.md | Flow row | [new flow / flow scope change] |
| WDP-HANDOVER.md | Confirmed Architectural Facts | [fact added / refined / superseded] |

**Candidate ADRs surfaced:** ADR-CAND-NNN, ADR-CAND-NNN

**Deviation flags:** [DEC-NNN PARTIAL/FULL/VOID per source verification]

**Open questions raised for architect:**
- [Question requiring architect decision before reconciliation can finalise the affected derivative]
```

---

*This log captures platform-level architectural change from v2.2 baseline forward. Pre-baseline component-audit history is absorbed into the v2.2 derivatives and is not tracked here.*
