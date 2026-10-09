# Huy — Individual Task Implementation Plan

> **For agentic workers:** Use `superpowers:executing-plans` when implementing these tasks yourself; use `superpowers:subagent-driven-development` only when delegation is explicitly selected. The shared workflow below applies to every task.

**Revision:** 2026-10-09. Task instructions only; no implementation or test execution is claimed.
**Owner:** Huy. **Owned tasks:** 11. Reviewers are specified on each card.
**Goal:** Deliver trusted catalog/rules/drafts and bounded honest optimization, plus receipts and concurrency evidence.
**Architecture:** Java owns authoritative hardware/catalog rules and final recommendation validation. Python searches trusted bounded snapshots; Java/Python share rule fixtures and test proof claims against an independent oracle.
**Tech Stack:** Java/Spring, Next.js/React/TypeScript, Python/FastAPI, PostgreSQL, Redis; toolchain values below.
**Spec:** [Canonical design specification](../superpowers/specs/2026-10-06-electronics-store-design.md).
**Workflow:** [Shared readiness, AI instructions and completion evidence](README.md).
**Assignment authority:** [Team ownership/review tables](../TEAM_TASK_ASSIGNMENTS.md).

## Global Constraints

- Java package `com.estore`; Java 21; Spring Boot 4.1.x; Maven 3.9.x wrapper.
- Next.js 16.x; Node.js 24 LTS; Python 3.13; PostgreSQL 17; Redis 8. Pin tested patches/digests and dependencies during scaffold in `docs/environment.md`.
- Currency: `VND`; application amounts are integer whole đồng. No float money or currency conversion.
- Slots: CPU, MOTHERBOARD, RAM, GPU, STORAGE, PSU, CASE. Goals: GAMING, PRODUCTIVITY, OFFICE, CONTENT_CREATION.
- Quote TTL five minutes; hold/payment-retry window ten minutes, never extended by a retry. Inventory retries: three total fresh transactions.
- Implement S01 types and S03 HTTP fields verbatim. A checkout key is user-scoped; it is not authentication or an inventory version.
- Core uses MoMo sandbox wallet `captureWallet`, `autoCapture=false`, and hosted redirects. Live credentials/actions require the production launch gates.
- All task cases below are requirements, not passed tests. Use the source task for its exact assertion fixtures and five-step implementation cycle.

## Review Focus

- Missing producer contracts cannot be replaced by invented API responses; review the schema/fixture before consuming it.
- Concurrency and external uncertainty need database-state and restart evidence, not only happy-path mocks.
- Browser/provider/AI output cannot override authoritative ownership, stock, price or payment state.
- Cross-owner files, migrations and transactional hooks require coordination before changes are merged.
- Missing live inputs or unexecuted tests remain explicit blockers, not generated success claims.

## Task index

| Task | Stage | Reviewer |
| --- | --- | --- |
| F4 | Core | Phat |
| F6 | Core | Phat |
| B1 | Core | Phat |
| B2 | Core | Phat |
| B3 | Core | Phat |
| B4 | Core | Phat |
| B5 | Core | Phat |
| B6 | Optional enhancement | Bảo |
| C7 | Core | Phat |
| D3 | Core | Phat |
| P3 | Production | Phat |

## Task cards

### F4 — Catalog reads and revisioned cache

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R01, R14; S03, S08, TRD10.
**Source task:** [F4 in the domain plan](../superpowers/plans/2026-10-07-backend-foundation.md).
**Business purpose:** Show correct availability and predictable discovery even when cached product facts are warm or Redis fails.
**Prerequisites and handoffs:** Phat/F1/F2 and S03 query contracts. The schema must be merged; fixtures can be prepared earlier. Review output with Bảo/U2.
**Expected output:** Active catalog/category/query APIs and trusted CatalogSnapshot with current stock and revisioned facts; B1/B5/U2 consume it.
**Files and tests:** Create backend/src/main/java/com/estore/catalog/ProductService.java, backend/src/main/java/com/estore/catalog/ProductController.java, backend/src/main/java/com/estore/catalog/CatalogSnapshotService.java, backend/src/main/java/com/estore/catalog/CatalogFactCache.java; test backend/src/test/java/com/estore/catalog/CatalogCacheIT.java.
**Interfaces:** Produces `getProduct(long id):ProductView`, `search(ProductQuery query):Page<ProductView>`, `snapshot():CatalogSnapshot`; snapshots include reviewed platforms and live stock; exposes fixed category tree.

