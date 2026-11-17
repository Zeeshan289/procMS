# Zeeshan — ProcMS v4.4.1 migration brief

**Track:** Finance operations + Central Registry correspondence + Mail Box + governance back-office (Audit, Court Cases, Decision Approval) + Document generation + Reporting Phase 1.
**Modules:** 8
**Start date:** 2026-11-17
**Ship target:** 5 weeks

---

## 1. Required reading (do this first, in order)

1. [`HANDOFF.md`](../../HANDOFF.md)
2. [`docs/INDENT_LIFECYCLE_STEPS.md`](../INDENT_LIFECYCLE_STEPS.md)
3. [`docs/SECONDARY_WORKFLOWS_STEPS.md`](../SECONDARY_WORKFLOWS_STEPS.md)
4. [`docs/MIGRATION_PLAN_FROM_V4.4.1.md`](../MIGRATION_PLAN_FROM_V4.4.1.md) — §4.4, §4.5, §4.6, §4.7, §4.27, §4.28, §4.29, §4.30, §4.31, §4.42, §6.1, §6.2
5. [`docs/OPEN_QUESTIONS.md`](../OPEN_QUESTIONS.md) — Q5 (IMF/EMF), Q6 (Decision Approval lock), Q11 (Audit/Legal on new app), Q12 (which reports)
6. [`docs/INTERN_ASSIGNMENTS.md`](../INTERN_ASSIGNMENTS.md) — shared conventions
7. The ProcMS v4.4.1 user manual, pages: **13–18** (Mail Box), **41–45** (Mail I/O), **46** (Minutes), **47** (Postal Stamp), **166–180** (PPRA Invoices + DGPRN Ack), **181–193** (Audit), **194–200** (Court Cases), **253–256** (Decision Approval)

## 2. Code patterns to copy

- Service → read [`src/infrastructure/execution/ExecutionAdminService.ts`](../../src/infrastructure/execution/ExecutionAdminService.ts) (pattern: a `const service = { method1, method2, ... }` singleton).
- Action buttons on a list page → read [`src/app/(app)/transparency/incidents/IncidentActions.tsx`](../../src/app/(app)/transparency/incidents/IncidentActions.tsx) and [`src/app/(app)/frameworks/call-offs/CallOffActions.tsx`](../../src/app/(app)/frameworks/call-offs/CallOffActions.tsx).
- List page shape → [`src/app/(app)/transparency/incidents/page.tsx`](../../src/app/(app)/transparency/incidents/page.tsx).
- Zod schema file → [`src/interfaces/http/schemas/specialCases.ts`](../../src/interfaces/http/schemas/specialCases.ts).
- Doc-gen reference (you own the Doc gen module; this was proved out for the one-pager): look at `scratchpad/build_one_pager.js` style using the `docx` npm.

## 3. Your 8 modules

### Z1 — Document generation helper (ship this FIRST — other tracks depend on it)  (§6.2)

Reusable server-side doc-gen so Mustafa / Fahad / your own reports can produce Word (`.docx`) files from templates.

