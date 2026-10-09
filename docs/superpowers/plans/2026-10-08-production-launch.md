# Production Commerce and Launch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Revision:** 2026-10-08; replaces the previous task bodies. Documentation work only; execution has not started.

**Goal:** Extend the integrated core into a Vietnam/VND MoMo application with secure accounts, correct commerce/refunds, and rehearsed operations.

**Architecture:** Keep the modular monolith and durable PostgreSQL operations; add versioned commerce policies, email outbox, MFA, refund/return records, private deployment and operational launch gates. Live payments are enabled only after evidence and merchant capabilities match the design.

**Tech Stack:** Core stack, MoMo live merchant account, transactional email provider, MFA, HTTPS ingress, managed PostgreSQL/backups, operational monitoring

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

Requires D4 integrated core evidence. P1 records merchant/operator/configuration inputs and creates fail-closed launch guard. P2 owns V4__identity_operations.sql; P3 owns V5__commerce_policy.sql, P4 owns V6__returns_refunds.sql, and P5 owns V7__fulfillment_privacy.sql; one coordinator sequences them without modifying applied migrations. Production hosting, provider account access, legal/tax reference cases, fulfillment/returns policies and operator names are required inputs before enabling live mode. Do not claim these have been supplied.

Paths below are proposed new application files; the current folder contains documentation only. Domain packages group controller/DTO/repository helpers with their owning service. The named service owns its transaction rules; split helpers by responsibility rather than building one application-wide service. Test paths and main deliverables are explicit in each task. Migration filenames are allocated by S02; do not create conflicting migration numbers in another plan.

## Review Focus

- Authorize/capture capability and wallet limits must match merchant account configuration — P1.
- Password reset/verification/MFA cannot replay or leak account existence — P2.
- Included tax must not be added to a total twice; unsupported zones fail before payment — P3.
- Concurrent partial refunds/residuals/restocks cannot exceed captured/refundable/received quantities — P4.
- Restore/deployment must reconcile provider effects before mutating durable jobs resume — P6/P7.

## Tasks

### P1 — Merchant readiness and live-mode guard

**Requirements:** R19, R24; S05, S09.
**Files:** Create docs/decisions/momo-live-capabilities.md, docs/operations/launch-inputs.md, backend/src/main/java/com/estore/config/LaunchReadiness.java; test backend/src/test/java/com/estore/config/LaunchReadinessIT.java.
**Interfaces:** Produces `validateLiveConfiguration(LaunchConfiguration):LaunchReadinessResult`, rejecting live startup unless required reviewed configuration/evidence is present; consumes S05 MoMo adapter.

- [ ] Step 1: Write failing tests in the stated test file: noMerchantApproval_liveDisabled; sandboxHostWithLiveMode_rejected; unavailableTwoStepCapability_rejected; missingProviderExpiryRetryStateReview_rejected; missingOperationsOwners_rejected; unsupportedWalletAmount_blocked; approvedConfig_hasNoSecretsInEvidence; testProfileCannotDispatchLive. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(missingMerchantConfiguration.liveEnabled()).isFalse();
assertThat(mixedEnvironment.startupAllowed()).isFalse();
assertThat(unsupportedCapturePolicy.startupAllowed()).isFalse();
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=LaunchReadinessIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Record merchant approval, autoCapture=false eligibility, hosted-return/IPN endpoints, state/query/replay/expiry/refund behavior, allowed methods/limits, operator contacts, and secret references. Run sandbox capability cases first. Enable only MoMo wallet methods whose terminal-state guarantees fit S05; no assumed waiting-wallet cancel or clock-based release. Future VNPAY requires its own adapter plan.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=LaunchReadinessIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: merchant readiness and live-mode guard`. Record test command/results in the task execution notes.

### P2 — Verification, recovery, MFA, and email outbox

**Requirements:** R19, R23; S09, TRD9.
**Files:** Create backend/src/main/resources/db/migration/V4__identity_operations.sql, backend/src/main/java/com/estore/auth/AccountRecoveryService.java, backend/src/main/java/com/estore/auth/MfaService.java, backend/src/main/java/com/estore/notification/EmailOutboxWorker.java, frontend/src/app/account/security/page.tsx, frontend/src/app/reset-password/page.tsx; test backend/src/test/java/com/estore/auth/ProductionIdentityIT.java and frontend/e2e/account-security.spec.ts.
**Interfaces:** Produces `verifyEmail(String token):void`, `resetPassword(String token,String password):void`, `challengeMfa(UUID sessionId,String code):MfaResult`, `enqueueMail(MailCommand):UUID`; MailCommand uses approved templates and recipient ID, no customer secrets in logs.

- [ ] Step 1: Write failing tests in the stated test file: verification24hExpiryAndReplay; reset30mReplayOneSuccess; unknownForgotEmail_same202; reset_revokesAllSessionsAndCredentialsVersion; oldAccessTokenAfterReset_denied; simultaneousTokenConsume_oneSuccess; unverifiedCheckout403; adminWithoutMfa_denied; stale5mStepUp_deniedMutation; recoveryCodeSingleUse; TOTPReplayRejected; emailWorkerRestart_noLostMessage; realCustomerEmailNotSentInCI. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(resetSecondUse.code()).isEqualTo("TOKEN_INVALID");
assertThat(unverifiedCheckout.status()).isEqualTo(403);
assertThat(adminWithoutMfa.status()).isEqualTo(403);
assertThat(resetRevokedSessions).isEqualTo(allSessionsBeforeReset);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=ProductionIdentityIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Add hashed random verification/reset tokens and single-use atomic transitions, credentials-version JWT invalidation, encrypted MFA secret/hash recovery codes, live admin MFA checks and5m reauthentication/step-up. Transactional outbox stores encrypted expiring links; duplicate sends are tolerated but financial/account transitions stay idempotent. Configure dev mailbox/HTTP stub for tests and approved real email sender for launch; implement Vietnamese verification/reset/security UI and quota/CSRF controls.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=ProductionIdentityIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: verification, recovery, mfa, and email outbox`. Record test command/results in the task execution notes.

