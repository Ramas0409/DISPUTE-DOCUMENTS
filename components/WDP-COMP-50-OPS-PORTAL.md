# WDP-COMP-50-OPS-PORTAL
**Worldpay Dispute Platform — Component Reference**
*Version: 3.0 STUB | May 2026*
*Status: Documentation pointer — content lives in COMP-49*

---

## ━━━ STUB POINTER ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## Identity

| Field | Value |
|-------|-------|
| **Name** | `WDP Ops Portal` (Ops mode of WDP Portal) |
| **Type** | Documentation stub — pointer to canonical file |
| **Canonical file** | [`WDP-COMP-49-WDP-PORTAL.md`](./WDP-COMP-49-WDP-PORTAL.md) |
| **Status** | `✅ Production` |
| **Doc status** | `📝 DRAFT 🔍 v3.0 (stub)` |

---

## Why this is a stub

WDP Merchant Portal (COMP-49) and WDP Ops Portal (COMP-50) are **two runtime modes of a single Angular 19 SPA built from a single repository (`wdp-portal`)**. Source verification on 2026-05-06 confirmed:

- Single build artifact — no separate `wdp-merchant-portal` or `wdp-ops-portal` repository
- No compile-time feature flag, no distinct entry point per mode
- Mode is determined at runtime by DNS hostname (`merchantBaseUrl` vs `opsBaseUrl`) plus IDP firm name plus `featurePermissionCheck()`

The architectural boundary in this system is **mode**, not **component**. Maintaining two component files for one codebase guarantees drift on the next source-verification pass — any change to the SPA would require parallel edits in two places. The two-file pattern was an artefact of v1.0 being hand-written from a product walkthrough, before source was inspected.

This stub is retained so that:

- The permanent-numbering convention from `WDP-HANDOVER.md` is honoured — COMP-50 remains a stable identifier
- Cross-references from other documents (`WDP-COMP-INDEX.md`, `WDP-ARCHITECTURE.md`, prior change-log entries) continue to resolve
- Readers landing on COMP-50 by name are directed to the canonical file

---

## Where to find Ops Portal documentation

All Ops-mode content lives in [`WDP-COMP-49-WDP-PORTAL.md`](./WDP-COMP-49-WDP-PORTAL.md). The canonical file labels every per-mode item inline as `[MERCHANT MODE]`, `[OPS MODE]`, or `[BOTH]`.

| If you are looking for... | See in COMP-49 |
|---|---|
| Ops Portal access path, deployment, user types | Identity table; "Access path — Ops mode" row |
| Ops mode boot flow and feature gating | Internal Processing Flow (single Mermaid covering both modes) |
| Ops-only Disputes-section differences (Codes-and-Descriptions view, More Actions, queue-context tabs) | Section A1, A2.2, A6 — items labelled `[OPS]` or `[OPS MODE]` |
| Queues section (Ops only) | Section B |
| Fax Matching section (Ops only, CORE only) | Section C |
| Fax Analytics section (Ops only, CORE only) | Section D |
| Administration section (NAP only, both modes) | Section E |
| Eleven More Actions sub-types (Ops only) | Section A6 |
| OPS super-user bypass amplifying DEC-018 | Risks table — top entry |
| Ops-only backend endpoints (queues, fax, more actions) | Outbound Interfaces table — `Mode = [OPS]` rows |

---

## Deviation flags and risks

All Ops-mode-specific risks are captured in the canonical file's Risks table, including:

- 🔴 OPS super-user UI bypass amplifying DEC-018
- 🔴 More Actions surface 5× larger than v1.0 documented
- 🟠 PCI-IN at form-input layer (Filter Fax Dispute, Update Transaction Search are Ops-only)
- 🟡 eViewer load-failure stuck state
- All 🟠 / 🟡 risks shared with Merchant mode

See `WDP-COMP-49-WDP-PORTAL.md` Risks section for the full table with severity, evidence, and DEC references.

---

## Boundaries

- **Owns no database state.** Confirmed.
- **Does not produce or consume Kafka.** Confirmed.
- **Backend service surface** — see canonical file Outbound Interfaces table; rows labelled `[OPS]` or `[BOTH]` are invoked in Ops mode.

---

*End of WDP-COMP-50-OPS-PORTAL.md (stub)*
*File status: 📝 DRAFT v3.0 STUB — pointer to `WDP-COMP-49-WDP-PORTAL.md`*
*Reason for stub: COMP-49 and COMP-50 are two runtime modes of a single codebase. Maintaining one canonical file prevents drift. See canonical file's footer for the consolidation rationale.*
