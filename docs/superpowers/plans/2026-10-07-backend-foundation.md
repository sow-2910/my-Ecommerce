# Backend Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Revision:** 2026-10-08; replaces the previous task bodies. Documentation work only; execution has not started.

**Goal:** Provide reliable identity, catalog, admin, and owned cart infrastructure for the builder and checkout.

**Architecture:** A Java modular monolith with PostgreSQL constraints and short domain transactions. Redis caches only stable facts and provides shared quotas; live availability remains in PostgreSQL.

**Tech Stack:** Java/Spring Boot, Spring Security, JPA, Flyway, PostgreSQL, Redis, JUnit, Testcontainers

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

Execute F1–F4 first. Builder B1 depends on F2/F4; F5/F6 depend on B1 for authoritative build/preview validation. F7 can follow F3/F4. The checkout plan consumes F1–F7. F1 owns the backend scaffold; F2 owns V1__foundation.sql. Builder B2 owns V2__saved_builds.sql.

Paths below are proposed new application files; the current folder contains documentation only. Domain packages group controller/DTO/repository helpers with their owning service. The named service owns its transaction rules; split helpers by responsibility rather than building one application-wide service. Test paths and main deliverables are explicit in each task. Migration filenames are allocated by S02; do not create conflicting migration numbers in another plan.

## Review Focus

- Concurrent duplicate registration/counter updates commit predictably — F3.
- Refresh response loss and benign multi-tab rotation do not corrupt sessions — F3.
- In-stock filters remain correct with warm Redis or catalog updates — F4.
- Two builds sharing a product remain independent from a standalone line — F5.
- Stock adjustment races and limiter outages cannot bypass invariants — F6/F7.

## Tasks

### F1 — Build and contract scaffold

**Requirements:** R15, R17; S01, S03.
**Files:** Create backend/pom.xml, backend/mvnw, backend/.mvn/wrapper/maven-wrapper.properties, backend/src/main/resources/application.yml, backend/src/main/java/com/estore/EStoreApplication.java, backend/src/main/java/com/estore/common/ApiError.java; test backend/src/test/java/com/estore/common/ContractSmokeTest.java and backend/src/test/java/com/estore/foundation/FoundationSmokeIT.java.
**Interfaces:** Produces ApiError from S01, `/actuator/health` private readiness, request correlation filter, and reproducible `./mvnw verify`.

- [ ] Step 1: Bootstrap the declared build/test tooling before invoking its commands; commit wrapper/lockfile support and example configuration without secrets.
- [ ] Step 2: Add the smoke checks: contract_errorSerialization_matchesS01; health_profileRequiresDatabase; verify_discoversUnitAndIT (create foundation/FoundationSmokeIT.java alongside the unit smoke). Unit/Failsafe separation must report counts; production configuration rejects missing keys.
- [ ] Step 3: Create the Boot 4.1-compatible dependency set, wrapper, Surefire/Failsafe configuration, Testcontainers JUnit support and PostgreSQL fixtures. Configure JSON to reject unknown request fields and float-to-integer coercion. Produce docs/environment.md with exact tested backend versions and a compile/integration smoke result. Use `ApiError(String code,String message,boolean retryable,Map<String,Object> details,UUID requestId)`; no old status/error/timestamp shape.
- [ ] Step 4: Run `cd backend && ./mvnw verify`. Expected: exit 0, declared checks execute, no missing test runner or ignored integration suite.
- [ ] Step 5: Review the diff and commit only this task's files with message `build: build and contract scaffold`. Initialize source control at implementation start if this folder is still not a repository.

### F2 — Foundation schema and exact money

**Requirements:** R01, R03, R10, R14; S01, S02.
**Files:** Create backend/src/main/resources/db/migration/V1__foundation.sql, backend/src/main/java/com/estore/catalog/Inventory.java, backend/src/main/java/com/estore/common/Money.java, backend/src/main/java/com/estore/common/MoneyMath.java; test backend/src/test/java/com/estore/foundation/FoundationSchemaIT.java and backend/src/test/java/com/estore/common/MoneyMathTest.java.
**Interfaces:** Produces Inventory with @Version long version; `MoneyMath.total(long unitPriceMinor,int quantity):long`; S02 foundation entities/repositories.