### P3 — Versioned Vietnamese delivery and tax totals

**Requirements:** R09, R20; S09.
**Files:** Create backend/src/main/resources/db/migration/V5__commerce_policy.sql, backend/src/main/java/com/estore/commerce/CommercePolicyService.java, backend/src/main/java/com/estore/commerce/QuoteCalculator.java, frontend/src/components/admin/CommercePolicyEditor.tsx, contracts/fixtures/commerce-vn.json; test backend/src/test/java/com/estore/commerce/CommerceTotalsIT.java.
**Interfaces:** Produces `calculate(CartPriceSnapshot,ShippingAddress,CommercePolicy):QuoteAmounts`, QuoteAmounts=(subtotalMinor,shippingMinor,taxMinor,additionalTaxMinor,totalMinor,currency,calculationVersion,lineBreakdown); consumes approved jurisdiction examples and versioned address/zone data.

- [ ] Step 1: Write failing tests in the stated test file: inclusiveTax_notAddedTwice; exclusiveTax_addedOnce; wholeDongHalfUpBoundary; shippingTaxClassified; mixedRates_residualsExact; missingTaxApproval_liveBlocked; unknownProvinceLocalityOrZone_rejected; policyEditForcesQuoteReview; historicalOrderCalculationUnchanged; providerMaximumAfterFees_rejectedBeforeHold. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(inclusive.totalMinor()).isEqualTo(inclusive.subtotalMinor() + inclusive.shippingMinor());
assertThat(exclusive.totalMinor()).isEqualTo(exclusive.subtotalMinor() + exclusive.shippingMinor() + exclusive.additionalTaxMinor());
assertThat(lineChargesPlusShipping).isEqualTo(order.totalMinor());
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=CommerceTotalsIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement immutable versioned flat-rate zones/current approved address dataset, policy classifications and exact rational/BigDecimal rounding. Store included-tax disclosure separately from additional payable tax; total=subtotal+shipping+additionalTax. Approval/effective dates/reference cases are required configuration, not guessed VAT. Extend quote/order snapshots and UI fee disclosures; changed policy forces re-review before hold.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=CommerceTotalsIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: versioned vietnamese delivery and tax totals`. Record test command/results in the task execution notes.

### P4 — Returns, partial refunds, and inspected restock

**Requirements:** R21; S02, S09.
**Files:** Create backend/src/main/resources/db/migration/V6__returns_refunds.sql; create backend/src/main/java/com/estore/returns/ReturnService.java, backend/src/main/java/com/estore/refund/RefundService.java, backend/src/main/java/com/estore/returns/RestockService.java, frontend/src/components/orders/ReturnRequest.tsx, frontend/src/components/admin/RefundReview.tsx; test backend/src/test/java/com/estore/commerce/ReturnsRefundsIT.java and frontend/e2e/returns.spec.ts.
**Interfaces:** Produces `requestReturn(UUID userId,UUID orderId,ReturnRequest,UUID key):ReturnView`, `requestRefund(UUID adminId,UUID orderId,RefundRequest,UUID key):RefundView`, `restock(UUID adminId,UUID returnId,RestockRequest,UUID key):RestockView`; consumes C3 provider refund/query and core compensation records.

- [ ] Step 1: Write failing tests in the stated test file: otherOwner404; eligibilityWindowPolicy; preShipmentCancellation_paidUnshippedOnly; duplicateReturn_noExcessQuantity; twoPartialRefunds_cappedIncludingPending; sameRefundKey_sameResult; below1000Residual_supportSettlementNoFalseSuccess; failedRefundRetry_queriesBeforeNewMutation; partialTaxFeeAllocations_exact; refundedButNotReceived_noRestock; inspectedReturn_restockOnce; duplicateRestockRace_noInflation; shippedOrderFullRefund_keepsFulfillmentHistory. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(successfulPlusPendingRefundAmount).isLessThanOrEqualTo(capturedAmount);
assertThat(stockAfterRefundOnly).isEqualTo(stockBeforeRefund);
assertThat(replayedRestock.totalDelta()).isEqualTo(receivedRestockableQuantity);
assertThat(unresolvedRefund.status()).isNotEqualTo(SUCCEEDED);
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=ReturnsRefundsIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement separate return/refund/restock state machines with RETURN and PRE_SHIPMENT_CANCELLATION request kinds, order-guarded refundable balance reservation, exact per-line/fee allocations, stable provider refund merchantRef/requestId, and durable query recovery. Refund failure releases the local pending refundable allocation only when terminal nonrefund is established. A later operator retry is a new audited command referencing the failed one, with new key/provider identity and a fresh balance check; never replace an uncertain refund. Small residuals create audited support settlement cases. Restock needs receipt/inspection and unique movement; return/refund does not automatically restore stock.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=ReturnsRefundsIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: returns, partial refunds, and inspected restock`. Record test command/results in the task execution notes.

