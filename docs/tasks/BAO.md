# Bảo — Individual Task Implementation Plan

> **For agentic workers:** Use `superpowers:executing-plans` when implementing these tasks yourself; use `superpowers:subagent-driven-development` only when delegation is explicitly selected. The shared workflow below applies to every task.

**Revision:** 2026-10-09. Task instructions only; no implementation or test execution is claimed.
**Owner:** Bảo. **Owned tasks:** 13. Reviewers are specified on each card.
**Goal:** Deliver faithful customer/admin journeys and coordinate reproducible integration and production operations.
**Architecture:** React consumes versioned backend contracts; browser state never authorizes stock, payment or fulfillment. Integration checks use the real application stack and distinguish provider stubs, sandbox and live evidence.
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
| U1 | Core | Phat |
| U2 | Core | Huy |
| U3 | Core | Huy |
| U4 | Core | Phat |
| U5 | Core | Phat |
| U6 | Core | Huy |
| D1 | Core | Huy |
| D2 | Core | Phat |
| D4 | Core | Huy |
| P1 | Production | Phat |
| P5 | Production | Huy |
| P6 | Production | Phat |
| P7 | Production | Huy |

## Task cards

### U1 — Frontend runtime and session client

**Owner:** Bảo. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R02, R15, R17; S03, S08.
**Source task:** [U1 in the domain plan](../superpowers/plans/2026-10-07-frontend.md).
**Business purpose:** Make authenticated frontend requests recover from concurrent expiry without leaking tokens or losing the customer’s destination.
**Prerequisites and handoffs:** Can scaffold now with S03 fixtures. Auth slice/fixtures are reviewed with Phat/F3; real acceptance needs F3/F7. Publish early contract fixtures under D2 instead of inventing DTO shapes.
**Expected output:** Next runtime/lock/test tooling, api/ensureAccessToken/safeReturnPath, real /login and /register routes and generated contract types.
**Files and tests:** Create frontend/package.json, frontend/package-lock.json, frontend/tsconfig.json, frontend/vitest.config.ts, frontend/src/lib/api.ts, frontend/src/lib/auth.ts, frontend/src/app/(auth)/login/page.tsx, frontend/src/app/(auth)/register/page.tsx; test frontend/src/lib/auth.test.ts.
**Interfaces:** Produces `api<T>(path:string,options:ApiOptions):Promise<T>`, `ensureAccessToken():Promise<string>`, `safeReturnPath(input:string):string`; generated types live frontend/src/lib/contracts.ts; auth memory only.

**Business rules and implementation boundary:**

Bootstrap Next16/Node24/test environment before invoking tests. Implement same-origin API/CSRF/cookie policy, memory access-token queue and Web Locks, thirty-second same refresh ID recovery. Generate DTO types from contracts; correct route groups; no refresh token in response body/storage.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Several requests receive 401 in one tab | Use one refresh promise; use Web Locks to coordinate tabs. |
| Refresh response was lost or another tab receives REFRESH_IN_PROGRESS | Reuse the same refresh request ID for recovery or wait/refetch without overwriting another tab’s cookie. |
| A login return path is //evil.example or tokens are inspected in storage | Reject unsafe destination and keep access tokens only in memory; no refresh token JSON/storage. |

**Required named cases from the source plan:** parallel401_oneRefreshPerTab; webLock_serializesTabs; refresh409_waitsWithoutCookieOverwrite; lostRefreshResponse_sameRequestIdReplay; unsafeReturnPathRejected; /loginAndRegister_resolve; noTokensInStorage; logout_clearsMemoryAndCookieSession.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd frontend && npm test -- --run src/lib/auth.test.ts)
```

**Explain before review:** Why does disabling a button not solve cross-tab authentication concurrency?
**Reviewer checks:** Phat checks backend cookie/CSRF parity and browser refresh tests against F3, including route paths and memory-only tokens.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### U2 — Catalog discovery and product views

**Owner:** Bảo. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R01, R16; S03, S08.
**Source task:** [U2 in the domain plan](../superpowers/plans/2026-10-07-frontend.md).
**Business purpose:** Help shoppers find active products with current stock and understandable loading/error states.
**Prerequisites and handoffs:** U1 scaffold/client; can start with reviewed F4/S03 fixtures. Connected completion needs Huy/F4 categories/products and real query semantics.
**Expected output:** Product list/detail/filter views with URL-backed queries, exact VND formatting and accessible mobile/keyboard states.
**Files and tests:** Create frontend/src/app/products/page.tsx, frontend/src/app/products/[id]/page.tsx, frontend/src/components/catalog/ProductList.tsx, frontend/src/components/catalog/ProductFilters.tsx; test frontend/src/components/catalog/ProductList.test.tsx.
**Interfaces:** Consumes ProductView/page/category tree via api<T>; produces URL-backed search/filter/page controls and exact VND product display.

**Business rules and implementation boundary:**

Implement catalog/search/details from live availability with debounced search and stable query navigation. Reject invalid UI query inputs consistently; render loading/empty/error states, labels/focus/mobile layout, and availability caveat.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Customer chooses page 2 in a human-facing control | Translate consistently to the zero-based API page contract and preserve filters in the URL. |
| Backend returns an empty page, inactive 404 or network error | Show distinct useful states rather than a false out-of-stock result. |
| Viewport is 360 px or an image fails | No horizontal overflow; labeled keyboard controls and an accessible image fallback remain usable. |

**Required named cases from the source plan:** filterAndPageUrls_matchZeroBasedContract; priceFormattedAsDong_noCents; inactive404; shortageDisplaysRequestedVsAvailable; emptyErrorLoadingStates; keyboardFilters; mobile360NoHorizontalOverflow; fallbackImageAccessible.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd frontend && npm test -- --run src/components/catalog/ProductList.test.tsx)
```

