# Fahad — ProcMS v4.4.1 migration brief

**Track:** Tender + Contract — IT Management (PPRA upload / newspaper / corrigendum), Contract tabbed detail, Contract Endorsement, Post-Contract deep tracking, Risk & Expense broadening.
**Modules:** 5
**Start date:** 2026-11-17
**Ship target:** 5 weeks (the Contract tabbed detail is the biggest single piece in the whole migration)

---

## 1. Required reading (do this first, in order)

1. [`HANDOFF.md`](../../HANDOFF.md)
2. [`docs/INDENT_LIFECYCLE_STEPS.md`](../INDENT_LIFECYCLE_STEPS.md)
3. [`docs/SECONDARY_WORKFLOWS_STEPS.md`](../SECONDARY_WORKFLOWS_STEPS.md)
4. [`docs/MIGRATION_PLAN_FROM_V4.4.1.md`](../MIGRATION_PLAN_FROM_V4.4.1.md) — your assignments: §4.21, §4.22, §4.23, §4.24, §4.25, §4.26
5. [`docs/OPEN_QUESTIONS.md`](../OPEN_QUESTIONS.md) — Q7 (Contract Endorsement precondition) blocks M3
6. [`docs/INTERN_ASSIGNMENTS.md`](../INTERN_ASSIGNMENTS.md) — shared conventions
7. The ProcMS v4.4.1 user manual, pages: **123–141** (Contract), **142–146** (Contract Endorsement), **147–158** (Post-Contract), **159–161** (Risk & Expense), **162–165** (IT Management)

## 2. Code patterns to copy

- Service method on existing singleton → read [`src/infrastructure/execution/ExecutionObligationService.ts`](../../src/infrastructure/execution/ExecutionObligationService.ts) (`issueTakingOverCertificate` and the new `closeContract`).
- Contract action button → read [`src/app/(app)/execution/contracts/IssueTakingOverButton.tsx`](../../src/app/(app)/execution/contracts/IssueTakingOverButton.tsx) and [`CloseContractButton.tsx`](../../src/app/(app)/execution/contracts/CloseContractButton.tsx).
- Multi-tab listing already exists for `/execution/contracts` — the LIST shape; the DETAIL page is what you'll build.
- IT Proforma detail (precedent for your IT Management work) → [`src/app/(app)/it-proformas/[id]/page.tsx`](../../src/app/(app)/it-proformas/[id]/page.tsx).

## 3. Your 5 modules

### F1 — IT Management: PPRA upload + newspaper publication + corrigendum  (§4.25 + §4.26, manual pp.162–165)

Tracks everything that happens to a tender on the outside — uploaded to PPRA, published in newspapers via DGPR(N), amended with corrigenda, or cancelled.

**Prisma:**
- `PPRAUpload { id, itPerformaId (unique), uploadedOn, ppraReferenceNumber, documentKey, status (DRAFT|UPLOADED|ACK_RECEIVED), ackReceivedOn? }`
- `NewspaperAdvertisement { id, itPerformaId, newspaperName, publishedOn, pageNumber?, costMinor?, currency, dgprnAcknowledgementRef? }` (M:1 with IT)
- `TenderCorrigendum { id, itPerformaId, corrigendumNumber, issuedOn, changeDescription, publishedOn?, documentKey? }`
- `TenderCancellation { id, itPerformaId (unique), cancelledOn, cancelledReason, documentKey? }`
- New status on `ITStatus`: `CANCELLED` (if not already there — check `src/domain/itproforma/ITStatus.ts`).

**Use cases / services** (all in `src/infrastructure/tender/ITPublicationService.ts`):
- `recordPPRAUpload`, `confirmPPRAAck`
- `recordNewspaperAd`
- `issueCorrigendum`
- `cancelTender` — transitions IT → CANCELLED and the parent indent → appropriate cancelled state.

**UI:**
- New page under IT detail: `/it-proformas/[id]/publication`
- Tabs: *PPRA Upload · Newspaper Ads · Corrigendum history · Cancellation*
- Buttons wired on the IT Proforma main page when the IT is in a publishable state.