### P5 — Shipment, customer policies, and governance

**Requirements:** R21, R23; S09.
**Files:** Create backend/src/main/resources/db/migration/V7__fulfillment_privacy.sql; create backend/src/main/java/com/estore/fulfillment/ShipmentService.java, backend/src/main/java/com/estore/privacy/CustomerDataService.java, frontend/src/app/help/page.tsx, frontend/src/app/policies/[policy]/page.tsx, frontend/src/components/orders/ShipmentStatus.tsx, docs/operations/data-retention.md; test backend/src/test/java/com/estore/commerce/FulfillmentGovernanceIT.java.
**Interfaces:** Produces `ship(UUID adminId,UUID orderId,ShipmentRequest,UUID key):ShipmentView`, `requestDataAction(UUID userId,DataActionRequest):DataActionView`; action=(EXPORT/DELETE), returns requestId/status/retentionExplanation; customer support policies versioned/visible.

- [ ] Step 1: Write failing tests in the stated test file: onlyPaidUnblockedShips; duplicateShipment_once; shippingUrlAllowlist; paidUnfulfilledCancel_entersRefundWorkflow; cancelVsShip_guardOnlyOneWins; deliveredTimestampStartsReturnWindow; customerCannotEditTracking; privacyExportOnlyOwnData; deletionPreservesRequiredTransactionRecordsWithExplanation; piiMaskedLogsAndAdminViews; optimizerNoPii; requiredPolicyPagesPresent; transactionalStatusEmailOutbox. Pin these assertions using the scenario fixtures, not mocked versions of the behavior under test:

```java
assertThat(unpaidShip.status()).isEqualTo(409);
assertThat(shipmentReplay.count()).isEqualTo(1);
assertThat(privacyExport.containsOtherUserData()).isFalse();
assertThat(piiInOptimizerPayload).isFalse();
```

- [ ] Step 2: Run `cd backend && ./mvnw -Dit.test=FulfillmentGovernanceIT verify`. Expected: named behavior tests fail for the missing behavior; repair fixture/build failures before treating this as the red step.
- [ ] Step 3: Implement one whole-order manual shipment/tracking and delivery confirmation with idempotency/audit, separate fulfillment history, and order-guarded pre-shipment cancellation blocking, status emails and support contact/policy pages. Record Vietnamese market-reviewed shipping/returns/privacy/terms and data retention/deletion exceptions with owners. Data deletion/export jobs are owned/authenticated, redacted and auditable; no blind erasure of unsettled financial records.
- [ ] Step 4: Rerun `cd backend && ./mvnw -Dit.test=FulfillmentGovernanceIT verify`. Expected: all task cases pass, with real PostgreSQL/Redis where specified and no skipped substitutes for the concurrency checks.
- [ ] Step 5: Review the diff and commit only this task's files with message `feat: shipment, customer policies, and governance`. Record test command/results in the task execution notes.