**Explain before review:** Why is displayed availability only informational until checkout allocation?
**Reviewer checks:** Huy compares UI query/stock behavior with F4 and checks stable navigation, exact currency and accessible states.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### U3 — Manual builder and draft revisions

**Owner:** Bảo. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R04, R05, R06, R07; S06–S08.
**Source task:** [U3 in the domain plan](../superpowers/plans/2026-10-07-frontend.md).
**Business purpose:** Guide a complete supported PC selection and show price/proof/staleness information honestly.
**Prerequisites and handoffs:** U1; fixtures can precede implementation. Connected acceptance needs B1/B2/B5 and F5 add-build; B6 is optional. Keep temporary storage bounded and free of PII.
**Expected output:** Manual seven-slot builder, owned draft views, faithful optimizer statuses, temporary selection recovery and atomic add-build submission.
**Files and tests:** Create frontend/src/app/builder/page.tsx, frontend/src/app/builder/drafts/page.tsx, frontend/src/components/builder/BuilderWorkspace.tsx, frontend/src/components/builder/Recommendation.tsx, frontend/src/lib/build-state.ts; test frontend/src/components/builder/BuilderWorkspace.test.tsx.
**Interfaces:** Consumes SlotSelection/BuildValidation/SolveView/SavedBuildView; produces bounded temporary builder state in sessionStorage and validated atomic add-build request.

**Business rules and implementation boundary:**

Implement sequential mobile/seven-slot desktop state, advisory filtering with skipped rules, authoritative selection/add validation, independent flags, draft revision/baseline notices. Render optimizer proof statuses and scope/score disclosures; never silently remove persisted draft data on GET.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A partial selection is compatible | Do not present it as complete or purchasable; all seven slots include a discrete GPU. |
| A saved price changed and the draft is opened twice | Keep the notice until explicit save; do not silently rewrite the saved baseline. |
| Optimizer returns FEASIBLE or TIMEOUT_NO_SOLUTION | Do not claim optimality or insufficient budget without the corresponding proof. |
| Login interrupts a build or explanation selectionHash differs | Restore temporary selections after login; hide stale explanation text. |

**Required named cases from the source plan:** sevenSlotsRequired_discreteGpuVisible; partialCompatible_notPurchasable; wrongCandidate_serverErrorShown; savedPriceNoticeRepeatedUntilSave; delistedReplacementNotice; optimizerFeasibleVsOptimal; timeoutUnknown_notInsufficientBudget; noCompatibleCatalog; knownAboveBudget_notCheapest; loginPreservesTemporarySelection; staleRulesRefreshed; explanationHashMismatchHidden.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd frontend && npm test -- --run src/components/builder/BuilderWorkspace.test.tsx)
```

**Explain before review:** How can misleading optimizer labels cause a customer to make the wrong purchase decision?
**Reviewer checks:** Huy verifies builder/draft/solver fixture parity, independent validation flags and error/recovery flows.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### U4 — Grouped cart and quote review

**Owner:** Bảo. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R03, R09, R16; S03, S04.
**Source task:** [U4 in the domain plan](../superpowers/plans/2026-10-07-frontend.md).
**Business purpose:** Keep groups visible and obtain explicit consent to server-calculated prices and fees.
**Prerequisites and handoffs:** U1; F5/C1 fixtures allow early UI work. Connected completion needs F5/C1 and F3/F7; P3 extends real production calculations later.
**Expected output:** Mixed grouped cart, revision-aware count edits, address form and QuoteReview that passes only reviewed quoteId downstream.
**Files and tests:** Create frontend/src/app/cart/page.tsx, frontend/src/components/cart/CartReview.tsx, frontend/src/components/checkout/QuoteReview.tsx; test frontend/src/components/cart/CartReview.test.tsx.
**Interfaces:** Consumes CartSnapshot and QuoteView; produces QuoteRequest(cartRevision,shippingAddress) and explicitly reviewed quoteId; groups edit count only.

**Business rules and implementation boundary:**

Implement persistent cart/groups, revisions, requested/available shortages and server quote/address form. Show exact VND fee/tax breakdown; require explicit new review on409 changes. No client-invented item prices or flat cart payload that loses groups.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Two builds share CPU X plus a standalone CPU X | Render separate groups/standalone entry; do not flatten their identity. |
| Quote expires, price changes or stock is insufficient | Preserve the cart, show the typed issue and require fresh review when needed. |
| Customer proceeds to initiate | Do not send UI prices or inventory versions; use the server quoteId and separate checkout identity flow. |

**Required named cases from the source plan:** mixedCartKeepsGroups; twoBuildsSharingCpu_standaloneSeparate; groupItemEditBlocked; shortageCartPreserved; quotePriceChange_requiresNewReview; quoteExpiry_beforeInitiate; feeTaxBreakdownExact; unsupportedAmountMessage; noClientPricesInInitiate; cartEditsDoNotMutatePendingOrder.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd frontend && npm test -- --run src/components/cart/CartReview.test.tsx)
```

