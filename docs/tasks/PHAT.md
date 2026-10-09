# Phat — Individual Task Implementation Plan

> **For agentic workers:** Use `superpowers:executing-plans` when implementing these tasks yourself; use `superpowers:subagent-driven-development` only when delegation is explicitly selected. The shared workflow below applies to every task.

**Revision:** 2026-10-09. Task instructions only; no implementation or test execution is claimed.
**Owner:** Phat. **Owned tasks:** 13. Reviewers are specified on each card.
**Goal:** Implement reliable backend identity, cart and purchase/payment recovery, then production identity and returns/refunds.
**Architecture:** Java is authoritative for money, ownership, inventory and payment lifecycle. Short transactions persist local intent and durable work; provider calls happen outside database locks.
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
| F1 | Core | Huy |
| F2 | Core | Huy |
| F3 | Core | Bảo |
| F5 | Core | Huy |
| F7 | Core | Huy |
| C1 | Core | Huy |
| C2 | Core | Huy |
| C3 | Core | Huy |
| C4 | Core | Huy |
| C5 | Core | Huy |
| C6 | Core | Huy |
| P2 | Production | Bảo |
| P4 | Production | Huy |

## Task cards

### F1 — Build and contract scaffold

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R15, R17; S01, S03.
**Source task:** [F1 in the domain plan](../superpowers/plans/2026-10-07-backend-foundation.md).
**Business purpose:** Give every contributor the same reproducible Java runtime, error envelope and test discovery behavior.
**Prerequisites and handoffs:** Can start now. Read S01/S03; coordinate the environment manifest with Huy/B3 and Bảo/U1. No catalog or payment implementation is needed.
**Expected output:** A runnable backend skeleton, committed Maven wrapper, private readiness endpoint, consistent ApiError and exact tested runtime manifest; Phat builds F2 next, and Huy/F4 consumes its schema.
**Files and tests:** Create backend/pom.xml, backend/mvnw, backend/.mvn/wrapper/maven-wrapper.properties, backend/src/main/resources/application.yml, backend/src/main/java/com/estore/EStoreApplication.java, backend/src/main/java/com/estore/common/ApiError.java; test backend/src/test/java/com/estore/common/ContractSmokeTest.java and backend/src/test/java/com/estore/foundation/FoundationSmokeIT.java.
**Interfaces:** Produces ApiError from S01, `/actuator/health` private readiness, request correlation filter, and reproducible `./mvnw verify`.

**Business rules and implementation boundary:**

Create the Boot 4.1-compatible dependency set, wrapper, Surefire/Failsafe configuration, Testcontainers JUnit support and PostgreSQL fixtures. Configure JSON to reject unknown request fields and float-to-integer coercion. Produce docs/environment.md with exact tested backend versions and a compile/integration smoke result. Use `ApiError(String code,String message,boolean retryable,Map<String,Object> details,UUID requestId)`; no old status/error/timestamp shape.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| An invalid public request is rejected | The error contains code, message, retryable, details and requestId; it does not use a second invented error shape. |
| The required signing configuration is missing in production | Startup fails; the application does not silently generate a new production signing key. |
| A declared FoundationSmokeIT is run through Maven verify | The integration suite is discovered and its count is nonzero; a green unit-only run is insufficient. |

**Required named cases from the source plan:** contract_errorSerialization_matchesS01; health_profileRequiresDatabase; verify_discoversUnitAndIT (create foundation/FoundationSmokeIT.java alongside the unit smoke).

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw verify)
```

**Explain before review:** Why can a green build still be misleading when integration tests were never discovered?
**Reviewer checks:** Huy runs the wrapper from a fresh checkout, inspects dependency/runtime pins and confirms both unit and integration discovery.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### F2 — Foundation schema and exact money

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R01, R03, R10, R14; S01, S02.
**Source task:** [F2 in the domain plan](../superpowers/plans/2026-10-07-backend-foundation.md).
**Business purpose:** Put money, ownership and inventory constraints in persistence so correctness survives competing requests.
**Prerequisites and handoffs:** F1 runnable backend/test tooling. Coordinate V1 with Huy/F4 and reserve V2/V3 sequencing with Huy/B2 and Phat/C1.
**Expected output:** V1 foundation tables, FK/index/check/uniqueness constraints, Inventory @Version and checked integer money helpers. Huy consumes the schema in F4/B1/B2; Phat consumes it in F3/F5.
**Files and tests:** Create backend/src/main/resources/db/migration/V1__foundation.sql, backend/src/main/java/com/estore/catalog/Inventory.java, backend/src/main/java/com/estore/common/Money.java, backend/src/main/java/com/estore/common/MoneyMath.java; test backend/src/test/java/com/estore/foundation/FoundationSchemaIT.java and backend/src/test/java/com/estore/common/MoneyMathTest.java.
**Interfaces:** Produces Inventory with @Version long version; `MoneyMath.total(long unitPriceMinor,int quantity):long`; S02 foundation entities/repositories.

**Business rules and implementation boundary:**

Implement checked arithmetic and all V1 table/index/partial uniqueness constraints. Reject unsupported currency/fractions before construction. Wire @Version for every inventory writer; no stock changes in catalog update setters.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Price is 19,990 VND and quantity is 3 | The line amount is exactly 59,970; there is no multiplication by 100 and no floating-point rounding. |
| Two transactions insert the same standalone product into one cart | Database uniqueness permits one standalone row, not two NULL-group duplicates. |
| A grouped row references a group in another cart, or inventory is set below zero | The database rejects it even if a controller-level validation was bypassed. |

**Required named cases from the source plan:** negativeInventory_rejected; duplicateStandalone_rejected; groupCrossCart_rejected; quantityOverflow_rejected; fractionalVnd_rejectedAtBoundary; fullV1MigratesEmptyDatabase.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dtest=MoneyMathTest -Dit.test=FoundationSchemaIT verify)
```

