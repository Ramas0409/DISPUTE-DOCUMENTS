
# 📘 WDP Portal — Dispute Management UI (Detailed Functional Summary)
**Version:** 1.1 (updated)  
**Scope:** End‑to‑end user experience and behaviors across Disputes List → Dispute Details (tabs & actions) → Export → Queues (architecture & dashboard).  
**Source:** Consolidation of product notes and shared screenshots from the latest walkthrough.

---

## 0) Global UI Notes
- **Keyboard & Accessibility**
  - **Quick View:** opens **by mouse click only** on a highlighted row. **Enter/Space do not open** Quick View.
- **Data Loading**
  - **Quick View** and row hover panels are **instant**: they read **cached UI data** (no API call).
- **Case Locking**
  - If a case is being edited elsewhere, user sees **“Case Locked”** modal with **Okay** action; further actions blocked until lock clears.

---

## 1) Disputes List (Search Grid)
The landing grid for dispute triage and bulk actions.

### 1.1 Toolbar & Controls
- **Status dropdown** (e.g., *Open*), **Case Owner** (e.g., *Merchant*), **Filter**, **Sort**, **Columns**, and **Export**.
- **Items per page** control with pagination.
- **Actions column** per-row: *Accept*, *Defend*, *… (More)*.

### 1.2 Row Interactions
- Click a row to open **Quick View** (cached) showing **dispute snippet** (e.g., dispute amt, reason code, recent progress).
- Click case number to navigate to **Dispute Details**.

### 1.3 Filter Panel
- **Open via:** Filter button → right drawer.
- **Default section** includes (representative list): Business Unit, Case Number, Report Date, **Due Date**, **Due Days**, **Currency**, **Min/Max Dispute Amount**, **Status**, **Reason Code**, **Scheme**, **Dispute Cycle**, **Dispute Actions**, **Case Owner**, **Case Liability**.  
- **Advanced Filters:** expands to additional merchant/OPS fields (e.g., Issuer BIN, PAN last 4, network refs, MCC, sequences, etc.).  
- **Currency + Amount behavior:** amount inputs are **formatted** and **tied** to a selected currency. Amount & currency move/activate together.
- **High Count of Results modal:** If filter returns a large set:
  - View a **reduced set** in UI and optionally **export** for full results.
  - Guidance to **refine filters** for concise list.
- **Result caps & export pathways (see §3)** are enforced from here.

### 1.4 Sort Menu
- **Sortable columns:**  
  - **Due Days:** Most→Least / Least→Most  
  - **Due Date:** Nearest→Farthest / Farthest→Nearest  
  - **Dispute Amount:** Highest→Lowest / Lowest→Highest  
  - **Reason Code:** Highest→Lowest / Lowest→Highest  
- **Reset Sort** option returns to the saved preset.

### 1.5 Columns Menu
- Toggle visibility, re‑order (drag handles), and **Reset Columns**.  
- **Default columns & order:** Case No., Status, Due Days, Dispute Cycle, Reason Code, Scheme, **Amount**, **Dispute Currency**, ARN, Refunded, Actions (pinned right).  
- **Additional columns (examples):** Dispute Action, Sequence, Due Date, Report Date, Action Date, Expiration Date, Source Case No., **Case Owner**, **Liability**, **Case Direction**, MID, Company ID, Group ID, Outlet ID, Issuer BIN, PAN last4, Merchant Ref No., Airline Ticket No.  
- **Special rule:** **Amount + Currency** behave as a pair (activate, pin, move together).

### 1.6 Accept from Grid
- Clicking **Accept** opens a confirmation modal summarizing **case no., amount, reason code** + a checkbox acknowledgment.  
- On confirm, toast confirms success.  
- **Edge case:** “**Case Locked**” modal prevents action if locked.

### 1.7 Notes Quick Access
- **Row kebab → Add Note** opens notes drawer for the case.  
- Users can **enter long notes**; field expands across lines; notes log shows author, timestamp, and content.

---

## 2) Export (Search Grid)
Supports exporting **current view**, **all results** in UI, or **selected rows**; plus **backend full-file export** when count is large.

### 2.1 UI Options
- **Export menu** (top-right):  
  - **Export This Page (N)** → immediate file (format chooser, default **CSV**; filename e.g., `Dispute-Search-Results-YYYY-MM-DD`).  
  - **Export All Results (M)** → exports all results **currently loaded in UI** (across pages seen so far).  
  - **Export Selected (K)** → exports box-checked rows.

### 2.2 Volume Rules (Critical)
- **≤ 5,000 results:** export directly via UI (page/all/selected).  
- **5,001 – 25,000:** user may **request a file**; processed **asynchronously by backend** and becomes **available later** in UI for download.  
- **> 25,000:** **refine filters** (export request is blocked).

> Note: Backend file‑export status screens will be added later (not yet available).

---

## 3) Dispute Details
Opens by selecting a case (from List or Queue). Two-column layout.

### 3.1 Left Column — Primary Tabs
1) **Case Details**  
   - **Dispute Information** (case number, sequence, cycle, action, due date, amount, chargeback ref, owner, **scheme**, status, action date, reason code, report date, case liability, product name, fraud indicators, issuer dispute amount, duplicate/partial flags).  
   - **Transaction Information** (PAN last4, issuer BIN, transaction date, processing date, auth code, AVS/CVC, region, merchant ref, input method/capability, scheme response codes).  
   - **Merchant Information** (merchant name, MID, company id, group id, outlet id, MCC).  
   - **Queues chip list** (e.g., High Value Disputes, Visa Disputes, +N).  
2) **Card Dispute History**  
3) **Card Transaction History**