**Business rules and implementation boundary:**

Query filtered/paged availability in PostgreSQL. Cache stable facts by DB catalog revision for600s, batch live quantities, and bypass failed cache reads. Implement size1–100/zero-based page/sort allowlist and ID tie. Build trusted snapshot, rejecting oversize candidate sets rather than truncating.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| The last GPU is held while its product facts are cached | It disappears from inStock-filtered database pages; old cached availability cannot keep it purchasable. |
| Redis cache fails | Read catalog facts/availability from the database rather than losing browsing. |
| Two equal-price products are paginated or page size 101 is requested | Use the stable ID tie-break; invalid size returns 400 rather than silently clamping. |

**Required named cases from the source plan:** warmCache_lastUnitHold_disappearsFromInStockPage; release_restoresAvailability; redisCacheDown_readsDatabase; rollback_doesNotAdvanceRevision; specEdit_changesFactKey; sort_equalPrice_stableId; size101_returns400; qNameAndSku; inactivePublic404; snapshotsOverLimit_failExplicitly.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=CatalogCacheIT verify)
```

**Explain before review:** Why must in-stock filtering happen before pagination in the database?
**Reviewer checks:** Phat checks real Redis warm-cache/outage cases, rollback-safe revisioning and bounded trusted snapshots.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### F6 — Admin publishing and audited stock adjustment

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R14, R13; S02, S03, S06.
**Source task:** [F6 in the domain plan](../superpowers/plans/2026-10-07-backend-foundation.md).
**Business purpose:** Prevent administrators from publishing unsupported hardware or overwriting stock from stale totals.
**Prerequisites and handoffs:** F2/F3/F4 and B1 publication validators; F7 supplies protections. Core admin product work can start here; connected order/overview acceptance also consumes C7. Coordinate UI fixtures with Bảo/U6.
**Expected output:** Audited product/tag/platform publication, no-write preview, soft delisting and idempotent signed stock adjustments. Phat reviews Java changes.
**Files and tests:** Create backend/src/main/java/com/estore/admin/AdminProductService.java, backend/src/main/java/com/estore/admin/AdminProductController.java, backend/src/main/java/com/estore/admin/StockAdjustmentService.java, backend/src/main/java/com/estore/admin/AdminOverviewController.java; test backend/src/test/java/com/estore/admin/AdminOperationsIT.java.
**Interfaces:** Consumes B1 publication/preview validators; produces `preview(ProposedComponent):BuildValidation`, `publish(ProductWriteRequest):ProductView`, `adjust(UUID actorId,long productId,StockAdjustmentRequest,UUID key):StockAdjustmentView`; admin order routes integrate C7.

**Business rules and implementation boundary:**

Implement controlled tag kinds/publication fields and reviewed platform evidence. Take exclusive catalog guard before catalog edits; audit mutations. Stock deltas use versioned fresh transaction/retry and unique movement sources; never overwrite stock from a stale admin total. Soft delist, read inactive products, expose redacted overview/recovery data.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| An admin previews proposed hardware | Return compatibility issues with no product, inventory or audit mutation that pretends a publication occurred. |
| An admin retries +5 units using the same command key | Apply one movement only, including when a checkout concurrently changes the inventory version. |
| An admin edits specs or submits an old absolute stock total | Advance catalog revision only on committed catalog edits; adjust stock by guarded delta, not stale overwrite. |

**Required named cases from the source plan:** preview_noWrites; missingRequiredMetadata_rejectPublication; wrongTagKind_reject; duplicateSku_one409; inactiveAdminVisible; categoryChange_reject; catalogEdit_advancesRevisionAfterCommitOnly; adjustmentRaceWithHold_honorsVersion; repeatedAdjustmentKey_oneMovement; lowStockAndHeld_dashboardCorrect; unauthenticatedAdminDenied.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=AdminOperationsIT verify)
```