**Explain before review:** Which invariants must the database enforce even when a service has already validated the request?
**Reviewer checks:** Huy migrates an empty real PostgreSQL database and checks actual constraints and JPA commit-time version behavior, not repository mocks.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### F3 — Account login and refresh lifecycle

**Owner:** Phat. **Reviewer:** Bảo. **Stage:** Core.
**Requirements:** R02, R15; S03, TRD9.
**Source task:** [F3 in the domain plan](../superpowers/plans/2026-10-07-backend-foundation.md).
**Business purpose:** Keep customer sessions usable under concurrent requests while preventing token replay and lost account-lock counters.
**Prerequisites and handoffs:** F1/F2. Cookie/CSRF/error fixtures must be reviewed with Bảo/U1 before either side connects. Configured issuer, audience and persistent signing keys are recorded in the environment contract.
**Expected output:** Register/login/me/refresh/logout behavior from S03, memory-compatible access JWTs and secure refresh cookies; Bảo integrates U1 without inventing token storage.
**Files and tests:** Create backend/src/main/java/com/estore/auth/AuthService.java, backend/src/main/java/com/estore/auth/RefreshService.java, backend/src/main/java/com/estore/auth/AuthController.java, backend/src/main/java/com/estore/auth/JwtService.java, backend/src/main/java/com/estore/config/SecurityConfig.java; test backend/src/test/java/com/estore/auth/AuthLifecycleIT.java.
**Interfaces:** Consumes V1 users/sessions/tokens; produces `login(LoginRequest):LoginResult`, `rotate(String token,UUID requestId):RefreshResult`, `logout(UUID sessionId):void`. Controllers expose S03 cookie/access-token contracts.

**Business rules and implementation boundary:**

Implement bcrypt12/min12/max72UTF8, normalized email, fifteen-minute JWT and seven-day absolute refresh sessions. Persist login-counter changes before returning generic401 AUTH_FAILED, including locked accounts; perform dummy bcrypt verification for unknown/locked users. Atomically consume/replace refresh; encrypted thirty-second same-request replay; no family revocation for benign concurrent IDs during grace. Configure shared RS256 keys and exact public/private route matchers; signed double-submit CSRF + Origin guard.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Five simultaneous bad passwords target one account | All five increments persist and the account locks for 15 minutes; returning 401 does not roll back the counters. |
| Two refreshes use different request IDs for the same old token within the 30-second grace | Exactly one rotates; the other receives 409 REFRESH_IN_PROGRESS without revoking the session or overwriting the cookie. |
| The same refresh request ID retries a lost response within grace | It replays the same replacement response; reuse after grace revokes that session. |
| An unknown, locked or wrong-password login is submitted | Return generic 401 AUTH_FAILED; perform the specified bcrypt work without logging password or token. |

