# Abdul Rafay — ProcMS v4.4.1 migration brief

**Track:** HR + Stores + Office operations + Departmental hierarchy UI + Dashboard charts.
**Modules:** 9
**Start date:** 2026-11-17
**Ship target:** 4 weeks

---

## 1. Required reading (do this first, in order)

1. [`HANDOFF.md`](../../HANDOFF.md)
2. [`docs/INDENT_LIFECYCLE_STEPS.md`](../INDENT_LIFECYCLE_STEPS.md)
3. [`docs/SECONDARY_WORKFLOWS_STEPS.md`](../SECONDARY_WORKFLOWS_STEPS.md)
4. [`docs/MIGRATION_PLAN_FROM_V4.4.1.md`](../MIGRATION_PLAN_FROM_V4.4.1.md) — §4.32, §4.33, §4.34, §4.35, §4.36, §4.37, §4.38, §4.39, §6.5
5. [`docs/OPEN_QUESTIONS.md`](../OPEN_QUESTIONS.md) — Q9 (HR ownership), Q10 (Telephone / Postal still used) touch this track
6. [`docs/INTERN_ASSIGNMENTS.md`](../INTERN_ASSIGNMENTS.md)
7. The ProcMS v4.4.1 user manual, pages: **201–203** (Departmental Hierarchy), **204–216** (Employees + Attendance), **217–220** (Demand Raising), **221–229** (Internal Procurement), **230–231** (Inventory Ledger), **232–235** (Vehicle Requisition), **236–241** (Work Order), **239–242** (Telephone Bills)

## 2. Code patterns to copy

- Admin-side listing + form → read [`src/app/(app)/admin/sections/`](../../src/app/(app)/admin/sections/) and [`src/app/(app)/admin/firms/`](../../src/app/(app)/admin/firms/).
- Bulk-operation UI pattern → the Committee member add/remove in [`src/app/(app)/governance/committees/CommitteeMembershipActions.tsx`](../../src/app/(app)/governance/committees/CommitteeMembershipActions.tsx).
- Service singleton pattern → [`src/infrastructure/governance/CommitteeService.ts`](../../src/infrastructure/governance/CommitteeService.ts).
- Dashboard existing → [`src/interfaces/ui/views/DashboardView.ts`](../../src/interfaces/ui/views/DashboardView.ts) and [`src/app/(app)/dashboard/page.tsx`](../../src/app/(app)/dashboard/page.tsx).

## 3. Your 9 modules

### AR1 — Departmental hierarchy as a tree view  (§4.32, manual pp.201–203)

Current `/admin/sections` works but renders flat. Convert to a collapsible tree (we have `parentId` + `children` on `Section` already).

**Prisma:** none.

**UI:** rewrite the list page to render recursively; keep the existing create/edit flow intact.

**Done quickly** — this is a UX polish, ship first as a warm-up.

### AR2 — Employees + Service / Leave / House Hiring / Disciplinary + Attendance  (§4.33, manual pp.204–216)

**Depends on Q9.** Default: build in-app. If Q9 says "owned by central HR", skip and move to AR3.

**Role:** add `HR_STAFF`.

**Prisma:**
- `Employee { id, userId? (unique nullable), personnelNumber (unique), rank, designation, section, dateOfJoining, dateOfBirth, bloodGroup, contact, maritalStatus, isActive, createdAt, updatedAt }`
- `EmployeeServiceRecord { id, employeeId, postingFrom, postingTo?, section, remarks }`
- `EmployeeLeave { id, employeeId, kind (CASUAL|EARNED|MEDICAL|ABSENT), fromDate, toDate, approvedByUserId?, remarks }`
- `EmployeeHouseHiring { id, employeeId, address, landlord, hireStart, hireEnd?, monthlyRentMinor, currency, documentKey? }`
- `EmployeeDisciplinaryAction { id, employeeId, kind, issuedOn, authority, decision, remarks, documentKey? }`

**UI:**
- `/hr/employees` — list + filter + create form.
- `/hr/employees/[id]` — tabbed: *Basic · Service · Leave · House Hiring · Disciplinary*.
- `/hr/attendance` — bulk-marking page: pick date → select section → select employees → mark Leave type / Absent.

### AR3 — Demand Raising  (§4.34, manual pp.217–220)

Internal item-demand (NOT the procurement indent — this is the upstream request from a section asking for items).

**Prisma:**
- `InternalDemand { id, demandNumber (unique generated), raisedByUserId, raisedForSectionId, raisedOn, status (OPEN|ISSUED|CLOSED), notes }`
- `InternalDemandItem { id, demandId, itemName, quantity, uom, remarks }`

**UI:** `/stores/demands` list + create + edit + close action.

### AR4 — Internal Procurement (small purchases)  (§4.35, manual pp.221–229)

Lightweight parallel to the main Indent→Contract flow for small discretionary spend.