**Explain before review:** Why must a newly calculated total require consent rather than being silently accepted?
**Reviewer checks:** Phat checks grouped quantities, exact subtotal/shipping/tax math and consent/revision fixtures against F5/C1.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### U5 — Checkout identity and payment recovery UI

**Owner:** Bảo. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R10, R11, R12, R13, R16; S03, S08.
**Source task:** [U5 in the domain plan](../superpowers/plans/2026-10-07-frontend.md).
**Business purpose:** Keep one intended purchase recoverable across double-clicks, lost responses, reload and external wallet return.
**Prerequisites and handoffs:** U1/U4 contracts can support early mocked states; connected completion needs C1–C7, F3/F7 and owned order history. Phat agrees identity/error fixtures before integration.
**Expected output:** Checkout/recovery/return/history pages and getOrCreateCheckoutIdentity; sessionStorage holds only quoteId/key/orderId and temporary parts.
**Files and tests:** Create frontend/src/app/checkout/page.tsx, frontend/src/app/checkout/[orderId]/page.tsx, frontend/src/app/payments/return/page.tsx, frontend/src/app/orders/page.tsx, frontend/src/app/orders/[id]/page.tsx, frontend/src/lib/checkout-attempt.ts; test frontend/src/components/checkout/CheckoutFlow.test.tsx.
**Interfaces:** Produces `getOrCreateCheckoutIdentity(quoteId:string):CheckoutIdentity`, identity=(quoteId,key,orderId nullable); consumes CheckoutView/OrderView and payment redirect action.

**Business rules and implementation boundary:**

Store only checkout identity/temporary parts, not PII/token/payUrl. Initiate once per intent, recover202 by status polling2s/backoff10s, handle typed409 errors, use hosted allowlisted redirect and backend-confirmed receipt. Return page strips query data, authenticates and retrieves owned order. Closing remains pending until backend outcome; payment retry/cancel are separate commands.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Double-click or lost initiation response repeats one intended purchase | Reuse the same UUID key; reload resumes the owned order rather than starting a second one. |
| Wallet return query says success but backend says pending | Strip provider query data and show backend pending state, not a paid receipt. |
| Countdown reaches zero while backend is CLOSING | Disable pay/retry, refresh state and continue the pending-closing explanation; do not locally cancel or release stock. |
| Payment is confirmed after another tab edited the cart | Display the immutable backend receipt; do not automatically clear the entire cart. |