**Required named cases from the source plan:** concurrentRegister_oneAccount; fiveParallelFailures_persistAndLock; expiredLock_resets; refresh_twoIds_oneSuccessOne409; refresh_sameIdLostResponse_replaysWithin30s; oldReuse_afterGrace_revokes; logout_currentSessionOnly; csrf_invalidOriginDenied; over72Utf8Bytes_rejected.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Bảo’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=AuthLifecycleIT verify)
```

**Explain before review:** Why is a failed login still a database write, and why must benign refresh concurrency differ from hostile reuse?
**Reviewer checks:** Bảo exercises concurrent refresh/cookie/CSRF fixtures against the real backend, checks logout behavior and confirms no token appears in browser storage.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### F5 — Owned cart and atomic build groups

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R03, R04, R13; S02, S03, S04.
**Source task:** [F5 in the domain plan](../superpowers/plans/2026-10-07-backend-foundation.md).
**Business purpose:** Preserve the identity of each PC build while maintaining an owned, editable cart.
**Prerequisites and handoffs:** F1/F2/F3/F4 and Huy/B1 validation. Core seven-slot fixtures and CartSnapshot fields must be shared with Bảo/U4. CartCleanup supports C7 but does not query nonexistent order tables before C1.
**Expected output:** Owned standalone operations, atomic independent build groups, revision-aware edits and exact purchased-entry cleanup. Cart adds never reserve stock.
**Files and tests:** Create backend/src/main/java/com/estore/cart/CartService.java, backend/src/main/java/com/estore/cart/CartController.java, backend/src/main/java/com/estore/cart/CartCleanup.java; test backend/src/test/java/com/estore/cart/CartGroupsIT.java.
**Interfaces:** Consumes B1 `CompatibilityService.validate(SlotSelection):BuildValidation`; produces `addItem(UUID userId,long productId,int quantity):CartSnapshot`, `addBuild(UUID userId,AddBuildRequest):CartSnapshot`, `snapshot(UUID userId):CartSnapshot`, `cleanupPurchased(OrderCartSnapshot):CleanupResult`.

**Business rules and implementation boundary:**

Lock cart before mutation, enforce DB uniqueness, copy validated seven-slot groups atomically, normalize quantities, and use independent group identity. Implement revision checks/bounds and no reservation at cart-add. Cleanup matches exact purchased revision/quantity and preserves changed entries intact.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Two builds share CPU X and there is also standalone CPU X | The cart retains two group identities plus one standalone entry; only checkout aggregates stock demand. |
| One selected slot is invalid during add-build | No group or partial group items are committed. |
| A saved draft is edited after being copied into the cart | The cart group remains unchanged. |
| Another tab changes an entry after checkout snapshots it | Payment cleanup preserves that changed entry intact; unchanged purchased entries are removed once. |

**Required named cases from the source plan:** parallelAdd_mergesUniqueStandalone; addBuild_invalidSlot_rollsBackAll; twoBuildsSharedCpu_andStandalone_preserveGroups; sourceDraftOtherOwner404; draftEdit_doesNotChangeCartGroup; groupCount_changesAllQuantities; individualGroupEdit409; shortageCart_retained; purchasedCleanup_preservesNewRevision.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=CartGroupsIT verify)
```

**Explain before review:** Why must group identity be preserved before inventory demand is aggregated?
**Reviewer checks:** Huy checks real concurrent adds, cross-cart/ownership rejection, all-or-none group writes and revision-aware cleanup; order integration is proved under C7/D2.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### F7 — Request quotas and backend security boundaries

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R15; S03, TRD9.
**Source task:** [F7 in the domain plan](../superpowers/plans/2026-10-07-backend-foundation.md).
**Business purpose:** Protect shared capacity and private actions consistently across all backend instances.
**Prerequisites and handoffs:** F3/F4 plus real Redis. Coordinate trusted ingress/origin configuration with Bảo/D1 and later P6; keep signed IPN protections distinct from shopper quotas.
**Expected output:** Shared atomic quota enforcement, body limits, exact public/private matchers, trusted proxy handling and redacted responses; all API consumers receive stable errors.
**Files and tests:** Create backend/src/main/java/com/estore/security/RateLimitFilter.java, backend/src/main/java/com/estore/security/OriginGuard.java, backend/src/main/java/com/estore/security/RequestLimits.java; test backend/src/test/java/com/estore/security/RequestProtectionIT.java.
**Interfaces:** Produces shared Redis quota checks, trusted proxy-IP resolution, bounded-body validation, redacted error/log responses; consumes F3 authentication and S03 security map.

**Business rules and implementation boundary:**

Implement TRD quotas through atomic Redis TTL counters and exact matchers, limit public bodies64KiB, use prepared queries/output escaping, and reject unsafe redirects/URLs. Separate provider notification protections from shopper quotas; Redis cache fallback does not bypass security mutation quotas.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| The 101st public read from an IP arrives inside one minute | Return 429 with Retry-After; the first 100 fit the public-read quota. |
| The 11th checkout initiation from a user arrives inside one minute | Return 429, including when requests reach different backend instances. |
| The shared security limiter is unavailable | Sensitive login/payment/refund mutations return 503; browsing can continue under the documented conservative policy. |
| A user supplies a spoofed forwarded IP or accesses another user’s order | Do not trust an unapproved proxy; a non-owned resource remains 404. |