### 3.2 Right Column — Context Tabs
1) **Progress**  
   - Shows **Dispute Cycles timeline** (e.g., Draft Retrieval, First/Second Chargeback, Representment, Pre‑Arb, Arbitration, Compliance, Outcome Win/Lose).  
   - Each stage may show **date**, **summary**, and **View Documents** when issuer/merchant docs exist; downloads originate from here.  
   - Outcome card (Win/Lose) provides brief reason text when available.
2) **Actions**  
   - Reverse‑chronological audit of **system/user actions** (e.g., “Automatic Business Rules Applied: VisaVal-R8”, “Document Attached: Issuer Documents”, “Merchant Response Documents”).  
   - Each action includes **actor**, **timestamp**, **description**.
3) **Notes**  
   - Threaded notes with **author chips** and timestamps; **Add a note** box with expanding textarea and send.  
   - Notes can be long; input grows dynamically.
4) **Documents (View‑Only)**  
   - **Intent:** viewing and downloading **attached documents** (Issuer or Merchant).  
   - Actions: **Check for Documents**, list existing PDFs/images; **click to open** modal viewer with paging and zoom.  
   - **No upload from this tab** (upload occurs inside **Defend** flows).

### 3.3 Primary Case Actions (top‑right)
- **Accept** → same confirmation pattern as grid; includes optional free‑text comments and **must‑check** acknowledgment before Submit.  
- **Defend** → two supported modes:
  - **Full Service Response** (direct submission to network)  
    1. **Questions** — dynamic per **scheme** (e.g., **Visa** has specific prompts). Choose **Amount Type** (*Full* or *Split*), plus **comments** for scheme.  
    2. **Documents** — upload evidence. **Accepted formats:** `.tiff, .tif, .pdf, .jpeg, .jpg, .png`; **max size:** 5MB; filename rules: letters, numbers, hyphens, underscores; **no spaces**; ≤ 80 chars including extension.  
    3. **Notes** — optional additional context.  
    4. **Submit Defense**.
  - **Add Response Documents** (let WDP handle defense)  
    1. **Documents** — same upload rules as above.  
    2. **Notes** — optional.  
    3. **Submit Defense** — hands off to **automations/WDP** to finalize response.
- **Refresh** icon re-fetches details.

### 3.4 More Actions (Ops tools)
- **Full Charge to Merchant** — Ops confirms merchant liability. UI: choose **CTM Template**, optional comments, mark as **Internal Note** and/or external **Note**, **Submit**.
- **Full Write Off** — Acquiring platform takes liability; merchant refunded. UI: choose **Write‑Off Reason**, enter **Write‑Off Note**, optional comments, internal/external note flags, **Submit**.
- **Split Action** — Liability split between merchant and acquirer. UI captures: **CTM Template**, **Write‑Off Direction**, **Charge to Merchant Amount**, **Write‑Off Amount**, **Write‑Off Reason/Note**, optional comments, note flags, **Submit**.
- **Advance Action** — Update or add case metadata:
  - **Add New Action** or **Update Existing Action**.
  - Fields: **Dispute Cycle**, **Dispute Action**, **Case Owner**, **Status**, **Case Direction**, **Workable Dispute Amount** (currency‑scoped), **NAP outcome (SRV118)**, **Post/Due/Expiration dates**.  
  - **Submit** applies and logs to Actions tab.
- **Route to Queue** — Assign the case to another queue via dropdown and optional comment; **Submit** routes it.  
- (**Queues screen variant only**) **Escalate to Supervisor** (if enabled).

---

## 4) Queues
Work allocator for Ops (and later Merchant). Two queue types; both listed in the **Queues** page’s left rail.

### 4.1 Queue Types
- **Physical Queues**  
  - Backed by `i_desk` value on the **case** record.  
  - **Business Rules Engine** assigns `i_desk` via rules to push cases into operational queues.
- **Logical Queues**  
  - Implemented as **pre‑defined search criteria** (filters) saved in DB.  
  - Ops can maintain multiple logical queues for different skill sets.  
  - **Merchant** users will have logical queues later (not yet developed).  
  - **Create‑Logical‑Queue** UI is **not implemented**; current logical queues are seeded in DB.

### 4.2 Queue Dashboard (UI Behavior)
- **Left rail:** list of all queues (logical + physical) with pill counts; search input for queues.  
- **Right pane:** **case grid** for the selected queue (columns similar to Disputes List).  
- **Start Queue** → opens the **first case** in the queue directly in **Dispute Details**.  
- **Dispute Details in Queue** mirrors the standard detail page **except**:
  - No **Accept** action (hidden/disabled).  
  - **Defend**, **More Actions** (including **Escalate to Supervisor** where applicable), **Progress/Actions/Notes/Documents** tabs are available.

### 4.3 Role Notes
- **Ops Users:** work both **physical** and **logical** queues.  
- **Merchant Users:** will work **logical** queues only (future).

---

## 5) Open Items / Future Enhancements
- **Backend export management UI** (status, history, retry, notifications).  
- **Document upload from Documents tab** (currently **view‑only**; uploads live inside **Defend** flows).  
- **Create/Edit Logical Queue UI** for Ops/Merchant; governance/permissions.  
- **Keyboard accessibility** for Quick View (Enter/Space) if later required.  
- **Per‑scheme questionnaire catalog** (Visa, MC, Amex, etc.) to document the precise prompts.

---

## 6) Appendix — Operational Rules & Limits (Quick Reference)
- **Export**: ≤5,000 = UI; 5,001–25,000 = backend file; >25,000 = must refine.  
- **Evidence files**: tiff/tif/pdf/jpeg/jpg/png; ≤5MB; names (A‑Z, 0‑9, hyphen, underscore), **no spaces**, ≤80 chars total.  
- **Case locks** prevent all write actions; UI shows lock modal.  
- **Documents tab** is view/download **only**.

---

**End of Document**