**Explain before review:** Why is a stock delta with identity safer than setting an absolute quantity copied from an old admin screen?
**Reviewer checks:** Phat checks publication metadata/evidence, actual audit rows and adjustment-versus-hold concurrency.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### B1 — Authoritative component validation

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R04, R06, R14; S06.
**Source task:** [B1 in the domain plan](../superpowers/plans/2026-10-07-pc-builder-optimizer.md).
**Business purpose:** Decide whether selected parts work together, form a complete build and can currently be purchased as three distinct answers.
**Prerequisites and handoffs:** F2/F4 and S06 metadata/evidence fixtures. Freeze shared Java/Python fixtures before B4; Phat/F5/F6 consume these validators.
**Expected output:** Authoritative Java validation/evaluation/publication APIs, versioned rules and shared positive/negative fixtures; missing data is reported rather than invented.
**Files and tests:** Create backend/src/main/java/com/estore/builder/CompatibilityService.java, backend/src/main/java/com/estore/builder/BuilderController.java, backend/src/main/java/com/estore/builder/rules/, backend/src/main/java/com/estore/builder/PublicationValidator.java, contracts/fixtures/compatibility.json; test backend/src/test/java/com/estore/builder/CompatibilityTest.java and backend/src/test/java/com/estore/builder/BuilderApiIT.java.
**Interfaces:** Produces `validate(SlotSelection selection):BuildValidation`, `evaluate(SlotSelection draft,Slot slot,List<Long> ids):List<CandidateEvaluation>`, `validatePublication(ProductWriteRequest):List<RuleIssue>`; CandidateEvaluation=(productId,allowed,violations).

**Business rules and implementation boundary:**

Implement S06 rule codes/category schema and reviewed CPU/board/case evidence with immutable version tuples. Estimate CPU+GPU+75W, integer5×load≤4×capacity. Explicit form-factor sets, kit module/capacity checks, vetted cooling and single NVMe support. Publish rule schema/version for advisory client filtering; never declare incomplete metadata compatible.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A partial draft lacks a motherboard | Skip counterpart-dependent checks; it may be compatible but is incomplete and not purchasable. |
| The selected CPU lacks required socket metadata | Report a violation, not an invented default or a skipped missing counterpart. |
| Estimated load is 520 W with a 650 W PSU, then 521 W | The first passes 5×load≤4×capacity; the second fails. |
| GPU length exactly equals clearance or the platform evidence versions changed | Exact length fits; outdated CPU/board/case review evidence invalidates the relevant platform rule. |

**Required named cases from the source plan:** partialDraft_skipsAbsentCounterpart; selectedMissingSocket_invalid; wrongCategory_invalid; allSeven_complete; stockBelowRequested_notPurchasable; exactGpuClearance_pass; load520_capacity650_pass; load521_capacity650_fail; unknownFormFactor_noImplicitRank; ramModuleAndCapacityRules; storageUnsupported_block; changedPlatformVersion_block; preview_noPublication.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dtest=CompatibilityTest -Dit.test=BuilderApiIT verify)
```

**Explain before review:** Why are compatible, complete and purchasable independent flags?
**Reviewer checks:** Phat checks rule order/codes, explicit form-factor membership, metadata versus absent-counterpart handling and fixture parity.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### B2 — Revisioned drafts and staleness

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R05; S02, S03.
**Source task:** [B2 in the domain plan](../superpowers/plans/2026-10-07-pc-builder-optimizer.md).
**Business purpose:** Keep a customer’s saved intention and price baseline stable while showing current changes.
**Prerequisites and handoffs:** F2/F3/F4/B1; reserve and merge V2 before shared V3 migration application. Agree SavedBuildView/revision fixtures with Bảo/U3.
**Expected output:** Owned partial saved drafts with baseline/current/effective selection and notices; explicit save advances baseline, while GET has no writes.
**Files and tests:** Create backend/src/main/resources/db/migration/V2__saved_builds.sql, backend/src/main/java/com/estore/builder/DraftService.java, backend/src/main/java/com/estore/builder/DraftController.java; test backend/src/test/java/com/estore/builder/DraftIT.java.
**Interfaces:** Produces `save(UUID userId,SaveDraftRequest):SavedBuildView`, `update(UUID userId,UUID id,UpdateDraftRequest):SavedBuildView`, `load(UUID userId,UUID id):SavedBuildView`; view contains selections/effectiveSelections/baselinePrices/currentProducts/notices/revision/validation.

**Business rules and implementation boundary:**

Persist normalized slot IDs/prices/catalog versions on explicit saves, permit partial drafts, check owner and expectedRevision. GET derives effective selection/current validation/notices with no write. Preserve historical IDs for replacement explanation; cart groups are independent copies.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A product’s price changed and the draft is read twice | Both reads retain the old saved price and change notice. |
| The customer explicitly saves the revised draft | Update the baseline and advance the draft revision once. |
| A selected product is delisted or two updates use one revision | Preserve historical selection/notices but exclude unavailable effective parts; concurrent updates permit one winner and one 409. |

**Required named cases from the source plan:** repeatedGet_keepsPriceBaselineAndNotice; explicitSave_advancesBaseline; delisted_selectionExcludedNoticePreserved; outOfStock_selectionRetained; otherOwner404; concurrentSave_one409Revision; invalidCategory_saveRejected; delete_doesNotMutateCopiedCartGroup.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=DraftIT verify)
```

