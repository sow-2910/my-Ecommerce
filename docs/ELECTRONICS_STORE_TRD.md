# Electronics Store — Technical Requirements

**Revision:** 3.0, 2026-10-08. **Market:** Vietnam; currency VND. **Provider:** MoMo first, VNPAY later. **Scope:** integrated learning milestone plus mandatory production release.

The [PRD](ELECTRONICS_STORE_PRD.md) defines outcomes. The [design specification](superpowers/specs/2026-10-06-electronics-store-design.md) defines schema, exact contracts, state transitions, and plan interfaces. This TRD defines engineering invariants. All three supersede the archived designs. Implementation and application tests remain outstanding.

## 1. Architecture and trust boundaries

Use a Java modular monolith for accounts, catalog, compatibility, carts, money, orders, reservations, payments, refunds, and fulfillment. Python handles bounded search and optional explanation. Next.js presents customer/admin journeys. PostgreSQL is authoritative; Redis is disposable for catalog facts and shared rate-limit counters. No distributed inventory lock or message broker is required initially.

Production serves the browser and `/api` from one HTTPS origin through an ingress. Local development uses the same API path via a proxy. Do not expose Java, Python, PostgreSQL, or Redis publicly. Internal Java→Python calls carry `X-Internal-Api-Key` and a request ID. The optimizer cannot modify inventory or authorize a payment. AI receives no customer information and cannot select parts.

There are five core Compose services: frontend, backend, optimizer, postgres, redis. Production ingress and a local email-test service are additional explicitly named services. A PostgreSQL durable-job/outbox worker runs in Java and is safe with multiple replicas.

### Runtime baseline

Select Java 21, Spring Boot 4.1.x, Maven 3.9.x wrapper, Next.js 16.x, Node.js 24 LTS, Python 3.13, PostgreSQL 17, and Redis 8. Pin current stable patches, image digests, and all third-party dependencies at scaffold time; commit the exact tested matrix in `docs/environment.md`. No floating container `latest` tags. This is a selection baseline, not a tested matrix.