**Required named cases from the source plan:** doubleClick_oneKey; lostInitiationResponse_reusesKey; reload_resumesStoredOwnedOrder; differentPurchase_newKey; providerReturnSuccess_queryDoesNotShowReceipt; awaitingAuthorization_notPaid; deadlineZero_disablesActionAndPolls; cancelledClosing202_visible; retrySameOrderAndExpiry; finalFailureRetry_newCommandKey; historyRecoversLostStorage; noAutomaticWholeCartClear; keyboardStatusAnnounced.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd frontend && npm test -- --run src/components/checkout/CheckoutFlow.test.tsx)
```

**Explain before review:** Why is a redirect URL or client countdown not payment authority?
**Reviewer checks:** Phat runs recovery fixtures with C1–C7, checks stable identity and confirms no token/PII/payUrl storage or client-trusted paid state.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### U6 — Admin operations and accessibility

**Owner:** Bảo. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R13, R14, R15, R16; S03, S08.
**Source task:** [U6 in the domain plan](../superpowers/plans/2026-10-07-frontend.md).
**Business purpose:** Expose safe administrative operations and recovery visibility without creating bypass buttons.
**Prerequisites and handoffs:** U1/F3/F7; fixtures may start early, connected completion requires F6/C7 and B1 preview rules. Live MFA behavior is added under P2.
**Expected output:** Controlled metadata/editor/preview, idempotent stock-delta form, redacted recovery queue and guarded fulfillment UI.
**Files and tests:** Create frontend/src/app/admin/page.tsx, frontend/src/components/admin/ProductEditor.tsx, frontend/src/components/admin/StockAdjustment.tsx, frontend/src/components/admin/RecoveryQueue.tsx; test frontend/src/components/admin/AdminOperations.test.tsx.
**Interfaces:** Consumes admin product/tag/platform/overview/order/exception APIs; produces controlled publication/preview and audited stock adjustment/reconcile commands.

**Business rules and implementation boundary:**

Build controlled category forms/tag selectors/platform review, nonpublishing preview, inactive product views, signed-delta stock audit, paid fulfillment and redacted recovery queue. Error focus/labels/announcements apply to customer and admin forms. Backend enforces RBAC regardless of hidden UI.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Admin previews metadata | Show validation without publishing. |
| Admin retries a +3 stock adjustment | Keep the command key stable and send a delta, not a copied absolute quantity. |
| A payment has unknown outcome or an unauthorized user opens admin | Display aged recovery/reconcile information; no force-paid/force-release shortcut. Backend rejects unauthorized actions regardless of hidden UI. |

**Required named cases from the source plan:** inactiveProductsListed; wrongCategoryMetadataErrors; previewNoPublish; stockDeltaNotOverwrite; adjustmentReplayKeyStable; recoveryAgeVisible; noForcePaidOrReleaseButton; fulfilConfirmedOnly; unauthorizedAdminRejected; focusAfterValidationError; mobileAdminUsable; redactedSensitiveData.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd frontend && npm test -- --run src/components/admin/AdminOperations.test.tsx)
```

**Explain before review:** Why must UI visibility never be the sole authorization control?
**Reviewer checks:** Huy checks F6/C7 API parity, command identity, redaction and keyboard focus on validation errors.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### D1 — Compose topology and deterministic seeds

**Owner:** Bảo. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R17; S01, S02, S08.
**Source task:** [D1 in the domain plan](../superpowers/plans/2026-10-08-integration-delivery.md).
**Business purpose:** Give every member the same integrated local topology and reproducible demo data.
**Prerequisites and handoffs:** F1/B3/U1 runnable scaffolds; full seeded startup also needs the corresponding migrations/schema and health contracts. Coordinate docs/environment.md with F1 and hardware seeds with Huy.
**Expected output:** Five-service Compose topology, Dockerfiles, safe environment example, persistent ignored local keys, repeatable seed/start/smoke scripts and demo runbook.
**Files and tests:** Create compose.yaml, backend/Dockerfile, optimizer/Dockerfile, frontend/Dockerfile, .env.example, scripts/demo-start.sh, scripts/seed-demo.sh, scripts/verify-compose.sh, docs/environment.md, docs/runbooks/demo.md.
**Interfaces:** Produces `scripts/demo-start.sh` and `scripts/verify-compose.sh`; consumes service health routes/migrations from component scaffolds; env example has no real secret.

**Business rules and implementation boundary:**

Create health-gated five-service startup, internal network and localhost-only development ingress. Generate ignored local RSA keys under .local/keys and reuse them across local restarts; never commit generated keys or production secrets. Seed80 PC parts and20 general products with positive/zero stocks, compatible/incompatible platforms, provenance labels, and synthetic demo accounts/addresses. Provider stub profile supports offline core demo; MoMo sandbox credentials are separate. Include compatible platform evidence instead of invented hardware claims.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A fresh volume starts or seed runs twice | Apply the coordinated available migrations, reach five healthy services and keep seed counts at 80 PC parts plus 20 general products without duplicates. |
| DB/Redis/optimizer are inspected from outside the app network | Their service ports are private; development ingress is localhost-only. |
| Demo uses provider stub or real sandbox credentials | Label them separately; live credentials/profile are rejected and local generated keys are never committed. |