**Build:**
- `src/infrastructure/docgen/DocxBuilder.ts` — thin wrapper around the `docx` npm package. Exposes a `buildDocument(template, data)` function and helpers for tables, headings, metadata.
- `src/infrastructure/docgen/templates/` — one file per template:
  - `indentTemplate.ts` (for Mustafa's print view)
  - `prTemplate.ts` (for Mustafa's PR print)
  - `contractTemplate.ts` (for Fahad's "Generate Final Contract" button)
  - `cstTemplate.ts`, `itTemplate.ts` — placeholders Mustafa / Fahad fill in
- New API route pattern: `GET /api/<entity>/[id]/download?format=docx` — returns the generated `.docx` with `content-type: application/vnd.openxmlformats-officedocument.wordprocessingml.document`.
- For PDF: wrap the generated `.docx` through a headless converter OR render an HTML print-view through puppeteer. Pick the simpler one; flag the choice in your commit message.

**Prisma:** none. This is infrastructure.

### Z2 — Central Registry: Mail Inward / Outward + Minutes + Postal Stamp  (§4.5 + §4.6 + §4.7, manual pp.41–47)

**Prisma:**
- `OfficialCorrespondence { id, direction (INWARD|OUTWARD), controlNumber (unique), subject, fromParty, toParty, receivedOrSentOn, assignedToUserId?, remarks, documentKey?, createdByUserId, createdAt }`
- `CorrespondenceMinute { id, correspondenceId, kind (IMF|EMF), text, authorUserId, createdAt }` — see Q5
- `PostalStamp { id, correspondenceId?, denomination, quantity, usedOn, enteredByUserId }` — see Q10

**Role:** add `CENTRAL_REGISTRY` to the Prisma `Role` enum. Migration will need backfill: no users get the role by default; admin assigns.

**UI:**
- `/registry/correspondence` list with filter + create.
- Detail page shows correspondence + minutes tab + postal-stamp table.
- `/registry/postal-stamps` separate flat list (for the stamp-register view).

### Z3 — Mail Box: internal user-to-user messaging  (§4.4, manual pp.13–18)

**Prisma:**
- `Message { id, fromUserId, toUserId, subject, body, sentAt, readAt?, parentMessageId? }`

**Services:** `send`, `reply`, `markRead`.

**UI:**
- `/mail/inbox`, `/mail/sent`, `/mail/compose`, `/mail/[id]`.
- Notification integration: new message writes a `Notification` row for the recipient (reuse existing `notificationService`).

### Z4 — PPRA Invoices  (§4.27, manual pp.166–174)

**Depends on Fahad's F1 (`PPRAUpload` model).** If Fahad has not shipped F1 yet, build the invoice without the `PPRAUpload` link and add it later.

**Prisma:**
- `PPRAInvoice { id, invoiceNumber (unique), receivedOn, amount, currency, status (RECEIVED|RETURNED|SENT_TO_DGPRN|PAID), returnedReason?, returnedOn?, sentToDGPRNOn?, documentKey? }`
- `PPRAInvoiceITLink { id, ppraInvoiceId, itPerformaId, ppraUploadId? }` (M:M between invoice and IT uploads)

**Services:** `createInvoice`, `returnToPPRA`, `sendToDGPRN`, `markPaid`.

**UI:** `/finance/ppra-invoices` list + create form + action buttons per row.

### Z5 — DGPRN Acknowledgement  (§4.28, manual pp.175–180)

**Prisma:**
- `DGPRNAcknowledgement { id, invoiceId?, itPerformaId?, receivedOn, referenceNumber, remarks }`

**UI:** `/finance/dgprn-acknowledgements` list + create form. Each ack can be viewed from the linked invoice's detail page.

### Z6 — Audit Objections + Audit Items + Draft Paras  (§4.29 + §4.30, manual pp.181–193)

**Role:** add `AUDIT_STAFF`.

**Prisma:**
- `AuditRecord { id, auditSection, fiscalYear, status (OPEN|CLOSED|SETTLED), subjectIndentId?, subjectContractId?, raisedOn, documentKey?, createdByUserId }`
- `AuditObjection { id, auditRecordId, ground, para, amountInvolvedMinor?, currency?, status (OPEN|REPLIED|SETTLED), replyText?, repliedOn?, repliedByUserId? }`
- `AuditItem { id, auditRecordId, itemText, status (AWAITED|REPLY|CLEARED), replyText? }`
- `AuditDraftPara { id, auditItemId, paraText, submittedOn }`
- `AuditItemObjectionLink { id, auditItemId, auditObjectionId }` (M:M)

**UI:**
- `/audit/records` list + filter + create.
- Detail page: *Audit Info · Objection List · Items · Draft Paras* tabs.

### Z7 — Court Cases + Proceeding History  (§4.31, manual pp.194–200)

**Role:** add `LEGAL_STAFF`.

**Prisma:**
- `CourtCase { id, caseNumber (unique), court, party, counterParty, status (PENDING|CLOSED|DISPOSED_OFF), relatedIndentId?, relatedContractId?, filedOn, documentKey? }`
- `CourtProceeding { id, courtCaseId, hearingDate, outcome, nextHearingDate?, notes, documentKey? }`

**UI:**
- `/legal/court-cases` list + filter + create.
- Detail: case summary + proceedings timeline + "Add proceeding" button.

### Z8 — Decision Approval (Communication lock)  (§4.42, manual pp.253–256)

**Depends on Q6 default.** Build as a separate lock from `CLOSED`. If Q6 answered differently, collapse.

**Role:** add `COMMUNICATION_STAFF`.

**Prisma:**
- Add nullable `decisionApprovalLocked: Boolean @default(false)` to `Indent` (and optionally `Contract`).
- `DecisionApprovalEvent { id, subjectType (INDENT|CONTRACT), subjectId, decidedByUserId, decidedOn, action (APPROVE|UN_APPROVE), remarks }` for audit history.

**Use cases:** `lockDecision`, `unlockDecision`.

**UI:** button on indent / contract detail: "Lock from further edits" (if role) / "Un-Approve" to reverse.

**Enforce**: every update API on the locked entity refuses while `decisionApprovalLocked = true`, EXCEPT the unlock action and comments. Pick a single enforcement point (recommend a guard clause in each affected use case's first line — `assertNotLocked(entity)`).

### Z9 — Dynamic Reporting Tool Phase 1  (§6.1)

**Depends on Q12.** Build as pre-built per-module pages. Not a query builder.

**Prisma:** none. Reports are read-only views over existing tables.

**UI:**
- `/reports` index page with cards:
  - **Indent Pipeline** — current status distribution of all indents, by section, by fiscal year.
  - **Contract Awards** — contracts awarded per month, by firm, total value.
  - **Compliance Snapshot** — count of open grievances / appeals / blacklists / misprocurement incidents.
  - **Budget Utilisation** — per budget head, allocated vs reserved vs spent.
  - **Firm Participation** — per firm, number of ITs invited vs participated vs awarded.
  - (5 more placeholder cards marked "Phase 2 — awaiting Q12 answer")
- Each report page: filters + a table or a chart from Recharts (already a React 19-compatible lib — install `recharts` if not present).
- Export-to-docx button on every report (uses Z1).

## 4. Your personal checklist

### Z1 Doc generation (SHIP FIRST)
- [ ] `npm install docx` (already installed if you look at `package.json` — confirm)
- [ ] `DocxBuilder.ts` + 5 template skeletons
- [ ] API pattern `GET /api/<entity>/[id]/download?format=docx`
- [ ] Pick PDF strategy (puppeteer vs docx-to-pdf). Document choice in commit.
- [ ] Walk: hit the download URL for one entity → verify .docx opens in Word
- [ ] Typecheck + commit: `feat(docgen): shared docx + pdf generation`

### Z2 Central Registry
- [ ] Read manual pp.41–47
- [ ] Add `CENTRAL_REGISTRY` to Role enum
- [ ] 3 new Prisma models
- [ ] `npx prisma migrate dev --name central_registry`
- [ ] Zod schemas + services + API routes
- [ ] 3 UI pages (correspondence list + detail + postal-stamp list)
- [ ] Walk: file inward mail → add minute → stamp a postal entry
- [ ] Typecheck + commit: `feat(registry): mail inward/outward + minutes + postal stamp`

### Z3 Mail Box
- [ ] Read manual pp.13–18
- [ ] `Message` model
- [ ] `npx prisma migrate dev --name mail_box`
- [ ] Zod + service + 4 API routes
- [ ] 4 pages + notification integration
- [ ] Walk: send message between two users → reply → mark read → notification bell
- [ ] Typecheck + commit: `feat(mail): internal user-to-user messaging`

### Z4 PPRA Invoices
- [ ] Read manual pp.166–174
- [ ] Coordinate with Fahad on `PPRAUpload` id shape
- [ ] 2 new models
- [ ] `npx prisma migrate dev --name ppra_invoices`
- [ ] Service with 4 methods
- [ ] API routes + Zod
- [ ] `/finance/ppra-invoices` page
- [ ] Walk: create invoice → return → re-create → send to DGPRN → mark paid
- [ ] Typecheck + commit: `feat(finance): PPRA invoices + return + DGPRN send`

### Z5 DGPRN Acknowledgement
- [ ] Read manual pp.175–180
- [ ] `DGPRNAcknowledgement` model
- [ ] `npx prisma migrate dev --name dgprn_ack`
- [ ] Service + route + page
- [ ] Walk: create ack against a PPRAInvoice
- [ ] Typecheck + commit: `feat(finance): DGPRN acknowledgement workflow`

### Z6 Audit Objections / Items / Draft Paras
- [ ] Read manual pp.181–193
- [ ] Add `AUDIT_STAFF` role
- [ ] 5 new Prisma models
- [ ] `npx prisma migrate dev --name audit`
- [ ] Services + schemas + routes
- [ ] `/audit/records` list + detail with 4 tabs
- [ ] Walk: create audit record → add objection list → add items → attach draft paras → link items to objections
- [ ] Typecheck + commit: `feat(audit): audit objections + items + draft paras module`

### Z7 Court Cases
- [ ] Read manual pp.194–200
- [ ] Add `LEGAL_STAFF` role
- [ ] 2 new Prisma models
- [ ] `npx prisma migrate dev --name court_cases`
- [ ] Services + schemas + routes
- [ ] `/legal/court-cases` list + detail with proceedings timeline
- [ ] Walk: create case → add proceeding → change status
- [ ] Typecheck + commit: `feat(legal): court cases + proceeding history`

### Z8 Decision Approval
- [ ] Read manual pp.253–256
- [ ] Add `COMMUNICATION_STAFF` role
- [ ] Add `decisionApprovalLocked` to Indent + Contract; `DecisionApprovalEvent` model
- [ ] `npx prisma migrate dev --name decision_approval_lock`
- [ ] `lockDecision` + `unlockDecision` services
- [ ] `assertNotLocked` guard called from every mutating use case on Indent/Contract
- [ ] Buttons on indent + contract detail
- [ ] Walk: lock an indent → try to edit → fails → unlock → edit succeeds
- [ ] Typecheck + commit: `feat(communication): decision approval lock on indent + contract`

### Z9 Reporting Phase 1
- [ ] `npm install recharts` (if not present)
- [ ] `/reports` index page
- [ ] 5 initial report pages (Indent Pipeline, Contract Awards, Compliance Snapshot, Budget Utilisation, Firm Participation)
- [ ] Charts rendering + filters
- [ ] Export-to-docx button (uses Z1)
- [ ] 5 Phase-2 placeholder cards
- [ ] Walk each report
- [ ] Typecheck + commit: `feat(reports): Phase 1 — 5 pre-built reports + export`

## 5. Done-when

- [ ] All 9 modules on `main`
- [ ] Typecheck clean
- [ ] Each demoed
- [ ] Doc-gen helper used by at least one Mustafa/Fahad module (coordinate the handoff)
- [ ] Q5 / Q6 / Q11 / Q12 clarified (or defaults confirmed)

---

## 6. Master prompt — paste this into Claude Code

```
You are helping Zeeshan, a software-developer intern on ProcMS (a Pakistan
Navy procurement management system, Next.js 16 + Prisma 5 + MySQL 8,
hexagonal architecture). Zeeshan owns 9 modules in the v4.4.1 migration
plan: Document generation (shared helper), Central Registry (mail I/O +
minutes + postal stamp), internal Mail Box, PPRA Invoices, DGPRN
Acknowledgement, Audit Objections + Items + Draft Paras, Court Cases,
Decision Approval lock (Communication Section), and Reporting Phase 1.

Before you do anything else, read these files in order:

1. HANDOFF.md
2. docs/MIGRATION_PLAN_FROM_V4.4.1.md — focus on §4.4, §4.5, §4.6, §4.7,
   §4.27, §4.28, §4.29, §4.30, §4.31, §4.42, §6.1, §6.2
3. docs/OPEN_QUESTIONS.md — Q5 (IMF/EMF), Q6 (Decision Approval), Q11
   (Audit/Legal in app), Q12 (which reports) touch this track
4. docs/INTERN_ASSIGNMENTS.md
5. docs/interns/zeeshan.md — the brief, the authoritative spec
6. src/infrastructure/execution/ExecutionAdminService.ts — service pattern
7. src/app/(app)/transparency/incidents/IncidentActions.tsx and
   src/app/(app)/transparency/incidents/page.tsx — list + actions pattern
8. src/interfaces/http/schemas/specialCases.ts — Zod pattern

Hard rules:

- No new hard-stop compliance refusals. Record state, do not block.
  ZeroLinkAI observes. See memory/procms_compliance_architecture.md.
- Work on main. No feature branches.
- Every schema edit followed by `npx prisma migrate dev --name <name>`.
- Every commit passes `npx tsc --noEmit -p .`.
- Module order: Z1 (doc-gen) ships FIRST because Mustafa and Fahad depend on
  it. Then Z2, Z3 (correspondence) in any order. Z4+Z5 together. Z6, Z7, Z8,
  Z9 in any order.
- Z4 (PPRA Invoices) links to Fahad's PPRAUpload model from his F1. If F1 is
  not yet on main, build Z4 without the link column and add it in a follow-up
  commit.
- Z8 (Decision Approval lock) adds an `assertNotLocked` guard call to every
  mutating use case on Indent and Contract. Do not add any OTHER hard-stop
  guards.
- Tick checklist items in docs/interns/zeeshan.md as you ship each module.

Start with Z1 (doc-gen helper). Then work the modules in the order above.

If you hit a scope question not answered by the brief or the manual, raise
it in docs/OPEN_QUESTIONS.md with a reasonable default and keep going; do
not block.

Begin by reading HANDOFF.md.
```