**Downstream:** Zeeshan's PPRA Invoices (§4.27) link to `PPRAUpload` (not `ITPerforma` directly) so the invoice-to-upload correspondence is clean.

### F2 — Contract tabbed detail page  (§4.21, manual pp.123–141)

**This is your biggest module.** Rework `/execution/contracts/[id]` (does not exist yet — we only have the list page `/execution/contracts/page.tsx`) into a 14-tab composite.

**Prisma additions:**
- `ContractAnnex { id, contractId, kind ('A'|'B'|'C'), body, templateId?, documentKey? }`
- `ContractStore { id, contractId, itemDescription, quantity, uom, deliveryByDate, locationOfDelivery, notes }`
  - (Copied from Mustafa's `PRScheduleOfStore` when the contract is created. Coordinate with Mustafa.)
- `ContractTrainingClause { id, contractId, description, location, duration, costMinor, currency }`
- `ContractDeliverySchedule { id, contractId, milestoneLabel, milestonePercent, byDate, plannedLocation }`
- `PaymentTermsClause { id, contractId, milestonePercent, trigger, days, notes }`
- `ContractJurisdictionClause { id, contractId (unique), governingLaw, courtsOfJurisdiction, arbitrationClause }`
- `ContractClaim { id, contractId, kind, amountMinor, currency, claimedOn, resolvedOn?, resolution }`
- `ContractAddress { id, contractId, kind (LEGAL|OFFICIAL|BILLING|DELIVERY), addressText }`
- `ContractSigner { id, contractId, side (BUYER|SELLER), userId?, firmSignatoryId?, title, signedOn }`
  - (The `firmSignatoryId` links to Mustafa's `FirmSignatory`. Wait for Mustafa's M1 to merge before wiring this.)

**UI (`/execution/contracts/[id]/page.tsx`):** 14 tabs matching the manual spec:
- **Basic Info** (read-only summary we already have via `WorkflowListViews`)
- **Annex-A / Annex-B / Annex-C** (CRUD per annex)
- **Stores** (editable schedule)
- **Bank Guarantee** (filtered view of existing `/execution/guarantees` for this contract)
- **Training** (CRUD)
- **Delivery** (schedule; separate from the post-contract Delivery tracking in F4)
- **Payment** (payment terms + filtered view of existing invoices/payments)
- **Jurisdiction** (single row)
- **Integrity Pact** (filtered view of existing `/compliance/integrity-pacts` for this contract)
- **Claims** (CRUD)
- **Legal / Official Addresses** (CRUD)
- **Signing Authority** (CRUD; reads `FirmSignatory` for seller side)
- **Final Contract** — button that calls Zeeshan's doc-gen (M3 — Doc Gen) to produce the Word / PDF.

**Copy-from-existing button:** "Copy terms from contract…" — pops a dialog, pick a prior contract, copy its Annex / Clause / Payment / Jurisdiction rows into this one.

### F3 — Contract Endorsement  (§4.22, manual pp.142–146)

**Prisma:** `ContractEndorsement { id, contractId (unique), endorsedByUserId, endorsedOn, endorsementRef, remarks, documentKey? }`.

**Use case:** `endorseContract` — called only when contract is `SIGNED`. One endorsement per contract.

**Guard change (coordinate with Q7):** if business confirms Q7 = "endorsement is prerequisite to TOC", add precondition to `executionObligations.issueTakingOverCertificate`: refuse if no endorsement row exists. If Q7 = "parallel sign-off, informational only", skip the guard.

**UI:** button on the contract detail page's Basic Info tab (your F2); shows the endorsement details once stamped.

### F4 — Post-Contract deep tracking  (§4.23, manual pp.147–158)

Live tracking of what happens AFTER taking-over.

**Prisma:**
- `GuaranteeMonitoringEvent { id, guaranteeId, eventType (RENEWED|EXPIRY_WARNING|RELEASED), on, notes, actedByUserId? }`
- `ContractDeliveryStatus { contractId (unique), status (NOT_STARTED|SHIPPED|PARTIAL_DELIVERED|DELIVERED), lastUpdatedAt, lastUpdatedByUserId }`
- Extend `LiquidatedDamages` with optional `scheduleOfStoreId?` so LDs can be per-store.
- Extend `ContractAmendmentScope` enum with: `DELIVERY | PAYMENT_TERMS | BANK_GUARANTEE | SCHEDULE_OF_STORE` (keeping existing values).

**Use cases / services:**
- `recordGuaranteeEvent` — manual entry; e.g. the BG was renewed.
- A scheduled job in `src/jobs/` (there's already a `src/app/api/jobs/` folder — follow its shape) that writes `EXPIRY_WARNING` rows N=30 days before each guarantee expires.
- `setContractDeliveryStatus` — manual override by Procurement.
- For the new amendment scopes, `ContractAmendment` already exists — no new use case, just teach the proposal form (`src/app/(app)/governance/contract-amendments/new/ContractAmendmentForm.tsx`) the new scope options.

**UI:** new page under contract detail: `/execution/contracts/[id]/post-contract`. Sections:
- **EM / BG Monitoring** — list of events + "Add event" form.
- **Delivery status** — single dropdown + save button.
- **Stores (LD)** — table of LDs on this contract, filtered from `/execution/liquidated-damages`.
- **Amendments** — list of this contract's amendments filtered by scope.
- **Closing / Termination** — links to existing Close + Terminate buttons.

### F5 — Risk & Expense broadening  (§4.24, manual pp.159–161)

Extend `RiskPurchaseCase` to cover R&E entries that aren't tied to a termination.

**Prisma:**
- Add to `RiskPurchaseCase`: `trigger (TERMINATION|DEFAULT|DELAY_RECOVERY|OTHER)`; default existing rows to `TERMINATION` in the migration.
- `RiskExpenseEntry { id, contractId, kind (NOTICE|RECOVERY|LITIGATION|OTHER), amountMinor, currency, incurredOn, notes, documentKey? }`

**Use cases / services:** `recordRiskExpenseEntry` — standalone, not tied to a `RiskPurchaseCase`.

**UI:** add a new section on the contract detail Basic Info tab: "Risk & Expense entries" + "+ Record entry" button.

## 4. Your personal checklist

### F1 IT Management
- [ ] Read manual pp.162–165
- [ ] 4 new Prisma models + `ITStatus.CANCELLED`
- [ ] `npx prisma migrate dev --name it_publication`
- [ ] `ITPublicationService` with 5 methods
- [ ] Zod schemas in `src/interfaces/http/schemas/itPublication.ts`
- [ ] API routes under `src/app/api/it-proformas/[id]/publication/` and `src/app/api/corrigenda/`
- [ ] `/it-proformas/[id]/publication` page with 4 tabs
- [ ] Buttons on main IT page
- [ ] Walk: dispatch IT → record PPRA upload → ack it → record a newspaper ad → issue corrigendum → cancel tender
- [ ] Typecheck + commit: `feat(it): PPRA upload + newspaper publication + corrigendum + cancellation`

### F2 Contract tabbed detail
- [ ] Read manual pp.123–141
- [ ] 10 new Prisma models (wait for Mustafa's `FirmSignatory` for the signer link)
- [ ] `npx prisma migrate dev --name contract_detail_tabs`
- [ ] Services (one per child entity)
- [ ] Zod schemas (one file per child)
- [ ] CRUD API routes
- [ ] `/execution/contracts/[id]/page.tsx` as 14 tabs
- [ ] "Copy terms from" dialog
- [ ] Walk every tab: create / edit one row per
- [ ] Typecheck + commit — several commits are fine, one per tab batch. Final commit: `feat(contract): tabbed detail page — 14 sub-tabs`

### F3 Contract Endorsement
- [ ] Read manual pp.142–146
- [ ] `ContractEndorsement` model
- [ ] `npx prisma migrate dev --name contract_endorsement`
- [ ] `endorseContract` service
- [ ] API route + Zod
- [ ] Button + display on contract Basic Info
- [ ] If Q7 answered "prerequisite": add guard to `issueTakingOverCertificate`
- [ ] Walk: sign contract → endorse → (if guard in place) try TOC without endorsement → fails → with endorsement → succeeds
- [ ] Typecheck + commit: `feat(contract): endorsement step before taking-over`

### F4 Post-Contract deep tracking
- [ ] Read manual pp.147–158
- [ ] 2 new models + 1 extend + 1 enum extension
- [ ] `npx prisma migrate dev --name post_contract_tracking`
- [ ] Services
- [ ] Guarantee-expiry-warning scheduled job
- [ ] `/execution/contracts/[id]/post-contract` page
- [ ] Teach `ContractAmendmentForm` the new scope options
- [ ] Walk: create guarantee → mark it renewed → check event logged. Set delivery to PARTIAL_DELIVERED → check it displays. Assess an LD → check it appears on the per-contract view
- [ ] Typecheck + commit: `feat(contract): post-contract EM/BG monitoring + delivery + per-store LD + typed amendments`

### F5 Risk & Expense broadening
- [ ] Read manual pp.159–161
- [ ] `trigger` on `RiskPurchaseCase` + `RiskExpenseEntry`
- [ ] `npx prisma migrate dev --name risk_expense_broadening`
- [ ] Service + route
- [ ] Button + display on contract detail
- [ ] Walk: record R&E entry on a non-terminated contract → check list rendering
- [ ] Typecheck + commit: `feat(contract): standalone risk & expense entries + trigger taxonomy on RiskPurchaseCase`

## 5. Done-when

- [ ] All 5 modules committed to `main`
- [ ] Typecheck clean
- [ ] Each module demoed to project lead
- [ ] Q7 clarified (or default assumption documented as confirmed)

---

## 6. Master prompt — paste this into Claude Code

```
You are helping Fahad, a software-developer intern on ProcMS (a Pakistan Navy
procurement management system, Next.js 16 + Prisma 5 + MySQL 8, hexagonal
architecture). Fahad owns 5 modules in the v4.4.1 migration plan: IT Management
(PPRA upload + newspaper publication + corrigendum / cancellation), Contract
tabbed detail page (14 sub-tabs — the biggest single module in the whole
migration), Contract Endorsement, Post-Contract deep tracking, and Risk &
Expense broadening.

Before you do anything else, read these files in order:

1. HANDOFF.md
2. docs/MIGRATION_PLAN_FROM_V4.4.1.md — focus on §4.21, §4.22, §4.23, §4.24,
   §4.25, §4.26
3. docs/OPEN_QUESTIONS.md — Q7 (Contract Endorsement precondition to TOC)
   shapes F3
4. docs/INTERN_ASSIGNMENTS.md — shared conventions
5. docs/interns/fahad.md — the brief, the authoritative spec
6. src/infrastructure/execution/ExecutionObligationService.ts — service pattern
7. src/app/(app)/execution/contracts/IssueTakingOverButton.tsx and
   CloseContractButton.tsx — action-button pattern
8. src/app/(app)/it-proformas/[id]/page.tsx — detail-page pattern
9. src/app/api/contracts/[id]/close/route.ts — API route pattern

Hard rules:

- No new hard-stop compliance refusals. Record state, do not block. ZeroLinkAI
  observes. See memory/procms_compliance_architecture.md.
- Work on main. No feature branches.
- Every schema edit followed by `npx prisma migrate dev --name <name>`.
- Every commit passes `npx tsc --noEmit -p .`.
- F2 (Contract Signing Authority) consumes Mustafa's FirmSignatory model.
  Coordinate with Mustafa — his M1 ships before your Signing Authority tab.
  Build the other 13 Contract tabs in parallel if Mustafa is still working.
- F1 (PPRA Upload) is what Zeeshan's PPRAInvoices will link to — pick a stable
  id shape before Zeeshan starts PPRAInvoices.
- Tick the checklist in docs/interns/fahad.md as you ship each module.

Start with F1 (IT Management). Then F3 (Endorsement), F4 (Post-Contract), F5
(Risk & Expense broadening) — F2 (Contract tabbed detail) is biggest and
should come last so the others don't get starved.

If you hit a scope question not answered by the brief or the manual, raise it
in docs/OPEN_QUESTIONS.md with a reasonable default and keep going; do not
block.

Begin by reading HANDOFF.md.
```
