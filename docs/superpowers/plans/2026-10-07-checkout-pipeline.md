# Checkout and Payment Recovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Revision:** 2026-10-08; replaces the previous task bodies. Documentation work only; execution has not started.

**Goal:** Create one identified order/allocation and safely reconcile MoMo authorization, capture, cancellation, and compensation.

**Architecture:** Commit the full order/hold plus durable operations in one transaction. Coordinators retry short database attempts; workers call MoMo outside transactions and observations finalize under one order guard.

**Tech Stack:** Java/Spring, PostgreSQL/JPA, MoMo sandbox, durable jobs, JUnit/Testcontainers, HTTP provider stub

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

Requires foundation F1–F7 and builder B1. C1 owns V3__checkout.sql, including core compensation refund records. C2 produces allocation/order snapshots used by C3–C7. No Stripe SDK/client-secret dependency. MoMo merchant sandbox credentials are optional for offline tests and required for D3 sandbox smoke.

Paths below are proposed new application files; the current folder contains documentation only. Domain packages group controller/DTO/repository helpers with their owning service. The named service owns its transaction rules; split helpers by responsibility rather than building one application-wide service. Test paths and main deliverables are explicit in each task. Migration filenames are allocated by S02; do not create conflicting migration numbers in another plan.

## Review Focus

- Creation readiness/authorization must not be mistaken for captured payment — C3/C4.
- Same-key concurrent requests and lost database commit responses yield one allocation — C2.
- Capture/closing races cannot restore paid stock — C4/C5.
- Terminal failed attempts can be replaced without extending/allocating stock — C6.
- Late released-stock success/refund uncertainty requires durable compensation — C7.

## Tasks

### C1 — Checkout persistence and reviewed quotes

**Requirements:** R09, R11; S01–S04.
**Files:** Create backend/src/main/resources/db/migration/V3__checkout.sql, backend/src/main/java/com/estore/checkout/QuoteService.java, backend/src/main/java/com/estore/checkout/CheckoutController.java; test backend/src/test/java/com/estore/checkout/QuoteIT.java.
**Interfaces:** Consumes F5 CartSnapshot and F4 product data; produces `quote(UUID userId,QuoteRequest):QuoteView`, persisted quote snapshot/fingerprint, core order/attempt/operation/refund entities.

- [ ] Step 1: Write failing tests in the stated test file: quote_exactVndTotal; quoteDoesNotReserve; foreignQuote404; priceChange409_beforeHold; expiredQuote409; cartChange409; policyChange409; providerAmountOutside1000To50000000_noHold; groupSnapshotPreserved. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(quote.totalMinor()).isEqualTo(19990L * 3);
assertThat(availableAfterQuote).isEqualTo(availableBeforeQuote);
assertThat(outOfRangeAllocationCount).isZero();
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=QuoteIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement owned five-minute quotes under cart/shared-catalog guard, exact subtotal/shipping/tax totals, per-item catalog versions and cart/calculation fingerprint. Reject unsupported MoMo totals before reservations. Migrate all S02 checkout constraints including one unresolved provider attempt and compensation refunds.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=QuoteIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: checkout persistence and reviewed quotes`. Record test command/results in the task execution notes.

### C2 — Atomic allocation and replay identity

**Requirements:** R10, R11; S04.
**Files:** Create backend/src/main/java/com/estore/checkout/CheckoutCoordinator.java, backend/src/main/java/com/estore/checkout/AllocationTransaction.java, backend/src/main/java/com/estore/checkout/CheckoutRequestRepository.java; test backend/src/test/java/com/estore/checkout/AllocationIT.java.
**Interfaces:** Produces `initiate(UUID userId,CheckoutRequest request,UUID key):CheckoutView`, `allocate(UUID userId,CheckoutRequest,UUID key):AllocatedOrder`; coordinator is nontransactional and allocation bean separately proxied.

- [ ] Step 1: Write failing tests in the stated test file: hold_decrementsOnce; multiItemShortage_noOrderNoHolds; reversedProductOrder_noDeadlock; duplicateProductAcrossGroups_aggregatesStock; sameKeyConcurrent_oneOrder; sameKeyDifferentQuote409; replayAfterQuoteExpiry_sameOrder; commitTimeConflict_freshRetry; stockStillAvailableConflict_succeeds; thirdConflict409Busy; ambiguousCommit_lookupOriginal. For 50 last-unit buyers exactly one succeeds, and ample stock succeeds for all independent healthy attempts. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(lastUnit.reservationSuccesses()).isEqualTo(1);
assertThat(sameKey.orderIds()).containsOnly(originalOrderId);
assertThat(shortage.persistedOrders()).isZero();
assertThat(retryExhaustion.code()).isEqualTo("INVENTORY_BUSY");
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=AllocationIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Use fresh backend inventory versions, deterministic ID order, whole-order transaction for snapshots/holds/identity/first CREATE job. Replays resolve before quote checks. Catch flush/commit concurrency errors outside transaction and cap3 attempts. Recover unknown commits by identity lookup, never provider/network retries inside allocation.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=AllocationIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: atomic allocation and replay identity`. Record test command/results in the task execution notes.

