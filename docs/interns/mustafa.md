# Mustafa — ProcMS v4.4.1 migration brief

**Track:** Procurement spine — Firm detail, Budget Card + Clearance, Pre-Purchase Proposal, Purchase Proposal sub-tabs.
**Modules:** 4
**Start date:** 2026-11-17
**Ship target:** 4 weeks

---

## 1. Required reading (do this first, in order)

1. [`HANDOFF.md`](../../HANDOFF.md) — what the project is and where it is
2. [`docs/INDENT_LIFECYCLE_STEPS.md`](../INDENT_LIFECYCLE_STEPS.md) — the spine as it works today
3. [`docs/SECONDARY_WORKFLOWS_STEPS.md`](../SECONDARY_WORKFLOWS_STEPS.md) — the 20 side flows already shipped
4. [`docs/MIGRATION_PLAN_FROM_V4.4.1.md`](../MIGRATION_PLAN_FROM_V4.4.1.md) — your assignments in context (§4.8, §4.11+§4.12, §4.19, §4.20)
5. [`docs/OPEN_QUESTIONS.md`](../OPEN_QUESTIONS.md) — Q3 (Budget Card) and Q4 (CFA) touch your track
6. [`docs/INTERN_ASSIGNMENTS.md`](../INTERN_ASSIGNMENTS.md) — shared conventions
7. The ProcMS v4.4.1 user manual, pages: **19–40** (Firm), **57–66** (Budget Card + Clearance), **104–107** (Pre-PP), **108–122** (Purchase Proposal)

## 2. Code patterns to copy

- Use-case shape → read [`src/application/indent/SubmitIndentUseCase.ts`](../../src/application/indent/SubmitIndentUseCase.ts) and [`src/application/indent/ForwardToBudgetUseCase.ts`](../../src/application/indent/ForwardToBudgetUseCase.ts).
- API route shape → read [`src/app/api/indents/[id]/forward-to-budget/route.ts`](../../src/app/api/indents/[id]/forward-to-budget/route.ts).
- Action button shape → read [`src/app/(app)/indents/[id]/IndentActions.tsx`](../../src/app/(app)/indents/[id]/IndentActions.tsx) (the whole file).
- Tabbed detail page → read [`src/app/(app)/indents/[id]/page.tsx`](../../src/app/(app)/indents/[id]/page.tsx).
- Admin-side form + page → read [`src/app/(app)/admin/firms/`](../../src/app/(app)/admin/firms/) as-is before you extend it.

## 3. Your 4 modules

### M1 — Firm detail tabbed page  (§4.8, manual pp.19–40)

Current `/admin/firms/[id]` shows Basic Info only. Add the missing sub-tabs.

**Prisma:**
- `FirmFinancial { id, firmId (unique), paidUpCapital, annualTurnover, bankName, bankAccountNumber, taxStatus, lastAuditedOn?, remarks }`
- `FirmItemCatalog { id, firmId, itemName, itemCode?, category, addedOn }`
- `FirmSignatory { id, firmId, personName, designation, phone?, email?, signatureImageKey?, addedOn, isActive }`
- `FirmPartner { id, firmId, partnerName, shareholdingPercent, addedOn, leftOn? }`
- `FirmAgent { id, firmId, agentName, localAddress, commissionPercent, agreementDocumentKey?, agreementFrom, agreementTo? }`
- `FirmDisciplinaryAction { id, firmId, kind (WARNING|NOTICE|PERFORMANCE_FAIL|OTHER), issuedOn, issuedByUserId, description, documentKey? }`
  - **Do not** collapse this into `Blacklist`; `Blacklist` is the hard-sanction. This is a lighter record.

**UI (`/admin/firms/[id]`):** convert to a tabbed page: *Basic Info · Financial · Items · Signatories · Partners · Agent · Discipline · Pre-qualifications* (last one links to existing `/registration`).

**APIs:** standard POST/PATCH per child entity under `/api/admin/firms/[id]/<child>`.

**Downstream consumer (do not build, just be aware):** Fahad's Contract work (§4.21) will read `FirmSignatory` when he builds the Signing Authority tab. Pick a stable id shape.

### M2 — Budget Card + Clearance / Commitment  (§4.11 + §4.12, manual pp.57–66)

New per-indent document distinct from `BudgetAllocation`.

**Prisma:**
- `BudgetCard { id, indentId (unique), controlNumber (unique, generated), fiscalYearId, budgetHeadId, reservedAmountMinor, currency, preparedByUserId, preparedAt, clearedAt?, clearedByUserId?, committedAt?, committedByUserId?, notes }`
  - Control number format: `BC/<FY>/<6-digit-seq>`. Add a seq helper in `src/infrastructure/common/`.