- [ ] Step 1: Write failing tests in the stated test file: negativeInventory_rejected; duplicateStandalone_rejected; groupCrossCart_rejected; quantityOverflow_rejected; fractionalVnd_rejectedAtBoundary; fullV1MigratesEmptyDatabase. VND 19,990×3=59,970, no ×100 conversion. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(MoneyMath.total(19990L, 3)).isEqualTo(59970L);
assertThatThrownBy(() -> persistInventory(-1)).isInstanceOf(RuntimeException.class);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dtest=MoneyMathTest -Dit.test=FoundationSchemaIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement checked arithmetic and all V1 table/index/partial uniqueness constraints. Reject unsupported currency/fractions before construction. Wire @Version for every inventory writer; no stock changes in catalog update setters.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dtest=MoneyMathTest -Dit.test=FoundationSchemaIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: foundation schema and exact money`. Record test command/results in the task execution notes.

### F3 — Account login and refresh lifecycle

**Requirements:** R02, R15; S03, TRD9.
**Files:** Create backend/src/main/java/com/estore/auth/AuthService.java, backend/src/main/java/com/estore/auth/RefreshService.java, backend/src/main/java/com/estore/auth/AuthController.java, backend/src/main/java/com/estore/auth/JwtService.java, backend/src/main/java/com/estore/config/SecurityConfig.java; test backend/src/test/java/com/estore/auth/AuthLifecycleIT.java.
**Interfaces:** Consumes V1 users/sessions/tokens; produces `login(LoginRequest):LoginResult`, `rotate(String token,UUID requestId):RefreshResult`, `logout(UUID sessionId):void`. Controllers expose S03 cookie/access-token contracts.

- [ ] Step 1: Write failing tests in the stated test file: concurrentRegister_oneAccount; fiveParallelFailures_persistAndLock; expiredLock_resets; refresh_twoIds_oneSuccessOne409; refresh_sameIdLostResponse_replaysWithin30s; oldReuse_afterGrace_revokes; logout_currentSessionOnly; csrf_invalidOriginDenied; over72Utf8Bytes_rejected. Access JWT signing survives restart/configured key rotation. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(successfulRotations).isEqualTo(1);
assertThat(conflictingRotation.code()).isEqualTo("REFRESH_IN_PROGRESS");
assertThat(reloadedUser.failedLoginAttempts()).isEqualTo(5);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=AuthLifecycleIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement bcrypt12/min12/max72UTF8, normalized email, fifteen-minute JWT and seven-day absolute refresh sessions. Persist login-counter changes before returning generic401 AUTH_FAILED, including locked accounts; perform dummy bcrypt verification for unknown/locked users. Atomically consume/replace refresh; encrypted thirty-second same-request replay; no family revocation for benign concurrent IDs during grace. Configure shared RS256 keys and exact public/private route matchers; signed double-submit CSRF + Origin guard.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=AuthLifecycleIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: account login and refresh lifecycle`. Record test command/results in the task execution notes.

### F4 — Catalog reads and revisioned cache

**Requirements:** R01, R14; S03, S08, TRD10.
**Files:** Create backend/src/main/java/com/estore/catalog/ProductService.java, backend/src/main/java/com/estore/catalog/ProductController.java, backend/src/main/java/com/estore/catalog/CatalogSnapshotService.java, backend/src/main/java/com/estore/catalog/CatalogFactCache.java; test backend/src/test/java/com/estore/catalog/CatalogCacheIT.java.
**Interfaces:** Produces `getProduct(long id):ProductView`, `search(ProductQuery query):Page<ProductView>`, `snapshot():CatalogSnapshot`; snapshots include reviewed platforms and live stock; exposes fixed category tree.

- [ ] Step 1: Write failing tests in the stated test file: warmCache_lastUnitHold_disappearsFromInStockPage; release_restoresAvailability; redisCacheDown_readsDatabase; rollback_doesNotAdvanceRevision; specEdit_changesFactKey; sort_equalPrice_stableId; size101_returns400; qNameAndSku; inactivePublic404; snapshotsOverLimit_failExplicitly. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(inStockPage.items()).noneMatch(p -> p.id() == heldProductId);
assertThat(invalidPage.status()).isEqualTo(400);
assertThat(cacheFailureProduct.availableQuantity()).isEqualTo(databaseAvailable);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=CatalogCacheIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Query filtered/paged availability in PostgreSQL. Cache stable facts by DB catalog revision for600s, batch live quantities, and bypass failed cache reads. Implement size1–100/zero-based page/sort allowlist and ID tie. Build trusted snapshot, rejecting oversize candidate sets rather than truncating.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=CatalogCacheIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: catalog reads and revisioned cache`. Record test command/results in the task execution notes.

### F5 — Owned cart and atomic build groups

**Requirements:** R03, R04, R13; S02, S03, S04.
**Files:** Create backend/src/main/java/com/estore/cart/CartService.java, backend/src/main/java/com/estore/cart/CartController.java, backend/src/main/java/com/estore/cart/CartCleanup.java; test backend/src/test/java/com/estore/cart/CartGroupsIT.java.
**Interfaces:** Consumes B1 `CompatibilityService.validate(SlotSelection):BuildValidation`; produces `addItem(UUID userId,long productId,int quantity):CartSnapshot`, `addBuild(UUID userId,AddBuildRequest):CartSnapshot`, `snapshot(UUID userId):CartSnapshot`, `cleanupPurchased(OrderCartSnapshot):CleanupResult`.

- [ ] Step 1: Write failing tests in the stated test file: parallelAdd_mergesUniqueStandalone; addBuild_invalidSlot_rollsBackAll; twoBuildsSharedCpu_andStandalone_preserveGroups; sourceDraftOtherOwner404; draftEdit_doesNotChangeCartGroup; groupCount_changesAllQuantities; individualGroupEdit409; shortageCart_retained; purchasedCleanup_preservesNewRevision. Use actual concurrent database transactions. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(cart.groups()).hasSize(2);
assertThat(cart.standaloneItems()).hasSize(1);
assertThat(failedBuildAdd.persistedSlots()).isZero();
assertThat(cleanupAfterOtherTab.changedEntryRetained()).isTrue();
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=CartGroupsIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Lock cart before mutation, enforce DB uniqueness, copy validated seven-slot groups atomically, normalize quantities, and use independent group identity. Implement revision checks/bounds and no reservation at cart-add. Cleanup matches exact purchased revision/quantity and preserves changed entries intact.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=CartGroupsIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: owned cart and atomic build groups`. Record test command/results in the task execution notes.

