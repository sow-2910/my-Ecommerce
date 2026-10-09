# Electronics Store with PC Builder — Product Requirements

**Revision:** 3.0, 2026-10-08. **Status:** revised design; no application implementation yet.

This revision includes the reviewed corrections and accepted inventory/reservation decisions. The user explicitly expanded the scope to **a larger application with live payments and production operations**, operating in Vietnam/VND with MoMo first. See the [documentation index](README.md) for precedence, decisions, and plans.

## 1. Purpose, users, and delivery scope

Create a single-seller electronics store that supports ordinary shopping and helps customers configure a supported PC, with safe scarce-stock checkout and recoverable payments. The project also teaches backend engineering through transactions, concurrency, authorization, external-service recovery, and measurable optimization.

Users are general electronics shoppers, PC builders, and store administrators. The initial team remains three students with Java/Python/React responsibilities. The original twelve-week semester is the **core milestone**, not a commitment to complete production launch in twelve weeks. Production work has separate mandatory gates and an estimate after the core milestone.

The initial operating model is one seller, one inventory location, VND as the single currency, and manual product curation. Browsing and temporary building are anonymous; saved builds, persistent carts, and checkout require an account. Guest checkout is a later enhancement. No marketplace, multi-warehouse allocation, currency conversion, or automated catalog scraping is included.

**Market decision:** the user selected Vietnam/VND. Amounts are whole đồng, without foreign-exchange conversion. The user selected MoMo first; VNPAY is a later adapter. MoMo merchant approval and two-step authorize/capture enablement are launch prerequisites. Production cannot default silently to free shipping or zero tax. Delivery zones, shipping rates, tax treatment, customer support, and return policy are also launch configuration decisions.

The product addresses uncertainty about hardware compatibility and scarce-stock checkout. Checkout and usability baselines must be measured during evaluation.

## 2. Priority and release definitions

| Level | Meaning | Included |
| --- | --- | --- |
| Core / P0 | Required integrated semester milestone | Catalog, accounts, grouped cart, manual/saved builder, optimizer, price review, reservations, test payments, recovery, admin, order history, CI/demo |
| Launch / L0 | Mandatory before accepting real customers or live money | Eligible provider, verified accounts/recovery, shipping/tax configuration, refunds/returns, fulfillment, legal/customer notices, backups, monitoring, incident procedures, security/load evaluation |
| Enhancement / P1 | After correctness and launch work | AI explanation, comparisons, saved addresses, guest checkout/merge, integrated graphics, separate cooler slot, multiple storage devices |

L0 is part of the requested application, not optional backlog. The documents define it in a separate production plan so production requirements do not disappear into the demo. P1 comparison/saved-address/guest/hardware features require their own follow-up designs; only optional AI is task-planned now.

## 3. Core requirements

### R01 — Product discovery (P0)

Browse a fixed three-level hierarchy such as Electronics → PC Components → GPU. Navigation labels are separate from component slot types. Search by name/SKU; filter category, brand, price, and live in-stock status; sort and paginate deterministically.

Acceptance: active list/detail views show name, brand, exact configured-currency price, image/fallback, relevant specifications, and available quantity. Delisted items disappear from discovery. Empty/loading/error states are useful. Availability is informational until checkout; adding to a cart does not reserve stock.

### R02 — Accounts and sessions (P0)

Register, log in, refresh, and log out with backend customer/admin permissions. Password minimum is twelve characters, with documented hashing/input limits. Five consecutive failed logins cause a fifteen-minute account lock; responses avoid exposing account existence.

Acceptance: concurrent duplicate registration yields one account; failed-login counters persist through rejected responses; refresh rotation yields one successor per token. Browser access tokens stay in memory; refresh tokens use protected HttpOnly cookies. Logout revokes the current refresh session; existing access tokens expire within fifteen minutes. Email verification and reset become mandatory in R19 before launch.

### R03 — Persistent grouped cart (P0)

One cart per customer. Standalone additions merge by product. Adding a full validated PC copies its seven selections into an independent immutable group; later draft edits do not change it.

Acceptance: all components are added or none. A group quantity changes the number of complete builds, not individual component ratios. Remove/rebuild to change a grouped component. Shared components across two builds and a standalone part remain distinguishable. Shortages stay visible with requested/available counts but block checkout. Concurrent adds preserve uniqueness. Editing the cart during payment does not modify the pending order.

### R04 — Manual builder (P0)

Seven required slots: CPU, MOTHERBOARD, RAM, GPU, STORAGE, PSU, CASE. RAM is one complete kit; STORAGE is one supported NVMe drive; CPU includes reviewed bundled cooling. Every initial build includes a discrete GPU, including OFFICE. Explain these limits before selection.