**Use cases:**
- `PrepareBudgetCardUseCase` — called from the indent detail page once status reaches `BUDGET_CHECK`. Allocates a control number.
- `ClearBudgetCardUseCase` — callable when indent reaches `PR_ISSUED`; moves amount from `reserved` to firmly-reserved on `BudgetAllocation`.
- `CommitBudgetCardUseCase` — callable when contract reaches `SIGNED`; moves amount from reserved → spent on `BudgetAllocation`.

**UI:**
- Button on `IndentActions` (when status = BUDGET_CHECK and no card exists yet, role BUDGET_STAFF/ADMIN): "Prepare Budget Card".
- New page `/indents/[id]/budget-card` — print-ready view of the card.
- Button on same page: "Mark Cleared" (visible when PR_ISSUED) and "Mark Committed" (visible when contract SIGNED).

**Blocks on:** Q3 in OPEN_QUESTIONS.md. Default assumption: Budget Card is a per-indent reservation worksheet separate from `BudgetAllocation`. If business says "same thing, different name", collapse into a column on the existing `BudgetAllocation`.

### M3 — Pre-Purchase Proposal + Single Offer Approval  (§4.19, manual pp.104–107)

Inserted between `CST_APPROVAL` and `PR_ISSUED` for single-quote and PROPRIETY indents.

**Prisma:**
- `PrePurchaseProposal { id, indentId (unique), kind (SINGLE_OFFER|PROPRIETARY|OTHER), justification, recommendedByUserId, recommendedOn, status (DRAFT|PENDING|APPROVED|REJECTED), approvalChainId?, rejectionReason? }`

**Use cases:**
- `CreatePrePPUseCase` — ADP_STAFF creates; aggregates existing CST + offers + indent data.
- `StartPrePPApprovalUseCase` — starts a senior-tier approval chain (reuse `CHAIN_TEMPLATES` with a new `PRE_PP` subject — add to `ApprovalSubject` enum).
- Chain outcome routes back through `ActOnApprovalUseCase` (switch add case `PRE_PP`).

**UI:** page `/indents/[id]/pre-pp`, create form, approval chain link.

**State machine hook:** add new status `PRE_PP_APPROVAL` to `IndentStatus` between `CST_APPROVAL` and `PR_ISSUED`. Only entered when `cst.singleQuote === true` or `indent.type === PROPRIETY`. Normal indents skip straight from `CST_APPROVAL` to `PR_ISSUED` as today.

### M4 — Purchase Proposal sub-tabs  (§4.20, manual pp.108–122)

Current `/prs/[id]` is lean. Add 8 tabs: *Indent · Tender · Advertisement · Suppliers Participation · Rates Received · Schedule of Store · Additional CFA · Approvals · Items · Document Attachment*.

**Prisma additions:**
- `PRAdvertisement { id, purchaseRequisitionId (unique), dgdpBulletinRef?, ppraUploadId?, dgprnPublishRef?, justificationIfNotPublished? }`
- `PRSupplierParticipation { id, purchaseRequisitionId, firmId, participated, reasonIfNotParticipated?, reasonIfNotInvited? }`
- `PRPriceAnalysis { id, purchaseRequisitionId, indentItemId, firmId, offerPriceMinor, currency, percentIncreaseVsEstimate?, justificationForIncrease? }`
- `PRScheduleOfStore { id, purchaseRequisitionId, itemDescription, quantity, uom, deliveryByDate, locationOfDelivery, totalContractValueMinor, currency, remarks }`
- `PRAdditionalCFA { id, purchaseRequisitionId (unique), cfaLevel, cfaUserId, financialAuthorityLimit, approvedOn?, remarks }`

**Downstream consumers:**
- Fahad's Contract work copies PR's `PRScheduleOfStore` into `ContractStore`.
- Zeeshan's Reports module reads `PRPriceAnalysis` for the award-report.

**Blocks on:** Q4 in OPEN_QUESTIONS.md (CFA meaning). Default: Competent Financial Authority.

## 4. Your personal checklist (tick as you go)

### M1 Firm detail
- [ ] Read manual pp.19–40
- [ ] Extend `prisma/schema.prisma` with 6 new models + relations on `Firm`
- [ ] `npx prisma migrate dev --name firm_detail_tabs`
- [ ] Zod schemas in `src/interfaces/http/schemas/firm.ts` (new file)
- [ ] CRUD services in `src/infrastructure/firm/FirmChildService.ts`
- [ ] 6 API routes under `src/app/api/admin/firms/[id]/`
- [ ] Rework `src/app/(app)/admin/firms/[id]/page.tsx` as tabbed
- [ ] Walk each tab in browser: create / edit / delete one row
- [ ] Typecheck + commit: `feat(firm): tabbed detail — Financial / Items / Signatories / Partners / Agent / Discipline`

