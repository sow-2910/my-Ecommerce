# PC Builder and Constraint Optimizer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Revision:** 2026-10-08; replaces the previous task bodies. Documentation work only; execution has not started.

**Goal:** Validate supported hardware and return bounded, honestly labeled budget recommendations from trusted catalog data.

**Architecture:** Java owns authoritative rules and drafts. Python searches the same curated schema with constraint-aware branch-and-bound in supervised processes; Java revalidates and enriches results.

**Tech Stack:** Java/Spring, Python/FastAPI/Pydantic, pytest, httpx, process supervision, Resilience4j

**Spec:** [Canonical design specification](../specs/2026-10-06-electronics-store-design.md).

## Global Constraints

- Java package `com.estore`; Java 21; Spring Boot 4.1.x; Maven 3.9.x wrapper.
- Next.js 16.x; Node.js 24 LTS; Python 3.13; PostgreSQL 17; Redis 8. Pin tested patches/digests and dependencies during scaffold in `docs/environment.md`.
- Currency: `VND`; application amounts are integer whole đồng. No float money or currency conversion.
- Slots: CPU, MOTHERBOARD, RAM, GPU, STORAGE, PSU, CASE. Goals: GAMING, PRODUCTIVITY, OFFICE, CONTENT_CREATION.
- Quote TTL five minutes; hold/payment-retry window ten minutes, never extended by a retry. Inventory retries: three total fresh transactions.
- Implement S01 types and S03 HTTP fields verbatim. A checkout key is user-scoped; it is not authentication or an inventory version.
- Core uses MoMo sandbox wallet `captureWallet`, `autoCapture=false`, and hosted redirects. Live credentials/actions require the production launch gates.
- Steps are future work. No checkbox or “Expected PASS” is evidence of an executed test.

## Dependencies and file ownership

B1 requires F2/F4 and unblocks cart/admin publication F5/F6 and checkout. B2 owns V2__saved_builds.sql. B3 establishes Python build/test tooling and shared contracts before B4. B5 consumes B4 and F4. B6 is optional P1 and cannot block the core milestone; selecting an actual AI provider/model is a separate recorded input before enabling it.

Paths below are proposed new application files; the current folder contains documentation only. Domain packages group controller/DTO/repository helpers with their owning service. The named service owns its transaction rules; split helpers by responsibility rather than building one application-wide service. Test paths and main deliverables are explicit in each task. Migration filenames are allocated by S02; do not create conflicting migration numbers in another plan.

## Review Focus

- Missing counterpart is different from missing selected-part metadata — B1.
- PSU capacity is excluded from estimated draw and boundary comparisons are exact — B1.
- Price notices survive repeated draft reads until explicit save — B2.
- Greedy failure/worker timeout never establish infeasibility — B4.
- Unknown IDs/stale stock/malformed proof flags cannot become purchasable recommendations — B5.

## Tasks

### B1 — Authoritative component validation

**Requirements:** R04, R06, R14; S06.
**Files:** Create backend/src/main/java/com/estore/builder/CompatibilityService.java, backend/src/main/java/com/estore/builder/BuilderController.java, backend/src/main/java/com/estore/builder/rules/, backend/src/main/java/com/estore/builder/PublicationValidator.java, contracts/fixtures/compatibility.json; test backend/src/test/java/com/estore/builder/CompatibilityTest.java and backend/src/test/java/com/estore/builder/BuilderApiIT.java.
**Interfaces:** Produces `validate(SlotSelection selection):BuildValidation`, `evaluate(SlotSelection draft,Slot slot,List<Long> ids):List<CandidateEvaluation>`, `validatePublication(ProductWriteRequest):List<RuleIssue>`; CandidateEvaluation=(productId,allowed,violations).