### C3 — MoMo adapter and durable worker dispatch

**Requirements:** R12, R19; S05.
**Files:** Create backend/src/main/java/com/estore/payment/PaymentGateway.java, backend/src/main/java/com/estore/payment/momo/MomoGateway.java, backend/src/main/java/com/estore/payment/PaymentWorker.java, backend/src/main/java/com/estore/payment/WorkerClaimTransaction.java, contracts/fixtures/momo/; test backend/src/test/java/com/estore/payment/MomoGatewayTest.java and backend/src/test/java/com/estore/payment/PaymentWorkerIT.java.
**Interfaces:** Produces PaymentGateway S05 interface and `runReadyOperations(Instant now,int limit):WorkerRunResult`; consumes persisted immutable command identities. Gateway START/QUERY/CAPTURE/CANCEL_AUTHORIZATION/REFUND/QUERY_REFUND map to MoMo protocol contexts.

- [ ] Step 1: Write failing tests in the stated test file: knownSignatureVectors_pass; alteredAmountSignature_fail; create0_returnsReadyNotPaid; authorized9000_notCaptured; decimalVndReject; unknownPayUrlHostReject; timeoutAfterCreate_reusesRequestId; crashAfterProviderBeforeAssociation_recovers; expiredLease_twoWorkers_oneEffect; neverDispatchedClosing_suppressesCreate; oldReplayBeyond30Days_review. 35s overall provider bound and no DB transaction during request are asserted. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(createSuccess.orderPaid()).isFalse();
assertThat(authorization.normalizedState()).isEqualTo(AUTHORIZED);
assertThat(allRetriedCreates.requestIds()).containsOnly(originalRequestId);
assertThat(providerCall.transactionActive()).isFalse();
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dtest=MomoGatewayTest -Dit.test=PaymentWorkerIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement operation-specific HMAC vectors with synthetic keys and immutable payloads. Use sandbox captureWallet/autoCapture=false/VND, hosted URL allowlist, context-aware states, encrypted action at rest. Claim order then operation with60s lease, call outside transaction max35s, persist result under guard. Stable request IDs, provider replay horizon and bounded retry/query cadence follow S05.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dtest=MomoGatewayTest -Dit.test=PaymentWorkerIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: momo adapter and durable worker dispatch`. Record test command/results in the task execution notes.

### C4 — Signed notifications and capture finalization

**Requirements:** R10, R12; S05.
**Files:** Create backend/src/main/java/com/estore/payment/MomoIpnController.java, backend/src/main/java/com/estore/payment/PaymentObservationService.java, backend/src/main/java/com/estore/payment/LifecycleTransaction.java; test backend/src/test/java/com/estore/payment/PaymentObservationIT.java.
**Interfaces:** Produces `observe(PaymentObservation):ObservationResult`, `acceptIpn(byte[] body):void`; consumes C3 VerifiedNotification; authoritative query and IPN share order guard/event receipt persistence.

- [ ] Step 1: Write failing tests in the stated test file: invalidSignature400; wrongMerchantOrAmount_quarantine; notification204AfterCommit; failedCommit_noReceiptAnd503; sameEventParallel_oneEffect; equivalentDifferentEvents_oneConfirmation; authorize_open_enqueuesCapture; authorize_closing_enqueuesCancel; captured_confirmsNoSecondDeduction; successThenOldFailure_noDowngrade; captureResultCrash_recovers; captureAfterDeadlineWithRetainedHolds_confirms. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(stockAfterCapture).isEqualTo(stockAfterHold);
assertThat(confirmedTransitions).isEqualTo(1);
assertThat(failedProcessing.receiptExists()).isFalse();
assertThat(authorizedReceiptShowsPaid).isFalse();
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=PaymentObservationIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Verify original notification representation/limits and persist receipt plus transitions/jobs atomically. One bounded IPN transaction/ten-second application budget; on failure return503 for redelivery, not a retry loop exceeding acknowledgement deadline. Code0 on CREATE is not payment; code9000 in two-step is authorization. Serialize capture/cancel dispatch, confirm only verified captured payment against retained holds; immutable paid attempt identity prevents old callbacks from changing a newer attempt/order.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=PaymentObservationIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: signed notifications and capture finalization`. Record test command/results in the task execution notes.

### C5 — Safe expiry and owned cancellation

**Requirements:** R12; S04, S05.
**Files:** Create backend/src/main/java/com/estore/checkout/ClosingService.java, backend/src/main/java/com/estore/checkout/ReservationScheduler.java, backend/src/main/java/com/estore/checkout/ReleaseTransaction.java; test backend/src/test/java/com/estore/checkout/ClosingIT.java.
**Interfaces:** Produces `cancel(UUID userId,UUID orderId):CheckoutView`, `closeExpired(Instant now):int`, `releaseIfProven(UUID orderId,NonpaymentEvidence):CloseResult`; provider proof comes from S05 adapter observations.

