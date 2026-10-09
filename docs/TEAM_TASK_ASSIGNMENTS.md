# Team task assignments

**Date:** 2026-10-09. Role allocation confirmed by the user: Phat leads backend, Bảo leads frontend, and Huy takes the remaining catalog/builder/optimizer track. Skills and weekly availability still need to be recorded. Implementation has not started. The twelve-week sequence is a core milestone estimate; production has a separate estimate and readiness gates.

Use this document for ownership and handoffs. Use the [implementation plans](README.md#implementation-plans-and-dependencies) for task files, interfaces, acceptance cases and verification commands. The [specification](superpowers/specs/2026-10-06-electronics-store-design.md) remains the canonical technical contract.

## Roles

| Member | Primary responsibility | Expected skills and support |
| --- | --- | --- |
| Phat | Java backend, accounts, cart, checkout and payment recovery | Java/SQL learning track: transactions, optimistic locking, idempotency and durable jobs. Huy pairs on concurrency and payment recovery. Phat coordinates migrations. |
| Huy | Catalog, hardware validation, saved builds and Python optimizer | Track requires Python and Java domain services; Huy’s experience is still to be recorded. Phat pairs on Java persistence and transaction boundaries; Bảo helps with connected journeys. |
| Bảo | React frontend, integration tooling and delivery coordination | React/TypeScript track plus API contracts, Docker, CI and operational evidence. Phat and Huy contribute service-specific deployment and security work. |

Task count does not measure difficulty. Phat’s checkout work and Huy’s solver work are substantial. Bảo can advance UI against reviewed fixtures while APIs are developed, then help with integration. Review capacity and learning time must be included when estimating. Rebalance at each milestone if one member is blocked or overloaded.

Each task has one accountable owner and a different reviewer. The owner integrates contributions and records evidence; the reviewer checks the named plan cases and contract compatibility. A task can have pairing support without having two owners.

## Assignment directly from the reviewed plans

The task IDs below refer to the existing plan tasks. Their files, interfaces, acceptance cases and verification commands stay in those plans. This document adds member ownership; it does not replace the implementation instructions. Phat leads the Java backend, with selected Java domain tasks assigned to Huy and reviewed by Phat to distribute the work.

| Reviewed plan | Phat's owned tasks | Bảo's owned tasks | Huy's owned tasks |
| --- | --- | --- | --- |
| [Backend foundation](superpowers/plans/2026-10-07-backend-foundation.md) | F1, F2, F3, F5, F7 | — | F4, F6 |
| [Checkout and payment recovery](superpowers/plans/2026-10-07-checkout-pipeline.md) | C1–C6 | — | C7 |
| [Builder and optimizer](superpowers/plans/2026-10-07-pc-builder-optimizer.md) | — | — | B1–B6; B6 optional |
| [Frontend](superpowers/plans/2026-10-07-frontend.md) | — | U1–U6 | — |
| [Integration and delivery](superpowers/plans/2026-10-08-integration-delivery.md) | Supports backend checks | D1, D2, D4 | D3 |
| [Production launch](superpowers/plans/2026-10-08-production-launch.md) | P2, P4 | P1, P5, P6, P7 | P3 |

## Phat — 13 owned tasks

**Detailed instructions:** [Phat's task cards](tasks/PHAT.md), using the [shared workflow](tasks/README.md).

Begin with F1, then F2. Review Huy’s Java changes and contribute backend evidence to integration and launch tasks.

| Task | Deliverable | Stage | Reviewer |
| --- | --- | --- | --- |
| F1 | Java scaffold and public contracts | Core | Huy |
| F2 | Foundation schema and exact VND money | Core | Huy |
| F3 | Login, refresh rotation and logout | Core | Bảo |
| F5 | Owned cart and atomic build groups | Core | Huy |
| F7 | Quotas and backend security boundaries | Core | Huy |
| C1 | Checkout schema and reviewed quotes | Core | Huy |
| C2 | Atomic stock allocation and checkout idempotency | Core | Huy |
| C3 | MoMo adapter and durable payment worker | Core | Huy |
| C4 | Signed notifications and capture finalization | Core | Huy |
| C5 | Safe expiry and cancellation | Core | Huy |
| C6 | Payment retry within the original hold | Core | Huy |
| P2 | Verified email, recovery, MFA and email outbox | Production | Bảo |
| P4 | Returns, partial refunds and inspected restock | Production | Huy |

## Bảo — 13 owned tasks

**Detailed instructions:** [Bảo's task cards](tasks/BAO.md), using the [shared workflow](tasks/README.md).

Begin with U1 using reviewed API fixtures. Service owners contribute to the integration and operations work Bảo coordinates.

| Task | Deliverable | Stage | Reviewer |
| --- | --- | --- | --- |
| U1 | Frontend scaffold and session client | Core | Phat |
| U2 | Product discovery and detail views | Core | Huy |
| U3 | Manual builder and saved drafts UI | Core | Huy |
| U4 | Grouped cart and quote review UI | Core | Phat |
| U5 | Checkout identity and payment recovery UI | Core | Phat |
| U6 | Admin operations and accessibility | Core | Huy |
| D1 | Compose topology and deterministic seeds | Core | Huy |
| D2 | Contract parity and connected user journeys | Core | Phat |
| D4 | CI, study metrics and core evidence ledger | Core | Huy |
| P1 | Merchant readiness and live-mode guard | Production | Phat |
| P5 | Shipment, customer policies and data governance | Production | Huy |
| P6 | Secure deployment, backup restore and recovery rehearsal | Production | Phat |
| P7 | Production evidence and controlled launch coordination | Production | Huy |

## Huy — 11 owned tasks

**Detailed instructions:** [Huy's task cards](tasks/HUY.md), using the [shared workflow](tasks/README.md).

Begin with B3; start F4 after F2. Defer optional B6 until the required core tasks and provider decision permit it.

| Task | Deliverable | Stage | Reviewer |
| --- | --- | --- | --- |
| F4 | Catalog reads, filters and revisioned cache | Core | Phat |
| F6 | Admin publication and audited stock changes | Core | Phat |
| B1 | Authoritative hardware compatibility rules | Core | Phat |
| B2 | Saved drafts, revisions and staleness notices | Core | Phat |
| B3 | Python scaffold and private optimizer contracts | Core | Phat |
| B4 | Constraint search, proof statuses and deadlines | Core | Phat |
| B5 | Java optimizer integration and outage handling | Core | Phat |
| B6 | Optional bounded AI explanation adapter | Optional enhancement | Bảo |
| C7 | Receipts, fulfillment and compensation | Core | Phat |
| D3 | Concurrency, restart recovery and MoMo sandbox checks | Core | Phat |
| P3 | Versioned Vietnamese shipping and tax totals | Production | Phat |

## Shared support and release gates

For example, Phat implements C2 using the checkout plan’s atomic allocation and replay cases; Bảo consumes that behavior in U5 using the frontend plan’s checkout recovery cases; Huy coordinates its concurrency verification in D3. These are producer, consumer and verification responsibilities for the same connected checkout journey.

B6 is optional: defer it until core correctness is demonstrated and the provider/model decision is recorded. Phat reviews the Java boundary and privacy controls if it is enabled. For D1, Phat contributes backend configuration and Huy supplies curated hardware seeds and optimizer configuration. For D2–D4, every service owner fixes failures in their service; the task owner coordinates the shared evidence.

For C2–C6 and D3, Phat and Huy pair on transaction and recovery cases. Bảo exercises reload, double-click, redirect and retry behavior through the UI. These cases must pass against real PostgreSQL and the connected stack where the plans require it.

Start production implementation after core evidence and a separate capacity estimate. P1 can gather merchant and business inputs during core development. Its completion still depends on the required evidence.

Bảo coordinates P1/P5/P7 but cannot supply merchant approval, business policy or legal review on behalf of their actual owners. Phat implements and reviews payment capability guards for P1 and contributes shipment/privacy backend endpoints to P5. Huy contributes commerce/retention cases to P5 and optimizer capacity checks to P6. Phat and Huy own their runtime secrets, health checks and recovery procedures under P6. P7 requires every service owner’s evidence and the launch approvals specified in the production plan.

## Dependencies and work sequence

| Stage | Phat | Huy | Bảo | Exit evidence |
| --- | --- | --- | --- | --- |
| Weeks 1–2 | F1, F2, F3 | B3; F4 after F2 | U1, then U2 fixtures; begin D1 after scaffolds exist | Reviewed contracts, runnable service scaffolds, foundation schema and account/catalog checks |
| Week 3 | F5 after B1; F7 after F3/F4 | B1 after F2/F4, then F6 | Connect U1/U2; start U3/U4 fixtures | Rules, cart groups, admin publication and protection checks |
| Weeks 4–5 | C1, then C2 | B2, B4 | U3, U4; continue D1 | Draft/search checks and reviewed quotes/atomic allocation |
| Weeks 6–7 | C3, C4, C5, C6 in dependency order | B5; pair on payment recovery | U5 fixtures and progressive integration | Sandbox authorization/capture/closing/retry evidence; optimizer boundary checks |
| Week 8 | Support C7 and review payment integration | C7 | Complete connected U5/U6 | Receipts, compensation and admin/fulfillment journeys |
| Weeks 9–10 | Pair on D3 and fix backend failures | D3; contribute D2 parity cases | D2; assemble D4 evidence | Connected journeys, concurrency, restarts and MoMo sandbox smoke |
| Weeks 11–12 | Fix remaining core failures and contribute metrics | Fix remaining core failures and contribute algorithm evidence | Complete D4 with CI, load/usability results | Core acceptance evidence and separately reviewed production estimate |
| After core | P2, P4; support P1/P5/P6/P7 | P3; support P4/P5/P6/P7 | Coordinate P1, P5, P6, P7 | All production-plan gates and owner approvals |

This table provides ordering rather than guaranteed completion dates. F1–F4 and B1/F5/F6/F7 are prerequisites for checkout. B4 consumes B3 and shared B1 rule fixtures; B5 consumes B4, F4 and B1. U5 can be designed against contracts earlier, but completes only with C1–C7. U6 completes with F6/C7. D4 completes after D2/D3. Production completion follows the production plan's dependencies and gates.

## First working session

1. Record each member’s Java/Python/React experience and available hours per week. Confirm Huy’s support needs for Java services and review work, then adjust task size and pairing time.
2. Read the PRD journeys and specification's shared types/API contracts together. Record API changes in the specification and matching plans before implementing consumers.
3. Establish the shared repository and PR workflow. Phat starts F1/F2; Huy starts B3 and prepares hardware fixtures; Bảo starts U1 against specification fixtures. F4 waits for F2's schema. Python rule implementation waits for agreed B1 fixtures.
4. Agree on one first connected slice: account login and catalog discovery through the real frontend/backend/database. Payment work follows foundation and builder prerequisites.
5. Estimate the next small batch with owners and reviewers; reserve pairing time for difficult backend cases. Compare actual progress weekly and adjust the next batch.

## Handoffs and shared files

- Bảo maintains `contracts/public.openapi.yaml` under D2, with Phat reviewing backend fields and errors. Publish the necessary specification-derived contracts/fixtures early for UI work; connected contract verification completes under D2. Huy owns `contracts/optimizer.openapi.yaml` under B3 and compatibility fixtures under B1. Bảo generates frontend types from reviewed public contracts. Every cross-service contract change is reviewed by its producer and consumer; fixture-only UI work is not connected acceptance evidence.
- Phat coordinates migrations: F2/Phat owns V1; B2/Huy owns V2; C1/Phat owns V3; P2/Phat owns V4; P3/Huy owns V5; P4/Phat owns V6; P5/Bảo coordinates V7 with Phat implementing its backend changes. Keep the migration filenames allocated in the specification. An applied migration is not rewritten; reserve the next corrective migration with the coordinator.
- Bảo owns shared Compose/CI changes under D1/D4. Phat and Huy provide their service commands, environment names, health checks and test requirements. Use task branches and review before merging changes to shared files.
- For each handoff, provide the reviewed API/schema, success and error fixtures, relevant passing checks and remaining limitations. Consumers may use mocks while waiting, then replace them with real-stack verification.

## Task completion and learning checkpoints

Record progress separately from this assignment: task ID, owner, status, blocker, review link, verification command/result and evidence link. Start with `Not started`; use `In progress`, `In review`, `Blocked` or `Done` based on actual work. Mark a task Done only when its planned acceptance cases pass, its reviewer has checked the change and connected checks required by that task have passed. Documentation plans remain unchecked until execution provides evidence.

At each backend milestone, the implementer should explain one failure case to another member:

- F2/F5: which changes belong in one transaction, and what rollback actually restores.
- C2: why the fresh backend version and stock condition prevent overselling, and why a stale version conflict differs from insufficient stock.
- C3–C6: why payment holds survive a committed transaction, how retries preserve identity, and how cancellation races with capture safely.
- B4/B5: why an interrupted search cannot prove infeasibility, and why Java revalidates Python results before purchase.
- D3: how a restart or delayed notification recovers without a second stock deduction or duplicate financial effect.

The allocation supports learning as well as delivery: pair on difficult cases, rotate explanations and reviews, and keep one accountable owner for each task.