Acceptance: select/replace/remove components, show subtotal and estimated power, identify missing slots and rule-specific problems. Instant client filtering assists selection; Java validates selections and final add-build. A compatible partial draft is not complete or purchasable. Keyboard and mobile interactions work.

### R05 — Saved draft history (P0)

Owned drafts save IDs and baseline prices. Loading compares against current data without changing the saved baseline.

Acceptance: repeated reads preserve price-change notices; out-of-stock selections stay identifiable; delisted parts are excluded from the effective selection but remain in replacement notices. Missing metadata is an error. Explicit save advances the snapshot; concurrent save returns a revision conflict. Other users cannot read or modify it.

### R06 — Supported compatibility checks (P0)

Check slot identity/required metadata, CPU/motherboard socket, RAM type/kit capacity, supported motherboard form factor, GPU clearance, storage interface, reviewed platform/cooling coverage, and PSU load.

Acceptance: consumption and PSU rated capacity are separate fields. Estimated load = CPU draw + GPU draw + 75 W allowance for the remaining supported parts; do not sum PSU capacity or count these parts twice. Require load ≤ 80% of capacity. Exact power/clearance boundaries pass. Required metadata and reviewed platform evidence are necessary for publication/purchase. Test valid and invalid combinations.

Say “passes supported compatibility checks,” not universal hardware compatibility. BIOS/connector/cooling details outside the rule model require curated combination evidence. Unsupported combinations are blocked instead of guessed. Catalog expansion beyond that evidence needs new rules or review.

### R07 — Budget recommendation (P0)

Inputs: maximum budget in configured currency and GAMING, PRODUCTIVITY, OFFICE, or CONTENT_CREATION. Use a trusted catalog snapshot and a documented educational utility model.

Acceptance: returned recommendations are complete, under budget, and revalidated by Java. `OPTIMAL` claims proof for the snapshot; `FEASIBLE` promises a valid result without proof of optimality. Timeout without a result says the search did not find a recommendation within its limit; it does not prove budget insufficiency. Offer retry/manual building.

Show a cheapest compatible build and exact shortfall only when minimum cost is proven. A known compatible above-budget example may be shown with an explicit non-minimum label. Distinguish a catalog with no compatible build from an insufficient budget. Optimization never reserves inventory. Explain score provenance and synthetic data; scores are not FPS guarantees.

### R08 — Optional AI explanation (P1)

Request explanation separately after displaying a verified result. Pass only its exact parts/workload, no customer or payment information.

Acceptance: build selection works without AI. Five-second total deadline/error omits explanation. Render bounded plain text with an AI label and direction to the supported checks/specifications. Treat catalog text as untrusted input. Changed parts invalidate the explanation. Prompting/disclaimers do not establish factual correctness.

### R09 — Quote and price consent (P0)

MoMo payment limits are checked before reservation; unsupported totals are rejected with an explanation, not split into unplanned payments. Create an owned five-minute quote for the current cart and validated shipping address. Show item prices, shipping, tax, currency, and final total. Demo profile uses VND/free shipping/no tax and is visibly labeled simulation; production uses approved R20 configuration.

Acceptance: initiation requires the reviewed quote. Changed cart, product price, component data, shipping/tax configuration, or expiry returns a new-review response before any allocation/charge. Inventory movement alone triggers live quantity validation, not a page-loaded version check. An initiated order freezes prices, groups, address, fees, and calculation versions. Item changes require closing that attempt safely before creating another.

### R10 — Atomic versioned reservations (P0)

`inventory.stock_quantity` means available, unreserved quantity. Hold subtracts requested quantity once; confirmation performs no further deduction; safe release restores once. The backend reads fresh inventory versions; the browser does not send page-loaded inventory versions.

Acceptance: one short transaction creates the order, price snapshots, deductions, and all holds. Any shortage/conflict rolls back the entire attempt. Retry conflicts in fresh transactions, at most three attempts total. Distinguish `INSUFFICIENT_STOCK` from retryable `INVENTORY_BUSY`. Fifty healthy competing attempts for one unit yield one reservation; plentiful-stock attempts can succeed too. Row locks end at transaction completion, while the reservation persists.

### R11 — Checkout idempotency (P0)

Generate one client UUID for an intended checkout; keep it with quote/order identity in that tab's sessionStorage. Double-click, reload, and uncertain-network retries reuse it. A genuinely new purchase gets a new key.

