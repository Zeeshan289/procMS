# Migration plan — from ProcMS v4.4.1 (legacy) to current Next.js build

**Purpose.** Compare feature-for-feature against the deployed *ProcMS & Automation of DP(Navy)* user manual v4.4.1 (Jan 2015, Maritime Technologies Complex, 260 pages, Java 8 / PrimeFaces 6) and spell out every module, screen, and sub-workflow that the current Next.js 16 rewrite is missing, lighter on, or has deliberately replaced.

**Non-goals.** This plan is not a redesign of the UI toolkit. We keep the modern Next.js look and feel we have built; the legacy PrimeFaces screens (three-tab *Search / Results / Details* layout, data grids with inline edit buttons, modal popups) are visual reference only — we will reproduce their **workflows and actions**, not their chrome.

**Compliance guard rail.** Per the ZeroLinkAI architecture decision ([[memory/procms_compliance_architecture.md]]), do not add any new hard-stop refusals when building any of the modules below. The existing four guards stay; new rule checks are data + events for ZeroLinkAI to observe.

---

## Status key

| Icon | Meaning |
|:---:|---|
| ✅ | Shipped in current build, at or beyond parity |
| 🟡 | Partially shipped — core exists, sub-tabs / actions missing |
| ❌ | Missing — no equivalent in the current build |
| 🆕 | New in current build; the legacy system did not have it |
| ❓ | Legacy term needs confirmation (assumed meaning marked) |

---

## 1. Executive summary

- The current build covers the **spine** of a procurement case (indent → IT → TSR → CST → PR → contract → post-contract) and has gone well beyond v4.4.1 on **compliance & transparency** workflows (grievances, appeals, integrity pacts, risk purchases, framework agreements, call-offs, FMS, disposals, misprocurement incidents, retention holds, annual plan, committees with parity, contract amendments) — these are modern PPRA / DPP&I-35 obligations added after 2015 that the legacy system did not model.
- At the same time, the current build is **missing 20+ modules** that the legacy system has and real operators still rely on: Central Registry (mail inward/outward + minutes + postal stamp), Mail Box (internal messaging), xMandate, Budget Card, Budget Clearance / Commitment distinction, Contract Endorsement, Post-Contract detail (EM/BG monitoring, deep delivery tracking, amendments on Delivery/Payment/BG/Schedule), IT Management (PPRA upload + newspaper publication), Corrigendum / Cancellation, PPRA Invoices, DGPRN Acknowledgement, Audit Objections, Audit Items (Draft Paras), Court Cases, Departmental Hierarchy, Employees + HR (service record, leave, disciplinary, house hiring, attendance), Demand Raising, Internal Procurement (with quotations + supply order + sanction cert + contingent bill), Inventory Ledger, Vehicle Requisition, Work Order, Telephone Bills, Decision Approval (Communication), DP-10 Commitment, and a **Dynamic Reporting Tool** (listed as a flagship feature in the legacy manual).
- Several modules we *do* have are **lighter than legacy** on sub-tabs: Firm (missing Financial Info / Items catalog / Agents / Signatories / Partners / Disciplinary Actions tabs), Purchase Proposal (missing Advertisement / Suppliers Participation / Rates Received / Schedule of Store / Additional CFA / Approvals / Items / Document Attachment tabs), Contract (missing Annex A/B/C, Stores, Bank Guarantee, Training, Delivery, Payment, Jurisdiction, Integrity Pact, Claims, Legal/Official Addresses, Signing Authority tabs and the "generate final contract" document render).
- Role Management in v4.4.1 is **dynamic, per-entity rights**; the current build has a fixed `Role` Prisma enum. That is a philosophy change, not a bug — but it should be an explicit decision.

Estimated backlog to reach legacy parity + keep what's new: **4 engineering tracks**, 32 deliverable modules, ~90 user stories. Rough order of magnitude at one-engineer-paced iterations: **12–16 weeks** if the schema migrations are sequenced sensibly.

---

## 2. Methodology

1. Extracted all 260 pages of text from the v4.4.1 PDF manual.
2. Catalogued every module by its page-title heading (38 top-level modules found).
3. Audited the current project at `D:\procurement-system` — all `src/app/(app)` pages, `src/app/api` routes, Prisma models.
4. Reconciled module by module in the sections below. Each section starts with the legacy spec (directly from the manual) and ends with the gap + proposed build-forward note.

Terms I could not confirm from the manual alone — **xMandate**, **DP-10 Commitment**, **DGPRN** (I assume DGPR(Navy) = Director General Public Relations, Navy), **CFA** (I assume Competent Financial Authority), **Budget Card** (I assume a per-indent budget worksheet distinct from the fiscal-year allocation) — are marked ❓ in each section and listed again at the end under **Open Questions**.

---

## 3. Legacy UI conventions (for context, not for copying)

The v4.4.1 manual (pp.4–5) describes a uniform three-tab layout on every form:

- **Search tab** — filter criteria, all filters AND-combined, `LIKE` on textbox fields
- **Results tab** — data grid of records matching the search
- **Details tab** — further sub-divided into tabs for the record's sub-entities

Every mutation shows a toast at the top of the data entry area. Mandatory fields are starred; incomplete submits are refused with red-bordered inputs. These are PrimeFaces-era conventions.

**What we keep in the new UI**: typed status pills, inline action buttons per row, server-rendered pages, native form controls with validation, toast on error. **What we drop**: the mandatory Search → Results → Details tab structure (our list pages jump straight to filterable tables; detail pages have their own route). Workflow fidelity > visual fidelity.

---

## 4. Module-by-module gap analysis

### 4.1 Authentication & Session

| Legacy (pp.1–6) | Current | Status |
|---|---|---|
| Login at `https://www.dpnpms.mil:8443/ProcMS/`, SSL cert required | NextAuth v5 (Credentials + optional OIDC), JWT sessions, TOTP 2FA | ✅ |
| Password encryption / decryption | bcrypt hashes; TOTP for second factor | ✅ (improved) |
| SSL on every connection | Deliberately relaxed for LAN/HTTP demo (see `src/middleware.ts` + `src/infrastructure/auth/nextauth.ts`) | 🟡 — revert the HTTP cookie hack before any public deploy |

**Build-forward**: see HARDENING plan (separate document, HTTPS deploy phase).

---

### 4.2 Home Page & Dashboard  (pp.7–9)

