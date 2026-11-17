# Intern assignments — ProcMS v4.4.1 migration

**Date:** 2026-11-17
**Four interns** picking up the migration backlog from [`MIGRATION_PLAN_FROM_V4.4.1.md`](./MIGRATION_PLAN_FROM_V4.4.1.md). Each intern owns a self-contained track. Tracks were split to keep file-level merge conflicts rare (each intern mostly touches different `src/app/(app)/...` sub-trees and different `src/application/...` sub-trees).

## Index

| Intern | Track | Modules | Brief |
|---|---|---|---|
| **Mustafa** | Procurement spine — Firm detail + Budget Card + Pre-PP + Purchase Proposal | 4 | [interns/mustafa.md](./interns/mustafa.md) |
| **Fahad** | Tender + Contract — IT Management, Contract tabs, Endorsement, Post-Contract, Risk & Expense | 5 | [interns/fahad.md](./interns/fahad.md) |
| **Zeeshan** | Finance ops + correspondence + governance back-office + doc gen + reports | 8 | [interns/zeeshan.md](./interns/zeeshan.md) |
| **Abdul Rafay** | HR + stores + office ops + dashboard charts | 9 | [interns/abdul_rafay.md](./interns/abdul_rafay.md) |

## Dependencies between tracks

- **Fahad's Contract work** uses the Firm Signatories schema **Mustafa** introduces. Mustafa ships §4.8 first; Fahad consumes it.
- **Zeeshan's PPRA Invoices** link to IT Proformas — can proceed in parallel with **Fahad's IT Management**.
- **Zeeshan's Document generation** is used by Mustafa (CST/PP print), Fahad (Contract), and Zeeshan himself. Zeeshan ships the doc-gen helper library first, then the templates.
- **Abdul Rafay's Dashboard charts** are optional polish — no one else depends on them.

## Shared conventions (every intern reads once)

- **Architecture**: hexagonal. `src/domain` owns aggregates + events, `src/application` owns use cases that run inside the UoW, `src/infrastructure` talks to Prisma, `src/interfaces` renders UI + HTTP.
- **Compliance rule**: see [`memory`](../memory/) → **no new hard-stop refusals** in any use case. New rule checks record state, do not block. ZeroLinkAI observes.
- **Git workflow**: work on `main` directly (no per-feature branches per the user's standing rule). Small frequent commits over big batches. Attribution per repo's CLAUDE.md.
- **Testing**: no test runner has been introduced; walk the UI after each module and record in each brief's checklist.
- **Prisma migrations**: `npx prisma migrate dev --name <descriptive_name>` after every schema edit, then commit the generated migration folder.
- **Typecheck before commit**: `npx tsc --noEmit -p .` must be clean.

## Required reading before starting (each intern)

1. [`HANDOFF.md`](../HANDOFF.md) — project state
2. [`docs/INDENT_LIFECYCLE_STEPS.md`](./INDENT_LIFECYCLE_STEPS.md) — the spine as it exists today
3. [`docs/SECONDARY_WORKFLOWS_STEPS.md`](./SECONDARY_WORKFLOWS_STEPS.md) — the 20 side flows already shipped
4. [`docs/MIGRATION_PLAN_FROM_V4.4.1.md`](./MIGRATION_PLAN_FROM_V4.4.1.md) — the plan this work is executing
5. [`docs/OPEN_QUESTIONS.md`](./OPEN_QUESTIONS.md) — what's still waiting on business owner
6. Your own brief (see Index above)

## Standard build checklist per module (every intern uses this)

Each intern's brief repeats this with module-specific details. The general shape:

- [ ] Read the manual pages for the module (cited in your brief)
- [ ] Update `prisma/schema.prisma` with new models / fields
- [ ] Run `npx prisma migrate dev --name <name>` — commit the generated migration
- [ ] (If new domain aggregate) Create `src/domain/<module>/<Aggregate>.ts` with typed commands + events
- [ ] Create `src/application/<module>/<UseCase>.ts` following the `SubmitIndentUseCase` pattern
- [ ] Register in `src/infrastructure/composition/container.ts`
- [ ] Create `src/interfaces/http/schemas/<module>.ts` with Zod schemas
- [ ] Create API route `src/app/api/<path>/route.ts` with `requireAuth([roles])` + `toHttpResponse`
- [ ] Create view loader `src/interfaces/ui/views/<ModuleView>.ts`
- [ ] Create list page `src/app/(app)/<path>/page.tsx`
- [ ] Create new form `src/app/(app)/<path>/new/<Form>.tsx` + page
- [ ] Create detail page / tabs as spec'd
- [ ] Add action buttons / `*Actions.tsx` client components
- [ ] Typecheck: `npx tsc --noEmit -p .`
- [ ] Walk the UI in the browser end-to-end
- [ ] Commit with `feat(<module>): <short description>` message

## Escalation

- A question about scope / what the business actually wants → raise in [`docs/OPEN_QUESTIONS.md`](./OPEN_QUESTIONS.md), add a row, flag project lead.
- A blocker from another intern's unshipped work → ping them directly, don't silently build around it.
- A bug in already-shipped code → fix in the same commit as your module, note in the commit message `fix(<other-module>): ...`.