**Prisma:**
- `InternalProcurement { id, reference (unique generated), raisedOn, raisedByUserId, status (DRAFT|QUOTED|ORDERED|CLOSED|CANCELLED), notes }`
- `InternalProcurementItem { id, procurementId, itemName, quantity, uom }`
- `InternalProcurementInvitedFirm { id, procurementId, firmId }`
- `InternalProcurementQuotation { id, procurementId, firmId, itemId, quotedPriceMinor, currency }`
- `SupplyOrder { id, procurementId (unique), firmId, orderedOn, totalValueMinor, currency, documentKey? }` — auto-suggest firm with lowest total.
- `SanctionCertificate { id, procurementId, certificateNumber, issuedOn, authority, documentKey? }`
- `DebitableVoucher { id, procurementId, voucherNumber, amountMinor, currency, issuedOn }`
- `ContingentBill { id, procurementId, firmId, billNumber, amountMinor, currency, submittedOn, paidOn? }`

**UI:**
- `/stores/procurement` — list + create.
- Detail: tabs for *Items · Invited Firms · Quotations · Supply Order · Sanction Cert · Debitable Voucher · Contingent Bill*.

### AR5 — Inventory Ledger  (§4.36, manual pp.230–231)

**Prisma:**
- `InventoryLedgerEntry { id, kind (ISSUANCE|RECEIVING), happenedOn, sourceDemandId?, sourceITPerformaId?, itemName, quantity, uom, enteredByUserId, remarks }`

**UI:** `/stores/ledger` — list + create. The create form picks source (demand or tender) + auto-fills items.

### AR6 — Vehicle Requisition  (§4.37, manual pp.232–235)

**Role:** no new role — any employee can raise.

**Prisma:**
- `VehicleRequisition { id, raisedByUserId, purpose, fromLocation, toLocation, requiredFrom, requiredTo, passengerCount, status (PENDING|APPROVED|REJECTED|COMPLETED), approvedByUserId?, decidedOn?, assignedVehicleRef?, remarks }`

**UI:** `/admin/transport/requisitions` — list + create. Approver actions on list rows.

### AR7 — Work Order  (§4.38, manual pp.236–241)

**Prisma:**
- `WorkOrder { id, workOrderNumber (unique generated), kind (MAINTENANCE|REPAIR|OTHER), subject, raisedOn, raisedByUserId, assignedTo, status (OPEN|IN_PROGRESS|COMPLETED|CANCELLED), completedOn?, costMinor?, currency?, notes }`

**UI:** `/admin/work-orders` — list + create + status progression buttons.

### AR8 — Telephone Bills  (§4.39, manual pp.239–242)

**Depends on Q10.** Default: build. If Q10 says deprecated, skip.

**Prisma:**
- `TelephoneBill { id, phoneNumber, billMonth, billAmountMinor, currency, dueDate, paidOn?, documentKey?, enteredByUserId }`

**UI:** `/admin/telephone-bills` — list + create + mark-paid action.

### AR9 — Dashboard charts / KPIs  (§6.5)