### P6 — Secure deployment and recovery rehearsal

**Requirements:** R22, R23; S09.
**Files:** Create deploy/compose.production.yaml, deploy/ingress.conf, scripts/backup.sh, scripts/restore-rehearsal.sh, scripts/deploy-smoke.sh, docs/runbooks/deploy.md, docs/runbooks/payments.md, docs/runbooks/restore.md, docs/operations/alerts.md.
**Interfaces:** Produces isolated production topology, readiness/metrics, encrypted backup/restore scripts and audited release/runbook evidence; consumes named hosting/secret/monitoring configuration.

- [ ] Step 1: Bootstrap the declared build/test tooling before invoking its commands; commit wrapper/lockfile support and example configuration without secrets.
- [ ] Step 2: Add the smoke checks: noPublicDbRedisOptimizerPorts; httpsOnlyAndSecureCookies; missingSecretFailClosed; readinessDegradedDependencies; failedBackupAlerts; agedClosing15mAlerts; jobAge5mAlerts; restoreMeetsAcceptedRpoRto; restartWorkersReconcile; appRollbackSchemaCompatible; restoredSnapshotJobsPausedUntilProviderReconciled; operatorIncidentExercise.
- [ ] Step 3: Configure HTTPS/private networking, managed secret rotation, least-privilege DB role, managed backups/PITR, redacted structured logs and metrics/alerts with real assigned receivers. Rehearse proposed15mRPO/4hRTO or record approved alternatives; test schema-compatible rollback/forward repair. Restore keeps mutating jobs paused until live provider state is reconciled. Include capacity/security scans and payment/refund/incident support runbooks.
- [ ] Step 4: Run `bash scripts/deploy-smoke.sh && bash scripts/restore-rehearsal.sh`. Expected: exit 0, declared checks execute, no missing test runner or ignored integration suite.
- [ ] Step 5: Review the diff and commit only this task's files with message `build: secure deployment and recovery rehearsal`. Initialize source control at implementation start if this folder is still not a repository.

### P7 — Production evidence and controlled launch

**Requirements:** R19–R24; S09, S10.
**Files:** Create docs/evidence/production-launch.md, scripts/verify-launch-gates.sh, tests/load/production-capacity.js, docs/runbooks/live-commissioning.md.
**Interfaces:** Produces owner-signed gate ledger for every R19–R24 acceptance case, approved objectives/capacity/market/provider configuration, and limited live commissioning/rollout record.

- [ ] Step 1: Bootstrap the declared build/test tooling before invoking its commands; commit wrapper/lockfile support and example configuration without secrets.
- [ ] Step 2: Add the smoke checks: criticalFindingBlocksLaunch; missingRestoreOrPolicyEvidenceBlocks; providerTwoStepCapabilityVerified; securityReviewRecorded; representativePlentifulAndScarceLoad; controlledLiveAuthorizeCaptureCancelRefundReconcile; alertReceiverAndOperatorConfirmed; rollbackRehearsed; noLiveTransactionsInOrdinaryCI.
- [ ] Step 3: Collect P1–P6 evidence and actual capacity/objectives, resolve critical findings, and validate all launch input documents. Execute controlled live commissioning only with the merchant/operator-approved account, bounded test purchases/refunds and designated owner present; ordinary verification script checks evidence without moving money. Limited rollout monitors reconciliation/aged holds/errors and records recovery/escalation thresholds. Do not mark launch complete from sandbox/demo success.
- [ ] Step 4: Run `bash scripts/verify-launch-gates.sh`. Expected: exit 0, declared checks execute, no missing test runner or ignored integration suite.
- [ ] Step 5: Review the diff and commit only this task's files with message `build: production evidence and controlled launch`. Initialize source control at implementation start if this folder is still not a repository.

## Completion evidence

Record actual commands, discovered test counts, results, and any failed cases. Run the component's full verification once after its tasks pass; broader integration belongs to the delivery plan. Keep the scope/contract changes synchronized with the specification. A provider stub proves application behavior under a modeled protocol; the sandbox and live gates require separate evidence.
