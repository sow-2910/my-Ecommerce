# Individual task guides and shared working rules

**Revision:** 2026-10-09. These guides expand the existing 37 tasks; ownership and scope are preserved. No implementation or application test execution is claimed.

The lead wants independently produced work, including AI-generated code, to fit together and satisfy the same business rules. Here, deterministic means the inputs, observable outputs, constraints and acceptance evidence are explicit. It does not mean every AI produces identical source code or every timed search proves an optimum.

## Member files

| Member | Detailed task file | Owned tasks | First task |
| --- | --- | --- | --- |
| Phat | [PHAT.md](PHAT.md) | 13 | F1, then F2 |
| Bảo | [BAO.md](BAO.md) | 13 | U1 |
| Huy | [HUY.md](HUY.md) | 11 | B3; F4 after F2 |

Each task card states its reviewer, business purpose, prerequisites, expected output, files, interfaces, rules, concrete acceptance scenarios, full source-plan case list, execution steps and verification commands. Cross-service tasks include the contribution needed from another member. The reviewer column identifies the accountable review; the card identifies any additional specialist review.

## Authority and task scope

1. The [PRD](../ELECTRONICS_STORE_PRD.md) defines product requirements and release scope.
2. The [TRD](../ELECTRONICS_STORE_TRD.md) and [specification](../superpowers/specs/2026-10-06-electronics-store-design.md) define technical invariants and canonical types/APIs/state transitions.
3. The [domain plans](../README.md#implementation-plans-and-dependencies) define task files, interfaces, exact assertion fixtures and implementation steps.
4. [Team assignments](../TEAM_TASK_ASSIGNMENTS.md) define the owner and reviewer. Individual files explain how to execute and assess those same tasks.

If documents conflict, record the conflict and align them before implementing the affected behavior. Do not silently pick whichever example an AI finds easiest. Helper functions and internal decomposition can vary; public fields, error meanings, money/state rules, deadlines and migration ownership must match the reviewed contract.

The individual guides reference existing tasks rather than introducing a second backlog. Their extra examples are acceptance scenarios derived from the specification. All source-plan cases still apply; three illustrative examples are not a replacement for the full case list.

## Readiness and parallel work

F1, U1 and B3 can start together. A task's position in a member file is not a promise it can start immediately. Use the readiness conditions on its card and the [shared sequence](../TEAM_TASK_ASSIGNMENTS.md#dependencies-and-work-sequence).

Before implementation, record:

- The task ID, owner, reviewer and starting commit.
- Which prerequisite deliverables are merged and verified, and their commit/fixture versions.
- The input/output schema and positive, invalid, conflict and unavailable fixtures agreed by producer and consumer.
- Whether work is scaffold/fixture-only, ready for component implementation or ready for connected acceptance.
- Any external input still missing, such as merchant capability, approved market policy, email sender or hosting configuration.

Frontend work may start with reviewed fixtures before the backend is complete. Mark it as fixture-based until real API checks pass. Payment adapter work may start with a controllable stub; sandbox and live outcomes require their own evidence. A missing input blocks only dependent work, not unrelated tasks.

## Decisions that AI must preserve

| Topic | Fixed business contract |
| --- | --- |
| Money and market | Vietnam/VND, integer whole đồng; exact totals and checked bounds. MoMo wallet checkout 1,000–50,000,000 VND. |
| Group identity | Seven-slot builds remain independent groups; cart add reserves no stock. Checkout aggregates product demand after preserving group snapshots. |
| Inventory | Backend reads current inventory versions; quote/browser versions are not allocation authority. The whole allocation commits or rolls back; three fresh transaction attempts total. |
| Hold and payment | Five-minute quote, ten-minute original hold window. Hold deducts once; captured payment only confirms. Safe release restores once after nonpayment proof. No provider call under database locks. |
| Purchase identity | User-scoped checkout UUID + canonical quote fingerprint. An uncertain retry preserves identity; a different intended purchase uses a new key. Payment retry keeps the order/amount/hold deadline. |
| Payment authority | MoMo readiness and authorization are not captured payment. Duplicate/old events cannot repeat effects or downgrade payment. Unknown outcomes retain proof obligations. |
| Optimizer | Trusted catalog only, joint rules, bounded search, explicit status/proof flags, Java fresh revalidation. A timeout is not an infeasibility proof. |
| Browser | Memory access token, secure refresh cookie, bounded nonsecret temporary state. Backend confirms receipts; URL parameters and countdowns cannot declare payment/cancellation. |
| Refund and restock | Pending + successful refunds reserve refundable balance. Restocking requires inspected physical receipt; a refund alone changes no availability. |
| Release gates | Core correctness, provider sandbox evidence and production merchant/business/operator approvals are distinct. Optional AI cannot block or redefine checkout. |

## Shared files and contract handoffs

Phat coordinates migrations. V1 belongs to F2/Phat; V2 to B2/Huy; V3 to C1/Phat; V4 to P2/Phat; V5 to P3/Huy; V6 to P4/Phat; V7 is coordinated by P5/Bảo with Phat's backend contribution. Merge and apply lower allocated migrations before later ones in shared environments. An applied migration is not rewritten. A corrective migration needs a newly coordinated number and matching documentation.

Bảo maintains public OpenAPI and generated frontend types under D2 with the API producer's review. Publish the auth/catalog slices and fixtures early to unblock frontend work; D2 completes only after full contract parity and real-stack journeys. Huy maintains the internal optimizer schema and rule fixtures under B3/B1. Keep producer and consumer types consistent; do not handwrite a competing DTO interpretation.

Some source types are internal helper names rather than fully serialized schemas. Implement their internal representation consistently with the canonical public fields. Before a consumer depends on any insufficiently specified public shape, such as detailed conflict payloads or a launch email landing route, record its exact fields/route/example in the specification and shared contract with Phat and the affected producer/consumer. Do not invent it independently on both sides.

Coordinate shared transactional hooks explicitly: C4/C7 use the same payment finalizer and F5 cleanup; D4 adds committed journey events to F5/C2/C4 transactions; P4/P5 share the order guard for paid cancellation versus shipment. Only one task owns a given migration or shared contract file at a time. A helper may contribute code, but the task owner integrates and records the result.

Missing business configuration is not a free implementation choice. P1 obtains real merchant capability; P2 records the approved mail sender/templates; P3 supplies approved current address/zones/tax cases; P4/P5 supply approved return/privacy/retention policies; P6 records hosting/operators/accepted recovery objectives. B6 stays disabled until its provider/model/cost/data decision is recorded. These inputs must not be fabricated by AI.

## Using AI on one task

Give AI the selected card, shared workflow, referenced specification sections and source task. Preserve the starting commit and contract/fixture versions. Use a prompt such as:

```text
Implement only task <ID> from <member guide>.
Read its source task and referenced specification sections first.
Confirm prerequisites and summarize the business invariants and observable outputs.
List the files/interfaces you will change and any missing producer contract.
Do not invent public fields, payment states, provider behavior or business policy.
Write the required acceptance checks before implementing business behavior.
Use real database concurrency and the prescribed provider boundary where required.
Run the listed checks and report actual case counts and outcomes.
Distinguish PASS, FAIL and NOT RUN. Do not claim sandbox/live evidence from a stub.
Finish with a diff summary, business-rule explanation and handoff evidence.
```

The implementer must inspect the result and explain the business question on the card before requesting review. Tests derived only from the generated implementation can share the same mistake; check outcomes against the specification, scenario fixtures and independent oracle where required. Use controlled clocks/barriers/fault injection for expiry, races and crashes instead of relying on lucky request timing.

Learning syntax still helps you read a diff, spot a transaction boundary and debug failures. The task guides focus that reading on the business decisions: who owns the data, what commits together, what changes stock, what evidence proves payment, and what a failed operation leaves behind.

## Completion evidence

Use this template in each task's PR or execution record; replace every bracketed field with actual evidence before requesting approval:

```text
Task / owner / reviewer:
Starting commit / reviewed result commit:
Prerequisite commits / schema and fixture versions:
Business outcome and invariant explanation:
Files changed / producer and consumer contracts affected:
Commands and environment:
Discovered cases / passed / failed / skipped:
Observed database, stock, journal or browser effects:
Evidence category: component / connected stack / provider stub / sandbox / live
NOT RUN checks and missing inputs:
Remaining failures or contract decisions:
Next consumer handoff and readiness:
```

Commands in the cards are future instructions. They become evidence only when they execute against the stated environment and actually discover the relevant suites. Bootstrap a missing runner before judging a behavioral test. Source-plan code examples are assertions to implement, not proof that those checks exist today. CI exit zero with no relevant tests is not acceptance.

Use `Not started`, `In progress`, `In review`, `Blocked` and `Done` for actual status. Component work can be ready for a consumer while connected acceptance remains outstanding; keep that distinction visible. A task is Done only after its required cases, cross-service deliverables and reviewer gate pass on the recorded commit. Live launch additionally requires the separate approved P7 evidence.

No task in these new files is currently marked complete.