**Required named cases from the source plan:** public101st_returns429RetryAfter; checkout11th_returns429; attackerForwardedIp_notTrusted; limiterDown_sensitiveMutation503_readAvailable; publicBuilder_accessAllowed_butDraftRequiresAuth; logoutRequiresCookieAndCsrf; oversizedBody400; secretFields_notLogged; ownResourceLookup404.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=RequestProtectionIT verify)
```

**Explain before review:** Why can Redis failure permit catalog reads while blocking payment mutations?
**Reviewer checks:** Huy tests atomic quotas with real Redis, checks exact security matchers and confirms provider IPN is not blocked by customer checkout quotas.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### C1 — Checkout persistence and reviewed quotes

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R09, R11; S01–S04.
**Source task:** [C1 in the domain plan](../superpowers/plans/2026-10-07-checkout-pipeline.md).
**Business purpose:** Obtain customer consent to a server-calculated price snapshot before allocating scarce stock.
**Prerequisites and handoffs:** Checkout foundation gate F1–F7 and B1. Huy/B2 must supply V2 before V3 is applied to shared environments; this is migration ordering, not a requirement to finish the whole draft UI. Freeze the labeled demo calculation/address configuration; real policies belong to P3.
**Expected output:** V3 checkout/order/attempt/operation/core-compensation schema and an owned five-minute QuoteView. Bảo/U4 displays the exact snapshot; C2 consumes quoteId.
**Files and tests:** Create backend/src/main/resources/db/migration/V3__checkout.sql, backend/src/main/java/com/estore/checkout/QuoteService.java, backend/src/main/java/com/estore/checkout/CheckoutController.java; test backend/src/test/java/com/estore/checkout/QuoteIT.java.
**Interfaces:** Consumes F5 CartSnapshot and F4 product data; produces `quote(UUID userId,QuoteRequest):QuoteView`, persisted quote snapshot/fingerprint, core order/attempt/operation/refund entities.

**Business rules and implementation boundary:**

Implement owned five-minute quotes under cart/shared-catalog guard, exact subtotal/shipping/tax totals, per-item catalog versions and cart/calculation fingerprint. Reject unsupported MoMo totals before reservations. Migrate all S02 checkout constraints including one unresolved provider attempt and compensation refunds.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| The customer requests a quote | Return the server’s line/group/address/fee snapshot and five-minute expiry; inventory is unchanged. |
| The cart, price, specification or calculation policy changes before initiation | Require new review with the appropriate typed 409; do not silently accept the new price. |
| The reviewed total exceeds MoMo’s 50,000,000 VND maximum | Reject before creating a stock hold. |
| Included tax is already part of displayed prices | Total is subtotal + shipping + additionalTaxMinor; included tax is disclosed and is not added again. |

**Required named cases from the source plan:** quote_exactVndTotal; quoteDoesNotReserve; foreignQuote404; priceChange409_beforeHold; expiredQuote409; cartChange409; policyChange409; providerAmountOutside1000To50000000_noHold; groupSnapshotPreserved.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=QuoteIT verify)
```

**Explain before review:** Why are quote freshness and inventory concurrency separate checks?
**Reviewer checks:** Huy compares quote amounts and immutable fields with the database, verifies no stock mutation and checks V3/core refund schema compatibility with later P4.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### C2 — Atomic allocation and replay identity

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R10, R11; S04.
**Source task:** [C2 in the domain plan](../superpowers/plans/2026-10-07-checkout-pipeline.md).
**Business purpose:** Create one complete order and hold without overselling, partial checkout or duplicate allocation.
**Prerequisites and handoffs:** C1 and the foundation/B1 gates. Use C1’s reviewed quote fixture and agree its consumption with Bảo/U4; publish CheckoutRequest/CheckoutView identity fixtures for Bảo/U5 and Huy/D3.
**Expected output:** A nontransactional coordinator plus separately proxied allocation transaction; immutable order/group/items, HELD reservations, checkout identity, first attempt and first CREATE job commit together.
**Files and tests:** Create backend/src/main/java/com/estore/checkout/CheckoutCoordinator.java, backend/src/main/java/com/estore/checkout/AllocationTransaction.java, backend/src/main/java/com/estore/checkout/CheckoutRequestRepository.java; test backend/src/test/java/com/estore/checkout/AllocationIT.java.
**Interfaces:** Produces `initiate(UUID userId,CheckoutRequest request,UUID key):CheckoutView`, `allocate(UUID userId,CheckoutRequest,UUID key):AllocatedOrder`; coordinator is nontransactional and allocation bean separately proxied.

**Business rules and implementation boundary:**

Use fresh backend inventory versions, deterministic ID order, whole-order transaction for snapshots/holds/identity/first CREATE job. Replays resolve before quote checks. Catch flush/commit concurrency errors outside transaction and cap3 attempts. Recover unknown commits by identity lookup, never provider/network retries inside allocation.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| 50 distinct healthy purchases compete for one remaining GPU | Exactly one order gets the hold; availability ends at zero, never negative. |
| One product in a multi-product order lacks stock | No order, reservation, checkout identity or CREATE job from that failed allocation remains. |
| Two simultaneous submissions use the same user/key/quoteId | They resolve to one order and one allocation, including replay after quote expiry. |
| The same key is submitted with another quoteId | Return 409 IDEMPOTENCY_CONFLICT; do not create another order. |
| Both transactions read inventory version 5 | Only an update expecting the still-current version can commit; a fresh whole-transaction retry may succeed if stock remains, with three attempts total. |