**Explain before review:** Why should reading a saved draft never silently accept a new price?
**Reviewer checks:** Phat compares persisted baselines across repeated GETs and checks ownership/revision races and cart-copy independence.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### B3 — Python service and trusted contracts

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R07, R15, R17; S01, S07.
**Source task:** [B3 in the domain plan](../superpowers/plans/2026-10-07-pc-builder-optimizer.md).
**Business purpose:** Create a private, bounded optimizer boundary that only accepts trusted backend catalog data.
**Prerequisites and handoffs:** Can start now from S01/S07. Publish internal OpenAPI and fixture definitions with Phat/B5; runtime pins align with F1/U1.
**Expected output:** Python runtime/lock/test tooling, strict request/result schemas, private API-key-protected route and shared internal fixtures. B4 supplies the real solver.
**Files and tests:** Create optimizer/pyproject.toml, optimizer/uv.lock, optimizer/app/main.py, optimizer/app/contracts.py, optimizer/app/security.py, optimizer/tests/test_contracts.py, contracts/optimizer.openapi.yaml.
**Interfaces:** Produces `SolveRequest`, `SolveResult`, `CatalogSnapshot`, `SlotSelection` models matching S07; private `/internal/optimizer/solve` and health route. B4 implements `solve(request: SolveRequest) -> SolveResult`.

**Business rules and implementation boundary:**

Bootstrap declared runtime/lock and pytest/httpx/respx dependencies first. Define exact Pydantic strict integer/enums/aliases and snapshot bounds; authenticate constant-time API key check. Produce versioned internal OpenAPI and shared success/failure JSON fixtures; no browser-supplied catalog input.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Budget is a floating-point JSON value | Reject it instead of coercing into integer VND. |
| Snapshot contains 201 products or 31 candidates in one slot | Reject explicitly rather than truncate silently. |
| A request lacks the internal API key | Reject it; the browser never sends its own prices or catalog to this service. |