- [ ] Step 1: Write failing tests in the stated test file: ownedCancel200or202_otherOwner404_paid409; repeatExpiry_restoresOnce; twoSchedulers_sameHoldsOnce; sharedProductRelease_aggregates; unknownCreate_keepsHold; queryNotFound_keepsHold; waitingWalletNotCancellable_keepsHold; lateAuthorization_closingCancels; captureVsClose_bothExecutionOrders_safe; releaseVersionConflict_retriesWholeFinalization; repeatedCancel_preservesFirstReason. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(secondExpiry.stockDelta()).isZero();
assertThat(ambiguousPayment.reservations()).allMatch(r -> r.status() == HELD);
assertThat(paidCancel.code()).isEqualTo("ORDER_ALREADY_PAID");
assertThat(twoWorkers.totalRestored()).isEqualTo(heldQuantity);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=ClosingIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Enter CLOSING under order guard, preserve closing reason, suppress new actions, and enqueue reconcile/cancel-authorization. Release only if every attempt is terminal nonpaying/no unresolved dispatch, or start never dispatched and durably suppressed. Transition HELD rows and grouped versioned restores/movements in one fresh retryable transaction. Scheduler every30s; never restore historical EXPIRED rows.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=ClosingIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: safe expiry and owned cancellation`. Record test command/results in the task execution notes.

### C6 — Payment retry within the original hold

**Requirements:** R11, R12; S03, S05.
**Files:** Create backend/src/main/java/com/estore/checkout/PaymentRetryService.java; test backend/src/test/java/com/estore/checkout/PaymentRetryIT.java.
**Interfaces:** Produces `retryPayment(UUID userId,UUID orderId,UUID retryKey):CheckoutView`; consumes order/attempt guards and durable QUERY/CREATE operations; at most3 sequential attempts.

- [ ] Step 1: Write failing tests in the stated test file: popupClose_reusesActiveAttempt; uncertainAttempt_noReplacement; finalNonpayment_newAttemptSameOrder; duplicateRetryKey_oneReplacement; twoRetryKeys_onlyOneActiveAttempt; expiredWindow_noRetry; thirdAttemptCap; oldAttemptLateCallback_noWrongConfirmation; retry_preservesItemsAmountAndDeadline. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(retry.orderId()).isEqualTo(originalOrderId);
assertThat(retry.reservationExpiresAt()).isEqualTo(originalDeadline);
assertThat(stockAfterRetry).isEqualTo(stockAfterHold);
assertThat(nonterminalAttempts).hasSize(1);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=PaymentRetryIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Persist per-order retry command identity. Reuse ready active action; query unknown state before deciding. Replace only conclusively failed/cancelled attempt before deadline, never while capture unresolved. Generate new provider merchantRef/requestId for a genuinely new terminal-replacement attempt; retain logical order/stock/amount/deadline.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=PaymentRetryIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: payment retry within the original hold`. Record test command/results in the task execution notes.

### C7 — Order receipts, fulfillment, and compensation

**Requirements:** R12, R13, R14; S03–S05.
**Files:** Create backend/src/main/java/com/estore/order/OrderService.java, backend/src/main/java/com/estore/order/OrderController.java, backend/src/main/java/com/estore/payment/CompensationService.java, backend/src/main/java/com/estore/order/AdminOrderController.java; test backend/src/test/java/com/estore/order/OrderLifecycleIT.java.
**Interfaces:** Produces `getOrder(UUID userId,UUID orderId):OrderView`, `fulfill(UUID adminId,UUID orderId,long expectedVersion,UUID key):OrderView`, `queueCompensation(UUID attemptId,String reason):UUID`; uses F5 cleanup and C3 refund/query worker.

- [ ] Step 1: Write failing tests in the stated test file: otherOwner404; catalogEdit_receiptUnchanged; confirm_cleanupOnlyUnchangedCartEntries; duplicateFulfill_once_noStockDelta; paymentException_blocksFulfillment; capturedAfterRelease_reviewNotPaidAndOneRefund; extraAttemptCapture_refundExtraKeepPaidOrder; refundTimeout_pendingThenQuery; compensationSuccess_setsRefundStatusNoRestock; adminView_masksPii. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(lateCaptured.status()).isEqualTo(PAYMENT_REVIEW_REQUIRED);
assertThat(compensationReplay.refundCount()).isEqualTo(1);
assertThat(stockAfterRefund).isEqualTo(stockBeforeRefund);
assertThat(newCartEntryStillPresent).isTrue();
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=OrderLifecycleIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement owned history/immutable receipt/admin fulfillment and review queue. Confirm cleanup exactly once under cart/order guard. Compensation uses one durable refund identity, captured transaction reference and guarded amount; confirm refund only from provider evidence, never restock automatically. Core compensation refund table exists in V3; production P4 adds ordinary return/allocation workflows.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=OrderLifecycleIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: order receipts, fulfillment, and compensation`. Record test command/results in the task execution notes.

## Completion evidence

Record actual commands, discovered test counts, results, and any failed cases. Run the component's full verification once after its tasks pass; broader integration belongs to the delivery plan. Keep the scope/contract changes synchronized with the specification. A provider stub proves application behavior under a modeled protocol; the sandbox and live gates require separate evidence.