**Required named cases from the source plan:** hold_decrementsOnce; multiItemShortage_noOrderNoHolds; reversedProductOrder_noDeadlock; duplicateProductAcrossGroups_aggregatesStock; sameKeyConcurrent_oneOrder; sameKeyDifferentQuote409; replayAfterQuoteExpiry_sameOrder; commitTimeConflict_freshRetry; stockStillAvailableConflict_succeeds; thirdConflict409Busy; ambiguousCommit_lookupOriginal.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=AllocationIT verify)
```

**Explain before review:** Why must an optimistic conflict be caught outside commit, and how does idempotency recover an uncertain commit?
**Reviewer checks:** Huy checks database counts/stock/journal after actual concurrent transactions, forces commit-time conflicts and verifies no provider call occurs under the allocation transaction.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### C3 — MoMo adapter and durable worker dispatch

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R12, R19; S05.
**Source task:** [C3 in the domain plan](../superpowers/plans/2026-10-07-checkout-pipeline.md).
**Business purpose:** Recover external payment work across timeouts and crashes without issuing another financial operation.
**Prerequisites and handoffs:** C2 persisted identities/jobs and S05 protocol fixtures. Offline work uses the deterministic provider stub; real sandbox smoke needs separately supplied merchant credentials. Do not infer live capability.
**Expected output:** The S05 PaymentGateway adapter, durable guarded worker/claim transactions, synthetic operation-specific HMAC vectors and stable command identities; C4/C5/C6/C7 consume validated observations.
**Files and tests:** Create backend/src/main/java/com/estore/payment/PaymentGateway.java, backend/src/main/java/com/estore/payment/momo/MomoGateway.java, backend/src/main/java/com/estore/payment/PaymentWorker.java, backend/src/main/java/com/estore/payment/WorkerClaimTransaction.java, contracts/fixtures/momo/; test backend/src/test/java/com/estore/payment/MomoGatewayTest.java and backend/src/test/java/com/estore/payment/PaymentWorkerIT.java.
**Interfaces:** Produces PaymentGateway S05 interface and `runReadyOperations(Instant now,int limit):WorkerRunResult`; consumes persisted immutable command identities. Gateway START/QUERY/CAPTURE/CANCEL_AUTHORIZATION/REFUND/QUERY_REFUND map to MoMo protocol contexts.

**Business rules and implementation boundary:**

Implement operation-specific HMAC vectors with synthetic keys and immutable payloads. Use sandbox captureWallet/autoCapture=false/VND, hosted URL allowlist, context-aware states, encrypted action at rest. Claim order then operation with60s lease, call outside transaction max35s, persist result under guard. Stable request IDs, provider replay horizon and bounded retry/query cadence follow S05.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| MoMo CREATE returns code 0 with a hosted action | Store readiness/action, not PAYMENT_CONFIRMED; authorization code 9000 is also not capture. |
| A worker crashes after provider success but before local association | Recover using the same logical mutating request identity and validated query/replay; do not blindly create another payment. |
| A lease expires | A 60-second lease and 35-second total call bound support guarded recovery; provider calls occur outside database transactions. |
| A payment response times out or replay reaches 30 days from first dispatch | Timeout is UNKNOWN; retain proof obligations. Beyond the replay horizon enter REVIEW rather than minting a new financial command. |

**Required named cases from the source plan:** knownSignatureVectors_pass; alteredAmountSignature_fail; create0_returnsReadyNotPaid; authorized9000_notCaptured; decimalVndReject; unknownPayUrlHostReject; timeoutAfterCreate_reusesRequestId; crashAfterProviderBeforeAssociation_recovers; expiredLease_twoWorkers_oneEffect; neverDispatchedClosing_suppressesCreate; oldReplayBeyond30Days_review.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dtest=MomoGatewayTest -Dit.test=PaymentWorkerIT verify)
```

**Explain before review:** Why cannot the database transaction and provider call be treated as one atomic transaction?
**Reviewer checks:** Huy inspects synthetic signature vectors, claimed-job locking order, restart recovery, identity persistence and separate stub/sandbox evidence.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### C4 — Signed notifications and capture finalization

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R10, R12; S05.
**Source task:** [C4 in the domain plan](../superpowers/plans/2026-10-07-checkout-pipeline.md).
**Business purpose:** Let verified payment evidence change local order state exactly once, despite duplicate or reordered notifications.
**Prerequisites and handoffs:** C3 adapter verification and C2 order/hold model. C4 exposes release/compensation integration points; C5/C7 complete those paths before the whole payment lifecycle is accepted.
**Expected output:** Signed IPN handler and shared observation finalizer that atomically commits receipt, state transitions and follow-up jobs. Query and IPN use the same lifecycle rules.
**Files and tests:** Create backend/src/main/java/com/estore/payment/MomoIpnController.java, backend/src/main/java/com/estore/payment/PaymentObservationService.java, backend/src/main/java/com/estore/payment/LifecycleTransaction.java; test backend/src/test/java/com/estore/payment/PaymentObservationIT.java.
**Interfaces:** Produces `observe(PaymentObservation):ObservationResult`, `acceptIpn(byte[] body):void`; consumes C3 VerifiedNotification; authoritative query and IPN share order guard/event receipt persistence.

**Business rules and implementation boundary:**