### F6 — Admin publishing and audited stock adjustment

**Requirements:** R14, R13; S02, S03, S06.
**Files:** Create backend/src/main/java/com/estore/admin/AdminProductService.java, backend/src/main/java/com/estore/admin/AdminProductController.java, backend/src/main/java/com/estore/admin/StockAdjustmentService.java, backend/src/main/java/com/estore/admin/AdminOverviewController.java; test backend/src/test/java/com/estore/admin/AdminOperationsIT.java.
**Interfaces:** Consumes B1 publication/preview validators; produces `preview(ProposedComponent):BuildValidation`, `publish(ProductWriteRequest):ProductView`, `adjust(UUID actorId,long productId,StockAdjustmentRequest,UUID key):StockAdjustmentView`; admin order routes integrate C7.

- [ ] Step 1: Write failing tests in the stated test file: preview_noWrites; missingRequiredMetadata_rejectPublication; wrongTagKind_reject; duplicateSku_one409; inactiveAdminVisible; categoryChange_reject; catalogEdit_advancesRevisionAfterCommitOnly; adjustmentRaceWithHold_honorsVersion; repeatedAdjustmentKey_oneMovement; lowStockAndHeld_dashboardCorrect; unauthenticatedAdminDenied. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(previewWriteCount).isZero();
assertThat(replayedAdjustment.movementCount()).isEqualTo(1);
assertThat(availableAfterRace).isGreaterThanOrEqualTo(0);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=AdminOperationsIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement controlled tag kinds/publication fields and reviewed platform evidence. Take exclusive catalog guard before catalog edits; audit mutations. Stock deltas use versioned fresh transaction/retry and unique movement sources; never overwrite stock from a stale admin total. Soft delist, read inactive products, expose redacted overview/recovery data.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=AdminOperationsIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: admin publishing and audited stock adjustment`. Record test command/results in the task execution notes.

### F7 — Request quotas and backend security boundaries

**Requirements:** R15; S03, TRD9.
**Files:** Create backend/src/main/java/com/estore/security/RateLimitFilter.java, backend/src/main/java/com/estore/security/OriginGuard.java, backend/src/main/java/com/estore/security/RequestLimits.java; test backend/src/test/java/com/estore/security/RequestProtectionIT.java.
**Interfaces:** Produces shared Redis quota checks, trusted proxy-IP resolution, bounded-body validation, redacted error/log responses; consumes F3 authentication and S03 security map.

- [ ] Step 1: Write failing tests in the stated test file: public101st_returns429RetryAfter; checkout11th_returns429; attackerForwardedIp_notTrusted; limiterDown_sensitiveMutation503_readAvailable; publicBuilder_accessAllowed_butDraftRequiresAuth; logoutRequiresCookieAndCsrf; oversizedBody400; secretFields_notLogged; ownResourceLookup404. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(limitExceeded.status()).isEqualTo(429);
assertThat(limiterOutageCheckout.status()).isEqualTo(503);
assertThat(logText).doesNotContain(password, refreshToken, signingKey);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=RequestProtectionIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement TRD quotas through atomic Redis TTL counters and exact matchers, limit public bodies64KiB, use prepared queries/output escaping, and reject unsafe redirects/URLs. Separate provider notification protections from shopper quotas; Redis cache fallback does not bypass security mutation quotas.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=RequestProtectionIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: request quotas and backend security boundaries`. Record test command/results in the task execution notes.

## Completion evidence

Record actual commands, discovered test counts, results, and any failed cases. Run the component's full verification once after its tasks pass; broader integration belongs to the delivery plan. Keep the scope/contract changes synchronized with the specification. A provider stub proves application behavior under a modeled protocol; the sandbox and live gates require separate evidence.
