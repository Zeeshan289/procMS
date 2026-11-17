# Open questions — waiting on business owner

**Status:** Parked 2026-11-17. Interns start work on the sensible defaults below; when any answer lands, edit the plan ([`MIGRATION_PLAN_FROM_V4.4.1.md`](./MIGRATION_PLAN_FROM_V4.4.1.md) §8) and the affected intern brief.

Each row: the question, the default assumption we are building against, the plan section it shapes, and who owns the affected module.

| # | Question | Default assumption (build on this until answered) | Blocks | Owner |
|:---:|---|---|---|---|
| Q1 | What exactly is **xMandate** (pp.67–71)? | Cross-section / external mandate approval for cross-cutting procurement. Treat as a sidebar approval with authority + document ref. | §4.13 | Deferred — do not build until answered |
| Q2 | What exactly is **DP-10 Commitment** (pp.257–260)? | DP(N) Form 10 annual budget commitment report, filed per fiscal year by Budget Section. | §4.43 | Deferred — do not build until answered |
| Q3 | Is **Budget Card** a per-indent reservation worksheet separate from `BudgetAllocation`? | Yes — per-indent `BudgetCard` child row with auto-generated Control Number, carries `reservedAmount` and the pointers back to fiscal year + budget head. | §4.11 | Mustafa |
| Q4 | Is **CFA** = Competent Financial Authority? | Yes. "Additional CFA" = the next-level-up approving authority beyond the standard chain. | §4.20 | Mustafa |
| Q5 | Is **IMF / EMF** = Internal / External Minute File? | Yes. Two kinds of minute attached to a correspondence record. | §4.6 | Zeeshan |
| Q6 | Is **Decision Approval (Communication)** a separate lock from `CLOSED`? | Yes — a `decisionApprovalLocked` boolean + role `COMMUNICATION_STAFF`, reversible with Un-Approve. Does not change the `CLOSED` semantics. | §4.42 | Zeeshan |
| Q7 | Is **Contract Endorsement** a prerequisite to taking-over? | Yes — must exist before TOC can be issued. Add it as a precondition on the `issueTakingOverCertificate` service. | §4.22 | Fahad |
| Q8 | Do you want **dynamic Role Management** of the legacy system? | No — keep the fixed Prisma enum. Document the roles clearly instead. If changed to yes, §4.41 becomes a large refactor. | §4.41 | None (deferred) |
| Q9 | Are HR / Employees / Attendance / House Hiring owned in-app by DP(N)? | Yes — build the full module. If owned by a central HR system, §4.33 drops out entirely. | §4.33 | Abdul Rafay |
| Q10 | Telephone Bills + Postal Stamps still manually tracked? | Yes — build the lightweight registers. If deprecated, §4.7 and §4.39 drop out. | §4.7, §4.39 | Abdul Rafay (bills) + Zeeshan (stamps) |
| Q11 | For **Audit Section** and **Legal Section** — are they really on ProcMS? | Yes — per "full parity" scope, we build Audit Objections / Items / Draft Paras and Court Cases in-app. | §4.29, §4.30, §4.31 | Zeeshan |
| Q12 | Which **10 reports** do operators actually run today? | Interns cannot guess. Zeeshan ships a Reports hub skeleton + 2 obvious reports (Indent Pipeline, Contract Awards) and collects a wish-list from users; the real 10 arrive in Phase 2. | §6.1 | Zeeshan |
| Q13 | Confirm business-owner contact + response-time expectations | Default: send questions via official channel; expect answers within 5 working days. If a block drags past that, raise with project lead. | Operational | Project lead |

## When an answer lands

1. Edit this file — change the "Default assumption" column to the confirmed answer, and mark the row ✅.
2. Edit [`MIGRATION_PLAN_FROM_V4.4.1.md`](./MIGRATION_PLAN_FROM_V4.4.1.md) §8 to reflect the confirmed answer.
3. Edit the owner intern's brief under [`docs/interns/`](./interns/) if the answer changes the scope.
4. Commit with message `docs(open-questions): confirm Q<n> — <answer>`.

## Last review

- **2026-11-17** — Parked all 9 original questions (+ 4 operational items) when interns started.