Verify original notification representation/limits and persist receipt plus transitions/jobs atomically. One bounded IPN transaction/ten-second application budget; on failure return503 for redelivery, not a retry loop exceeding acknowledgement deadline. Code0 on CREATE is not payment; code9000 in two-step is authorization. Serialize capture/cancel dispatch, confirm only verified captured payment against retained holds; immutable paid attempt identity prevents old callbacks from changing a newer attempt/order.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A valid captured payment arrives while all reservations are HELD | Confirm the order and HELD→CONFIRMED; availability stays at its post-hold value. |
| The same event arrives twice or equivalent events have different identities | At most one confirmation and cleanup occur; business-state guards complement receipt uniqueness. |
| Receipt/state persistence fails | Return 503 and leave no durable receipt; return 204 only after a successful commit. |
| Authorization arrives while open before expiry, or while closing | Open valid holds enqueue capture; closing enqueues cancellation of authorization. Neither branch treats authorization as payment. |
| A failure from an old attempt arrives after confirmation | Do not downgrade the paid order or overwrite its paid attempt. |

**Required named cases from the source plan:** invalidSignature400; wrongMerchantOrAmount_quarantine; notification204AfterCommit; failedCommit_noReceiptAnd503; sameEventParallel_oneEffect; equivalentDifferentEvents_oneConfirmation; authorize_open_enqueuesCapture; authorize_closing_enqueuesCancel; captured_confirmsNoSecondDeduction; successThenOldFailure_noDowngrade; captureResultCrash_recovers; captureAfterDeadlineWithRetainedHolds_confirms.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=PaymentObservationIT verify)
```

**Explain before review:** Why must the notification receipt and its business effects commit in the same transaction?
**Reviewer checks:** Huy observes IPN acknowledgement timing, forces rollback/redelivery, verifies stock is not deducted twice and checks capture/closing interleavings with C5.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### C5 — Safe expiry and owned cancellation

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R12; S04, S05.
**Source task:** [C5 in the domain plan](../superpowers/plans/2026-10-07-checkout-pipeline.md).
**Business purpose:** Release scarce stock only when the order can no longer take payment, using compensation rather than a long database lock.
**Prerequisites and handoffs:** C3/C4 guarded dispatch/observations and C2 holds. Pair with Huy on capture-versus-close cases and supply Bảo/U5 with CANCELLED versus CLOSING fixtures.
**Expected output:** Owned cancellation, 30-second expiry scheduler and versioned proof-gated release transaction. First closing reason is preserved; new payment actions are suppressed.
**Files and tests:** Create backend/src/main/java/com/estore/checkout/ClosingService.java, backend/src/main/java/com/estore/checkout/ReservationScheduler.java, backend/src/main/java/com/estore/checkout/ReleaseTransaction.java; test backend/src/test/java/com/estore/checkout/ClosingIT.java.
**Interfaces:** Produces `cancel(UUID userId,UUID orderId):CheckoutView`, `closeExpired(Instant now):int`, `releaseIfProven(UUID orderId,NonpaymentEvidence):CloseResult`; provider proof comes from S05 adapter observations.

**Business rules and implementation boundary:**

Enter CLOSING under order guard, preserve closing reason, suppress new actions, and enqueue reconcile/cancel-authorization. Release only if every attempt is terminal nonpaying/no unresolved dispatch, or start never dispatched and durably suppressed. Transition HELD rows and grouped versioned restores/movements in one fresh retryable transaction. Scheduler every30s; never restore historical EXPIRED rows.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A ten-minute deadline passes while provider outcome is UNKNOWN | Enter CLOSING and retain HELD stock until terminal nonpayment is proved; elapsed time alone does not restore availability. |
| Two schedulers close the same conclusively unpaid hold of quantity 2 | Restore exactly 2 once; repeated expiry/cancellation restores zero more. |
| The owned order is already paid | Return 409 ORDER_ALREADY_PAID; do not route it through unpaid cancellation. |
| CREATE was never dispatched and is durably suppressed | Local no-payment proof can permit release. A missing provider ID or not-found query after possible dispatch cannot. |
| Capture was already dispatched before closing and later succeeds | Retained holds allow confirmation; do not release stock while capture is unresolved. |

**Required named cases from the source plan:** ownedCancel200or202_otherOwner404_paid409; repeatExpiry_restoresOnce; twoSchedulers_sameHoldsOnce; sharedProductRelease_aggregates; unknownCreate_keepsHold; queryNotFound_keepsHold; waitingWalletNotCancellable_keepsHold; lateAuthorization_closingCancels; captureVsClose_bothExecutionOrders_safe; releaseVersionConflict_retriesWholeFinalization; repeatedCancel_preservesFirstReason.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=ClosingIT verify)
```

**Explain before review:** What evidence is sufficient to restore stock, and why is a ten-minute hold not a ten-minute row lock?
**Reviewer checks:** Huy checks both race orders, all-attempt proof and whole-finalization retries; verifies only newly transitioned HELD rows generate restoration movements.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### C6 — Payment retry within the original hold

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R11, R12; S03, S05.
**Source task:** [C6 in the domain plan](../superpowers/plans/2026-10-07-checkout-pipeline.md).
**Business purpose:** Give customers a safe second payment attempt without recreating the purchase or extending its reservation.
**Prerequisites and handoffs:** C3/C4/C5 plus C2. Retry fixtures are agreed with Bảo/U5; retry command keys are distinct from the original checkout identity.
**Expected output:** Guarded retryPayment on the same owned order, durable retry-key replay and at most three sequential provider attempts with one unresolved attempt.
**Files and tests:** Create backend/src/main/java/com/estore/checkout/PaymentRetryService.java; test backend/src/test/java/com/estore/checkout/PaymentRetryIT.java.
**Interfaces:** Produces `retryPayment(UUID userId,UUID orderId,UUID retryKey):CheckoutView`; consumes order/attempt guards and durable QUERY/CREATE operations; at most3 sequential attempts.