**Required named cases from the source plan:** schema_acceptsVndIntegers_rejectsFloat; wrongGoal422InternalMaps400Public; missingInternalKey401; wrongKey401; snapshotOver200_rejects; perSlotOver30_rejects; missingMetadata_rejects; internalPortsPrivate.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd optimizer && uv run pytest tests/test_contracts.py -q)
```

**Explain before review:** Why does optimizer correctness depend on the catalog’s trust boundary?
**Reviewer checks:** Phat checks strict aliases/types, bounds, internal authentication and fixtures before consuming the Python result.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### B4 — Constraint search and bounded execution

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R07; S06, S07.
**Source task:** [B4 in the domain plan](../superpowers/plans/2026-10-07-pc-builder-optimizer.md).
**Business purpose:** Find a valid whole build under budget and honestly state what the bounded search proved.
**Prerequisites and handoffs:** B3, B1 rule fixtures and F4 curated candidate evidence. The exhaustive small-instance oracle must be defined independently of the solver; B5 consumes its results.
**Expected output:** Joint constraint search and supervised process execution with retained incumbent, exact tie-breaks and S07 status/proof envelope.
**Files and tests:** Create optimizer/app/rules.py, optimizer/app/solver.py, optimizer/app/supervisor.py, optimizer/tests/test_solver.py, optimizer/tests/test_supervisor.py.
**Interfaces:** Produces `solve(request: SolveRequest) -> SolveResult`, `SearchSupervisor.solve(request: SolveRequest) -> SolveResult`; rule parity consumes contracts/fixtures/compatibility.json; statuses/proof flags are S07.

**Business rules and implementation boundary:**

Use joint partial-state rules, admissible lower-price/upper-utility bounds, deterministic score/price/ID ties and retained complete valid incumbent. Base/upgrades/minimum search share3000ms monotonic deadline with checks≤256nodes. Two supervised worker-process slots/no queue; kill unresponsive work at3500ms, reap safely, return no unsupported proof. No blocking CPU loop in async handler; no independent MCKP compatibility guarantee.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Score/price/ID ties occur in an exhaustively checked small catalog | Choose higher integer utility, then lower price, then lexicographically lower IDs in fixed slot order. |
| Search times out with a valid incumbent, or without one | Return FEASIBLE without optimality proof, or TIMEOUT_NO_SOLUTION without infeasibility proof. |
| Two jobs occupy both slots and a third arrives | Do not queue it; keep health responsive. All phases share 3,000 ms and a stuck worker is killed at 3,500 ms. |
| Greedy selection fails but another compatible combination fits | Search must not label greedy failure INFEASIBLE or label an unproven above-budget example cheapest. |

**Required named cases from the source plan:** exhaustiveSmallOracle_sameOptimalAndTie; aggregatePowerTrap_rejected; gpuCaseTrap_rejected; greedy700Gpu_trap_finds900Build; timeoutWithIncumbent_feasible; timeoutWithoutIncumbent_unknown; missingCategory_provenNoCompatible; belowMinimum_onlyClaimMinimumWhenProven; allSearchPhasesSameDeadline; thirdWorker_rejectedNoQueue; blockedWorker_terminated; healthResponsiveDuringTwoSearches.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd optimizer && uv run pytest tests/test_solver.py tests/test_supervisor.py -q)
```

**Explain before review:** What evidence distinguishes “did not find a build” from “proved no build exists”?
**Reviewer checks:** Phat compares small-instance results to exhaustive enumeration, checks adversarial power/fit traps and validates deadline/proof reporting.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### B5 — Java optimizer boundary and outage behavior

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R07, R15, R16; S07, TRD8.
**Source task:** [B5 in the domain plan](../superpowers/plans/2026-10-07-pc-builder-optimizer.md).
**Business purpose:** Convert advisory optimizer output into a trustworthy public recommendation without letting an outage block the store.
**Prerequisites and handoffs:** F4/B1/B4, Phat/F7 public protection and agreed internal/public schema fixtures. Revalidation must use fresh catalog data.
**Expected output:** Public autoBuild service, trusted snapshot client, resilient bounded calls and enriched/revalidated SolveView; Bảo/U3 displays the result.
**Files and tests:** Create backend/src/main/java/com/estore/builder/optimizer/OptimizerClient.java, backend/src/main/java/com/estore/builder/optimizer/AutoBuildService.java, backend/src/main/java/com/estore/builder/optimizer/OptimizerResilience.java; test backend/src/test/java/com/estore/builder/OptimizerBoundaryIT.java.
**Interfaces:** Produces `autoBuild(AutoBuildRequest request):SolveView`; AutoBuildRequest=(budgetMinor,currency,goal); consumes F4 CatalogSnapshot, B1 validation, and Python SolveResult.

**Business rules and implementation boundary:**

Send trusted snapshot only, enforce500ms connect/10s overall, TIME_BASED10s/min10calls/50%/30sopen/2probes and2permit no-queue bulkhead. Recompute current price/power/stock/rules before public enrichment; stale result409 with manual/retry action, no hidden repeated solver calls. Explain proof statuses faithfully.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Python returns unknown IDs, wrong slots or impossible proof flags | Return 503 OPTIMIZER_INVALID_RESULT, not a recommendation. |
| Stock, price or specs drift while Python searches | Return 409 BUILD_STALE; do not silently accept changed parts or rerun without bounds. |
| A valid no-solution result or transport failure arrives | No-solution is a business result, not a breaker fault; transport/malformed/timeout errors follow the breaker policy. |
| Both concurrency permits are busy | Reject excess work without queuing; enforce 500 ms connect and 10-second total boundary. |

