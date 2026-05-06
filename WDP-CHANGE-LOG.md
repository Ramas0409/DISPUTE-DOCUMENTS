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
| *(no reconciliation sessions yet — first session against post-baseline entries)* | | | |

---

## Pending Entries

*Each per-component chat appends a new block here at the end of Phase 2.*

### 2026-04-30 · COMP-42 BENConsumer · documentation correction (post-baseline)

**Entry type:** Documentation correction — not a component-update pass and not a version transition. The component file `WDP-COMP-42-BEN-CONSUMER.md` is already correct; this entry tracks an inconsistency in `WDP-ARCHITECTURE.md` that should have been caught during the v2.2 reconciliation.

**Source:** Discovered during v2.2 baseline approval review (cross-check between §7.4 and §8.4 of `WDP-ARCHITECTURE.md`).

**Platform-level findings:**
- §7.4 BEN Consumer correctly describes BEN delivery as Kafka publish to a BEN-owned AWS MSK cluster, with separate SASL/JAAS credentials. v2.2 reconciliation rewrote this section.
- §8.4 Notification Targets carries forward v1.0's incorrect "Webhook" label in two places: the Mermaid node `BEN[BEN\nWebhook]` and the Notification Targets table row labelling BEN's protocol as "Webhook." v2.2 reconciliation marked §8 as unchanged, so the v1.0 label was inherited verbatim from v2.1.
- The v2.2 baseline document is internally inconsistent: §7.4 and §8.4 disagree on BEN's delivery protocol.
- Likely siblings: any §6, §11.1, or §12 reference that mentions BEN's delivery mechanism may carry the same stale label and needs a sweep.

**Affected derivatives:**
| Document | Section | Change required |
|----------|---------|-----------------|
| WDP-ARCHITECTURE.md | §8.4 Mermaid node | Replace `BEN\nWebhook` with a label reflecting Kafka delivery to BEN-owned MSK |
| WDP-ARCHITECTURE.md | §8.4 Notification Targets table — BEN row, "Protocol" column | Replace `Webhook` with `Kafka publish (BEN-owned MSK)` or similar |
| WDP-ARCHITECTURE.md | §6 / §7 / §11.1 / §12 | Sweep for sibling references that may still carry the stale "Webhook" label |

**Candidate ADRs surfaced:** None — documentation correction, not architectural change.

**Deviation flags:** None.

**Open questions raised for architect:** None — the architectural fact (BEN-via-Kafka) is settled per COMP-42 source verification and was correctly captured in §7.4 and §12. Documentation needs to catch up in §8.4.

**Resolution path:** Fold into the next reconciliation pass that touches COMP-42 source verification or §8 architectural review. If no such pass is scheduled within a reasonable window, run a targeted §8 / BEN-references sweep as a standalone reconciliation.

---

## Reconciled Archive

*Compressed entries from completed reconciliation sessions, in reverse-chronological order.*

*(none yet — first reconciliation session will populate this section)*

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