### M2 Budget Card + Clearance
- [ ] Read manual pp.57–66
- [ ] `BudgetCard` model + control-number sequencer
- [ ] `npx prisma migrate dev --name budget_card`
- [ ] `PrepareBudgetCardUseCase` + `ClearBudgetCardUseCase` + `CommitBudgetCardUseCase`
- [ ] Register in container
- [ ] 3 API routes + Zod schemas
- [ ] `/indents/[id]/budget-card` page with print view
- [ ] Buttons wired into `IndentActions` for status gates
- [ ] Walk: create indent → BUDGET_CHECK → prepare card → check card page renders → clear after PR → commit after contract
- [ ] Typecheck + commit: `feat(budget): per-indent budget card + clearance + commitment`

### M3 Pre-Purchase Proposal
- [ ] Read manual pp.104–107
- [ ] Add `IndentStatus.PRE_PP_APPROVAL` + domain transitions
- [ ] Add `ApprovalSubject.PRE_PP` + chain template
- [ ] `PrePurchaseProposal` model
- [ ] `npx prisma migrate dev --name pre_pp`
- [ ] `CreatePrePPUseCase` + `StartPrePPApprovalUseCase`
- [ ] Switch case in `ActOnApprovalUseCase` for `PRE_PP`
- [ ] API routes
- [ ] `/indents/[id]/pre-pp` page + form
- [ ] Walk: single-quote CST → indent enters PRE_PP_APPROVAL → chain approved → moves to PR_ISSUED
- [ ] Typecheck + commit: `feat(indent): Pre-Purchase Proposal step for single-quote + propriety indents`

### M4 Purchase Proposal sub-tabs
- [ ] Read manual pp.108–122
- [ ] 5 new Prisma models on PR
- [ ] `npx prisma migrate dev --name pr_subtabs`
- [ ] Zod schemas per tab
- [ ] CRUD services
- [ ] API routes
- [ ] Rework `/prs/[id]/page.tsx` as tabbed (8 tabs)
- [ ] Walk each tab
- [ ] Typecheck + commit: `feat(pr): tabbed detail — Advertisement / Suppliers / Rates / Schedule / CFA / Approvals / Items / Attachments`

## 5. Done-when

- [ ] All 4 modules committed to `main`
- [ ] Typecheck clean
- [ ] Each module demoed to project lead
- [ ] Any Q3 / Q4 clarification merged into the plan + this brief

---

## 6. Master prompt — paste this into Claude Code

Open Claude Code in the repo root (`D:\procurement-system`). Paste this prompt verbatim as your first message:

```
You are helping Mustafa, a software-developer intern on ProcMS (a Pakistan Navy
procurement management system, Next.js 16 + Prisma 5 + MySQL 8, hexagonal
architecture). Mustafa owns 4 modules from the v4.4.1 migration plan: Firm
detail tabbed page, Budget Card + Clearance/Commitment, Pre-Purchase Proposal
(with Single Offer Approval), and Purchase Proposal sub-tabs.

Before you do anything else, read these files in order:

1. HANDOFF.md
2. docs/MIGRATION_PLAN_FROM_V4.4.1.md — focus on sections §4.8, §4.11, §4.12,
   §4.19, §4.20
3. docs/OPEN_QUESTIONS.md — Q3 (Budget Card) and Q4 (CFA) touch this track
4. docs/INTERN_ASSIGNMENTS.md — shared conventions
5. docs/interns/mustafa.md — the brief, the authoritative spec for the work
6. src/application/indent/SubmitIndentUseCase.ts and
   src/application/indent/ForwardToBudgetUseCase.ts — the use-case pattern to
   copy
7. src/app/(app)/indents/[id]/IndentActions.tsx — the action-button pattern
8. src/app/(app)/indents/[id]/page.tsx — the tabbed-detail pattern

Hard rules:

- No new hard-stop compliance refusals in any use case. New rule checks record
  state and do nothing else; ZeroLinkAI observes. See
  memory/procms_compliance_architecture.md.
- Work on main. No feature branches.
- Every schema edit is followed by `npx prisma migrate dev --name <name>`.
- Every commit passes `npx tsc --noEmit -p .`.
- Keep files small; copy the existing use-case shape, do not refactor existing
  code unless the brief says to.
- When you finish a module, tick its checklist items in docs/interns/mustafa.md.

Start with M1 (Firm detail tabbed page). When M1 is on main, start M2. Work the
checklist in docs/interns/mustafa.md item by item. If you hit a scope question
(what field does X need, what role gates Y) that is not answered by the brief
or the manual pages, raise it as a new row in docs/OPEN_QUESTIONS.md and keep
going with a reasonable default; do not block.

Begin by reading HANDOFF.md.
```