| Legacy | Current | Status |
|---|---|---|
| Statistics counters on left pane; click count → relevant list | `/dashboard` with KPI cards + burndown + turnaround | ✅ (richer) |
| "Dynamic menu creation according to assigned rights" | Fixed menu, role-gated | 🟡 — fine unless we move to dynamic roles (§4.40) |

**Build-forward**: none required; our dashboard is already stronger than legacy.

---

### 4.3 Notifications  (pp.10–12)

| Legacy | Current | Status |
|---|---|---|
| Notification bell + notification list | `NotificationBell.tsx` + `/notifications` page + `notificationService` | ✅ |

**Build-forward**: none.

---

### 4.4 Mail Box — internal messaging  (pp.13–18)

Legacy: a user-to-user inbox with Message Icon List, replies ("New Reply tab"), and read tracking.

Current: ❌ No DM/mail-box feature. `/comments` is per-indent threaded discussion, not user-to-user.

**Gap**: full module.

**Build-forward**:
- Prisma: `Message { id, fromUserId, toUserId, subject, body, sentAt, readAt?, parentMessageId? }`
- Pages: `/mail/inbox`, `/mail/sent`, `/mail/[id]`, `/mail/compose`
- APIs: `POST /api/messages`, `POST /api/messages/[id]/mark-read`, `POST /api/messages/[id]/reply`
- Notification integration: new message → notification row for recipient

---

### 4.5 Central Registry — Mail Inward / Outward  (pp.41–45)

Legacy: Central Registry users record every piece of correspondence coming in or out of DP(N), with a filterable search page and create/edit forms.

Current: ❌ Missing.

**Gap**: full module.

**Build-forward**:
- Prisma: `OfficialCorrespondence { id, direction (INWARD|OUTWARD), controlNumber (unique), subject, fromParty, toParty, receivedOrSentOn, assignedToUserId?, remarks, documentKey?, createdByUserId, createdAt }`
- Role: `CENTRAL_REGISTRY` (new in Prisma Role enum)
- Pages: `/registry/correspondence` (list), `/registry/correspondence/new`, `/registry/correspondence/[id]`
- APIs: standard CRUD + a `POST /api/correspondence/[id]/assign` action

---

### 4.6 Central Registry — Minutes (IMF / EMF)  (p.46)

Legacy: a minutes record linked to a Control Number (presumably from the Mail Inward/Outward above). IMF = Internal Minute File, EMF = External Minute File — **assumed**.

Current: ❌ Missing.

**Gap**: full module.

**Build-forward**:
- Prisma: `Minute { id, correspondenceId, kind (IMF|EMF), text, authorUserId, createdAt }`
- UI: nested under correspondence detail page — add-minute button + minute list

---

### 4.7 Central Registry — Postal Stamp  (p.47)

Legacy: records affixing of postal stamps (literally; for outgoing physical mail).

Current: ❌ Missing.

**Gap**: thin but present in legacy — probably a stamp-register for DAK (official dispatch) accounting.

**Build-forward**:
- Prisma: `PostalStamp { id, correspondenceId?, denomination, quantity, usedOn, enteredByUserId }`
- UI: `/registry/postal-stamps` list + new form.
- Low priority — flag for business owner: *do you still need postal stamp tracking or has DAK gone electronic?*

---

### 4.8 Firm Management  (pp.19–40)

Legacy sub-tabs per firm: **Basic Info · Financial Info · Items · Registration · Signatories · Partners · Agent Info · Disciplinary Actions**.

