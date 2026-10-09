# Integration and Delivery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Revision:** 2026-10-08; replaces the previous task bodies. Documentation work only; execution has not started.

**Goal:** Produce a reproducible integrated core milestone and evidence that each requirement has an owner and test.

**Architecture:** Contracts and shared fixtures align Java, Python, and Next before integration. Five core services run together; offline provider simulations, real sandbox validation, and live launch are separate gates.

**Tech Stack:** Docker Compose, CI, OpenAPI/JSON Schema, Playwright, k6, Maven, pytest, Node tooling

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

D1 follows F1/B3/U1 scaffolds; D2 follows connected core API implementations. D3 needs checkout/builder/UI; D4 needs D2/D3. No application source exists at plan creation. Three students own backend, algorithm, and frontend work; designate one migration coordinator before execution.

Paths below are proposed new application files; the current folder contains documentation only. Domain packages group controller/DTO/repository helpers with their owning service. The named service owns its transaction rules; split helpers by responsibility rather than building one application-wide service. Test paths and main deliverables are explicit in each task. Migration filenames are allocated by S02; do not create conflicting migration numbers in another plan.

## Review Focus

- Empty-database startup must work without hidden manually created files — D1.
- Public/internal/client DTOs must agree on VND and proof states — D2.
- Healthy plentiful-stock checkout must succeed, not just avoid negatives — D3.
- Signed provider replay/stub tests are not evidence of real sandbox money flow — D3.
- Zero discovered tests, unsupported runtimes, and incomplete study attempts cannot be reported as success — D4.

## Tasks

### D1 — Compose topology and deterministic seeds

**Requirements:** R17; S01, S02, S08.
**Files:** Create compose.yaml, backend/Dockerfile, optimizer/Dockerfile, frontend/Dockerfile, .env.example, scripts/demo-start.sh, scripts/seed-demo.sh, scripts/verify-compose.sh, docs/environment.md, docs/runbooks/demo.md.
**Interfaces:** Produces `scripts/demo-start.sh` and `scripts/verify-compose.sh`; consumes service health routes/migrations from component scaffolds; env example has no real secret.

- [ ] Step 1: Bootstrap the declared build/test tooling before invoking its commands; commit wrapper/lockfile support and example configuration without secrets.
- [ ] Step 2: Add the smoke checks: freshVolume_bootMigrationsAndSeed; fiveCoreServicesHealthy; seedRerun_noDuplicates; privateOptimizerDbRedisNotPublished; missingSigningKey_startupFailure; liveSecretInDemoProfile_rejected; imageDigestsRecorded.
- [ ] Step 3: Create health-gated five-service startup, internal network and localhost-only development ingress. Generate ignored local RSA keys under .local/keys and reuse them across local restarts; never commit generated keys or production secrets. Seed80 PC parts and20 general products with positive/zero stocks, compatible/incompatible platforms, provenance labels, and synthetic demo accounts/addresses. Provider stub profile supports offline core demo; MoMo sandbox credentials are separate. Include compatible platform evidence instead of invented hardware claims.
- [ ] Step 4: Run `bash scripts/verify-compose.sh`. Expected: exit 0, declared checks execute, no missing test runner or ignored integration suite.
- [ ] Step 5: Review the diff and commit only this task's files with message `build: compose topology and deterministic seeds`. Initialize source control at implementation start if this folder is still not a repository.

### D2 — Contract parity and connected user journeys

**Requirements:** R01–R17; S01–S08.
**Files:** Create contracts/public.openapi.yaml, contracts/fixtures/checkout/, scripts/verify-contracts.sh, frontend/playwright.config.ts, frontend/e2e/core.spec.ts, frontend/e2e/payment-recovery.spec.ts.
**Interfaces:** Produces versioned public/internal schema fixtures and generated frontend contracts; consumes real Java/Python/PostgreSQL/Redis with controllable HTTP provider stub.

- [ ] Step 1: Write failing tests in the stated test file: catalogToStandalonePurchase; builderToAtomicGroupedMixedCart; savedDraftPriceChange; quoteRepriceReview; doubleClickReloadOneOrder; authorizeNotPaid_captureReceipt; pendingCancelSafeRelease; providerReturnFakeSuccess_noReceipt; otherUser404; adminStockAudit; keyboard360Layout; sharedRulesJavaPythonFrontendParity; malformedProofRejected. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```typescript
expect(new Set(submittedCheckoutKeys).size).toBe(1);
expect(await backendOrder.status).toBe("PAYMENT_CONFIRMED");
expect(await backendInventory.available).toBe(initialAvailable - purchasedQuantity);
```