Boot's official requirements support Java 21; Next 16 is the active supported line and Node 24 is LTS at review time. [Spring requirements](https://docs.spring.io/spring-boot/system-requirements.html), [Next support](https://nextjs.org/support-policy), [Node releases](https://nodejs.org/en/about/previous-releases). Use programmatic Resilience4j decorators so an old Boot starter is not assumed compatible. Verify Flyway/PostgreSQL, JWT, Redis, test tooling, React, and FastAPI/Pydantic integration during scaffold. Select patches against [PostgreSQL's policy](https://www.postgresql.org/support/versioning/) and [Python runtime status](https://devguide.python.org/versions/).

## 2. Exact money and authoritative review

All prices/budgets/totals are `amountMinor` integers with `currency: VND`. In this application one minor unit is one đồng. PostgreSQL stores BIGINT amounts, Java uses checked long arithmetic, and Python uses integers. Frontend accepts safe bounded integers and formats `vi-VN`/VND. Reject fractions, unsupported currency, negative values, overflow, and provider-specific out-of-range amounts. Never parse through binary float or multiply application amounts by 100 globally.

A provider adapter is solely responsible for its wire representation. MoMo receives the whole-đồng integer directly. A future VNPAY adapter converts VND × 100 with checked arithmetic and exact reverse validation; this conversion remains adapter-local. [MoMo payment](https://developers.momo.vn/v3/docs/payment/api/wallet/onetime/), [VNPAY PAY protocol](https://sandbox.vnpayment.vn/apis/docs/thanh-toan-pay/pay.html). This does not make the application's currency have two decimal places.

Quote the entire server-owned cart for five minutes. Freeze line/group IDs and revisions, unit prices, product/spec versions, shipping address, shipping/tax configuration versions, and exact totals. Initiation rechecks these against current authoritative values before allocating. A quote is customer price consent, not an inventory version or reservation. Same-key checkout replay is resolved before quote-expiry/current-cart validation.

Demo uses clearly labeled VND/free-shipping/zero-tax simulation. Production requires signed-off Vietnamese delivery/tax configuration and provider limits. Do not infer legal rates or assume zero taxes. Keep calculation components and versions on receipts/refunds.

## 3. Inventory and transaction invariants

`inventory.stock_quantity` is available and unreserved. `version` is a non-null JPA `@Version` counter. Product catalog data is editable and separately versioned; separating tables reduces unrelated contention but does not eliminate competition for stock.

| Action | Availability delta | Reservation transition |
| --- | --- | --- |
| Cart add / quote / optimize | 0 | None |
| Allocate quantity q | −q | New HELD |
| Successful payment | 0 | HELD → CONFIRMED |
| Safe cancel | +q once | HELD → RELEASED |
| Safe expiry | +q once | HELD → EXPIRED |
| Fulfillment | 0 | No inventory transition |
| Inspected return/restock | +q once | Separate audited stock movement |

Inventory writers include allocation, safe release, admin adjustment, and inspected restock. All use the same version/check protocol. Bulk SQL must include expected version and increment it; `@Version` does not fix arbitrary SQL. Database constraints enforce nonnegative available stock and positive reservation quantities.

Normalize repeated product IDs after preserving build-group identity. Process inventory in ascending product ID. One transaction persists idempotency identity, immutable order/items/groups, all inventory updates/holds, and the durable payment-start operation. No per-item commits. Compatibility/group/quantity validation precedes allocation in that transaction.

A transactional allocation bean is called by a separate nontransactional coordinator. Catch flush/commit conflicts outside the proxy boundary, discard the persistence context, and retry the whole attempt at most three times total with fresh reads. Retry only concurrency/deadlock failures, never provider calls, unknown commits, or validation failures. An ambiguous database commit is resolved by idempotency lookup before any new attempt. Fresh shortage is `409 INSUFFICIENT_STOCK`; exhausted contention is `409 INVENTORY_BUSY`, `retryable: true`.

Transactions use READ COMMITTED with explicit lifecycle/cart/catalog guards where stated in the specification. Ordinary reads do not serialize checkouts. Database writes take locks until commit/rollback; optimistic locking detects stale versions and does not choose customer fairness. [PostgreSQL isolation](https://www.postgresql.org/docs/17/transaction-iso.html), [locking](https://www.postgresql.org/docs/17/explicit-locking.html), [JPA Version](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/version).

## 4. Idempotency and durable payment operations

Client key: cryptographically random UUID per intended checkout, stored in sessionStorage with quote/order ID; reuse after double-click, reload, or uncertain response. Auth scopes it; the key grants no access. Server key: UNIQUE `(user_id, idempotency_key)`, canonical SHA-256 request fingerprint, immutable order reference. Keep successful identity records with the order; never expire them on the ten-minute hold boundary.

Concurrent same-key allocation yields one order. Existing same fingerprint returns/resumes it; different fingerprint returns `IDEMPOTENCY_CONFLICT`. Failed allocation rolls back the identity too. Payment retry reads the existing order; it does not initiate a new allocation. A replacement attempt requires the terminal-state guard above. Separate new keys can create separate purchases if stock exists.

A durable `payment_operations` row carries operation kind, immutable provider request identity/payload hash, status, attempt count, next-run time, lease, and failure code. Commit before external calls. Workers briefly claim work, call outside transactions, then persist/associate results in a new transaction under the order guard. Leases are not sufficient for external exactly-once effects: the adapter must provide duplicate-safe provider identity or reconcile uncertainty without replaying a charge/refund.

Use one persisted merchant reference per provider attempt, stable request IDs per create/capture/cancel/refund operation, fresh IDs per separately scheduled query, and a separate refund merchant reference per refund. An order may have at most three attempts, with only one nonterminal attempt at a time. Replacement requires verified terminal nonpayment and an open original reservation window; it does not allocate again or extend that window. Do not import Stripe idempotency-header assumptions into another provider. Retry retained identity only within the provider's documented guarantees; beyond that, reconcile or enter manual review instead of risking another payment. Stripe's ≥24-hour key-retention behavior is historical comparison, not the Vietnamese adapter contract. [Stripe idempotency](https://docs.stripe.com/api/idempotent_requests).

## 5. Provider integration and payment closing

The user selected MoMo as the first Vietnamese provider; VNPAY is a later integration. Design against `PaymentGateway` rather than a `stripe_payment_intent_id` column or mandatory client secret. The customer action is a validated hosted `REDIRECT` URL or NONE. Local stub supports deterministic tests; real provider sandbox validates its protocol.

MoMo uses hosted wallet payment, `requestType: captureWallet`, `autoCapture: false`. Persist the create request's orderId/requestId before dispatch. Whole-đồng payment range is initially 1,000–50,000,000 VND; reject unsupported totals before holds. The worker allows a 35-second total provider call, with 2-second connect and 30-second response budget; browser initiation returns 202 and polls instead of waiting under a transaction. Creating a payUrl only means an attempt is ready, not that funds were captured. [MoMo create](https://developers.momo.vn/v3/docs/payment/api/wallet/onetime/).

An authorized observation triggers a durable CAPTURE only while the order's holds remain and the deadline is open; otherwise schedule CANCEL_AUTHORIZATION. Capture and cancellation share the order/operation guard so they cannot both be dispatched as new conflicting commands. In-flight capture uncertainty retains holds. Confirm inventory only on verified captured payment. Authorization alone is not a paid receipt. [MoMo confirm](https://developers.momo.vn/v3/vi/docs/payment/api/payment-api/confirm/).

Map statuses by operation context: create success is readiness; code 9000 is authorization in this two-step flow; code 0 on capture/verified captured transaction is payment success; 1000/7000/7002 and unknown results remain pending. Codes 1001/1002/1003/1004/1005/1006/1007/1017 may represent terminal nonpayment only after verifying the specific attempt/current provider state. Query “not found” and transport errors remain ambiguous. Maintain the reviewed full mapping as fixtures. [MoMo result codes](https://developers.momo.vn/v3/docs/payment/api/result-handling/resultcode/), [query](https://developers.momo.vn/v3/docs/payment/api/payment-api/query/).

Reuse MoMo requestId on uncertain mutating retries, never replace it merely to bypass a duplicate/in-progress response. Treat duplicate/in-progress as a reason to query. Stop automatic mutating replay after 30 days from first dispatch, inside the documented minimum 31-day retention, and escalate uncertainty. Reuse never authorizes a new provider attempt while the old one may pay. [MoMo idempotency](https://developers.momo.vn/v3/docs/payment/api/result-handling/idempotency/).

Verify MoMo's prescribed HMAC-SHA256 notification representation and acknowledge with HTTP 204 within fifteen seconds after durable processing; invalid notifications are not acknowledged as processed. No PAN/CVV collection. Use only HTTPS hosted payUrl on a configured MoMo host allowlist; no QR/deep-link permission is assumed. [MoMo notifications](https://developers.momo.vn/v3/docs/payment/api/result-handling/notification/).

The ten-minute deadline stops application payment retries. Explicit cancel or expiry enters CLOSING with a durable first closing reason. Cancel a verified authorization when possible. An attempt still waiting for wallet authorization is not assumed cancellable through the authorization-confirm API: hide its hosted action and reconcile until authoritative terminal nonpayment is established. A late authorization for a closing order is cancelled, not captured. A clock, “not found,” a failed attempt, or an absent local provider ID alone is insufficient. Retain ambiguous holds beyond ten minutes; alert on aged cases. Never promise immediate release for every provider.

Before live enablement, obtain/verify a provider-specific state table covering payment retry with the same merchant reference, expiry/in-flight settlement, authoritative terminal statuses, refund replay/query behavior, and notification acknowledgements. If these guarantees do not satisfy safe release, restrict methods/change adapter or keep cases in review; do not weaken the inventory invariant.

### Required lifecycle behavior

| Observation | Permitted result |
| --- | --- |
| Payment started, holds exist, window open | DRAFT → PAYMENT_PENDING; provide hosted action |
| Retryable transport/popup failure | Keep order/active attempt/holds/deadline; query before exposing retry |
| Explicit cancel/deadline | DRAFT/PAYMENT_PENDING → CLOSING; no new hosted action |
| Verified success, all holds HELD | Confirm once, including while CLOSING; no stock delta |
| Verified terminal nonpayment, no future/in-flight start | CLOSING → CANCELLED; release only HELD once |
| Success for previously released stock | PAYMENT_REVIEW_REQUIRED; compensation refund/reconciliation; never fulfill |
| Success/failure duplicate or older failure after success | No downgrade/no repeated stock side effect |
| Identity/amount/currency/merchant/environment mismatch | Quarantine/payment review; no fulfillment or speculative release |
| Refund completion | Record refund amount/state; restock only through separate inspected-return command |

All finalizers take the same short order-row guard; release still uses inventory versions and whole-transaction retry. Provider responses are observations with timestamp/revision; a finalizer checks current order/operation state rather than blindly applying a stale response. CONFIRMED holds never expire. IPN applies one bounded transaction with a ten-second application budget; lock/commit failure returns503 for redelivery, with no provider call in that request. Duplicate event receipts and local transitions commit together; failed processing rolls back the receipt so a retry is possible.

Verify signatures over the provider-prescribed original representation before mutation. Reject duplicate query keys/ambiguous encodings; verify merchant/reference, exact amount/currency, environment/endpoint configuration, and allowed state. Use the provider's event ID when present, otherwise a stable normalized notification identity. Business-state guards protect separate equivalent events too. Do not acknowledge successful application processing before its commit. Use provider-specific acknowledgement bodies, not the generic public API error format.

## 6. Cart/build/draft integrity

Use independent cart groups and slot items, not a nullable reference to a mutable saved build. A group snapshot stores seven slot/product selections and one group count. Cart/order grouping survives aggregation by product for inventory. Atomic add-build validates owner/current selected products/rules and persists all slots or none.

One cart per user, partial unique index for standalone `(cart_id, product_id)`, and unique grouped `(group_id, slot)`. Lock the cart for mutations; creation uniqueness is database-enforced. Quantity is 1–99 and total expanded cart lines ≤50. Group count produces each component quantity; individual grouped-line edits are rejected.

Saved drafts use revisioned normalized slot rows with saved unit price/product version. GET computes effective selections and notices without writes. Explicit save advances snapshot. Compare expected draft revision for updates. Delisted products are soft-deleted; immutable order name/SKU/spec/price snapshots keep receipt history useful.

After successful payment, lock the cart and remove only source entries with matching purchased revision/quantity. A changed entry remains intact, with a notice to review it; never blindly clear a cart that another tab edited.

## 7. Hardware validation and data policy

Java is authoritative. Shared rule fixtures drive Java, Python, and client filtering; client filtering cannot approve purchase. Slots must match category. Partial builds return evaluated/skipped/violated rule states plus independent `isCompatible`, `isComplete`, `isPurchasable` values. Missing counterpart means skipped; missing required metadata on a selected product means violation.

Initial rules: socket equality; RAM type equality/kit capacity and module count; motherboard form factor membership in case supported set; GPU length ≤ clearance; supported single NVMe interface; curated CPU/motherboard/case platform coverage including bundled cooling; integer power inequality `5 * estimatedLoadW <= 4 * ratedWattageW`. Estimate = CPU + GPU draw + 75 W base. Supported set membership replaces unknown form-factor ranks.

Publication validates category-specific fields, tag kinds, positive numeric values, and provenance. A controlled string alone cannot establish BIOS/cooling compatibility. Curated combination records include CPU/board/case, evidence reference, reviewer, and reviewed catalog versions; changing pertinent specs invalidates that coverage. Only reviewed combinations are selectable/purchasable. Store manufacturer/evidence links and score provenance; synthetic academic scores are explicitly labeled.

## 8. Search, deadlines, and resilience

Use constraint-aware branch-and-bound with partial-state validation, safe lower price bounds and upper utility bounds, deterministic tie-breaks, and a retained feasible incumbent. Motherboard-anchor pruning is an optimization, not proof that independent category choices satisfy aggregate constraints. Greedy upgrades can improve a known complete valid incumbent; greedy failure is not infeasibility.

Search statuses: OPTIMAL, FEASIBLE, TIMEOUT_NO_SOLUTION, INFEASIBLE, NO_COMPATIBLE_BUILD. Only exhaustive/pruned proof may mark optimality/infeasibility/minimum cost. The same total three-second cooperative search budget covers feasible baseline, score search, and any minimum-cost search. Python CPU work runs in a bounded process pool, off the async event loop, with monotonic checks at most every 256 nodes. Supervisor terminates unresponsive work at 3.5 seconds and replaces the worker; this is a hard wall limit, not a promise of a salvaged result. Cooperative completion normally returns any retained incumbent.

Bound to two search workers, zero queued jobs, ≤200 active component candidates with ≤30 per slot; reject oversize snapshots explicitly rather than silently truncating. Use exact integer workload scores/weights. Validate with exhaustive small-instance oracles and adversarial fit/power/greedy traps.

Public route is `POST /api/builder/auto-build`; Python route is `/internal/optimizer/solve`. Java supplies trusted IDs/specs/prices/stock/catalog revision/rule version and enriches selected IDs after fresh authoritative validation. Never accept catalog prices/specs from the browser. A stale result yields `409 BUILD_STALE`; no automatic unbounded retry or AI explanation for different parts.

Java has a ten-second total optimizer deadline, 500 ms connect timeout, and bounded concurrency. Resilience4j: TIME_BASED ten-second window, minimum ten calls, 50% failure threshold, thirty-second open wait, two half-open probes. Invalid inputs and valid no-solution results are not dependency failures; transport/timeout/malformed results are. Circuit breaking does not limit concurrency; use a two-permit no-queue bulkhead. [Circuit breaker behavior](https://resilience4j.readme.io/docs/circuitbreaker).

Optional explanation is a separate route with a five-second total deadline; it cannot delay the solver result. Phase-specific HTTP timeouts do not replace a total deadline. Bound output to 2,000 characters, plain text; invalidate on selection/snapshot mismatch. CPU cancellation is not implemented by merely wrapping synchronous work in `asyncio.wait_for`. [Python asyncio](https://docs.python.org/3.13/library/asyncio-task.html).

## 9. Authentication, email, and request protection

Access JWT: RS256, fifteen minutes, issuer/audience/sub/role/sessionId/jti/credentialsVersion, persisted/shared keys with rotation. Refresh: random 256-bit opaque token, hash at rest, seven-day absolute session lifetime, HttpOnly Secure SameSite=Lax cookie on `/api/auth`. No refresh token in JSON/localStorage. Same-origin signed double-submit CSRF token plus exact Origin checks protect browser mutations. Local HTTP development has an explicit nonproduction cookie profile.

Refresh consumption and replacement occur atomically under a token/session guard. Frontend uses one refresh promise per tab and Web Locks across tabs. `X-Refresh-Request-Id` supports thirty-second encrypted replay of the same lost response; another request ID during that grace period returns `409 REFRESH_IN_PROGRESS` without revoking the family or overwriting cookies. Reuse after grace revokes the session; recovery is re-login. This bounded exception is documented and tested, not unlimited token reuse.

Normalize email consistently. Passwords are not trimmed; bcrypt cost 12, minimum twelve characters, maximum 72 UTF-8 bytes (reject excess, never truncate). Account-lock counters update and commit before returning an authentication failure. Five parallel failures cannot lose increments. Lock expires after fifteen minutes. Error responses are generic401 AUTH_FAILED for unknown/locked/bad-password cases, with dummy bcrypt verification for unknown/locked users; rate-limit also by account/IP.

Launch: email verification before payment, hashed single-use verification/reset tokens, verification TTL 24 hours, reset TTL 30 minutes, password reset revokes all refresh sessions and increments credentialsVersion, verified against the database on every private request to invalidate old access tokens. Live admin reads require an MFA-verified session; sensitive mutations require a five-minute fresh challenge. Admin TOTP MFA/recovery codes and sensitive-operation step-up within five minutes. Avoid raw MFA secrets at rest; encrypt through managed secret storage. Transactional email outbox with bounded retries, dead-letter visibility, redacted content, and no production email in automated tests.

Parameterized queries, DTO validation, output escaping, bounded URLs, same-origin return-path allowlist, content-security policy appropriate for hosted redirects, and secure response headers are required. Do not claim generic “sanitization” solves SQL/XSS. [OWASP session guidance](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html).

Rate limits: public reads 100/min/IP; authenticated mutations 30/min/user; login 10/min/IP and 5/min/account; checkout initiation 10/min/user; explanation 5/min/user/IP. Use shared Redis atomic counters with TTL, trusted ingress IP parsing, HTTP 429/Retry-After. Security-sensitive login/payment/refund mutations fail closed with 503 when the shared limiter is unavailable; browsing can continue with a local conservative limit. Signed provider notifications have separate bounded ingress protection and are not subject to shopper quotas.

## 10. Catalog caching and revisions

For this small catalog, query filtered pages and live availability from PostgreSQL. Cache only stable product/spec/rules facts in Redis, keyed by authoritative catalog revision with ten-minute TTL. Read stock in a batch from the database. Do not cache `inStock` results or stale pages then filter them after pagination.

Advance `catalog_state.revision` within each committed admin data change. Stock-only changes do not advance catalog revision. Admin publication and checkout quote validation use the common catalog guard so prices/specs cannot change between validation and commit. Cache invalidation after commit is an optimization; revision keys preserve correctness if deletion fails. Redis cache failure falls back to the database; security rate-limit policy remains as above. Real Redis integration tests prove revisioning and outage behavior.

## 11. Production commerce and operations

Shipping starts with versioned manual zone/flat-rate configuration for approved Vietnamese addresses and explicit unsupported-zone rejection. Tax calculation uses versioned inclusive/exclusive rules and deterministic whole-đồng rounding against approved reference examples; no guessed VAT rates. Quote/order snapshots retain all calculation inputs and component amounts. `taxMinor` discloses total included/additional tax; `additionalTaxMinor` is only the tax not already in displayed prices/fees. `totalMinor = subtotalMinor + shippingMinor + additionalTaxMinor`; do not add included tax twice. Refund targets include their corresponding tax charge exactly once. Address data is versioned province/locality identifiers plus freeform lines; validate against the approved current address dataset, not a stale assumed district list.

Payments, order lifecycle, fulfillment, returns, and refunds have separate state fields. MoMo refunds use the captured transId, a new persisted refund orderId, and stable requestId; query unresolved refund results rather than issuing another refund. Initial wallet refunds accept 1,000–50,000,000 VND; smaller residuals enter an audited support settlement case, not a false successful refund. [MoMo refunds](https://developers.momo.vn/v3/docs/payment/api/payment-api/refund/). Partial refunds are durable records with per-line/fee allocation; amounts reserved by pending refunds count against remaining refundable capture. Refund completion is distinct from stock restock. Restock is an explicit versioned adjustment with unique movement source and inspection evidence. All admin commands check role/MFA freshness, carry request identity, and append audit records.

Deploy with private networking, managed secrets, encrypted backups, least-privilege database roles, health/readiness probes, structured redacted logs, metrics/correlation IDs, and alert delivery to named operators. Suggested initial objectives: 99.5% monthly core API availability, 15-minute backup RPO, 4-hour restore RTO; record owner acceptance and prove the latter two in a rehearsal. These are proposed objectives, not achieved guarantees. Define performance envelope with measured load before launch; provisional catalog p95 <500 ms, quote/allocation p95 <2 s excluding provider wait, and solve response <5 s for normal cooperative search on the agreed deployment.

Alert on any negative-stock invariant failure immediately, aged CLOSING holds >15 minutes, oldest runnable job >5 minutes, failed refunds/reconciliation, elevated 5xx, saturation, failed backups, and expired certificates. Reconcile provider transactions/refunds daily and pending states continuously; no speculative release to improve dashboard numbers. Runbook owns ambiguous cases and controls audited recovery commands.

Migrations are forward-compatible, one coordinator, additive changes before removal. Release rollback reverts application only when schema-compatible; database recovery is forward repair or rehearsed restore, not blind destructive downgrade. No payment replay in restored production data without reconciliation to the live provider. Test restart and restore with durable jobs.

## 12. Verification and evidence

Maven wrapper plus Surefire unit/Failsafe `*IT` suites and Testcontainers/PostgreSQL; pytest with declared HTTP mocking dependencies; Vitest/Testing Library, TypeScript build, and Playwright browser tests; k6 scarce/plentiful stock scenarios. CI validates API fixtures/schema compatibility across all services, migrations on empty database, Redis behavior, provider contract stubs, and clean Compose startup.

Tests must demonstrate all-or-none allocation, two independent last-unit requests, true commit-time optimistic conflict, replay identity, external timeout/crash recovery, ordered/unordered duplicate events, safe closing, grouped cart integrity, unchanged/new cart cleanup, exact money/wire conversion, supported-rule parity, exhaustive optimizer proof, auth rotation/counters, rate limiting, and authorization.

Production adds email/MFA, shipping/tax fixtures, partial refund/return races, backup restore, deployment rollback, capacity/security review, incident exercise, and controlled live commissioning. Evidence is stored with commands, environment, sample sizes, outcomes, and outstanding failures. “Expected PASS” in a plan is not execution evidence.

The [integration plan](superpowers/plans/2026-10-08-integration-delivery.md) and [production plan](superpowers/plans/2026-10-08-production-launch.md) assign verification work and gates. The [documentation index](README.md#requirement-coverage) maps requirements to tasks and evidence; [team assignments](TEAM_TASK_ASSIGNMENTS.md) name task owners and reviewers.