**Required named cases from the source plan:** freshVolume_bootMigrationsAndSeed; fiveCoreServicesHealthy; seedRerun_noDuplicates; privateOptimizerDbRedisNotPublished; missingSigningKey_startupFailure; liveSecretInDemoProfile_rejected; imageDigestsRecorded.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
bash scripts/verify-compose.sh
```

**Explain before review:** Why is a repeatable seed with reviewed compatibility evidence part of correctness rather than just convenience?
**Reviewer checks:** Huy starts from fresh volumes, checks seed/evidence/port boundaries and compares exact runtime/image pins.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### D2 — Contract parity and connected user journeys

**Owner:** Bảo. **Reviewer:** Phat. **Stage:** Core.
**Requirements:** R01–R17; S01–S08.
**Source task:** [D2 in the domain plan](../superpowers/plans/2026-10-08-integration-delivery.md).
**Business purpose:** Make independently written services agree on actual API fields and end-to-end business outcomes.
**Prerequisites and handoffs:** Publish spec-derived public schemas/fixtures early with producer review to unblock U1–U5. Completion needs D1 and connected required core APIs/UI; mocked screenshots do not complete this task.
**Expected output:** Public OpenAPI, shared fixture/type parity script and real-stack browser journeys with a controllable signed provider stub.
**Files and tests:** Create contracts/public.openapi.yaml, contracts/fixtures/checkout/, scripts/verify-contracts.sh, frontend/playwright.config.ts, frontend/e2e/core.spec.ts, frontend/e2e/payment-recovery.spec.ts.
**Interfaces:** Produces versioned public/internal schema fixtures and generated frontend contracts; consumes real Java/Python/PostgreSQL/Redis with controllable HTTP provider stub.

**Business rules and implementation boundary:**

Generate schemas/types from actual backend contracts, check committed fixtures with every service, and run browser journeys against the full stack. Test public routes and owned/private boundaries. Provider stub controls IPN/query/capture interleavings and realistic signatures; it does not bypass application verification.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Java/Python/frontend drift in a field or enum | Contract verification fails before integration is accepted. |
| A customer buys a mixed standalone/build cart | The browser journey preserves groups through quote/allocation and reaches a backend-confirmed immutable receipt. |
| Provider return fakes success or another user’s UUID is requested | No false paid receipt; ownership is enforced by the server. |
| Authorization precedes capture | The browser still displays pending until verified capture, with correct cancel/recovery paths. |

**Required named cases from the source plan:** catalogToStandalonePurchase; builderToAtomicGroupedMixedCart; savedDraftPriceChange; quoteRepriceReview; doubleClickReloadOneOrder; authorizeNotPaid_captureReceipt; pendingCancelSafeRelease; providerReturnFakeSuccess_noReceipt; otherUser404; adminStockAudit; keyboard360Layout; sharedRulesJavaPythonFrontendParity; malformedProofRejected.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(bash scripts/verify-contracts.sh && cd frontend && npm run test:e2e)
```

**Explain before review:** Why do independently passing component tests not prove a working checkout journey?
**Reviewer checks:** Phat checks schemas against S01/S03/S07, runs connected E2E and rejects provider stubs that bypass signature/lifecycle handling.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### D4 — CI, study metrics, and milestone evidence

**Owner:** Bảo. **Reviewer:** Huy. **Stage:** Core.
**Requirements:** R17, R18; S10.
**Source task:** [D4 in the domain plan](../superpowers/plans/2026-10-08-integration-delivery.md).
**Business purpose:** Give the lead repeatable evidence of implemented requirements instead of AI-generated success claims.
**Prerequisites and handoffs:** Can prepare CI and telemetry contracts early. Completion requires D2/D3 real evidence and all required core task checks; coordinate V3 journey columns and business-transaction hooks with Phat.
**Expected output:** Clean-build CI/verify-all, nonzero discovered suites, trusted journey metrics, evaluation protocol and R01–R07/R09–R18 core evidence ledger.
**Files and tests:** Create .github/workflows/ci.yml, scripts/verify-all.sh, docs/evidence/core-milestone.md, docs/evaluation/protocol.md, backend/src/main/java/com/estore/observability/JourneyEvents.java, backend/src/main/java/com/estore/observability/JourneyController.java, frontend/src/lib/journey-events.ts; modify backend/src/main/java/com/estore/cart/CartService.java, backend/src/main/java/com/estore/checkout/AllocationTransaction.java, backend/src/main/java/com/estore/payment/LifecycleTransaction.java; test backend/src/test/java/com/estore/observability/JourneyEventsIT.java.
**Interfaces:** Produces clean-build CI/verification command, discovered-test counters, deduplicated started/completed journey metrics, and requirement evidence ledger.

**Business rules and implementation boundary:**