Current:
- `/admin/firms` and `/admin/firms/[id]` — Basic Info only ✅
- `/registration` — Firm Registration (pre-qualification; richer than legacy's Registration tab, tied to PPRA Rule 15-16) ✅
- `Blacklist` model — covers disciplinary actions in the modern compliance sense ✅ (but see below)

**Gaps**:

| Sub-tab | Status | Notes |
|---|---|---|
| Basic Info | ✅ | Name, NTN, address, city, country, email, phone |
| Financial Info | ❌ | Legacy captures firm paid-up capital, annual turnover, bank details, tax status |
| Items catalog (what the firm supplies) | ❌ | Legacy allowed per-firm item linking — drives which firms are IT-eligible per indent |
| Registration | 🟡 | We have `/registration` (PPRA pre-qual). Legacy also had a separate per-firm registration tab for DP(N)-specific registration — possibly duplicated by our `FirmRegistration` |
| Signatories | ❌ | Authorised signatories per firm, with names + designations + signature image |
| Partners | ❌ | Partnership deed / directors of the firm |
| Agent Info | ❌ | Local agent name, commission %, agreement doc — different from the per-contract `AgentCommission` we built |
| Disciplinary Actions | 🟡 | We have `Blacklist`. Legacy also has lighter "disciplinary action" records that are not full blacklists (warnings, performance notices) |

**Build-forward**:
- Extend `Firm` with child tables: `FirmFinancial`, `FirmItemCatalog` (M2M with an `Item` reference), `FirmSignatory`, `FirmPartner`, `FirmAgent`, `FirmDisciplinaryAction` (distinct from `Blacklist`).
- Rework `/admin/firms/[id]` as a tabbed page.
- Pre-award IT invite screen reads `FirmItemCatalog` to default the firm list for the indent's items.

---

### 4.9 Indent  (pp.48–52)

Legacy: simple create / edit on the indent record.

Current: ✅ Full typed lifecycle (23 statuses, see `src/domain/indent/IndentStatus.ts`), items child table, approval chains, observations, comments, attachments, pipeline view.

**Build-forward**: none — current exceeds legacy.

---

### 4.10 Budget Allocation  (pp.53–56)

Legacy: budget-allocation record maintained by Procurement Budget section.

Current: ✅ `/admin/budget-allocations` with fiscal-year + budget-head + section + allocated / reserved / spent amounts.

**Build-forward**: none.

---

### 4.11 Budget Card  ❓  (pp.57–62)

Legacy: a per-indent budget worksheet that auto-fills indent number + date + a Control Number ("as per standard format must be displayed on screen automatically"), with budgetary information. **Assumed**: this is the Budget section's internal file-card for an indent — different from the fiscal-year allocation, closer to a reservation voucher.

Current: ❌ Missing as an explicit document.

**Gap**: medium.

**Build-forward**:
- Prisma: `BudgetCard { id, indentId (unique), controlNumber (generated), fiscalYearId, budgetHeadId, reservedAmount, currency, notes, preparedByUserId, preparedAt }`
- Page: `/indents/[id]/budget-card` with a print-ready view (legacy prints these).
- Button on the Budget's indent-detail view: "Prepare Budget Card".
- **Confirm with business owner**: what exactly belongs on a Budget Card vs the existing `BudgetAllocation` row? The manual isn't explicit.

---

### 4.12 Budget Clearance / Commitment  (pp.63–66)

Legacy explicitly separates two transitions:
- **Mark as Cleared** — once Purchase Proposal is complete
- **Mark as Committed** — once Contract is signed

Current: the pipeline has `BUDGET_CHECK → COORD_VETTING` and that's it. "Cleared" and "Committed" are not separately recorded.

**Gap**: medium — a legitimate business distinction (cleared = money set aside pending award; committed = money bound by the contract) that drives the budget-head's `reserved` vs `spent` fields.

**Build-forward**:
- `BudgetCard` (above) grows two timestamps: `clearedAt?`, `committedAt?`.
- Two actions: `POST /api/budget-cards/[id]/clear` (gated on PR_ISSUED), `POST /api/budget-cards/[id]/commit` (gated on contract SIGNED).
- Clearing moves the budget-allocation from `reserved` to firmly-reserved; committing moves it to `spent`.

---

### 4.13 xMandate  ❓  (pp.67–71)

Legacy: a module for entering/editing "xMandate" information for an indent. Includes a "Fresh xMandate" request and an approval-authority popup. **Assumed**: an out-of-service / cross-service mandate (e.g. one Navy section procuring on behalf of another, or a GHQ-approved cross-cutting mandate for naval acquisitions beyond the normal budget head).

Current: ❌ Missing.

**Gap**: unclear priority until the term is confirmed.

**Build-forward (tentative)**:
- Prisma: `ExternalMandate { id, indentId, mandateType, authority, requestedOn, approvedOn?, documentKey?, notes }`
- `/indents/[id]/mandate` tab + approval action.
- **Blocked on confirmation** — do not build until the business owner defines xMandate.

---

### 4.14 Invitation to Tender (IT)  (pp.72–85)

Legacy flow: create IT record, Annex-A provision, edit IT, publish via DGDP Bulletin.

Current: ✅ IT Proforma module with approval chain (AD → DD → DP(N)), dispatch, invited firms.

**Gaps**:

| Legacy feature | Current | Status |
|---|---|---|
| Annex-A provision (standard terms) | ❌ | The old system attached an Annex-A to every IT — our IT does not |
| DGDP Bulletin publication | ❌ | Legacy publishes IT to a DGDP bulletin; we do not |
| Edit IT after creation | 🟡 | Our IT becomes immutable at IN_APPROVAL — legacy allowed edits further along |

**Build-forward**:
- Prisma: `ITAnnex { id, itPerformaId, kind ('A'|'B'|'C'), templateId?, body, attachedDocumentKey? }` — references the `BiddingDocumentTemplate` we already have.
- Prisma: `DGDPBulletinEntry { id, itPerformaId, issueNumber, publishedOn, notes }` — record of each publication.
- UI on IT detail: "Attach Annex", "Publish to DGDP Bulletin".

---

### 4.15 Technical Opening  (pp.86–92)

Legacy sub-tabs: Technical Opening details, Opening Information, Participated Firms, Document Attachment.

Current: ✅ Technical offers submission + opening; Participated Firms is implicit from `TechnicalOffer` rows; attachments are on the indent.

**Gap**: minor — a dedicated Technical Opening *minutes* record (who opened, when, in whose presence) is missing.

**Build-forward**:
- Prisma: `TechnicalOpeningMinutes { id, itPerformaId (unique), openedOn, chairmanUserId, membersPresentJson, remarks, documentKey? }`
- UI: single-use form on the IT Proforma detail page once offers close.

---

### 4.16 TSR  (pp.93–95)

Legacy: Technical Scrutiny Report linked to the IT.

Current: ✅ Full TSR module (start / evaluate / query / answer / complete).

**Build-forward**: none.

---

### 4.17 Commercial Opening  (pp.96–100)

Legacy: Commercial Proposal Opening + Commercial Proposal Information sub-tabs.

Current: ✅ Commercial offers module + opening action.

**Gap**: same as Technical — no dedicated Commercial Opening Minutes record.

**Build-forward**: `CommercialOpeningMinutes` parallel to the Technical one.

---

### 4.18 CST — Comparative Statement  (pp.101–103)

Legacy: CST tab.

Current: ✅ CST module with full approval chain, single-quote escalation to senior chain.

**Build-forward**: none.

---

### 4.19 Pre-Purchase Proposal + Single Offer Approval  (pp.104–107)

Legacy: a dedicated "Pre-PP" step for single-offer / proprietary indents before the PP is generated.

Current: ⚠️ The CST chain auto-escalates on single-quote, but there is no distinct "Pre-PP" record. Single-offer approvals are implicit.

**Gap**: medium.

**Build-forward**:
- Prisma: `PrePurchaseProposal { id, indentId (unique), kind (SINGLE_OFFER|PROPRIETARY|OTHER), justification, recommendedByUserId, recommendedOn, approvedByUserId?, approvedOn?, status (DRAFT|PENDING|APPROVED|REJECTED), rejectionReason? }`
- UI: step inserted between CST_APPROVAL and PR_ISSUED for single-quote and PROPRIETY indents.
- Approval chain reuses the senior chain machinery.

---

### 4.20 Purchase Proposal  (pp.108–122)

Legacy sub-tabs: **Indent · Tender · Advertisement · Suppliers Participation · Rates Received · Schedule of Store · Additional CFA · Approvals · Items · Document Attachment**.

Current: 🟡 `PurchaseRequisition` is a lean record (PR number, status, firm, send/acknowledge/close).

**Gap**: significant.

**Build-forward**:
- Add tabs to `/prs/[id]` detail page:
  - **Indent** — read-only summary (we have the data)
  - **Tender** — type (open / closed / re-tender / single / proprietary); read-only
  - **Advertisement** — DGDP Bulletin ref + justification-if-not-published + PPRA upload ref + DGPR(N) publish ref (ties into §4.25 and §4.14)
  - **Suppliers Participation** — all invited firms with participation flag; "reason for not issuing IT to last supplier" field
  - **Rates Received** — per-item price analysis, "justification for increase" field
  - **Schedule of Store** — mandatory delivery schedule with Total Contract Value
  - **Additional CFA** ❓ — Competent Financial Authority information, approval level breakdown. **Confirm meaning**
  - **Approvals** — approval-level history
  - **Items** — item-level breakdown
  - **Document Attachment** — already exists on indent; mirror here
- Prisma additions: `PRPriceAnalysis`, `PRScheduleOfStore`, `PRAdvertisement`, `PRSupplierParticipationNote`.

---

### 4.21 Contract  (pp.123–141)

Legacy tabs on Contract Generation: **Basic Info · Annex-A · Annex-B · Annex-C · Stores · Bank Guarantee · Training · Delivery · Payment · Jurisdiction · Integrity Pact · Claims · Legal/Official Addresses · Signing Authority · Final Contract (generate)**. Plus a "Copy terms from existing contract" button.

Current: 🟡 Thin — `Contract` model has indentId, firmId, contractNumber, value, dates, status, taking-over cert, closedOn. We have `Guarantee`, `Invoice`, `Payment`, `LC`, `LCAmendment` as sibling records.

**Gap**: large. The legacy contract is a tabbed composite document with 14 sub-tabs; we have ~6 of them scattered.

**Build-forward**:
- Rework `Contract` as a tabbed detail page with:
  - **Basic Info** — ✅ have
  - **Annex A / B / C** — ❌ add `ContractAnnex { id, contractId, kind, body, templateId? }` (reuse templates)
  - **Stores** — the schedule-of-store copied from the PR, editable here
  - **Bank Guarantee** — our `Guarantee` ✅ (but filter to this contract only)
  - **Training** ❌ — `ContractTrainingClause { id, contractId, description, location, duration, cost }`
  - **Delivery** — moved from Post-Contract to planned Delivery schedule (we have shipments; make a `ContractDeliverySchedule` entity)
  - **Payment** — our `Invoice`/`Payment` ✅; also need `PaymentTermsClause { id, contractId, milestonePercent, trigger, days }`
  - **Jurisdiction** ❌ — `ContractJurisdictionClause { id, contractId, governingLaw, courtsOfJurisdiction, arbitrationClause }`
  - **Integrity Pact** — our `IntegrityPact` ✅; link from here
  - **Claims** ❌ — `ContractClaim { id, contractId, kind, amount, claimedOn, resolvedOn?, resolution }`
  - **Legal / Official Addresses** ❌ — `ContractAddress { id, contractId, kind (LEGAL|OFFICIAL|BILLING|DELIVERY), addressText }`
  - **Signing Authority** ❌ — `ContractSigner { id, contractId, side (BUYER|SELLER), userId?, firmSignatoryId?, title, signedOn }`
  - **Final Contract** — generate Word / PDF document (see §6 Reports)
- "Copy terms from existing contract" button.

---

### 4.22 Contract Endorsement  (pp.142–146)

Legacy: a fresh endorsement step on a signed contract. Appears to be a formal sign-off endorsement by competent authority before the contract becomes active.

Current: ❌ Missing. Our `ContractAmendment` is for *later* changes, not for the initial endorsement.

**Gap**: medium.

**Build-forward**:
- Prisma: `ContractEndorsement { id, contractId (unique), endorsedByUserId, endorsedOn, endorsementRef, remarks }`
- UI: button on contract detail when status=SIGNED → "Endorse contract".
- Confirm with business owner whether endorsement is a prerequisite to taking-over, or informational only.

---

### 4.23 Post Contract  (pp.147–158)

Legacy sub-tabs:
- **Basic Info** — read-only summary
- **EM / BG Monitoring** — monitor Earnest Money and Bank Guarantee validity
- **Delivery** — status ∈ {shipped, partial delivered, delivered}; actual-delivery-date
- **Stores (LD)** — stores-wise LD assessment
- **Amendments** — four kinds: on Delivery / on Payment Terms / on Bank Guarantee / on Schedule of Store
- **Closing / Termination** — general-status section

Current: 🟡
- EM/BG Monitoring: ❌ (we have Guarantee but no "monitoring" workflow — expiry alerts, renewal, release)
- Delivery status: 🟡 (`Shipment` has IN_TRANSIT / RECEIVED, but legacy has shipped / partial / delivered on the *contract* not the shipment)
- Stores-wise LD: ❌ (our `LiquidatedDamages` is contract-wide; legacy allows per-store LD)
- Amendments (typed into four categories): 🟡 — our `ContractAmendment` is single-purpose
- Closing / Termination: ✅ (new `closeContract` action from this session + `terminateContract`)

**Build-forward**:
- `GuaranteeMonitoringEvent { id, guaranteeId, eventType (RENEWED|EXPIRY_WARNING|RELEASED), on, notes }` + a scheduled job that writes EXPIRY_WARNING rows N days before expiry.
- `ContractDeliveryStatus { contractId, status, lastUpdated }` — computed from shipments, settable manually by Procurement.
- `LiquidatedDamages.scheduleOfStoreId?` to make it per-store.
- Add `scope` enum to `ContractAmendment`: `DELIVERY | PAYMENT_TERMS | BANK_GUARANTEE | SCHEDULE_OF_STORE | SCOPE | VALUE | DP | OTHER`. Our current schema already has `ContractAmendmentScope`, just extend.

---

### 4.24 Risk and Expense  (pp.159–161)

Legacy: a separate module recording Risk-and-Expense (R&E) for contracts where a defaulter firm's default is being recovered.

Current: 🟡 We have `/frameworks/risk-purchases` (post-termination risk purchase). The legacy module is broader — any R&E entry on any contract, not just post-termination.

**Build-forward**:
- Extend `RiskPurchaseCase` to allow `trigger ∈ {TERMINATION | DEFAULT | DELAY_RECOVERY | OTHER}`, defaulting trigger = TERMINATION for the current flow.
- Add `RiskExpenseEntry { id, contractId, kind (NOTICE|RECOVERY|LITIGATION), amount, incurredOn, notes }` for standalone entries not tied to a replacement contract.

---

### 4.25 IT Management (PPRA upload + newspaper publication)  (pp.162–165)

Legacy sub-tabs:
- **PPRA Uploading** — upload tender to PPRA website
- **IT Publication** — send to DGPR(N) for newspaper publication, record DGPR(N) acknowledgement, record newspaper advertisement details (add/remove entries)
- **Corrigendum / Cancellation** — amend or cancel after publication; also a "finalize tender process" button

Current: ❌ We have `/execution/advertisements` as an evidence row (date published, medium) but not the IT-side upload + newspaper publication tracking.

**Gap**: significant.

**Build-forward**:
- Prisma:
  - `PPRAUpload { id, itPerformaId (unique), uploadedOn, ppraReferenceNumber, documentKey, status (DRAFT|UPLOADED|ACK_RECEIVED) }`
  - `NewspaperAdvertisement { id, itPerformaId, newspaperName, publishedOn, pageNumber?, cost?, dgprnAcknowledgementRef? }` (can have multiple per IT)
  - `TenderCorrigendum { id, itPerformaId, corrigendumNumber, issuedOn, changeDescription, publishedOn? }`
- UI: new "IT Management" page under `/it-proformas/[id]/ppra`.
- "Finalize tender" button (locks IT from further corrigendum).

---

### 4.26 Corrigendum / Cancellation  (p.165)

Legacy: part of IT Management (§4.25). Separate cancellation action.

**Build-forward**: already covered in §4.25. Add `TenderCancellation { id, itPerformaId (unique), cancelledOn, cancelledReason, documentKey? }` + a state transition on the IT proforma to `CANCELLED`.

---

### 4.27 PPRA Invoices  (pp.166–174)

Legacy: records invoices DP(N) pays to PPRA (tender upload fees etc.), with:
- Create invoice record
- Return to PPRA (incorrect invoice)
- Send to DGPR(N)
- Edit / delete

Current: ❌ Missing entirely. Our `Invoice` is for firm invoices against contracts, not PPRA-side.

**Gap**: full module.

**Build-forward**:
- Prisma: `PPRAInvoice { id, invoiceNumber, issuedOn, amount, currency, status (RECEIVED|RETURNED|SENT_TO_DGPRN|PAID), returnedReason?, documentKey? }`
- M2M link to `ITPerforma` ("add tender information with the invoice").
- UI: `/finance/ppra-invoices`.

---

### 4.28 DGPRN Acknowledgement  (pp.175–180)

Legacy: records DGPR(N) acknowledging the IT publication or PPRA invoice forwarding. Linkable to invoice numbers.

Current: ❌ Missing.

**Build-forward**:
- Prisma: `DGPRNAcknowledgement { id, invoiceId? (nullable), itPerformaId? (nullable), receivedOn, referenceNumber, remarks }`
- UI: `/finance/dgprn-acknowledgements` + link from PPRA Invoice detail.

---

### 4.29 Audit Objections  (pp.181–186)

Legacy: Audit Section creates audit records with an Audit Info tab + Objection List tab. Each objection list has line items.

Current: ❌ Missing. Our `/admin/audit` is a *log* view, not an audit-objection workflow.

**Build-forward**:
- Prisma:
  - `AuditRecord { id, auditSection, fiscalYear, status (OPEN|CLOSED|SETTLED), subjectIndentId?, subjectContractId?, raisedOn, documentKey? }`
  - `AuditObjection { id, auditRecordId, ground, para, amountInvolved?, status (OPEN|REPLIED|SETTLED), replyText?, repliedOn? }`
- Role: `AUDIT_STAFF`.
- Pages: `/audit/objections`, `/audit/objections/new`, `/audit/objections/[id]`.

---

### 4.30 Audit Items (Draft Paras)  (pp.187–193)

Legacy: audit items with a (Draft) Paras tab — items can be linked to selected Objection Lists.

Current: ❌ Missing.

**Build-forward**:
- Prisma: `AuditItem { id, auditRecordId, itemText, status (AWAITED|REPLY|CLEARED), replyText? }` + `AuditDraftPara { id, auditItemId, paraText, submittedOn }`.
- M2M: `AuditItemObjectionLink`.
- UI: nested under the Audit Record detail.

---

### 4.31 Court Cases  (pp.194–200)

Legacy: Legal Section tracks litigation with:
- Case Information (status: Pending / Closed / Disposed Off)
- Court Proceeding History (many entries per case)

Current: ❌ Missing. Our `Grievance` + `Appeal` cover PPRA 48/50 grievances — a different thing.

**Build-forward**:
- Role: `LEGAL_STAFF`.
- Prisma:
  - `CourtCase { id, caseNumber, court, party, counterParty, status (PENDING|CLOSED|DISPOSED_OFF), relatedIndentId?, relatedContractId?, filedOn, documentKey? }`
  - `CourtProceeding { id, courtCaseId, hearingDate, outcome, nextHearingDate?, notes, documentKey? }`
- Pages: `/legal/court-cases`, `/legal/court-cases/[id]` with proceedings history timeline.

---

### 4.32 Departmental Hierarchy  (pp.201–203)

Legacy: Application Administrator can search, create, edit, delete departments.

Current: 🟡 `/admin/sections` exists with `kind` + `parentId` (tree); create / edit is wired.

**Gap**: hierarchy *display* — legacy renders a tree; our page is flat. Minor UX miss.

**Build-forward**: render `/admin/sections` as a collapsible tree (we already have `parentId` + `children`).

---

### 4.33 Employees + HR  (pp.204–216)

Legacy: a full HR module.
- Employee Basic Info
- Service Record
- Leave Record (view)
- House Hiring Info
- Disciplinary Actions
- Edit / delete employee
- **Attendance** sub-module: mark leave / absent for multiple employees by date (bulk); leave types = Casual / Earned / Medical

Current: ❌ Missing entirely. Our `User` is an auth identity, not an HR record.

**Gap**: whole module. In scope per the user's direction (full parity).

**Build-forward**:
- Prisma:
  - `Employee { id, userId?, personnelNumber (unique), rank, designation, section, dateOfJoining, dateOfBirth, bloodGroup, contact, maritalStatus, isActive }`
  - `EmployeeServiceRecord { id, employeeId, postingFrom, postingTo?, section, remarks }`
  - `EmployeeLeave { id, employeeId, kind (CASUAL|EARNED|MEDICAL|ABSENT), fromDate, toDate, approvedByUserId?, remarks }`
  - `EmployeeHouseHiring { id, employeeId, address, landlord, hireStart, hireEnd?, monthlyRent, currency, documentKey? }`
  - `EmployeeDisciplinaryAction { id, employeeId, kind, issuedOn, authority, decision, remarks }`
- Pages under `/hr/employees`, `/hr/attendance`.
- APIs for CRUD + bulk attendance marking.
- Role: `HR_STAFF`.

---

### 4.34 Demand Raising  (pp.217–220)

Legacy: Administration Section raises item-demand records. Multiple items per demand.

Current: ❌ Missing. Different from Indent — Indent is procurement-driven; Demand is internal consumption-driven.

**Build-forward**:
- Prisma: `InternalDemand { id, demandNumber, raisedByUserId, raisedForSectionId, raisedOn, status (OPEN|ISSUED|CLOSED), notes }` + `InternalDemandItem { id, demandId, itemName, quantity, uom, remarks }`
- Pages: `/stores/demands`.

---

### 4.35 Procurement (internal / small purchases)  (pp.221–229)

Legacy: a lightweight internal procurement workflow:
- Invitation to Tender (list of items to procure + list of firms to invite)
- Quotations (per-firm prices per item)
- Supply Order (auto-suggests firm from lowest quoted price)
- Sanction Certificate / Debitable Voucher
- Contingent Bill

Current: ❌ Missing. Different from the main Indent → Contract flow; this is for small, discretionary spends.

**Build-forward**:
- Prisma:
  - `InternalProcurement { id, reference, raisedOn, raisedByUserId, status (DRAFT|QUOTED|ORDERED|CLOSED), notes }`
  - `InternalProcurementItem { id, procurementId, itemName, quantity, uom }`
  - `InternalProcurementInvitedFirm { id, procurementId, firmId }`
  - `InternalProcurementQuotation { id, procurementId, firmId, itemId, quotedPrice, currency }`
  - `SupplyOrder { id, procurementId (unique), firmId, orderedOn, totalValue, currency }`
  - `SanctionCertificate { id, procurementId, certificateNumber, issuedOn, authority, documentKey? }`
  - `DebitableVoucher { id, procurementId, voucherNumber, amount, issuedOn }`
  - `ContingentBill { id, procurementId, firmId, billNumber, amount, submittedOn, paidOn? }`
- Pages: `/stores/procurement`.

---

### 4.36 Inventory Ledger  (pp.230–231)

Legacy: records item issuance (against a demand number) and receiving (against a tender number). All stores movement is logged.

Current: ❌ Missing.

**Build-forward**:
- Prisma: `InventoryLedgerEntry { id, kind (ISSUANCE|RECEIVING), happenedOn, sourceRef (demandId|tenderId), itemId, quantity, uom, enteredByUserId, remarks }`
- Pages: `/stores/ledger`.

---

### 4.37 Vehicle Requisition  (pp.232–235)

Legacy: any DP(N) employee can raise a vehicle requisition; approval workflow.

Current: ❌ Missing.

**Build-forward**:
- Prisma: `VehicleRequisition { id, raisedByUserId, purpose, fromLocation, toLocation, requiredFrom, requiredTo, passengerCount, status (PENDING|APPROVED|REJECTED|COMPLETED), approvedByUserId?, decidedOn?, assignedVehicleId?, remarks }`
- Pages: `/admin/transport/requisitions`.

---

### 4.38 Work Order  (pp.236–241)

Legacy: maintenance / service work orders (likely for vehicles, buildings, equipment).

Current: ❌ Missing.

**Build-forward**:
- Prisma: `WorkOrder { id, workOrderNumber, kind (MAINTENANCE|REPAIR|OTHER), subject, raisedOn, raisedByUserId, assignedTo, status (OPEN|IN_PROGRESS|COMPLETED|CANCELLED), completedOn?, cost?, currency?, notes }`
- Pages: `/admin/work-orders`.

---

### 4.39 Telephone Bills  (pp.239–242)

Legacy: records monthly telephone bills (likely for DP(N) official lines).

Current: ❌ Missing.

**Build-forward**:
- Prisma: `TelephoneBill { id, phoneNumber, billMonth, billAmount, currency, dueDate, paidOn?, documentKey?, enteredByUserId }`
- Pages: `/admin/telephone-bills`.
- Low priority — flag with business owner: *still needed, or has this gone to a separate utility system?*

---

### 4.40 User Management  (pp.243–247)

Legacy: standard CRUD, password-complexity rule (≥5 chars, ≥1 uppercase, ≥1 lowercase, ≥1 digit), must link to Employee record if user is DP(N) employee, at least one Role.

Current: ✅ `/admin/users` with user CRUD and role assignment.

**Gap**:
- Password complexity rule not enforced the legacy way (we rely on bcrypt but no explicit minimum-complexity check).
- No required User ↔ Employee link (because Employee module doesn't exist — see §4.33).

**Build-forward**:
- Add complexity validator at user create/update (matching the legacy rule exactly, so operators notice no behavioural change).
- When Employee module ships (§4.33), require a `User.employeeId` link for DP(N)-employee users.

---

### 4.41 Role Management  (pp.248–252)

Legacy: **dynamic** roles — create a role, assign it entities with per-entity access rights.

Current: ❌ Fixed `Role` enum in Prisma. Role assignment per user is dynamic; the roles themselves are compile-time.

**Gap**: philosophical. Our current build is simpler and harder to misconfigure. Dynamic roles would allow operators to create new roles without a deploy.

**Build-forward (if business wants dynamic roles)**:
- Replace `Role` enum with `Role { id, code, name, isSystem }` table and `RoleEntityRight { roleId, entity, canRead, canWrite, canDelete }`.
- Keep system roles (ADMIN etc.) as `isSystem=true` so they can't be deleted.
- Rewrite `requireRoles` / `requireAuth` to check the DB instead of the enum.
- Significant refactor — every `hasRole(ctx, Role.X)` call site.

**Recommendation**: keep the enum. The legacy dynamic-role system was flexible but led to configuration drift in practice (my inference from the pattern — confirm). Instead, document the fixed roles clearly. Flag this for business decision.

---

### 4.42 Decision Approval  (pp.253–256)

Legacy: Communication Section marks a tender as Approved / Un-Approved with a "prevent further editing" lock button.

Current: ❌ Missing as a distinct step. Our approval chains handle in-flight approval, but the final "lock from further editing" / "Un-Approve" by Communication is not modelled.

**Build-forward**:
- Add `decisionApprovalLocked: boolean` to indent / tender / contract (contextually).
- UI action on completed records: "Lock from further edits" (reversible with Un-Approve by Communication section).
- Role: `COMMUNICATION_STAFF` (new).

**Note**: the semantics here overlap with our existing "contract closed" state. Confirm with business owner whether Decision Approval is a separate lock, or whether we can treat `CLOSED` as the lock.

---

### 4.43 DP-10 Commitment  ❓  (pp.257–260)

Legacy: records "DP-10" commitments under Budget Section. **Assumed**: DP form 10 — a specific DP(N) form for budget commitment reporting, probably analogous to "Budget Card" but for a different ledger. Could be an annual commitment statement filed with Finance.

Current: ❌ Missing.

**Build-forward (tentative)**:
- Prisma: `DP10Commitment { id, fiscalYearId, budgetHeadId, committedAmount, currency, filedOn, filedByUserId, documentKey?, notes }`
- Page: `/admin/dp10-commitments`.
- **Blocked on confirmation** — do not build until business owner defines DP-10.

---

## 5. Modules in the current build that legacy did NOT have

These are 🆕 additions driven by post-2015 PPRA and DPP&I-35 obligations. Keep all of them.

| Module | Reason |
|---|---|
| Firm pre-qualification (`/registration`) | PPRA Rule 15-16 |
| Grievances + Appeals | PPRA Rule 48 / 50 — formal grievance redressal mechanism |
| Risk Purchase cases | DPP&I-35 Ch XV |
| Framework Agreements + Call-offs | DPP&I-35 Ch XXIII |
| FMS Cases | DPP&I-35 Ch XXII |
| Disposal Cases | DPP&I-35 Ch XXV |
| Misprocurement Incidents | PPRA Rule 50 / ZeroLinkAI observation |
| Retention Holds | DPP&I-35 Ch XIII |
| Annual Procurement Plan | PPRA Rule 8 |
| Bidding-document Templates | Operational need |
| Inspections + Discrepancies | DPP&I-35 Ch XVI |
| Integrity Pacts | PPRA Rule 7 |
| GRC / TEC / FEC Committees with parity enforcement | PPRA Rule 48 |
| Contract Amendments | DPP&I-35 Ch XXI |
| LD Waiver | DPP&I-35 Ch XIII |
| Agent Commissions | DPP&I-35 Ch XIX |
| Shipments | DPP&I-35 Ch XVII |
| End-User Certificates | DPP&I-35 Ch XIX |
| Insurance records | DPP&I-35 Ch XVII |
| TOTP 2FA | Modern security baseline |
| CSRF double-submit | Modern security baseline |
| OpenAPI route | Integration with ZeroLinkAI |
| Dashboards: burndown, turnaround | Modern operational view |

**Build-forward**: none — these are already there.

---

## 6. Cross-cutting concerns

### 6.1 Dynamic Reporting Tool  ❌

Legacy manual lists this as a flagship feature on page 2 ("Dynamic reporting tool to generate reports based on vast criteria"). The current build has views but **no user-facing report builder**.

**Build-forward**:
- Pick an approach: (a) pre-built report pages per module (fastest), (b) a query-builder UI that lets operators pick entity + filters + columns (true legacy parity, significant effort), or (c) wire a BI tool (Metabase / Superset) with read-only access to the Prisma DB.
- Recommend (a) for Phase 1 — identify the 10 most-used legacy reports from business owner, ship as dedicated pages. Then (c) for ad-hoc.

### 6.2 Document generation (Word / PDF)  🟡

Legacy generates "Invitation to Tender, Comparative Statement, Purchase Proposal, Contract etc." as Word / PDF documents at runtime (manual p.2, feature i).

Current: Print-view HTML pages only (`/indents/[id]/print`); no docx / pdf renderer.

**Build-forward**:
- Server-side docx generation (we proved out `docx` npm in this session for the one-pager).
- Templates: IT, CST, PP, Contract, Amendment, Endorsement, Minutes, Budget Card.
- PDF via headless Chrome on the server, or via a managed service.

### 6.3 Dynamic menu  🟡

Legacy: "Dynamic menu creation according to assigned rights over one or many roles" — menu shrinks/grows based on the user's roles.

Current: `MobileNav.tsx` and the sidebar are role-filtered already, but entries are hard-coded.

**Build-forward**: only if we go dynamic-roles (§4.41). Otherwise keep as-is.

### 6.4 Logs / Transparency  ✅

Legacy: "Logs are maintained at each level to enhance transparency in operations".

Current: `AuditLog` written by every use case via `UnitOfWork.audit.appendFromEvents`. Better than legacy.

### 6.5 Graphical charts / KPIs  🟡

Legacy: "Graphical charts are used for different KPIs to help management in effective decision management".

Current: Dashboard has KPI cards + a burndown. Missing: pie / bar / trend charts per module, drill-down.

**Build-forward**: add charts to dashboard (Recharts; we already use React 19).

### 6.6 Field-level validation / UX feedback  ✅

Legacy: starred mandatory fields, red border on invalid, toast at top of data entry area.

Current: Zod validation server-side, inline errors, form components already highlight invalid fields. Matches or exceeds legacy.

### 6.7 Runtime sorting of data grids  ✅

Legacy: "Runtime sorting (both Ascending and Descending) over a column in data grid".

Current: Server-sorted tables via query params; most list pages support it. Confirm every table — add where missing.

### 6.8 HTTPS deploy + reverse proxy  🟡

Covered in §4.1. Separate hardening document will own this.

---

## 7. Prioritized backlog

Four tracks, ordered by business impact. One engineer-track per column; can run in parallel.

### Track A — Deepen the procurement spine (highest priority)

1. **Firm detail tabbed page** (§4.8) — Financial, Items, Signatories, Partners, Agent, Disciplinary — this is read by every IT invite and contract flow.
2. **Purchase Proposal sub-tabs** (§4.20) — Advertisement, Suppliers Participation, Rates Received, Schedule of Store, Additional CFA, Approvals, Items, Document Attachment.
3. **Contract tabbed detail page** (§4.21) — the 14 sub-tabs. Biggest chunk; may need its own mini-plan.
4. **Contract Endorsement** (§4.22) step before taking-over.
5. **Post-Contract deep tracking** (§4.23) — EM/BG Monitoring, Delivery status, per-store LD, typed amendments.
6. **Pre-Purchase Proposal + Single Offer Approval** (§4.19).
7. **IT Management: PPRA upload + newspaper publication + corrigendum** (§4.25).
8. **Budget Card + Clearance/Commitment** (§4.11, §4.12).

### Track B — Finance & correspondence

9. **PPRA Invoices + DGPRN Acknowledgement** (§4.27, §4.28).
10. **Central Registry: Mail Inward/Outward + Minutes + Postal Stamp** (§4.5–§4.7).
11. **Mail Box internal messaging** (§4.4).
12. **Document generation (Word/PDF)** (§6.2).
13. **Dynamic Reporting Tool** (§6.1) — Phase 1 hard-coded reports.

### Track C — Compliance & governance back-office

14. **Audit Objections + Audit Items + Draft Paras** (§4.29, §4.30).
15. **Court Cases + Court Proceeding History** (§4.31).
16. **Decision Approval (Communication lock)** (§4.42).
17. **xMandate** (§4.13) — after confirmation.
18. **DP-10 Commitment** (§4.43) — after confirmation.
19. **Dynamic Role Management** (§4.41) — only if business wants it.

### Track D — HR + admin + office ops

20. **Employees + Service Record + Leave + House Hiring + Disciplinary** (§4.33).
21. **Attendance bulk marking** (§4.33).
22. **Demand Raising** (§4.34).
23. **Internal Procurement + Supply Order + Sanction Cert + Debitable Voucher + Contingent Bill** (§4.35).
24. **Inventory Ledger** (§4.36).
25. **Vehicle Requisition** (§4.37).
26. **Work Order** (§4.38).
27. **Telephone Bills** (§4.39) — confirm still needed.

---

## 8. Open questions — ask business owner before building

| # | Question | Blocks |
|:---:|---|---|
| Q1 | What exactly is **xMandate**? (pp.67–71) — guess: external/cross-mandate approval for cross-section procurement | §4.13 |
| Q2 | What exactly is **DP-10 Commitment**? (pp.257–260) — guess: DP(N) form 10 for annual budget commitment | §4.43 |
| Q3 | Is **Budget Card** (pp.57–62) a per-indent budget worksheet separate from `BudgetAllocation`, or the same thing with a print view? | §4.11 |
| Q4 | Is **CFA** = Competent Financial Authority? (pp.119–120) | §4.20 |
| Q5 | Is **IMF / EMF** = Internal / External Minute File? (p.46) | §4.6 |
| Q6 | Is **Decision Approval** (pp.253–256) a separate lock from `CLOSED`, or can we treat the two as the same? | §4.42 |
| Q7 | Is **Contract Endorsement** (pp.142–146) a prerequisite to taking-over, or a parallel formal sign-off? | §4.22 |
| Q8 | Do you still need **Telephone Bills** tracked inside ProcMS, or has that moved to a utility billing system? | §4.39 |
| Q9 | Do you still need **Postal Stamp** register, or has DAK gone electronic? | §4.7 |
| Q10 | Do you want the **dynamic Role Management** of the legacy system, or keep the fixed enum we have? | §4.41 / §6.3 |
| Q11 | Do you want **Audit Section** and **Legal Section** on the new app, or will those stay in separate systems? (In scope per *full parity*, but worth confirming) | §4.29, §4.30, §4.31 |
| Q12 | For the **Dynamic Reporting Tool** — which 10 legacy reports do operators actually run? (so we can ship those first) | §6.1 |
| Q13 | For **Employees / HR / Attendance / House Hiring** — is this really owned by DP(N) or by a central HR system? If central, we skip §4.33. | §4.33 |

---

## 9. Legend of inferred terminology

Everything marked ❓ in the plan uses one of these assumed meanings. Confirm before building.

| Term | Assumed meaning | Source |
|---|---|---|
| **xMandate** | External / cross-mandate approval — an authority granting procurement power across sections or beyond standard budget head | Context on pp.67–70 |
| **DP-10 Commitment** | DP(N) Form 10 — annual budget-commitment report | Context on pp.257–260 (listed under Budget Section) |
| **Budget Card** | Per-indent budget worksheet with auto-generated Control Number, distinct from the fiscal-year `BudgetAllocation` | pp.57–62 |
| **CFA** | Competent Financial Authority | pp.119–120 ("Additional CFA Information") |
| **IMF / EMF** | Internal Minute File / External Minute File | p.46 (Minutes module) |
| **DGPR(N)** | Director General Public Relations, Navy | pp.164, 170, 175+ |
| **DGDP Bulletin** | Director General Defence Procurement's tender bulletin | p.82 |
| **Annex-A** | Standard terms annexure attached to every IT | p.79 |

---

## 10. Appendix — legacy manual's module index

For traceability. Each row is a legacy module → its page range → the section of this plan that covers it.

| Legacy module | Pages | This plan |
|---|---|---|
| Login & UI | 1–6 | §4.1 |
| Home Page & Dashboard | 7–9 | §4.2 |
| Notifications | 10–12 | §4.3 |
| Mail Box | 13–18 | §4.4 |
| Firm | 19–40 | §4.8 |
| Mail Inward / Outward | 41–45 | §4.5 |
| Minutes | 46 | §4.6 |
| Postal Stamp | 47 | §4.7 |
| Indent | 48–52 | §4.9 |
| Budget Allocation | 53–56 | §4.10 |
| Budget Card | 57–62 | §4.11 |
| Budget Clearance / Commitment | 63–66 | §4.12 |
| xMandate | 67–71 | §4.13 |
| Invitation to Tender | 72–85 | §4.14 |
| Technical Opening | 86–92 | §4.15 |
| TSR | 93–95 | §4.16 |
| Commercial Opening | 96–100 | §4.17 |
| CST | 101–103 | §4.18 |
| Pre-Purchase Proposal | 104–107 | §4.19 |
| Purchase Proposal | 108–122 | §4.20 |
| Contract | 123–141 | §4.21 |
| Contract Endorsement | 142–146 | §4.22 |
| Post Contract | 147–158 | §4.23 |
| Risk and Expense | 159–161 | §4.24 |
| IT Management | 162–165 | §4.25, §4.26 |
| PPRA Invoices | 166–174 | §4.27 |
| DGPRN Acknowledgement | 175–180 | §4.28 |
| Audit Objections | 181–186 | §4.29 |
| Audit Items | 187–193 | §4.30 |
| Court Cases | 194–200 | §4.31 |
| Departmental Hierarchy | 201–203 | §4.32 |
| Employees (+ HR tabs) | 204–213 | §4.33 |
| Attendance | 214–216 | §4.33 |
| Demand Raising | 217–220 | §4.34 |
| Procurement (internal) | 221–229 | §4.35 |
| Inventory Ledger | 230–231 | §4.36 |
| Vehicle Requisition | 232–235 | §4.37 |
| Work Order | 236–241 | §4.38 |
| Telephone Bills | 239–242 | §4.39 |
| User Management | 243–247 | §4.40 |
| Role Management | 248–252 | §4.41 |
| Decision Approval | 253–256 | §4.42 |
| DP-10 Commitment | 257–260 | §4.43 |

---

## 11. Changelog

- **2026-11-12** — Initial draft. Comparison against ProcMS User Manual v4.4.1 (© 2015 Maritime Technologies Complex, 260 pp.).