**Business rules and implementation boundary:**

Persist per-order retry command identity. Reuse ready active action; query unknown state before deciding. Replace only conclusively failed/cancelled attempt before deadline, never while capture unresolved. Generate new provider merchantRef/requestId for a genuinely new terminal-replacement attempt; retain logical order/stock/amount/deadline.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| The customer closes the wallet popup and clicks retry | Reuse an eligible active attempt/action; stock, amount, order and original deadline are unchanged. |
| The active attempt has an uncertain outcome or unresolved capture | Schedule reconciliation rather than creating a replacement attempt. |
| Terminal nonpayment is established before the original deadline | Create a genuinely new provider attempt with new provider identity on the same order; allocate no additional stock. |
| Two retry keys arrive concurrently | At most one unresolved attempt is created; repeating the same retry key returns the same result. |
| The original window ended or three attempts already exist | Do not create another attempt or extend expiry. |

**Required named cases from the source plan:** popupClose_reusesActiveAttempt; uncertainAttempt_noReplacement; finalNonpayment_newAttemptSameOrder; duplicateRetryKey_oneReplacement; twoRetryKeys_onlyOneActiveAttempt; expiredWindow_noRetry; thirdAttemptCap; oldAttemptLateCallback_noWrongConfirmation; retry_preservesItemsAmountAndDeadline.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=PaymentRetryIT verify)
```

**Explain before review:** Why does a new provider attempt need a new provider identity while the purchase keeps its original order?
**Reviewer checks:** Huy checks attempt uniqueness, sequential cap, original snapshots/deadline and old-attempt callback safety; Bảo exercises the corresponding UI flow.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### P2 — Verification, recovery, MFA, and email outbox

**Owner:** Phat. **Reviewer:** Bảo. **Stage:** Production.
**Requirements:** R19, R23; S09, TRD9.
**Source task:** [P2 in the domain plan](../superpowers/plans/2026-10-08-production-launch.md).
**Business purpose:** Require verified customer identity and stronger admin authentication before operating with real customers.
**Prerequisites and handoffs:** D4 core evidence, F3/F7 and V4 coordination. Record approved sender/templates and verification/reset landing routes with Bảo; development uses a stub mailbox. Email-provider selection is an input, not permission for AI to choose one silently.
**Expected output:** V4 single-use verification/reset tokens, MFA and encrypted email outbox, plus account-security/reset UI. Phat owns the task; Bảo contributes/reviews its frontend files.
**Files and tests:** Create backend/src/main/resources/db/migration/V4__identity_operations.sql, backend/src/main/java/com/estore/auth/AccountRecoveryService.java, backend/src/main/java/com/estore/auth/MfaService.java, backend/src/main/java/com/estore/notification/EmailOutboxWorker.java, frontend/src/app/account/security/page.tsx, frontend/src/app/reset-password/page.tsx; test backend/src/test/java/com/estore/auth/ProductionIdentityIT.java and frontend/e2e/account-security.spec.ts.
**Interfaces:** Produces `verifyEmail(String token):void`, `resetPassword(String token,String password):void`, `challengeMfa(UUID sessionId,String code):MfaResult`, `enqueueMail(MailCommand):UUID`; MailCommand uses approved templates and recipient ID, no customer secrets in logs.

**Business rules and implementation boundary:**

Add hashed random verification/reset tokens and single-use atomic transitions, credentials-version JWT invalidation, encrypted MFA secret/hash recovery codes, live admin MFA checks and5m reauthentication/step-up. Transactional outbox stores encrypted expiring links; duplicate sends are tolerated but financial/account transitions stay idempotent. Configure dev mailbox/HTTP stub for tests and approved real email sender for launch; implement Vietnamese verification/reset/security UI and quota/CSRF controls.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A verification token is used after 24 hours or a reset token after 30 minutes | Reject with TOKEN_INVALID; simultaneous consumption permits only one success. |
| A password reset succeeds | Revoke all refresh sessions, advance credentialsVersion and reject already-issued old access tokens on private requests. |
| An unverified customer attempts live checkout or an admin lacks MFA | Return 403; sensitive admin mutations also require step-up no older than five minutes. |
| The email worker restarts or sends a duplicate message | The outbox remains recoverable; the account token stays single-use, and ordinary CI sends no real customer mail. |

**Required named cases from the source plan:** verification24hExpiryAndReplay; reset30mReplayOneSuccess; unknownForgotEmail_same202; reset_revokesAllSessionsAndCredentialsVersion; oldAccessTokenAfterReset_denied; simultaneousTokenConsume_oneSuccess; unverifiedCheckout403; adminWithoutMfa_denied; stale5mStepUp_deniedMutation; recoveryCodeSingleUse; TOTPReplayRejected; emailWorkerRestart_noLostMessage; realCustomerEmailNotSentInCI.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Bảo’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=ProductionIdentityIT verify)
(cd frontend && npm run test:e2e -- e2e/account-security.spec.ts)
```