**Depends on:** `recharts` installed (Zeeshan's Z9 will install it; coordinate).

**Build:**
- Add 3 charts to the existing dashboard:
  - **Indent pipeline bar chart** (stacked by status, split by section)
  - **Contract awards trend** (line chart, month over month)
  - **Budget utilisation pie** (allocated / reserved / spent, per fiscal year)
- Each chart links to its full report (Zeeshan's Z9 reports).

## 4. Your personal checklist

### AR1 Departmental hierarchy tree
- [ ] Rewrite `/admin/sections/page.tsx` as collapsible tree
- [ ] Walk: see nesting render; create / edit still works
- [ ] Typecheck + commit: `feat(sections): tree view for departmental hierarchy`

### AR2 Employees + HR
- [ ] Confirm Q9 before building; if "central HR", skip this module
- [ ] Add `HR_STAFF` role
- [ ] 5 Prisma models
- [ ] `npx prisma migrate dev --name hr_employees`
- [ ] Services + schemas + routes
- [ ] `/hr/employees` + `/hr/employees/[id]` tabbed + `/hr/attendance`
- [ ] Walk: create employee → add service record → mark leave → bulk attendance
- [ ] Typecheck + commit: `feat(hr): employees + service/leave/house-hiring/disciplinary + attendance`

### AR3 Demand Raising
- [ ] Read manual pp.217–220
- [ ] 2 Prisma models + sequencer
- [ ] `npx prisma migrate dev --name internal_demand`
- [ ] Service + schemas + routes
- [ ] `/stores/demands` list + create
- [ ] Walk: raise demand → add items → close
- [ ] Typecheck + commit: `feat(stores): internal item demand raising`

### AR4 Internal Procurement
- [ ] Read manual pp.221–229
- [ ] 8 Prisma models
- [ ] `npx prisma migrate dev --name internal_procurement`
- [ ] Services + schemas + routes (one per child entity)
- [ ] `/stores/procurement` list + tabbed detail
- [ ] Walk: create procurement → invite firms → enter quotations → auto-suggest firm → place supply order → record sanction cert → debitable voucher → contingent bill
- [ ] Typecheck + commit: `feat(stores): internal procurement + supply order + sanction cert + debitable voucher + contingent bill`

### AR5 Inventory Ledger
- [ ] Read manual pp.230–231
- [ ] `InventoryLedgerEntry` model
- [ ] `npx prisma migrate dev --name inventory_ledger`
- [ ] Service + schema + route
- [ ] `/stores/ledger` page
- [ ] Walk: issuance against demand → receiving against tender
- [ ] Typecheck + commit: `feat(stores): inventory ledger — issuance / receiving entries`

### AR6 Vehicle Requisition
- [ ] Read manual pp.232–235
- [ ] `VehicleRequisition` model + sequencer
- [ ] `npx prisma migrate dev --name vehicle_requisition`
- [ ] Service + schemas + routes
- [ ] `/admin/transport/requisitions` page
- [ ] Walk: raise → approve → mark completed
- [ ] Typecheck + commit: `feat(transport): vehicle requisition workflow`

### AR7 Work Order
- [ ] Read manual pp.236–241
- [ ] `WorkOrder` model + sequencer
- [ ] `npx prisma migrate dev --name work_order`
- [ ] Service + schemas + routes
- [ ] `/admin/work-orders` page with status buttons
- [ ] Walk: create → assign → progress → complete
- [ ] Typecheck + commit: `feat(admin): work order workflow`

### AR8 Telephone Bills
- [ ] Confirm Q10 before building; if deprecated, skip this module
- [ ] `TelephoneBill` model
- [ ] `npx prisma migrate dev --name telephone_bills`
- [ ] Service + routes + page
- [ ] Walk: create bill → mark paid
- [ ] Typecheck + commit: `feat(admin): telephone bills register`

### AR9 Dashboard charts
- [ ] Wait for Zeeshan to install `recharts` (or install it yourself)
- [ ] 3 charts on the existing dashboard
- [ ] Links to reports index (Zeeshan Z9)
- [ ] Walk the dashboard; charts render in both themes
- [ ] Typecheck + commit: `feat(dashboard): 3 KPI charts — indent pipeline, awards trend, budget utilisation`

## 5. Done-when

- [ ] All 9 modules on `main` (or skip-reason documented for AR2 / AR8 if Q9 / Q10 answered negatively)
- [ ] Typecheck clean
- [ ] Each demoed
- [ ] Q9, Q10 confirmed or defaults recorded

---

## 6. Master prompt — paste this into Claude Code

```
You are helping Abdul Rafay, a software-developer intern on ProcMS (a
Pakistan Navy procurement management system, Next.js 16 + Prisma 5 + MySQL
8, hexagonal architecture). Abdul Rafay owns 9 modules in the v4.4.1
migration plan: Departmental hierarchy tree UI, Employees + Service / Leave
/ House Hiring / Disciplinary + Attendance, Demand Raising, Internal
Procurement (with Supply Order + Sanction Cert + Debitable Voucher +
Contingent Bill), Inventory Ledger, Vehicle Requisition, Work Order,
Telephone Bills, and Dashboard KPI charts.

Before you do anything else, read these files in order:

1. HANDOFF.md
2. docs/MIGRATION_PLAN_FROM_V4.4.1.md — focus on §4.32, §4.33, §4.34, §4.35,
   §4.36, §4.37, §4.38, §4.39, §6.5
3. docs/OPEN_QUESTIONS.md — Q9 (HR in-app?) and Q10 (Telephone bills still
   used?) touch this track. Confirm Q9 BEFORE starting AR2 (if answer is
   central-HR, skip AR2 entirely); confirm Q10 before AR8.
4. docs/INTERN_ASSIGNMENTS.md
5. docs/interns/abdul_rafay.md — the brief, the authoritative spec
6. src/app/(app)/admin/sections/ and src/app/(app)/admin/firms/ — admin
   list/form pattern
7. src/app/(app)/governance/committees/CommitteeMembershipActions.tsx —
   bulk-ops UI pattern
8. src/infrastructure/governance/CommitteeService.ts — service pattern

Hard rules:

- No new hard-stop compliance refusals. Record state, do not block.
  ZeroLinkAI observes. See memory/procms_compliance_architecture.md.
- Work on main. No feature branches.
- Every schema edit followed by `npx prisma migrate dev --name <name>`.
- Every commit passes `npx tsc --noEmit -p .`.
- Module order: AR1 is a 1-hour warm-up — ship it first. Then AR3 → AR4 →
  AR5 (stores track). Then AR6 → AR7 (admin track). Then AR2 (big HR
  module) and AR8 (small) in parallel if time. AR9 last — depends on
  Zeeshan's recharts install.
- Tick checklist items in docs/interns/abdul_rafay.md as you ship each
  module.

If you hit a scope question not answered by the brief or the manual, raise
it in docs/OPEN_QUESTIONS.md with a reasonable default and keep going; do
not block.

Begin by reading HANDOFF.md.
```