**Required named cases from the source plan:** anonymousAutoBuildAllowed; browserCatalogFieldRejected400; unknownSelectedId503; wrongSlot503; changedPriceStockSpec409BuildStale; resultOverBudget503; invalidProofFlag503; validNoSolution_notBreakerFault; transportFailures_tripBreaker; tenSecondOverallBound; twoPermits_noQueue; halfOpenProbes_recover.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=OptimizerBoundaryIT verify)
```

**Explain before review:** Why must Java validate a result that already passed Python’s compatibility checks?
**Reviewer checks:** Phat checks the exact breaker/bulkhead policy, stale-result handling and consistency of enriched prices, rules and proof flags.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### B6 — Optional bounded explanation boundary

**Owner:** Huy. **Reviewer:** Bảo. **Stage:** Optional enhancement.
**Requirements:** R08; S07, S08.
**Source task:** [B6 in the domain plan](../superpowers/plans/2026-10-07-pc-builder-optimizer.md).
**Business purpose:** Explain a verified build optionally without changing selection, price or checkout correctness.
**Prerequisites and handoffs:** Optional only; B1/B3/B5 trusted selection/context, Phat/F7 quotas and Bảo/U3 selectionHash rendering. Live model/provider/cost/data decision must be recorded; a disabled stub is the default.
**Expected output:** Separate explanation boundary, nullable text, selectionHash, five-second total deadline and 2,000-character plain-text cap. Solver response does not wait for it.
**Files and tests:** Create optimizer/app/explanation.py, backend/src/main/java/com/estore/builder/ExplanationService.java, optimizer/tests/test_explanation.py, backend/src/test/java/com/estore/builder/ExplanationBoundaryTest.java.
**Interfaces:** Produces `explain(selection: SlotSelection, goal: Goal, catalog: CatalogSnapshot) -> ExplanationResult`; ExplanationResult=(text nullable,selectionHash). `ExplanationProvider.generate(VerifiedExplanationInput):String` is the configured provider boundary; disabled mode returns null.

**Business rules and implementation boundary:**

Implement separate explanation endpoint/provider port, fixed structured trusted input, five-second total deadline and2,000-character plain-text output. Default disabled; test with deterministic provider stub. Enabling a real AI adapter requires its model/API/cost/data decision and corresponding integration test; this optional task is not evidence of a live AI integration.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Provider is disabled, fails or streams past five seconds | Return no explanation while leaving the build result usable. |
| The user changes the build while an explanation is in flight | The stale selectionHash explanation is not displayed. |
| Catalog text contains instructions or provider output contains markup | Treat catalog text as data and render bounded plain text; send no customer PII. |

**Required named cases from the source plan:** disabledProvider_returnsNull; slowStreaming_overall5s_returnsNull; providerError_buildUnaffected; changedSelection_invalidatesHash; textOver2000_bounded; catalogPromptInjection_treatedAsData; noPiiInPayload; plainTextRendering.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Bảo’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd optimizer && uv run pytest tests/test_explanation.py -q)
(cd backend && ./mvnw -Dtest=ExplanationBoundaryTest test)
```

**Explain before review:** Why should explanation failure have no effect on compatibility or purchase eligibility?
**Reviewer checks:** Bảo checks hash invalidation/plain-text UI; Phat additionally reviews the Java deadline/privacy boundary.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### C7 — Order receipts, fulfillment, and compensation

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R12, R13, R14; S03–S05.
**Source task:** [C7 in the domain plan](../superpowers/plans/2026-10-07-checkout-pipeline.md).
**Business purpose:** Keep receipts stable and handle exceptional payments without fulfilling unsupported orders or losing new cart edits.
**Prerequisites and handoffs:** C1–C6, F5 cleanup and F6/admin security. Coordinate LifecycleTransaction/cleanup integration with Phat; do not duplicate C4 payment finalization.
**Expected output:** Owned immutable receipts/history, guarded admin fulfillment and durable compensation/refund exceptions; Bảo completes U5/U6 on these APIs.
**Files and tests:** Create backend/src/main/java/com/estore/order/OrderService.java, backend/src/main/java/com/estore/order/OrderController.java, backend/src/main/java/com/estore/payment/CompensationService.java, backend/src/main/java/com/estore/order/AdminOrderController.java; test backend/src/test/java/com/estore/order/OrderLifecycleIT.java.
**Interfaces:** Produces `getOrder(UUID userId,UUID orderId):OrderView`, `fulfill(UUID adminId,UUID orderId,long expectedVersion,UUID key):OrderView`, `queueCompensation(UUID attemptId,String reason):UUID`; uses F5 cleanup and C3 refund/query worker.