- [ ] Step 2: Run `bash scripts/verify-contracts.sh && cd frontend && npm run test:e2e`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Generate schemas/types from actual backend contracts, check committed fixtures with every service, and run browser journeys against the full stack. Test public routes and owned/private boundaries. Provider stub controls IPN/query/capture interleavings and realistic signatures; it does not bypass application verification.
- [ ] Step 4: Rerun `bash scripts/verify-contracts.sh && cd frontend && npm run test:e2e`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: contract parity and connected user journeys`. Record test command/results in the task execution notes.

### D3 — Concurrency, restart recovery, and MoMo sandbox

**Requirements:** R10, R11, R12, R17; S04, S05.
**Files:** Create tests/load/last-unit.js, tests/load/plentiful-stock.js, scripts/verify-recovery.sh, scripts/momo-sandbox-smoke.sh, docs/evidence/core-checkout.md.
**Interfaces:** Produces k6 safety/success metrics and recorded sandbox evidence; consumes real checkout schema/HTTP/provider stub and separately configured MoMo sandbox merchant credentials.

- [ ] Step 1: Write failing tests in the stated test file: fiftyBuyersOneUnit_oneHold; sameKeyConcurrent_oneAllocation; plentifulStockAllHealthyAttemptsSucceed; killAfterCreateBeforeAssociation_recovers; killDuringCapture_noRelease; duplicateIpn_noDoubleConfirm; twoExpiryWorkers_once; actualSandboxAuthorizeCaptureAndRefund; missingSandboxCredentials_reportNotRun. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```javascript
check(summary, { "one last-unit hold": s => s.successfulHolds === 1 });
check(summary, { "zero negatives": s => s.negativeInventoryCount === 0 });
check(plentiful, { "healthy attempts succeed": s => s.successfulHolds === s.attempts });
```

- [ ] Step 2: Run `k6 run tests/load/last-unit.js && k6 run tests/load/plentiful-stock.js && bash scripts/verify-recovery.sh`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Run tests with quotas explicitly raised in a nonproduction load profile; retain real inventory/version semantics and successful payment stub behavior. Separately run `bash scripts/momo-sandbox-smoke.sh` only with sandbox credentials to validate hosted authorize/capture/cancel/refund/query, amount/signatures and notification acknowledgement. Store timings/outcomes; no live calls and no invented PASS when credentials are absent.
- [ ] Step 4: Rerun `k6 run tests/load/last-unit.js && k6 run tests/load/plentiful-stock.js && bash scripts/verify-recovery.sh`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: concurrency, restart recovery, and momo sandbox`. Record test command/results in the task execution notes.

### D4 — CI, study metrics, and milestone evidence

**Requirements:** R17, R18; S10.
**Files:** Create .github/workflows/ci.yml, scripts/verify-all.sh, docs/evidence/core-milestone.md, docs/evaluation/protocol.md, backend/src/main/java/com/estore/observability/JourneyEvents.java, backend/src/main/java/com/estore/observability/JourneyController.java, frontend/src/lib/journey-events.ts; modify backend/src/main/java/com/estore/cart/CartService.java, backend/src/main/java/com/estore/checkout/AllocationTransaction.java, backend/src/main/java/com/estore/payment/LifecycleTransaction.java; test backend/src/test/java/com/estore/observability/JourneyEventsIT.java.
**Interfaces:** Produces clean-build CI/verification command, discovered-test counters, deduplicated started/completed journey metrics, and requirement evidence ledger.

- [ ] Step 1: Write failing tests in the stated test file: unitAndITDiscovered_notZero; pythonTestsDiscovered; frontendBuildAndTypecheck; sharedSchemasDrift_fail; emptyDatabaseMigrationsPass; telemetryReplay_dedup; clientPaymentComplete_rejected; transactionRollback_noCompletionMetric; cohortWindow30mIncompleteAttemptsCounted; secretsPiiAbsent; unsupportedRuntimeManifest_fail; optimizerOracleAndRuntimeReport. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```json
{"checkoutWindowMinutes":30,"deduplicationKey":"journeyId:eventType","includeIncompleteAttempts":true,"reportLoginTimeSeparately":true}
```

- [ ] Step 2: Run `bash scripts/verify-all.sh`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: CI runs Maven verify, Python tests, frontend unit/type/build/E2E, migrations/Redis/contracts and Compose smoke with provider stub. Include dependency/secrets checks and nonzero test-discovery gates. Implement S10 client-start telemetry and durable backend-only completion events using X-Journey-Id and persisted snapshot IDs; never trust browser payment-complete events. Define checkout/build timing denominators, pilot baseline/counterbalanced study and exact participant counts. Core exit requires R01–R07/R09–R18 evidence, with R08 optional; production requirements remain open until P7.
- [ ] Step 4: Rerun `bash scripts/verify-all.sh`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: ci, study metrics, and milestone evidence`. Record test command/results in the task execution notes.

## Completion evidence

Record actual commands, discovered test counts, results, and any failed cases. Run the component's full verification once after its tasks pass; broader integration belongs to the delivery plan. Keep the scope/contract changes synchronized with the specification. A provider stub proves application behavior under a modeled protocol; the sandbox and live gates require separate evidence.