Acceptance: backend uniqueness is per authenticated user, with a durable request fingerprint and order reference in the allocation transaction. Same key/same request resumes the original order even after quote expiry. Same key/different request returns `IDEMPOTENCY_CONFLICT`. The key is not authentication. Owned pending-order history recovers lost browser state. Payment retry reuses the order/hold without another allocation. A terminally failed MoMo attempt may be replaced only after authoritative nonpayment, with at most three provider attempts inside the original window. Distinct keys may represent distinct purchases; idempotency is not a universal duplicate-purchase detector.

### R12 — Payment lifecycle and recovery (P0)

Ten-minute payment window starts at allocation. Retryable payment failure or a closed wallet popup retains the same order and active provider attempt and deadline. Explicit cancel/expiry begins closing and disables new payment attempts.

Acceptance: provider calls are outside database locks/transactions. Durable operations and stable provider keys recover creation/association crashes. Missing provider ID or timeout is not proof of nonpayment. Reconcile/cancel before release; ambiguous outcomes keep holds and show “Closing — verifying payment.” Safe release transitions only current holds once. Success while holds remain can confirm during closing. Duplicate/out-of-order events cannot downgrade payment, repeat restoration, or confirm released stock. Backend state, not a redirect query, establishes success.

Exceptional success after release enters payment review and a durable refund/reconciliation path, never automatic fulfillment. Amount/currency/account/mode mismatches are quarantined. Admin sees unresolved cases and age. Unpaid cancellation is not a paid-order refund endpoint. Exact live-provider behavior must pass R19's adapter tests before launch.

### R13 — Order history and fulfillment (P0)

Customers view only their own orders, resume eligible payments, inspect closing/refund status, and see immutable paid receipts. Admin marks confirmed orders fulfilled once; launch shipping adds tracking under R21.

Acceptance: distinct messages for pending, closing, confirmed, cancelled, review-required, refunded, and fulfilled. Price/delisting changes do not rewrite receipts. Purchase cleanup removes only unchanged purchased cart entries; newer edits in another tab remain. Fulfillment never deducts inventory again.

### R14 — Catalog and inventory administration (P0)

Admin list/detail includes inactive products. Create/edit controlled metadata and tags, preview specifications without publication, and soft-delist. No hard delete of referenced products or in-place component-type change.

Acceptance: enforce SKU uniqueness, exact positive prices, category metadata, tag kind, and reviewed platform evidence. Catalog changes advance revision in the same transaction. Adjust available stock by signed delta with actor/reason audit; all writers honor the same version protocol. Negative availability is blocked. Display available/held quantities, low-stock items, orders, and payment exceptions. Existing order snapshots/holds survive delisting.

### R15 — Security and privacy (P0)

Enforce ownership/RBAC, CSRF/origin protection for cookies, bounded inputs/rate limits, private internal APIs, and verified raw-body payment signatures. Exclude passwords, tokens, keys, addresses, and payment client secrets from logs.

Acceptance: automated negative tests cover invalid IDs/quantities, unauthorized admin operations, open redirects, invalid signatures, payment amount/currency/mode mismatch, and oversized inputs. Public deployment uses HTTPS/private data ports. Demo data is synthetic; production retention/access controls are in R23.

### R16 — Accessible recovery UX (P0)

Every shortage, contention, price change, stale recommendation, outage, expiry, and uncertain payment has an actionable state that preserves useful work.

Acceptance: labels/associated errors, visible focus, keyboard flow, announced checkout status, 360 px layouts, and loading/empty states are checked. Countdown reaching zero initiates backend reconciliation; the UI does not claim stock was released locally.

### R17 — Reproducible integration (P0)

Start frontend, Java, Python, PostgreSQL, and Redis through one documented command; repeatably seed the demo. CI builds all services and runs unit, database, service-contract, and browser journeys.

Acceptance: fresh-start migrations/builds pass; versions and lockfiles are recorded. Test signed event replays, outages, and restart recovery. Offline provider stubs and the selected provider’s sandbox smoke tests are separate evidence. Production deployment/restore/alerts require R22.

### R18 — Honest evaluation (P0)

Correctness goals: zero negative stock, duplicate allocation, or duplicate restoration; correct accept/reject behavior for all supported-rule fixtures. Compare optimizer validity/score to exhaustive small-instance solutions; report feasible-result rate and runtime.

Pilot aspirations: median build completion under two minutes and observed checkout abandonment below 40%. These are hypotheses, not launch guarantees or industry-baseline comparisons. Define start/complete events, deduplication, thirty-minute observation window, participants/experience, incomplete attempts, and counterbalanced baseline tasks. Include login time and report it separately. Small student samples cannot establish general business conversion claims.

## 4. Production requirements — mandatory before live launch

### R19 — Eligible payments and customer account recovery (L0)