Frontend E2E uses the runner established by D2 and the connected stack; backend success alone does not complete this cross-service task.

**Explain before review:** Why can password reset invalidate an old JWT immediately while ordinary logout follows the documented access-token TTL?
**Reviewer checks:** Bảo runs token/refresh/MFA flows against backend and named account-security E2E, checks routes/templates and verifies no secrets or live mail in tests.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### P4 — Returns, partial refunds, and inspected restock

**Owner:** Phat. **Reviewer:** Huy. **Stage:** Production.
**Requirements:** R21; S02, S09.
**Source task:** [P4 in the domain plan](../superpowers/plans/2026-10-08-production-launch.md).
**Business purpose:** Treat returns, refunds and physical stock receipt as separate auditable workflows.
**Prerequisites and handoffs:** D4, C3/C7, approved returns/support policy, P3 charge allocations and V4/V5 before V6. Coordinate the shipment-versus-cancellation contract with Bảo/P5; full acceptance needs that connected guard. Bảo helps on the named frontend files.
**Expected output:** V6 extension of core refunds, return/refund/restock services and UI, per-line/fee allocation ledger, proof-based refund recovery and inspection-controlled stock movements.
**Files and tests:** Create backend/src/main/resources/db/migration/V6__returns_refunds.sql; create backend/src/main/java/com/estore/returns/ReturnService.java, backend/src/main/java/com/estore/refund/RefundService.java, backend/src/main/java/com/estore/returns/RestockService.java, frontend/src/components/orders/ReturnRequest.tsx, frontend/src/components/admin/RefundReview.tsx; test backend/src/test/java/com/estore/commerce/ReturnsRefundsIT.java and frontend/e2e/returns.spec.ts.
**Interfaces:** Produces `requestReturn(UUID userId,UUID orderId,ReturnRequest,UUID key):ReturnView`, `requestRefund(UUID adminId,UUID orderId,RefundRequest,UUID key):RefundView`, `restock(UUID adminId,UUID returnId,RestockRequest,UUID key):RestockView`; consumes C3 provider refund/query and core compensation records.

**Business rules and implementation boundary:**

Implement separate return/refund/restock state machines with RETURN and PRE_SHIPMENT_CANCELLATION request kinds, order-guarded refundable balance reservation, exact per-line/fee allocations, stable provider refund merchantRef/requestId, and durable query recovery. Refund failure releases the local pending refundable allocation only when terminal nonrefund is established. A later operator retry is a new audited command referencing the failed one, with new key/provider identity and a fresh balance check; never replace an uncertain refund. Small residuals create audited support settlement cases. Restock needs receipt/inspection and unique movement; return/refund does not automatically restore stock.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Captured amount is 100,000 VND; pending refund is 60,000 and another request asks for 50,000 | Reject the excess: pending plus successful refunds cannot exceed capture or target allocations. |
| A refund is confirmed but no goods have been received/inspected | Availability does not change. |
| An inspection marks 2 returned units restockable and the restock command is replayed | Stock increases by exactly 2 once. |
| A refund times out or a residual is below 1,000 VND | Query uncertain refund state; route unsupported small residuals to audited support settlement without reporting false success. |
| A paid pre-shipment cancellation races with shipping | The shared order guard allows a consistent winner; unpaid cancel does not handle paid returns/refunds. |

**Required named cases from the source plan:** otherOwner404; eligibilityWindowPolicy; preShipmentCancellation_paidUnshippedOnly; duplicateReturn_noExcessQuantity; twoPartialRefunds_cappedIncludingPending; sameRefundKey_sameResult; below1000Residual_supportSettlementNoFalseSuccess; failedRefundRetry_queriesBeforeNewMutation; partialTaxFeeAllocations_exact; refundedButNotReceived_noRestock; inspectedReturn_restockOnce; duplicateRestockRace_noInflation; shippedOrderFullRefund_keepsFulfillmentHistory.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=ReturnsRefundsIT verify)
(cd frontend && npm run test:e2e -- e2e/returns.spec.ts)
```

Frontend E2E uses the runner established by D2 and the connected stack; backend success alone does not complete this cross-service task.

**Explain before review:** Why is refund completion not evidence that goods can be sold again?
**Reviewer checks:** Huy checks concurrent pending-balance reservations, exact tax/fee allocation, existing V3 refund compatibility, inspection evidence and duplicate restock races.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

## Completion evidence

Use the shared evidence template per task. All owned task checkboxes remain unchecked until implementation is verified. A reviewer’s approval concerns the recorded commit; later contract changes require matching producer/consumer checks. Production approvals come from the named merchant/business/operator owners.