**Business rules and implementation boundary:**

Implement owned history/immutable receipt/admin fulfillment and review queue. Confirm cleanup exactly once under cart/order guard. Compensation uses one durable refund identity, captured transaction reference and guarded amount; confirm refund only from provider evidence, never restock automatically. Core compensation refund table exists in V3; production P4 adds ordinary return/allocation workflows.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Catalog name/price changes after purchase | Receipt keeps the original snapshots. |
| A capture arrives after holds were released | Mark PAYMENT_REVIEW_REQUIRED, block fulfillment and enqueue one compensation identity; do not deduct another unit. |
| A second attempt captures after another already paid the order | Keep the paid order/paid attempt and refund the extra capture. |
| Refund succeeds or fulfillment is replayed | Do not change availability automatically; cleanup and fulfillment remain once-only and preserve changed cart entries. |

**Required named cases from the source plan:** otherOwner404; catalogEdit_receiptUnchanged; confirm_cleanupOnlyUnchangedCartEntries; duplicateFulfill_once_noStockDelta; paymentException_blocksFulfillment; capturedAfterRelease_reviewNotPaidAndOneRefund; extraAttemptCapture_refundExtraKeepPaidOrder; refundTimeout_pendingThenQuery; compensationSuccess_setsRefundStatusNoRestock; adminView_masksPii.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=OrderLifecycleIT verify)
```

**Explain before review:** Why must exceptional payment compensation preserve both order history and the original paid attempt?
**Reviewer checks:** Phat checks immutable snapshots, exact cleanup conditions, captured-refund identity and late/extra payment recovery.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### D3 — Concurrency, restart recovery, and MoMo sandbox

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R10, R11, R12, R17; S04, S05.
**Source task:** [D3 in the domain plan](../superpowers/plans/2026-10-08-integration-delivery.md).
**Business purpose:** Prove both stock safety and legitimate-purchase success under real concurrency and restart failures.
**Prerequisites and handoffs:** D1 real stack, F1–F7/B1, C1–C7 and connected U5; builder validation fixtures available. Backend quotas are raised only in the explicit nonproduction load profile. Actual sandbox cases need merchant credentials.
**Expected output:** k6 last-unit/plentiful scenarios, crash/restart recovery scripts and a core-checkout evidence ledger with separate stub and MoMo sandbox results.
**Files and tests:** Create tests/load/last-unit.js, tests/load/plentiful-stock.js, scripts/verify-recovery.sh, scripts/momo-sandbox-smoke.sh, docs/evidence/core-checkout.md.
**Interfaces:** Produces k6 safety/success metrics and recorded sandbox evidence; consumes real checkout schema/HTTP/provider stub and separately configured MoMo sandbox merchant credentials.

**Business rules and implementation boundary:**

Run tests with quotas explicitly raised in a nonproduction load profile; retain real inventory/version semantics and successful payment stub behavior. Separately run `bash scripts/momo-sandbox-smoke.sh` only with sandbox credentials to validate hosted authorize/capture/cancel/refund/query, amount/signatures and notification acknowledgement. Store timings/outcomes; no live calls and no invented PASS when credentials are absent.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| 50 distinct buyers target one unit | Exactly one holds the unit, and no negative inventory or partial order exists. |
| Stock is plentiful for all independent healthy requests | All should succeed; rejecting everyone is not evidence of correct concurrency. |
| A worker is killed after CREATE or while capture is unresolved | Recover identities and keep uncertain holds; duplicate notifications/schedulers cause no repeated effect. |
| Sandbox credentials are absent | Mark sandbox checks NOT RUN with the missing input; do not report stub success as provider evidence. |

**Required named cases from the source plan:** fiftyBuyersOneUnit_oneHold; sameKeyConcurrent_oneAllocation; plentifulStockAllHealthyAttemptsSucceed; killAfterCreateBeforeAssociation_recovers; killDuringCapture_noRelease; duplicateIpn_noDoubleConfirm; twoExpiryWorkers_once; actualSandboxAuthorizeCaptureAndRefund; missingSandboxCredentials_reportNotRun.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
k6 run tests/load/last-unit.js && k6 run tests/load/plentiful-stock.js && bash scripts/verify-recovery.sh
bash scripts/momo-sandbox-smoke.sh
```