CI runs Maven verify, Python tests, frontend unit/type/build/E2E, migrations/Redis/contracts and Compose smoke with provider stub. Include dependency/secrets checks and nonzero test-discovery gates. Implement S10 client-start telemetry and durable backend-only completion events using X-Journey-Id and persisted snapshot IDs; never trust browser payment-complete events. Define checkout/build timing denominators, pilot baseline/counterbalanced study and exact participant counts. Core exit requires R01–R07/R09–R18 evidence, with R08 optional; production requirements remain open until P7.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A test command exits zero but discovers no integration suite | Fail the gate. |
| The allocation/confirmation transaction rolls back | No committed completion metric exists; duplicate backend events deduplicate by stable identity. |
| Browser submits ORDER_CONFIRMED | Reject it; only server business transactions produce trusted completion events. |
| Thirty-minute cohorts include abandoned starts | Count incomplete attempts in denominators; report actual sample sizes and keep optional R08 and production requirements separate. |

**Required named cases from the source plan:** unitAndITDiscovered_notZero; pythonTestsDiscovered; frontendBuildAndTypecheck; sharedSchemasDrift_fail; emptyDatabaseMigrationsPass; telemetryReplay_dedup; clientPaymentComplete_rejected; transactionRollback_noCompletionMetric; cohortWindow30mIncompleteAttemptsCounted; secretsPiiAbsent; unsupportedRuntimeManifest_fail; optimizerOracleAndRuntimeReport.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
bash scripts/verify-all.sh
```

**Explain before review:** Why can “no errors” or an attractive conversion percentage be misleading evidence?
**Reviewer checks:** Huy runs the clean verifier, checks suite counts/schema parity/oracle evidence and independently validates metric denominators and source authority.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### P1 — Merchant readiness and live-mode guard

**Owner:** Bảo. **Reviewer:** Phat. **Stage:** Production.
**Requirements:** R19, R24; S05, S09.
**Source task:** [P1 in the domain plan](../superpowers/plans/2026-10-08-production-launch.md).
**Business purpose:** Ensure the merchant and operating model actually support the payment protocol before live dispatch is possible.
**Prerequisites and handoffs:** Gather merchant/operator inputs during core development; completion requires D4 and reviewed S05 sandbox behavior. Phat implements/reviews backend launch guards; merchant/provider approval remains external.
**Expected output:** MoMo capability decision, named launch inputs and fail-closed LaunchReadiness validation using reviewed evidence and secret references, not secrets in documents.
**Files and tests:** Create docs/decisions/momo-live-capabilities.md, docs/operations/launch-inputs.md, backend/src/main/java/com/estore/config/LaunchReadiness.java; test backend/src/test/java/com/estore/config/LaunchReadinessIT.java.
**Interfaces:** Produces `validateLiveConfiguration(LaunchConfiguration):LaunchReadinessResult`, rejecting live startup unless required reviewed configuration/evidence is present; consumes S05 MoMo adapter.

**Business rules and implementation boundary:**

Record merchant approval, autoCapture=false eligibility, hosted-return/IPN endpoints, state/query/replay/expiry/refund behavior, allowed methods/limits, operator contacts, and secret references. Run sandbox capability cases first. Enable only MoMo wallet methods whose terminal-state guarantees fit S05; no assumed waiting-wallet cancel or clock-based release. Future VNPAY requires its own adapter plan.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| Merchant approval, two-step capability or operator ownership is absent | Live mode remains disabled. |
| Live mode is paired with sandbox hosts or ordinary test profile | Reject mixed configuration and any live financial dispatch. |
| CREATE/authorization/query/expiry behavior is reviewed | Record actual terminal-state/idempotency guarantees; do not assume waiting-wallet cancellation from a popup or deadline. |

**Required named cases from the source plan:** noMerchantApproval_liveDisabled; sandboxHostWithLiveMode_rejected; unavailableTwoStepCapability_rejected; missingProviderExpiryRetryStateReview_rejected; missingOperationsOwners_rejected; unsupportedWalletAmount_blocked; approvedConfig_hasNoSecretsInEvidence; testProfileCannotDispatchLive.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=LaunchReadinessIT verify)
```

**Explain before review:** Why does a successful sandbox payment not prove production merchant readiness?
**Reviewer checks:** Phat checks environment/amount/capability guards and merchant-backed evidence; Bảo records inputs without claiming to supply merchant/business/operator approvals.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### P5 — Shipment, customer policies, and governance