Record business country, currency, eligible merchant account/provider, supported payment methods, two-step authorization/capture policy, webhook capabilities, cancellation/expiry guarantees, refund support, and provider retry-retention limits. Test and live secrets/environments are separate; startup fails closed on mismatched configuration. Enable verified email before checkout, resend verification, single-use password reset, session revocation after password change, and MFA for administrators.

Acceptance: sandbox and controlled live commissioning verify amount/currency, duplicate/out-of-order events, creation timeout, cancellation/payment race, refund failure, and reconciliation after restart. Non-cancellable delayed methods are excluded until a safe hold/reconciliation policy is designed. Verification/reset responses avoid account enumeration; links expire and cannot replay. Email is delivered through a durable transactional outbox. No live transactions run as part of ordinary CI.

### R20 — Delivery, tax, and checkout totals (L0)

Approve served delivery zones, supported address format, manual flat rates per zone, tax-inclusive/exclusive treatment, rounding method, and return/shipping policy for the chosen jurisdiction. Version calculation configuration and preserve it on quotes/orders. Unsupported addresses are rejected before payment. No carrier-rate integration is required initially.

Acceptance: approved jurisdiction examples match exact totals; quote price includes all applicable charges; configuration changes force review before allocation; item/shipping/tax breakdown survives refunds and later rule changes. Production startup/launch is blocked if this configuration is missing. Legal rates/requirements must be confirmed for the selected market rather than invented in these documents.

### R21 — Shipment, returns, and refunds (L0)

Admins record shipment/tracking and a fulfillment audit; customers see status. Define return eligibility, request/approval/rejection, received/inspected outcomes, and partial/full refund amounts. Include paid but unfulfilled cancellation through this workflow. Unpaid cancellation remains R12.

Acceptance: refund commands are idempotent, capped by captured amount minus prior successful/pending refunds, and recover ambiguous provider results. Duplicate callbacks cannot refund twice. Record per-line refund allocation including applicable shipping/tax. Refund completion alone does not restock: only an explicit inspected-return/restock operation adjusts inventory once. Customers receive status updates and support contact. Released-stock late payment compensation remains separate from ordinary returns.

### R22 — Operations and resilience (L0)

Assign deployment, migration, payment reconciliation, support, and incident owners. Define workload capacity, service objectives, backup/restore objectives, alerts, and release/rollback procedures. Start with a single-region deployment; scaling must preserve durable jobs, shared sessions/rate limits, and versioned inventory semantics.

Acceptance: encrypted backups restore in a rehearsal; deployment rollback and forward database-repair procedures are tested; durable workers recover after crashes; alerts detect aged holds, payment/refund exceptions, errors, and queue saturation. Run a representative load test with both scarce and plentiful stock and report measured capacity. Zero data-loss/availability promises must not be claimed without evidence. Live launch requires completed runbooks and observed smoke tests.

### R23 — Customer trust and data governance (L0)

Publish shipping/returns, privacy, terms, and customer-support information approved for the market. Define access/retention/deletion rules for accounts, shipping data, payment references, logs, and purchase records, including records that cannot be deleted immediately. Use hosted provider payment components; store no PAN/CVV. Separate admin accounts, require MFA, audit sensitive actions, and manage/rotate secrets.

Acceptance: security review exercises authorization, session theft/reuse, CSRF, injection, webhook replay, dependency findings, and recovery privileges. PII is masked in operational views/logs and absent from optimizer/AI payloads. Customer-data requests and support escalation have owners. Live launch requires market-specific review of these policies.

### R24 — Measured launch gate (L0)

Launch only when P0 and L0 acceptance evidence, provider approval, approved configuration/policies, security findings, restore/load/incident exercises, and operational ownership are documented. First live commissioning uses controlled orders and verified reconciliation/refund outcomes; then a limited rollout with monitoring.

Acceptance: each gate has an owner, evidence, and pass/fail state. Open critical safety/security findings block launch. Production completion is not inferred from a passing demo or a document revision.

## 5. Journey and boundaries

Browse → configure or select items → authenticate → add to grouped cart → review exact quote → initiate one identified checkout → pay within retained hold → read backend status → receipt/order history → shipment/support/return if needed.

Manual inventory/catalog/fulfillment is allowed; manual guesses about payment success or restoring uncertain holds are not. Deferred capabilities are marketplace, multi-warehouse, dynamic carrier rates, multi-currency conversion, automated scraping, universal hardware support, and advanced rollout infrastructure. Optional AI never authorizes compatibility, money, or stock.

The [TRD](ELECTRONICS_STORE_TRD.md), [design specification](superpowers/specs/2026-10-06-electronics-store-design.md), and [delivery plan](superpowers/plans/2026-10-08-integration-delivery.md) define implementation detail and evidence. Requirements here specify future checks; no application behavior or passing tests are claimed.