The sandbox command is a separate manual, credential-gated run. Ordinary CI uses the stub; absent sandbox credentials mean NOT RUN, not PASS.

**Explain before review:** Why must a concurrency test prove successful purchases as well as no overselling?
**Reviewer checks:** Phat reruns real-stack races and checks database/journal/count evidence and the authenticity of separate sandbox records.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### P3 — Versioned Vietnamese delivery and tax totals

**Owner:** Huy. **Reviewer:** Phat. **Stage:** Production.
**Requirements:** R09, R20; S09.
**Source task:** [P3 in the domain plan](../superpowers/plans/2026-10-08-production-launch.md).
**Business purpose:** Charge exactly the approved Vietnamese delivery/tax amount and preserve historical calculations.
**Prerequisites and handoffs:** D4, C1, P1 market inputs, approved address/zones/tax reference cases and V4 before V5. Bảo helps with the commerce editor; Phat reviews backend calculations. No tax rate is invented.
**Expected output:** V5 immutable policy/address versioning, quote calculator, fee disclosures and reviewed commerce-vn fixtures; P4 consumes exact refundable allocations.
**Files and tests:** Create backend/src/main/resources/db/migration/V5__commerce_policy.sql, backend/src/main/java/com/estore/commerce/CommercePolicyService.java, backend/src/main/java/com/estore/commerce/QuoteCalculator.java, frontend/src/components/admin/CommercePolicyEditor.tsx, contracts/fixtures/commerce-vn.json; test backend/src/test/java/com/estore/commerce/CommerceTotalsIT.java.
**Interfaces:** Produces `calculate(CartPriceSnapshot,ShippingAddress,CommercePolicy):QuoteAmounts`, QuoteAmounts=(subtotalMinor,shippingMinor,taxMinor,additionalTaxMinor,totalMinor,currency,calculationVersion,lineBreakdown); consumes approved jurisdiction examples and versioned address/zone data.

**Business rules and implementation boundary:**

Implement immutable versioned flat-rate zones/current approved address dataset, policy classifications and exact rational/BigDecimal rounding. Store included-tax disclosure separately from additional payable tax; total=subtotal+shipping+additionalTax. Approval/effective dates/reference cases are required configuration, not guessed VAT. Extend quote/order snapshots and UI fee disclosures; changed policy forces re-review before hold.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Subtotal is 100,000, shipping 20,000 and included tax disclosure is 10,000 with zero additional tax | Total is 120,000, not 130,000. These are synthetic arithmetic values, not Vietnamese tax rates. |
| Additional payable tax is 5,000 on the same synthetic example | Total is 125,000 and allocated charges reconcile exactly. |
| Policy changes or locality is unsupported | Require quote re-review or reject before holding stock; historical order totals stay unchanged. |

**Required named cases from the source plan:** inclusiveTax_notAddedTwice; exclusiveTax_addedOnce; wholeDongHalfUpBoundary; shippingTaxClassified; mixedRates_residualsExact; missingTaxApproval_liveBlocked; unknownProvinceLocalityOrZone_rejected; policyEditForcesQuoteReview; historicalOrderCalculationUnchanged; providerMaximumAfterFees_rejectedBeforeHold.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=CommerceTotalsIT verify)
```

**Explain before review:** Why is taxMinor disclosure different from additionalTaxMinor payable money?
**Reviewer checks:** Phat checks approved reference inputs, deterministic whole-đồng rounding, exact residuals and provider limits after fees.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

## Completion evidence

Use the shared evidence template per task. All owned task checkboxes remain unchecked until implementation is verified. A reviewer’s approval concerns the recorded commit; later contract changes require matching producer/consumer checks. Production approvals come from the named merchant/business/operator owners.