- [ ] Step 1: Write failing tests in the stated test file: partialDraft_skipsAbsentCounterpart; selectedMissingSocket_invalid; wrongCategory_invalid; allSeven_complete; stockBelowRequested_notPurchasable; exactGpuClearance_pass; load520_capacity650_pass; load521_capacity650_fail; unknownFormFactor_noImplicitRank; ramModuleAndCapacityRules; storageUnsupported_block; changedPlatformVersion_block; preview_noPublication. Shared fixtures check both false acceptance and false rejection. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(partial.isCompatible()).isTrue();
assertThat(partial.isComplete()).isFalse();
assertThat(missingMetadata.isCompatible()).isFalse();
assertThat(powerBoundary520.isCompatible()).isTrue();
assertThat(powerBoundary521.isCompatible()).isFalse();
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dtest=CompatibilityTest -Dit.test=BuilderApiIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement S06 rule codes/category schema and reviewed CPU/board/case evidence with immutable version tuples. Estimate CPU+GPU+75W, integer5×load≤4×capacity. Explicit form-factor sets, kit module/capacity checks, vetted cooling and single NVMe support. Publish rule schema/version for advisory client filtering; never declare incomplete metadata compatible.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dtest=CompatibilityTest -Dit.test=BuilderApiIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: authoritative component validation`. Record test command/results in the task execution notes.

### B2 — Revisioned drafts and staleness

**Requirements:** R05; S02, S03.
**Files:** Create backend/src/main/resources/db/migration/V2__saved_builds.sql, backend/src/main/java/com/estore/builder/DraftService.java, backend/src/main/java/com/estore/builder/DraftController.java; test backend/src/test/java/com/estore/builder/DraftIT.java.
**Interfaces:** Produces `save(UUID userId,SaveDraftRequest):SavedBuildView`, `update(UUID userId,UUID id,UpdateDraftRequest):SavedBuildView`, `load(UUID userId,UUID id):SavedBuildView`; view contains selections/effectiveSelections/baselinePrices/currentProducts/notices/revision/validation.

- [ ] Step 1: Write failing tests in the stated test file: repeatedGet_keepsPriceBaselineAndNotice; explicitSave_advancesBaseline; delisted_selectionExcludedNoticePreserved; outOfStock_selectionRetained; otherOwner404; concurrentSave_one409Revision; invalidCategory_saveRejected; delete_doesNotMutateCopiedCartGroup. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(secondRead.baselinePrices()).isEqualTo(firstRead.baselinePrices());
assertThat(secondRead.notices()).contains(priceChangeNotice);
assertThat(savedAgain.revision()).isEqualTo(oldRevision + 1);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=DraftIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Persist normalized slot IDs/prices/catalog versions on explicit saves, permit partial drafts, check owner and expectedRevision. GET derives effective selection/current validation/notices with no write. Preserve historical IDs for replacement explanation; cart groups are independent copies.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=DraftIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: revisioned drafts and staleness`. Record test command/results in the task execution notes.

### B3 — Python service and trusted contracts

**Requirements:** R07, R15, R17; S01, S07.
**Files:** Create optimizer/pyproject.toml, optimizer/uv.lock, optimizer/app/main.py, optimizer/app/contracts.py, optimizer/app/security.py, optimizer/tests/test_contracts.py, contracts/optimizer.openapi.yaml.
**Interfaces:** Produces `SolveRequest`, `SolveResult`, `CatalogSnapshot`, `SlotSelection` models matching S07; private `/internal/optimizer/solve` and health route. B4 implements `solve(request: SolveRequest) -> SolveResult`.

- [ ] Step 1: Write failing tests in the stated test file: schema_acceptsVndIntegers_rejectsFloat; wrongGoal422InternalMaps400Public; missingInternalKey401; wrongKey401; snapshotOver200_rejects; perSlotOver30_rejects; missingMetadata_rejects; internalPortsPrivate. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```python
assert payload.currency == "VND"
assert payload.budgetMinor == 20000000
assert validate_float_budget().is_error
assert unauthenticated_response.status_code == 401
```

- [ ] Step 2: Run `cd optimizer && uv run pytest tests/test_contracts.py -q`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Bootstrap declared runtime/lock and pytest/httpx/respx dependencies first. Define exact Pydantic strict integer/enums/aliases and snapshot bounds; authenticate constant-time API key check. Produce versioned internal OpenAPI and shared success/failure JSON fixtures; no browser-supplied catalog input.
- [ ] Step 4: Rerun `cd optimizer && uv run pytest tests/test_contracts.py -q`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: python service and trusted contracts`. Record test command/results in the task execution notes.

### B4 — Constraint search and bounded execution

**Requirements:** R07; S06, S07.
**Files:** Create optimizer/app/rules.py, optimizer/app/solver.py, optimizer/app/supervisor.py, optimizer/tests/test_solver.py, optimizer/tests/test_supervisor.py.
**Interfaces:** Produces `solve(request: SolveRequest) -> SolveResult`, `SearchSupervisor.solve(request: SolveRequest) -> SolveResult`; rule parity consumes contracts/fixtures/compatibility.json; statuses/proof flags are S07.

- [ ] Step 1: Write failing tests in the stated test file: exhaustiveSmallOracle_sameOptimalAndTie; aggregatePowerTrap_rejected; gpuCaseTrap_rejected; greedy700Gpu_trap_finds900Build; timeoutWithIncumbent_feasible; timeoutWithoutIncumbent_unknown; missingCategory_provenNoCompatible; belowMinimum_onlyClaimMinimumWhenProven; allSearchPhasesSameDeadline; thirdWorker_rejectedNoQueue; blockedWorker_terminated; healthResponsiveDuringTwoSearches. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```python
assert oracle_result.selections == solved.selections
assert timed_with_incumbent.status == "FEASIBLE"
assert timed_empty.status == "TIMEOUT_NO_SOLUTION"
assert not timed_empty.minimumCostProven
assert solved.totalMinor <= request.budgetMinor
```

- [ ] Step 2: Run `cd optimizer && uv run pytest tests/test_solver.py tests/test_supervisor.py -q`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Use joint partial-state rules, admissible lower-price/upper-utility bounds, deterministic score/price/ID ties and retained complete valid incumbent. Base/upgrades/minimum search share3000ms monotonic deadline with checks≤256nodes. Two supervised worker-process slots/no queue; kill unresponsive work at3500ms, reap safely, return no unsupported proof. No blocking CPU loop in async handler; no independent MCKP compatibility guarantee.
- [ ] Step 4: Rerun `cd optimizer && uv run pytest tests/test_solver.py tests/test_supervisor.py -q`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: constraint search and bounded execution`. Record test command/results in the task execution notes.