**Owner:** Bảo. **Reviewer:** Huy. **Stage:** Production.
**Requirements:** R21, R23; S09.
**Source task:** [P5 in the domain plan](../superpowers/plans/2026-10-08-production-launch.md).
**Business purpose:** Provide traceable fulfillment and customer policies while preserving required financial records and data ownership.
**Prerequisites and handoffs:** D4/C7, P2 identity/outbox, P3 approved market inputs, P4 cancellation/returns interfaces and V4–V6 before V7. Phat contributes backend shipment/privacy code and Huy helps policy/data cases.
**Expected output:** V7 shipments/privacy/outbox, whole-order shipment/delivery confirmation, customer support/policy pages and owned export/delete workflow with retention explanations.
**Files and tests:** Create backend/src/main/resources/db/migration/V7__fulfillment_privacy.sql; create backend/src/main/java/com/estore/fulfillment/ShipmentService.java, backend/src/main/java/com/estore/privacy/CustomerDataService.java, frontend/src/app/help/page.tsx, frontend/src/app/policies/[policy]/page.tsx, frontend/src/components/orders/ShipmentStatus.tsx, docs/operations/data-retention.md; test backend/src/test/java/com/estore/commerce/FulfillmentGovernanceIT.java.
**Interfaces:** Produces `ship(UUID adminId,UUID orderId,ShipmentRequest,UUID key):ShipmentView`, `requestDataAction(UUID userId,DataActionRequest):DataActionView`; action=(EXPORT/DELETE), returns requestId/status/retentionExplanation; customer support policies versioned/visible.

**Business rules and implementation boundary:**

Implement one whole-order manual shipment/tracking and delivery confirmation with idempotency/audit, separate fulfillment history, and order-guarded pre-shipment cancellation blocking, status emails and support contact/policy pages. Record Vietnamese market-reviewed shipping/returns/privacy/terms and data retention/deletion exceptions with owners. Data deletion/export jobs are owned/authenticated, redacted and auditable; no blind erasure of unsettled financial records.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| An unpaid or blocked order is submitted for shipment | Reject it; paid pre-shipment cancellation and shipping use the same order guard. |
| Shipment command is replayed or delivery is recorded | One shipment effect; recorded delivery time starts the approved return window. |
| A customer exports/deletes data | Access only that customer’s data; preserve required transaction records with a reviewed explanation rather than blind deletion. |

**Required named cases from the source plan:** onlyPaidUnblockedShips; duplicateShipment_once; shippingUrlAllowlist; paidUnfulfilledCancel_entersRefundWorkflow; cancelVsShip_guardOnlyOneWins; deliveredTimestampStartsReturnWindow; customerCannotEditTracking; privacyExportOnlyOwnData; deletionPreservesRequiredTransactionRecordsWithExplanation; piiMaskedLogsAndAdminViews; optimizerNoPii; requiredPolicyPagesPresent; transactionalStatusEmailOutbox.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
(cd backend && ./mvnw -Dit.test=FulfillmentGovernanceIT verify)
```

**Explain before review:** Why does a deletion request require a retention policy instead of immediate erasure of every row?
**Reviewer checks:** Huy checks reviewed policies and owned data fixtures; Phat checks backend guards/audit and the P4 cancel-versus-ship integration.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### P6 — Secure deployment and recovery rehearsal

**Owner:** Bảo. **Reviewer:** Phat. **Stage:** Production.
**Requirements:** R22, R23; S09.
**Source task:** [P6 in the domain plan](../superpowers/plans/2026-10-08-production-launch.md).
**Business purpose:** Operate the application with controlled secrets, alerts and rehearsed recovery instead of relying on successful startup.
**Prerequisites and handoffs:** D4, P1 actual hosting/operator/budget inputs, accepted service/RPO/RTO objectives and service-owner runbooks. Rehearsals require disposable or explicitly designated infrastructure; production live effects remain gated.
**Expected output:** Private HTTPS topology, secret references, encrypted backups/PITR capability, deploy/restore scripts, assigned alerts and tested incident/payment/restore runbooks.
**Files and tests:** Create deploy/compose.production.yaml, deploy/ingress.conf, scripts/backup.sh, scripts/restore-rehearsal.sh, scripts/deploy-smoke.sh, docs/runbooks/deploy.md, docs/runbooks/payments.md, docs/runbooks/restore.md, docs/operations/alerts.md.
**Interfaces:** Produces isolated production topology, readiness/metrics, encrypted backup/restore scripts and audited release/runbook evidence; consumes named hosting/secret/monitoring configuration.

**Business rules and implementation boundary:**

Configure HTTPS/private networking, managed secret rotation, least-privilege DB role, managed backups/PITR, redacted structured logs and metrics/alerts with real assigned receivers. Rehearse proposed15mRPO/4hRTO or record approved alternatives; test schema-compatible rollback/forward repair. Restore keeps mutating jobs paused until live provider state is reconciled. Include capacity/security scans and payment/refund/incident support runbooks.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A restored database contains jobs whose effects may already exist at MoMo | Keep financial mutation jobs paused until live state is reconciled. |
| A rollback is requested after a migration | Roll back only schema-compatible application code; use forward repair or rehearsed restore for data. |
| A hold is CLOSING over 15 minutes or runnable job age exceeds 5 minutes | Deliver an alert to a named operator; the alert does not force stock release. |
| Restore objectives are reported | Provide measured evidence for accepted objectives; proposed 15-minute RPO/4-hour RTO are not automatically achieved. |

**Required named cases from the source plan:** noPublicDbRedisOptimizerPorts; httpsOnlyAndSecureCookies; missingSecretFailClosed; readinessDegradedDependencies; failedBackupAlerts; agedClosing15mAlerts; jobAge5mAlerts; restoreMeetsAcceptedRpoRto; restartWorkersReconcile; appRollbackSchemaCompatible; restoredSnapshotJobsPausedUntilProviderReconciled; operatorIncidentExercise.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Phat’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
bash scripts/deploy-smoke.sh && bash scripts/restore-rehearsal.sh
```