### B5 — Java optimizer boundary and outage behavior

**Requirements:** R07, R15, R16; S07, TRD8.
**Files:** Create backend/src/main/java/com/estore/builder/optimizer/OptimizerClient.java, backend/src/main/java/com/estore/builder/optimizer/AutoBuildService.java, backend/src/main/java/com/estore/builder/optimizer/OptimizerResilience.java; test backend/src/test/java/com/estore/builder/OptimizerBoundaryIT.java.
**Interfaces:** Produces `autoBuild(AutoBuildRequest request):SolveView`; AutoBuildRequest=(budgetMinor,currency,goal); consumes F4 CatalogSnapshot, B1 validation, and Python SolveResult.

- [ ] Step 1: Write failing tests in the stated test file: anonymousAutoBuildAllowed; browserCatalogFieldRejected400; unknownSelectedId503; wrongSlot503; changedPriceStockSpec409BuildStale; resultOverBudget503; invalidProofFlag503; validNoSolution_notBreakerFault; transportFailures_tripBreaker; tenSecondOverallBound; twoPermits_noQueue; halfOpenProbes_recover. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(stale.code()).isEqualTo("BUILD_STALE");
assertThat(malformed.code()).isEqualTo("OPTIMIZER_INVALID_RESULT");
assertThat(validNoSolutionBreakerFailureCount).isZero();
assertThat(concurrentPermitsUsed).isLessThanOrEqualTo(2);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=OptimizerBoundaryIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Send trusted snapshot only, enforce500ms connect/10s overall, TIME_BASED10s/min10calls/50%/30sopen/2probes and2permit no-queue bulkhead. Recompute current price/power/stock/rules before public enrichment; stale result409 with manual/retry action, no hidden repeated solver calls. Explain proof statuses faithfully.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=OptimizerBoundaryIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: java optimizer boundary and outage behavior`. Record test command/results in the task execution notes.

### B6 — Optional bounded explanation boundary

**Requirements:** R08; S07, S08.
**Files:** Create optimizer/app/explanation.py, backend/src/main/java/com/estore/builder/ExplanationService.java, optimizer/tests/test_explanation.py, backend/src/test/java/com/estore/builder/ExplanationBoundaryTest.java.
**Interfaces:** Produces `explain(selection: SlotSelection, goal: Goal, catalog: CatalogSnapshot) -> ExplanationResult`; ExplanationResult=(text nullable,selectionHash). `ExplanationProvider.generate(VerifiedExplanationInput):String` is the configured provider boundary; disabled mode returns null.

- [ ] Step 1: Write failing tests in the stated test file: disabledProvider_returnsNull; slowStreaming_overall5s_returnsNull; providerError_buildUnaffected; changedSelection_invalidatesHash; textOver2000_bounded; catalogPromptInjection_treatedAsData; noPiiInPayload; plainTextRendering. Optional live provider/model selection is recorded before configuring its adapter, not invented by this task. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```python
assert timeout_result.text is None
assert failed_provider_result.text is None
assert result.selectionHash == canonical_selection_hash
assert "shippingAddress" not in provider_payload
```

- [ ] Step 2: Run `cd optimizer && uv run pytest tests/test_explanation.py -q`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement separate explanation endpoint/provider port, fixed structured trusted input, five-second total deadline and2,000-character plain-text output. Default disabled; test with deterministic provider stub. Enabling a real AI adapter requires its model/API/cost/data decision and corresponding integration test; this optional task is not evidence of a live AI integration.
- [ ] Step 4: Rerun `cd optimizer && uv run pytest tests/test_explanation.py -q`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: optional bounded explanation boundary`. Record test command/results in the task execution notes.

## Completion evidence

Record actual commands, discovered test counts, results, and any failed cases. Run the component's full verification once after its tasks pass; broader integration belongs to the delivery plan. Keep the scope/contract changes synchronized with the specification. A provider stub proves application behavior under a modeled protocol; the sandbox and live gates require separate evidence.