**Explain before review:** Why can replaying jobs from a restored database cause duplicate external effects?
**Reviewer checks:** Phat verifies service-owner configuration, financial replay suppression, restore measurement and schema-compatible rollback evidence.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

### P7 — Production evidence and controlled launch

**Owner:** Bảo. **Reviewer:** Huy. **Stage:** Production.
**Requirements:** R19–R24; S09, S10.
**Source task:** [P7 in the domain plan](../superpowers/plans/2026-10-08-production-launch.md).
**Business purpose:** Make launch an evidence-backed business/operator decision rather than an automatic result of generated code.
**Prerequisites and handoffs:** P1–P6 complete evidence plus real merchant/operator approval for bounded live commissioning. Ordinary gate verification reads evidence and must not itself move money.
**Expected output:** Signed R19–R24 gate ledger, capacity/security/restore/policy/provider evidence and controlled commissioning/limited rollout record.
**Files and tests:** Create docs/evidence/production-launch.md, scripts/verify-launch-gates.sh, tests/load/production-capacity.js, docs/runbooks/live-commissioning.md.
**Interfaces:** Produces owner-signed gate ledger for every R19–R24 acceptance case, approved objectives/capacity/market/provider configuration, and limited live commissioning/rollout record.

**Business rules and implementation boundary:**

Collect P1–P6 evidence and actual capacity/objectives, resolve critical findings, and validate all launch input documents. Execute controlled live commissioning only with the merchant/operator-approved account, bounded test purchases/refunds and designated owner present; ordinary verification script checks evidence without moving money. Limited rollout monitors reconciliation/aged holds/errors and records recovery/escalation thresholds. Do not mark launch complete from sandbox/demo success.

**Acceptance scenarios:**

| Given / when | Required observable result |
| --- | --- |
| A critical finding or required policy/restore evidence is missing | Gate fails and launch remains disabled. |
| Ordinary CI runs verify-launch-gates | It inspects evidence only; it makes no live purchase/refund or customer-mail calls. |
| Controlled live commissioning is approved | Designated owners perform bounded authorization/capture/cancel/refund/reconciliation with evidence and escalation thresholds; no sandbox result is relabeled live. |

**Required named cases from the source plan:** criticalFindingBlocksLaunch; missingRestoreOrPolicyEvidenceBlocks; providerTwoStepCapabilityVerified; securityReviewRecorded; representativePlentifulAndScarceLoad; controlledLiveAuthorizeCaptureCancelRefundReconcile; alertReceiverAndOperatorConfirmed; rollbackRehearsed; noLiveTransactionsInOrdinaryCI.

**Implementation steps:**

- [ ] Step 1: Confirm the listed prerequisites/fixtures and write the source task’s named assertions, including the business scenarios above.
- [ ] Step 2: Bootstrap this task’s runner if necessary, then record a meaningful failing check for missing behavior; a missing runner is not a business test failure.
- [ ] Step 3: Implement the stated files/interfaces and business rules; coordinate shared files before editing another owner’s integration points.
- [ ] Step 4: Run the commands below and inspect actual discovered cases, outcomes and persisted effects; include each named case rather than only the happy path.
- [ ] Step 5: Prepare the evidence packet and business explanation below, commit the task scope and request Huy’s review; unresolved connected checks keep the task open.

**Verification commands:** run from the repository root; commands are future instructions, not executed results.

```bash
bash scripts/verify-launch-gates.sh
```

**Explain before review:** Who can approve live financial operation, and what proof must exist before that approval is useful?
**Reviewer checks:** Huy checks evidence completeness and measured capacity/security records; merchant/operator owners approve live steps and Phat checks payment readiness.
**Handoff evidence:** task ID + commit/PR, contract/fixture version, actual command/environment/test counts, observed business results, failing or NOT RUN cases, and the next consumer’s readiness. Follow the [shared evidence template](README.md#completion-evidence).

## Completion evidence

Use the shared evidence template per task. All owned task checkboxes remain unchecked until implementation is verified. A reviewer’s approval concerns the recorded commit; later contract changes require matching producer/consumer checks. Production approvals come from the named merchant/business/operator owners.
