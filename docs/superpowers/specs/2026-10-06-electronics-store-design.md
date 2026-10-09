# Electronics Store with PC Builder — Design Specification

**Revision:** 3.0, reconciled 2026-10-08. Filename retained to preserve references.
**Sources:** [PRD](../../ELECTRONICS_STORE_PRD.md), [TRD](../../ELECTRONICS_STORE_TRD.md). Accepted inventory and reservation decisions are specified in S04/S05 below.
**Scope:** Vietnam/VND, MoMo first, larger application with a twelve-week core milestone and separate mandatory launch work. This document is the canonical implementation contract. No code/test completion is implied.

## S01 — Global constraints and domain types

- Java package: `com.estore`; Java 21; Spring Boot 4.1.x; Maven 3.9.x wrapper.
- Next.js 16.x; Node.js 24 LTS; Python 3.13; PostgreSQL 17; Redis 8. Pin tested patches/digests and dependencies during scaffold in `docs/environment.md`.
- Currency: `VND`; application amounts are integer whole đồng. No float money or currency conversion.
- Prices: `priceMinor` in 1–1,000,000,000; order arithmetic checked and total ≤1,000,000,000. MoMo checkout total must additionally be 1,000–50,000,000. Budget is 1,000–50,000,000. Unsupported payment totals fail before allocation; split payments are excluded.
- Quantities/group count: 1–99. Expanded cart lines ≤50; quantity sums checked before allocation. Public page is zero-based, size 1–100, default 20; invalid values return 400, never silently clamp.
- UUIDs identify users, carts, groups, drafts, quotes, orders, attempts, operations, refunds, and returns. Product IDs are positive BIGINT constrained to ≤9,007,199,254,740,991 and serialized as safe integers. Dates are UTC ISO-8601; provider-specific timestamps are adapter-local.
- Slots: CPU, MOTHERBOARD, RAM, GPU, STORAGE, PSU, CASE. Goals: GAMING, PRODUCTIVITY, OFFICE, CONTENT_CREATION.
- Order states: DRAFT, PAYMENT_PENDING, CLOSING, PAYMENT_CONFIRMED, PAYMENT_REVIEW_REQUIRED, CANCELLED, FULFILLED. `refundStatus` is separate: NONE, PENDING, PARTIAL, FULL, REVIEW_REQUIRED. Do not turn a shipped order into an unpaid order after a refund.
- Reservation states: HELD, CONFIRMED, RELEASED, EXPIRED. Closing reasons: CANCELLATION, EXPIRY; first reason is preserved.
- Attempt states: NEW, STARTING, AWAITING_USER, AUTHORIZED, CAPTURE_PENDING, PROCESSING, SUCCEEDED, FAILED_FINAL, CANCELLED_FINAL, UNKNOWN. All except the three terminal states count as nonterminal.
- Shared clocks are injectable. Database deadlines are authoritative. Inventory retries: three total fresh transactions. Quote TTL five minutes; hold/payment-retry window ten minutes, never extended by a retry.
- Default customer UI is Vietnamese, VND formatting, and Asia/Ho_Chi_Minh display time; backend contracts/errors use stable English codes with localized frontend messages.

Request record names used in plans: `QuoteRequest(cartRevision,shippingAddress)`, `AddBuildRequest(name,selections,quantity,sourceDraftId?)`, `ProductWriteRequest(sku,category,name,brand,priceMinor,imageUrl?,isActive,spec?,expectedCatalogVersion?)`, `ProposedComponent(category,spec)`, `StockAdjustmentRequest(delta,reason)`, `SaveDraftRequest(name,selections)`, `UpdateDraftRequest(expectedRevision,name,selections)`, `AutoBuildRequest(budgetMinor,currency,goal)`. New products start at zero available stock; audited adjustment adds stock. SavedBuildView fields are id/name/revision/selections/effectiveSelections/baselinePrices/currentProducts/notices/validation. Update/delist referenced products uses expected catalog version; new creation has no expected version.

Core records used by plans (Java records/enums; JSON camelCase; Python Pydantic aliases identical):

| Type | Fields |
| --- | --- |
| `ApiError` | code:String, message:String, retryable:boolean, details:object, requestId:UUID |
| `Money` | amountMinor:long, currency:String=VND |
| `ShippingAddress` | recipientName:String, phone:String, country:String=VN, provinceCode:String, localityCode:String, addressLine:String, postalCode:String? |
| `SlotSelection` | selections:Map<Slot,Long>; unselected slots absent |
| `ProductView` | id:long, sku:String, category:String, name:String, brand:String, priceMinor:long, currency:String, availableQuantity:int, imageUrl:String?, catalogVersion:long, spec:ComponentSpec? |
| `RuleIssue` | code:String, slots:List<Slot>, message:String, details:object |
| `BuildValidation` | isCompatible:boolean, isComplete:boolean, isPurchasable:boolean, totalMinor:long, estimatedLoadW:int, missingSlots:List<Slot>, violations:List<RuleIssue>, skippedRules:List<String>, catalogRevision:long, rulesVersion:String |
| `CartEntry` | id:UUID, revision:long, kind:STANDALONE/BUILD, quantity:int, selections:Map<Slot,Long>?, productId:long? |
| `CartSnapshot` | id:UUID, revision:long, entries:List<CartEntry>, products:List<ProductView>, subtotalMinor:long, issues:List<RuleIssue> |
| `QuoteView` | quoteId:UUID, cartRevision:long, expiresAt:Instant, subtotalMinor:long, shippingMinor:long, taxMinor:long, additionalTaxMinor:long, totalMinor:long, currency:String, shippingAddress:ShippingAddress, calculationVersion:long, lineSnapshots:List<object> |
| `CheckoutRequest` | quoteId:UUID; idempotency UUID is an HTTP header |
| `PaymentAction` | type:NONE/REDIRECT, url:String?; url must match configured HTTPS MoMo hosts |
| `CheckoutView` | orderId:UUID, status:OrderStatus, refundStatus:RefundStatus, reservationExpiresAt:Instant, serverTime:Instant, totalMinor:long, currency:String, paymentAction:PaymentAction, canRetryPayment:boolean, nextPollAfterMs:int=2000 |
| `OrderView` | checkout fields + immutable line/group/address/fee snapshots, fulfillment:object, paymentNotices:List<String> |

`ShippingAddress` lengths: recipient 1–100, address 1–300, province/locality 1–30; phone normalized to E.164 for supported Vietnamese numbers. Customer-data launch routes are POST `/api/account/data-requests` with `{action:EXPORT/DELETE}` and GET `/api/account/data-requests/{requestId}`, owned/authenticated; response has requestId/status/retentionExplanation, with secure expiring export access after processing. No required obsolete district field. The production address dataset/configuration is versioned and approved; demo uses clearly labeled sample codes. Login rejection uses the same generic401 for unknown/locked/bad-password cases; verify a dummy bcrypt hash for unknown/locked users to avoid a cheap account-timing distinction. Public request bodies reject unknown fields and numeric-float coercion; JSON body ≤64 KiB except provider IPN ≤128 KiB. Sensitive fields are never logged.

## S02 — Persistence model, migrations, and indexes

All timestamps use TIMESTAMPTZ; all enum fields use NOT NULL plus CHECK; all FK columns used in queries are indexed. All money/quantity fields have nonnegative/positive checks as appropriate. Referenced catalog data is soft-deleted. Entities below have created/updated timestamps where mutable.

### Foundation migration `V1__foundation.sql`

| Table | Required columns and constraints |
| --- | --- |
| users | id UUID PK; email VARCHAR(255) normalized UNIQUE; email_verified_at nullable; password_hash; role CUSTOMER/ADMIN; failed_login_attempts≥0; locked_until nullable; credentials_version≥0 |
| auth_sessions | id UUID PK; user_id FK; absolute_expires_at; revoked_at nullable; last_mfa_at nullable; last_reauth_at nullable |
| refresh_tokens | id UUID PK; session_id FK; token_hash UNIQUE; expires_at; consumed_at; replacement_id nullable FK; replay_request_id nullable; encrypted_replay nullable; replay_expires_at nullable |
| catalog_state | singleton id=1 PK; revision BIGINT≥0 |
| products | id BIGINT identity PK; sku UNIQUE; category; name; brand; price_minor BIGINT>0; currency=VND; image_url; is_active; catalog_version BIGINT≥0 |
| inventory | product_id PK/FK; stock_quantity INT≥0; version BIGINT≥0, JPA @Version |
| constraint_tags | id BIGINT PK; kind SOCKET/RAM_TYPE/FORM_FACTOR/STORAGE_INTERFACE; value; UNIQUE(kind,value); referenced tags cannot be deleted/renamed |
| pc_component_specs | product_id PK/FK; category-typed fields in S06; metadata_schema_version; provenance; synthetic_score flag |
| reviewed_platforms | CPU/board/case IDs and catalog versions, evidence_ref, reviewer_id FK, reviewed_at; UNIQUE tuple of IDs+versions; invalidated by relevant edits |
| carts | id UUID PK; user_id UNIQUE FK; revision BIGINT≥0 |
| cart_groups | id UUID PK; cart_id FK; name; quantity 1–99; revision≥0; builder_journey_id nullable UUID; source_draft_id nullable informational only |
| cart_items | id UUID PK; cart_id FK; product_id FK; group_id nullable FK; slot nullable; quantity positive; revision≥0; CHECK(group_id/slot both null or both set) |
| inventory_movements | id UUID PK; product_id FK; delta nonzero; reason; actor_id nullable; source_kind; source_id; resulting_available≥0; UNIQUE(source_kind,source_id,product_id) |
| admin_audit | id UUID PK; actor_id FK; action; resource_type/id; request_id; redacted before/after JSONB; timestamp |

Standalone uniqueness: partial unique `(cart_id,product_id) WHERE group_id IS NULL`. Grouped uniqueness: `(group_id,slot) WHERE group_id IS NOT NULL`. Composite FK `(group_id,cart_id)` references `cart_groups(id,cart_id)` to prevent cross-cart grouping. Grouped quantity is derived from group count and must not diverge; persist one unit per slot and expand at read/checkout. DB constraints plus the cart lock enforce seven unique slots for committed build groups. Create cart with unique insert/reload, not unguarded find-then-create.

Inventory movement records document hold/release/admin/restock deltas, with reservation IDs as unique hold/release sources. Confirmation has no stock movement. This journal supports reconciliation; it is not an additional quantity to subtract.

### Builder migration `V2__saved_builds.sql`

`saved_builds(id UUID PK,user_id FK,name,revision≥0,saved_at)` and `saved_build_items(build_id FK,slot,product_id FK,saved_price_minor>0,saved_catalog_version,PRIMARY KEY(build_id,slot))`. Drafts may be partial. Ownership comes from the parent; all APIs query it by user ID. GET has no side effects. Price baselines change only on explicit POST/PUT save.

### Checkout migration `V3__checkout.sql`

| Table | Required columns and constraints |
| --- | --- |
| checkout_quotes | id UUID PK; user_id/cart_id FK; cart_revision; immutable snapshot JSONB; price/spec fingerprint; calculation_version; address snapshot; totals; expires_at |
| checkout_requests | id UUID PK; user_id FK; idempotency_key UUID; request_hash; order_id UNIQUE FK; UNIQUE(user_id,idempotency_key) |
| orders | id UUID PK; user_id FK; source_quote_id FK; checkout_journey_id nullable UUID; status; closing_reason nullable; reservation_expires_at; address snapshot; subtotal/shipping/tax/additional_tax/total_minor; currency=VND; calculation_version; refund_status; active_attempt_id nullable FK; paid_attempt_id nullable FK; cart_cleanup_done boolean; version≥0 |
| order_groups | id UUID PK; order_id FK; name_snapshot; quantity; source_cart_entry_id/revision |
| order_items | id UUID PK; order_id FK; product_id FK; order_group_id nullable FK; slot nullable; source_cart_entry_id/revision; name/SKU/category/spec snapshot; quantity>0; locked_price_minor>0; line_total_minor; line_tax_minor; additional_line_tax_minor; UNIQUE(order_id,source_cart_entry_id,slot) with standalone partial uniqueness |
| inventory_reservations | id UUID PK; order_id/product_id FK; quantity>0; status; expires_at; UNIQUE(order_id,product_id) |
| payment_attempts | id UUID PK; order_id FK; sequence>0; provider=MOMO/STUB; merchant_ref UNIQUE; create_request_id UNIQUE; retry_key nullable UUID; expected_amount/currency; status; provider_trans_id nullable UNIQUE; encrypted_pay_url nullable; first_dispatched_at; last_observed_at; UNIQUE(order_id,sequence); UNIQUE(order_id,retry_key) |
| payment_operations | id UUID PK; attempt_id FK nullable; refund_id nullable FK to core refunds; kind CREATE/QUERY/CAPTURE/CANCEL_AUTHORIZATION/REFUND/QUERY_REFUND; provider_request_id UNIQUE; request_hash; immutable payload JSONB without raw secrets; status READY/RUNNING/RETRY/DONE/REVIEW; attempt_count; next_run_at; lease_until; first_dispatched_at; result_code |
| payment_event_receipts | id UUID PK; provider; notification_key; payload_hash; attempt_id FK; observed_at; normalized state; UNIQUE(provider,notification_key) |
| journey_events | id UUID PK; journey_id UUID; event_type BUILD_STARTED/BUILD_ADDED_TO_CART/CHECKOUT_STARTED/ORDER_CONFIRMED; event_key UNIQUE; occurred_at; no raw user/address/token fields |
| payment_exceptions | id UUID PK; order_id/attempt_id FK; kind; state OPEN/RESOLVED; reason; resolution_ref; reviewed_by; timestamps |

Create order before its checkout_request FK; all remain inside one allocation transaction. Partial unique `(order_id) WHERE payment_attempts.status IN (nonterminal states)` permits only one unresolved attempt. Mutating operation uniqueness `(attempt_id,kind)` for CREATE/CAPTURE/CANCEL_AUTHORIZATION; a partial unique `(refund_id,kind) WHERE kind=REFUND` guards compensation/refund dispatch. Each separately scheduled QUERY/QUERY_REFUND has a new operation/request ID. Capture/cancel dispatch decisions are additionally mutually exclusive under the order guard. A refunded extra attempt must not overwrite `paid_attempt_id`.

Indexes: reservations `(status,expires_at)` and product; orders `(user_id,created_at,id)` and `(status,reservation_expires_at)`; operations `(status,next_run_at,lease_until)`; events unique identity; cart child FKs; catalog `(is_active,category,id)`, `(brand,id)`, `(price_minor,id)`. Case-insensitive contains search over the small curated catalog can initially scan; add trigram index only after measurement.

Core V3 also creates `refunds` for exceptional payment compensation: UUID PK, order_id/attempt_id FK, business_key UNIQUE per order, refund merchant_ref UNIQUE, stable request_id UNIQUE, amount_minor>0, status REQUESTED/PROCESSING/SUCCEEDED/FAILED_FINAL/REVIEW, reason, provider_ref, timestamps. V6 extends these records for ordinary refund allocations and optional return references; it does not create a second incompatible refunds table. Create FK targets before operation rows and add circular orders.active_attempt/paid_attempt FKs after both tables exist.

### Launch migrations (single coordinator)

`V4__identity_operations.sql`: verification/reset token hashes, encrypted MFA secrets/recovery-code hashes, email outbox.
`V5__commerce_policy.sql`: immutable commerce policies, delivery zones and tax rules.
`V6__returns_refunds.sql`: return requests/items, refund allocations/return references, inspection/restock records; extends core refunds.
`V7__fulfillment_privacy.sql`: shipments, owned customer-data requests and notification outbox. Each task owns its new migration; never edit an already-applied migration.

Core refund fields remain unchanged. Allocations link refund to order line/fee with exact amounts. Order guard reserves pending+successful refund totals before dispatch. Restock unique `(return_item_id,inspection_id)` and inventory movement source makes replay harmless.

## S03 — Public HTTP contracts and ownership

JSON uses the types in S01. Pagination returns `{items,page,size,totalElements,totalPages}`. Sort allowlist: PRICE_ASC, PRICE_DESC, NAME_ASC, NEWEST; append ID as stable secondary key. Search `q`≤100 chars; no arbitrary SQL sort. Public errors use ApiError; standard statuses: 400 invalid, 401 unauthenticated, 403 forbidden, 404 unknown/non-owned, 409 state/revision conflict, 429 quota, 503 unavailable. No provider secret is sent as a frontend environment variable.

| Access | Method/path | Request and response |
| --- | --- | --- |
| Public | GET /api/auth/csrf | Signed double-submit cookie/token |
| Public, CSRF/origin | POST /api/auth/register | email,password → 201 userId; duplicate 409 EMAIL_EXISTS |
| Public, CSRF/origin | POST /api/auth/login | email,password → accessToken,expiresIn=900,user; refresh cookie; generic 401 AUTH_FAILED for invalid credentials, unknown account, or locked account |
| Cookie, CSRF/origin | POST /api/auth/refresh | empty body; X-Refresh-Request-Id UUID → new accessToken and cookie; 409 REFRESH_IN_PROGRESS |
| Cookie, CSRF/origin | POST /api/auth/logout | empty → 204; revoke current refresh session, clear cookie |
| Customer/admin | GET /api/auth/me | user ID/role/email/verification state |
| Public, quota | POST /api/telemetry/journeys | {journeyId,event:BUILD_STARTED/CHECKOUT_STARTED} →202, deduplicated server timestamp; no client payment-complete event |
| Public | GET /api/categories | fixed three-level navigation tree |
| Public | GET /api/products | q,category,brand,minPriceMinor,maxPriceMinor,inStock,page,size,sort → page ProductView |
| Public | GET /api/products/{id} | active ProductView; inactive 404 |
| Public | GET /api/builder/rules | rulesVersion,catalogRevision,slot schema,rule definitions,scope disclaimer |
| Public | POST /api/builder/validate | SlotSelection → BuildValidation |
| Public | POST /api/builder/evaluate | {draft:SlotSelection,slot:Slot,candidateIds:List<long>≤100} → candidate ID/allowed/violations |
| Public | POST /api/builder/auto-build | {budgetMinor:long,currency:VND,goal:Goal} → SolveView in S07 |
| Public, quota | POST /api/builder/explanation | {selection:SlotSelection,goal:Goal,catalogRevision:long} → {text:String?,selectionHash:String}; stale 409 |
| Customer | GET/POST /api/builder/drafts | owned list / {name,selections} → SavedBuildView with revision/baseline/notices |
| Customer | GET/PUT/DELETE /api/builder/drafts/{id} | read effective draft; PUT {expectedRevision,name,selections}; delete 204 |
| Customer | GET /api/cart | CartSnapshot |
| Customer | POST /api/cart/items | {productId,quantity} → CartSnapshot; merge standalone |
| Customer | PUT/DELETE /api/cart/items/{id} | PUT {quantity,expectedRevision}; DELETE requires If-Match with quoted entry revision; standalone only; grouped mutation 409 GROUP_EDIT_REQUIRED |
| Customer | POST /api/cart/builds | {name,selections,quantity,sourceDraftId?} → atomic new group, CartSnapshot |
| Customer | PUT/DELETE /api/cart/builds/{id} | PUT {quantity,expectedRevision}; DELETE requires If-Match with quoted group revision; group count only |
| Customer | DELETE /api/cart | explicit clear; If-Match with quoted cart revision required; missing/malformed header400 |
| Customer | POST /api/checkout/quote | {cartRevision,shippingAddress} → 201 QuoteView |
| Customer | POST /api/checkout/initiate | CheckoutRequest + Idempotency-Key UUID → 202 CheckoutView until action ready; replay 200/202 original |
| Customer | GET /api/checkout/{orderId} | CheckoutView, no provider calls; reads persisted action only when eligible |
| Customer | POST /api/checkout/{orderId}/retry-payment | empty + Idempotency-Key UUID for retry command → 200/202 same order; replacement only if prior attempt terminal and window open |
| Customer | POST /api/checkout/{orderId}/cancel | empty → 200 CANCELLED or 202 CLOSING; paid 409 ORDER_ALREADY_PAID |
| Customer | GET /api/orders; GET /api/orders/{id} | owned paginated summaries / OrderView |
| Provider signed only | POST /api/payments/momo/ipn | MoMo signed body → 204 after committed receipt/state/jobs; invalid 400; processing failure 503 |
| Public | GET /payments/return | frontend status page; strips provider query and loads owned backend status after login |
| Admin | GET/POST /api/admin/products | includeInactive,page → page / full validated product input →201 |
| Admin | GET/PUT/DELETE /api/admin/products/{id} | PUT expectedCatalogVersion and input; soft delist 204 |
| Admin | POST /api/admin/products/preview | proposed category/spec + candidate counterparts → validation; no writes |
| Admin | GET/POST /api/admin/tags | controlled list / {kind,value} →201; referenced values immutable |
| Admin | GET/POST /api/admin/platforms | reviewed combination evidence/version tuple →201; advance catalog revision |
| Admin | POST /api/admin/inventory/{productId}/adjustments | {delta,reason} + Idempotency-Key → audit/result; shortage409 |
| Admin | GET /api/admin/overview; GET /api/admin/orders | low stock/held/recovery counts / paginated orders |
| Admin | POST /api/admin/orders/{id}/fulfill | expected order version + Idempotency-Key → fulfilled once; launch shipment data included |
| Admin | GET /api/admin/payment-exceptions | redacted unresolved cases/age |
| Admin | POST /api/admin/payment-exceptions/{id}/reconcile | Idempotency-Key →202 query work; never force paid/release |

Exact security matchers implement this table. Public builder rules/validate/evaluate/auto-build do not accidentally make drafts/cart/order routes public. Refresh/logout require the refresh cookie/CSRF policy despite living under auth. Admin checks apply both controller and service boundaries. Non-owned UUIDs are 404. Signed provider IPN is CSRF-exempt with its own authentication.

Launch endpoints: POST `/api/auth/verify-email`, `/api/auth/resend-verification`, `/api/auth/forgot-password`, `/api/auth/reset-password`; POST `/api/auth/mfa/enroll`, `/api/auth/mfa/confirm`, `/api/auth/mfa/challenge`, `/api/auth/reauth`; GET/PUT `/api/admin/commerce-policy`; POST `/api/admin/orders/{id}/shipments`; POST `/api/admin/shipments/{id}/delivered`; POST customer `/api/orders/{id}/returns`; GET customer `/api/returns/{id}`; POST admin `/api/returns/{id}/approve`, `/reject`, `/receive`, `/restock`; POST admin `/api/orders/{id}/refunds`; GET admin `/api/refunds/{id}`. All launch request/response schemas are defined in S09; core setup reserves their paths but does not stub successful responses.

## S04 — Checkout algorithm and concurrency guards

1. Resolve `(user,key)` before inspecting current quote/cart. Existing fingerprint must equal SHA-256 of canonical `{quoteId}` (fixed field order); return its order only to owner. Never hash a new current cart for replay.
2. On new identity, begin fresh transaction: lock cart, lock catalog_state FOR SHARE, read selected products/current revisions, validate quote TTL/ownership/cart snapshot/prices/specs/config, validate every complete group, calculate exact totals and provider range. Anonymous/unverified launch accounts fail before allocation.
3. Aggregate quantities across standalone/group slots by product; reject bounds/shortages; read inventory fresh and update managed entities in ascending ID order. Flush before commit, with version checking. Inventory expected versions are backend values, never a quote/client version.
4. Persist immutable order groups/items, one aggregated HELD reservation per product, checkout request identity, first attempt, and CREATE operation in the same transaction. Expiry = allocation clock +10 minutes. Commit all or none.
5. A nontransactional coordinator handles retries outside commit boundary (three attempts total); unique-key conflict reloads the existing request. Unknown commit result triggers lookup/reconciliation, not blind allocation replay.
6. Worker claim and provider dispatch occur after commit. Browser gets 202 with order/deadline and no action until CREATE result association commits. GET status returns eligible persisted redirect with no network work.

Price-review errors: QUOTE_EXPIRED, CART_CHANGED, PRICE_CHANGED, CATALOG_CHANGED, COMMERCE_POLICY_CHANGED; return current review details but do not auto-consent or reserve. New quote/new intended submission uses a new key; uncertain retry of the original submission keeps its original key. No inventory-page reload is required.

Guard order: cart mutations/quotes/allocation take cart then shared catalog then sorted inventory where needed; admin catalog writes take exclusive catalog guard before products; inventory adjustments take only sorted inventory. Payment lifecycle locks order, then cart (if cleanup), then sorted inventory (if release). No network call while these locks are held. No path acquires an order guard after holding a cart guard. Workers claiming jobs lock order before operation; candidate discovery is unlocked then conditional claim/recheck. This avoids order/operation lock inversion.

Short database transactions have a five-second timeout, one-second PostgreSQL lock timeout, and three-second statement timeout. IPN processing has a ten-second application budget and makes one bounded database attempt; on lock/commit failure return503 for redelivery instead of retrying until the fifteen-second provider acknowledgement deadline is exceeded. Durable workers/coordinators may use the specified bounded fresh-transaction retries outside this HTTP acknowledgement.

Use separate Spring proxied transaction beans (`AllocationTransaction`, `LifecycleTransaction`, `WorkerClaimTransaction`) called by coordinators. Self-invocation does not create retry transactions. Keep transient data and failure counters out of transactions that are deliberately rolled back.

Purchase cleanup: when confirming, lock current cart and remove a source standalone/group only if revision and quantity exactly match the purchased snapshot; otherwise leave it intact and return a notice. Set cleanup_done in the same confirmation transaction. Group deletion is all slots. Cart revision advances once for actual cleanup. No automatic clear on browser success.

## S05 — Payment operations, observations, and safe finalization

`PaymentGateway` Java interface:

```java
StartResult start(StartCommand command);
PaymentObservation query(QueryCommand command);
PaymentObservation capture(CaptureCommand command);
PaymentObservation cancelAuthorization(CancelCommand command);
RefundObservation refund(RefundCommand command);
RefundObservation queryRefund(RefundQueryCommand command);
VerifiedNotification verifyNotification(byte[] originalBody);
```

Commands include attempt/refund UUID, persisted merchantRef/requestId, expected amount/currency, provider configuration version, and immutable approved payload. StartResult is `{ready:boolean,action:PaymentAction,observation:PaymentObservation?}`. PaymentObservation includes attempt ID, normalized state, original result code, expected merchant match, amount/currency, providerTransId, observedAt, and evidence source. Its normalized states are READY, WAITING, AUTHORIZED, CAPTURED, TERMINAL_NONPAYMENT, UNKNOWN. RefundObservation is PENDING, SUCCEEDED, TERMINAL_FAILED, UNKNOWN with identity/amount.

For MoMo create/capture/query/cancel/refund signatures, implement operation-specific prescribed canonical strings, never generic sorted JSON. Store synthetic known-answer fixtures in `contracts/fixtures/momo/`; no real secret/sample customer information. The authoritative adapter reference is the [TRD payment section](../../ELECTRONICS_STORE_TRD.md#5-provider-integration-and-payment-closing), with official sources. Core uses only wallet `captureWallet` with `autoCapture=false`, VND, hosted payUrl, and sandbox credentials. Two-step live enablement is a launch gate.

CREATE success/readiness is not CAPTURED. AUTHORIZED enqueues CAPTURE only if all holds remain HELD, the order is open, and the deadline has not passed. Otherwise enter CLOSING and enqueue CANCEL_AUTHORIZATION. Before dispatch, briefly recheck the same guard/deadline; mark the chosen command RUNNING and commit. An in-flight capture may later win even after local closing; retain holds until its outcome is known. Never dispatch cancellation simultaneously as a new competing command while capture is unresolved.

Each worker operation has a 60-second lease, 35-second total call limit, and stable mutating request identity; heartbeat is unnecessary within this bound. A restarted/expired lease queries/replays under provider idempotency within its retention policy. Claim at most four provider jobs concurrently; no transaction spans a call. Retry delays 5s,30s,2m,5m capped, with jitter; query pending orders at most once/30s/attempt. Cap automatic mutating replay at thirty days from first dispatch, then REVIEW with alert. A no-response timeout is UNKNOWN, not failure/nonpayment. A done operation cannot be reclaimed. Suppress never-dispatched CREATE under the order/operation guard when closing; in-flight/ever-dispatched CREATE must be resolved before release.

One order can have up to three sequential provider attempts. `retryPayment(userId,orderId,retryKey)` locks order, resolves retry command identity, queries unresolved attempt through durable work, and only creates a replacement after terminal nonpayment, complete retained holds, open deadline, and no unresolved capture. Reuse active redirect when awaiting user. Closed wallet popup needs no new attempt. Replacement keeps order price/stock/deadline unchanged. Never reuse a terminal provider orderId to manufacture a new payment or mint a new ID while outcome is ambiguous.

IPN: verify signature/merchant/body bounds first; derive notification key from merchantRef/providerTransId/resultCode/responseTime plus canonical signed-field payload hash (or provider event ID if supplied). Within one transaction lock order, insert receipt uniquely, apply observation/transition, and persist follow-up work. Duplicate receipt returns204 after no repeated effect. If transaction fails, no receipt remains; return503 so processing can retry. Query worker uses the same observation finalizer. Server-side query results are trusted only after validated TLS endpoint/response identity and adapter-prescribed checks.

| Order/hold condition | Observation/event | Transition/side effect |
| --- | --- | --- |
| DRAFT + HELD | start ready | PAYMENT_PENDING, persist encrypted hosted URL |
| Open + HELD + before deadline | authorized | persist AUTHORIZED; enqueue CAPTURE |
| Closing/expired + HELD | authorized | enqueue CANCEL_AUTHORIZATION; no capture |
| Any open/closing + all HELD | verified captured | PAYMENT_CONFIRMED; all HELD→CONFIRMED; stock unchanged; snapshot cleanup once |
| Confirmed/fulfilled | same captured observation / old failure | no downgrade/stock side effect |
| Open + final attempt failure | nonpayment | retain holds/window; enable guarded retry or close on user action/deadline |
| Closing + all attempts conclusively nonpaying, no unresolved dispatch | terminal nonpayment | CANCELLED; HELD→RELEASED or EXPIRED by stored reason; aggregate restores once |
| Cancelled/released | captured | PAYMENT_REVIEW_REQUIRED, no fulfillment; enqueue compensation refund |
| Already paid by another attempt | extra captured payment | keep paid order, open exception/refund for extra attempt; no extra confirmation |
| Any | mismatch or contradictory terminal observation | open payment exception; disable fulfillment/capture until reconciled; retain held stock |

Release transaction rechecks proof/guard, transitions only newly HELD rows, sums those quantities by product, version-updates stock in sorted order, records unique release movements, and commits order cancellation with releases. It never selects all historical EXPIRED rows. Conflict retries the entire finalization (three total), then leaves durable work runnable. Scheduler runs every30s, enters closing for expired unpaid orders, and schedules reconciliation; multiple workers are safe.

If no attempt was ever dispatched and CREATE is durably suppressed, no-payment proof can be local. Otherwise only adapter-reviewed terminal nonpayment with no in-flight commands permits release. Missing ID, code42/not-found, system failure, or elapsed ten minutes do not meet that proof. After fifteen minutes of closing, alert; escalation does not automatically restore stock. Exceptional released-stock success creates one refund business identity; do not claim refunded until verified completion. Minimum-refund limitations route to audited support settlement, not fake refund completion.

## S06 — Compatibility schema and rules

Required ComponentSpec fields by category:

| Slot | Required metadata |
| --- | --- |
| CPU | socketTagId; estimatedDrawW>0; bundledCooler=true; evidenceRef |
| MOTHERBOARD | socketTagId; ramTypeTagId; formFactorTagId; supportedStorageInterfaces; dimmSlots>0; maxRamGb>0 |
| RAM | ramTypeTagId; moduleCount>0; capacityGb>0; one-kit selection |
| GPU | estimatedDrawW>0; lengthMm>0 |
| STORAGE | interfaceTagId=reviewed supported NVME_PCIE; capacityGb>0 |
| PSU | ratedWattageW>0; connector/platform evidenceRef; no consumption field |
| CASE | supportedFormFactorTagIds nonempty; maxGpuLengthMm>0; platform/cooling evidenceRef |

Every PC product also has workload scores 0–1000, metadata schema version, manufacturer/evidence reference, source date, and synthetic flag. Tags are FK IDs; publication verifies correct kind. Unused category fields can be NULL, but selected required fields cannot. Revalidate inactive/incomplete catalog data; do not manufacture defaults. Product category cannot change after creation.

Rule order/codes: SLOT_CATEGORY, REQUIRED_METADATA, SOCKET_MISMATCH, RAM_TYPE_MISMATCH, RAM_CAPACITY, RAM_MODULE_COUNT, CASE_FORM_FACTOR, GPU_CLEARANCE, STORAGE_INTERFACE, PLATFORM_NOT_REVIEWED, PSU_CAPACITY. Counterpart absent → skipped; selected required metadata absent → violation. Exclude delisted parts from effective draft and report PRODUCT_UNAVAILABLE; shortages yield INSUFFICIENT_STOCK separately.

`isCompatible` = no supported-rule violations; `isComplete` = seven slots populated with available catalog identities; `isPurchasable` = compatible + complete + active products + quantity availability. A draft with an incomplete pair can be compatible but incomplete. Checkout validates group quantities and current available stock in the allocation transaction.

Power estimate CPUdraw+GPUdraw+75. Integer rule `5*load <=4*capacity`; exact650W/520W passes,521W fails. GPU exact length equality passes. Explicit form-factor membership means new tags acquire no implicit rank. Platform reviewed tuple must match current CPU/board/case catalog versions and evidence. Checkout holds freeze reviewed specs; later edits do not retroactively invalidate an already-created payment/order.

Client evaluation is advisory; shared JSON fixtures test Java/Python parity with Java final validation. Preview reuses these rules, returns issues, and writes neither product nor inventory. Admin platform publication is audited and advances catalog revision.

## S07 — Constraint search and contracts

Workload score model is an educational utility, not measured FPS. Per-product `scores[goal]` is category-normalized 0–1000 with provenance; do not combine raw PassMark/Cinebench scales. Versioned weights (sum1000) for CPU/GPU/RAM/STORAGE respectively: GAMING200/650/100/50; PRODUCTIVITY550/150/200/100; OFFICE450/100/250/200; CONTENT_CREATION400/300/200/100. Motherboard/PSU/case weight0; select their price/fit by constraints. Utility = integer sum(weight*score). Tie: larger utility, then smaller total price, then lexicographically smaller product IDs in fixed slot order.

`CatalogSnapshot`: `{catalogRevision,rulesVersion,scoreModelVersion,currency:VND,products:List<ProductView>,reviewedPlatforms:List<object>}` with live positive availability and required metadata. Java builds it; browser cannot supply it. Limits two workers/no queue, total≤200 and≤30 per slot. Over-limit returns503 CATALOG_TOO_LARGE without silent truncation.

Internal `POST /internal/optimizer/solve`, API-key protected:
request `{requestId,budgetMinor,currency,goal,catalog:CatalogSnapshot,deadlineMs:3000}`;
response `{status,selections,totalMinor,utility,provenOptimal,minimumCostProven,minimumBuild?,catalogRevision,rulesVersion,scoreModelVersion,elapsedMs}`.
Public `SolveView`: same proof/status fields plus `products`, `validation:BuildValidation?`, `minimumBuild` enriched/validated if present, `shortfallMinor?`, and `warnings`. No AI text in solve. Unknown IDs/category mismatch/wrong revision/malformed success are503 OPTIMIZER_INVALID_RESULT; current price/stock/spec drift is409 BUILD_STALE.

Search keeps slot identity and aggregate power/clearance/RAM/platform state. Prune only with admissible bounds. Share one monotonic deadline among finding a complete feasible incumbent, maximizing utility, and optional minimum-price proof. Check at most every256 nodes. Always validate a complete selection before recording it. A seeded reviewed baseline is usable only if active/available/under budget; upgrade greedily only from a valid full incumbent. Exhaustive small-instance oracle proves pruning/ties.

| Status | Required evidence/returned fields |
| --- | --- |
| OPTIMAL | in-budget compatible incumbent, provenOptimal=true, exhaustive/admissible completion of score search |
| FEASIBLE | in-budget compatible incumbent; provenOptimal=false; deadline interrupted proof |
| TIMEOUT_NO_SOLUTION | no in-budget incumbent before limit; no proof of infeasibility; selections empty |
| INFEASIBLE | under-budget search exhausted/proven; minimum example optional and never labeled cheapest without minimumCostProven=true |
| NO_COMPATIBLE_BUILD | search proves no compatible complete selection at any price in the snapshot |

Minimum-price search cannot run outside the same three-second budget. A known above-budget example can be returned only labeled `minimumCostProven=false`; it is not a purchase recommendation under the entered budget. On hard worker kill at3.5s, return TIMEOUT_NO_SOLUTION with no unsupported proof. `asyncio.wait_for` around blocking CPU work is insufficient; supervise a process and keep event loop responsive. Search supervisor replaces terminated worker.

Java revalidates each returned build with current product data, recomputes amount/load, and verifies budget/proof-envelope consistency before enrichment. No automatic retry; UI may retry within quota. Circuit/bulkhead/timeouts use TRD values. Python explanation route `/internal/explanation`, separate five-second total deadline, bounded output. Java validates selections/revision before forwarding; explanation response includes selectionHash for UI invalidation.

## S08 — Frontend, cache, and runtime behavior

Routes: `/`, `/products`, `/products/[id]`, `/builder`, `/builder/drafts`, `/cart`, `/checkout`, `/checkout/[orderId]`, `/orders`, `/orders/[id]`, `/login`, `/register`, `/payments/return`, `/admin`. Next `(auth)` route grouping does not add `/auth` to URLs. Only allow configured relative return destinations; reject protocol-relative/external URLs.

Browser fetch client attaches memory access token, current CSRF token, request ID, and credentials for auth cookies. One refresh queue per tab; Web Locks coordinate tabs. Memory token expires naturally; sessionStorage stores only checkout identity and temporary builder selection, never access/refresh token, shipping PII, or hosted payment URL. On reload, refresh auth and resume stored/owned order. Return page strips query data and polls backend. A successful-looking query alone cannot show a paid receipt.

Countdown uses serverTime/reservationExpiresAt. On zero disable pay/retry and request backend state, display closing until confirmed cancellation/payment. Poll every2s while foreground, back off to10s after1min, stop terminal states, refresh on focus. GET status has no provider network call; reconciliation runs durably. Available payment action is suppressed once closing/expired.

Client filtering uses rulesVersion/catalogRevision; refresh stale rule context on409. Never hide all alternatives from a partial draft merely because counterparts are missing. Draft/builder totals use integer amounts. Display pricing notices, supported scope, exact totals, and distinct optimizer statuses. Payment attempt replacement is a backend decision behind one “Retry payment” action.

Redis fact keys `catalog:{revision}:product:{id}` / `rules:{rulesVersion}:{revision}`, TTL600s. Resolve current revision from database; after-commit invalidation optional. Live stock/inStock/pagination query database. Cache outage falls back to DB; security quota outage follows TRD fail-closed mutations. Warm-cache tests must use actual Redis.

## S09 — Launch-specific contracts and operational gates

Production remains disabled until `launchReady=true` is backed by the signed gate evidence, MoMo merchant credentials/capabilities, verified email/MFA, approved commerce/policies, and rehearsed operations. Demo/sandbox never accepts live configuration. VNPAY is not included in first-release task implementation.

Launch DTOs:
- `VerifyEmailRequest(token)`; `ResetPasswordRequest(token,newPassword)`; `EmailRequest(email)`. Forgot/resend always202 generic; valid verify/reset204; expired/replayed tokens400 TOKEN_INVALID. Verification token24h, reset30m, hashes only; reset revokes all refresh sessions. Email outbox encrypts sensitive link payloads and deletes them after delivery/expiry.
- `MfaConfirmRequest(code)`; enrollment returns one-time QR/secret only after reauth; encrypted TOTP secret; hashed single-use recovery codes. Challenge success updates session last_mfa_at. All live admin routes require an MFA-verified session; sensitive mutations require MFA age≤5m. `ReauthRequest(password)` on `/api/auth/reauth` updates session last_reauth_at; enrollment/secret recovery requires password reauth age≤5m. Enroll/challenge endpoints are authenticated, CSRF-protected, and rate-limited; successful login alone never grants live admin operations. Reset increments credentials_version, checked against the access JWT on each private request, so reset invalidates existing access tokens too. Ordinary current-session logout still permits its already-issued JWT until TTL expiry as documented.
- `CommercePolicyRequest(expectedRevision,deliveryZones,taxRules,addressDatasetVersion,returnsPolicyVersion,legalApprovalRef)`; creates immutable policy version, no in-place historical edit. Demo policy distinctly labeled; production requires approved data.
- `ShipmentRequest(expectedOrderVersion,carrierName,trackingNumber,trackingUrl?)` + Idempotency-Key. Accept only paid, unblocked order; tracking URL HTTPS allowlist; customer view redacts admin fields. Initial release uses one whole-order shipment, no partial shipping. Admin `DeliveryConfirmationRequest(expectedShipmentRevision,deliveredAt,evidenceRef)` marks delivery once with audit; the approved return window starts at recorded delivery. All shipment/cancellation commands share the order guard, so a pending pre-shipment cancellation blocks shipment.
- `ReturnRequest(kind:RETURN/PRE_SHIPMENT_CANCELLATION,items:[{orderItemId,quantity}],reason)` + Idempotency-Key; RETURN validates owned delivered items, unreturned quantities and approved eligibility window. PRE_SHIPMENT_CANCELLATION validates paid/unshipped items and atomically blocks fulfillment while requested; race with shipment returns409 if already shipped. No request cancels provider payment through the unpaid endpoint. Accepted unshipped cancellation may refund, but stock still requires audited warehouse inspection/restock. Return states REQUESTED/APPROVED/REJECTED/RECEIVED/CLOSED; approve/reject body `{reason}`; receive `{inspectionId,items:[{returnItemId,receivedQuantity,restockable}]}`.
- `RefundRequest(allocations:[{orderItemId?,feeType?,amountMinor}],reason,returnId?)` + Idempotency-Key. Fee target enum is SHIPPING (including its applicable additional tax); each line target includes its charge/tax allocation. Exactly one line/fee target per allocation; sum checked; pending+successful refunds cannot exceed captured amount or target allocation. MoMo wallet refund range checked; out-of-range becomes support settlement case. API202 with refundId/status; queried verified success drives refundStatus, never availability.
- `RestockRequest(inspectionId,items:[{returnItemId,quantity}],reason)` + Idempotency-Key. Only received/restockable un-restocked quantities; lock order first, then return/inspection, then sorted inventory; unique restock source, audited positive deltas. Refund may be pending/failed independently; inspection controls stock. Automatic retries after transport uncertainty keep the same refund identity. After verified terminal nonrefund, an audited operator may submit a new refund command/key with a reference to the failed command and a fresh guarded refundable-balance check; never replace an uncertain refund to bypass a duplicate result.

Shipping: manual zone/flat-rate tables keyed to approved versioned Vietnamese address codes. Explicit reject unknown/unsupported zones. Taxes: approved inclusive/exclusive product/shipping classifications, rate/rule effective dates and invoice rounding. Money contract: subtotalMinor=sum(display unit price×quantity); shippingMinor=display delivery fee; taxMinor=total tax disclosure (included+additional); additionalTaxMinor=only tax payable beyond those displayed amounts; totalMinor=subtotalMinor+shippingMinor+additionalTaxMinor. Included tax is never added again. Order line_total_minor includes its additional_line_tax_minor; line_tax_minor is total informational tax. Shipping refundable charge similarly includes any additional shipping tax, so tax is not refunded twice. Whole-đồng line calculations use BigDecimal or integer rational arithmetic, HALF_UP at approved calculation boundaries; preserve the approved policy/version/allocation residuals. Production task supplies approved reference cases and tests; no tax rate is fabricated here. Support partial refunds and exact residual allocation to avoid refund rounding exceeding capture.

Operations baseline: HTTPS ingress; managed secrets; private database/network; migration coordinator; daily full plus point-in-time backup capability for proposed15mRPO; proposed4hRTO and99.5% monthly API availability pending operator acceptance/evidence. Alert thresholds as TRD. Store runbooks for deployment, migration/rollback compatibility, payment ambiguity, aged holds, refund uncertainty, restore/reconciliation, and customer support. Restore must reconcile live financial state before resuming mutating jobs. Validate PII retention/deletion/customer policies against current Vietnamese requirements before launch; operational/legal approvals are real launch inputs, not blank code TODOs.

## S10 — Evidence, delivery ownership, and traceability

Owners: backend engineer owns money/auth/inventory/payment/commerce; algorithm engineer owns catalog evidence/rules/search/shared fixtures; frontend engineer owns UX/accessibility/browser tests; all share integration, with one designated migration/deployment coordinator. Named production operators/legal/merchant contacts are added to the launch record before go-live.

Core fixtures include: mixed standalone/build cart, two groups sharing CPU, direct incompatible grouped checkout, malformed category/spec, exact power/fit boundaries, greedy budget trap, no compatible catalog, timeout with/without incumbent, three-attempt conflict retry, ambiguous commit lookup, fifty last-unit buyers, ample stock successes, same-key concurrent initiation, held→paid no deduction, repeated release/expiry, authorization vs capture, cancel/capture races both orders, terminal-failure replacement, old-attempt callbacks, replay/identity mismatches, creation/capture/refund crash recovery, price policy change, cross-tab cart preservation, auth counter/rotation races, real Redis outage/revision checks.

Journey instrumentation: optional validated `X-Journey-Id` UUID accompanies add-build/quote/initiation; persist it with the group/order snapshot. A request-scoped JourneyContext carries it to events; workers use the saved order value. BUILD_ADDED_TO_CART and ORDER_CONFIRMED event rows are recorded in the backend business transaction and become visible/published only after commit; stable group/order-derived event_key prevents replay. Client start events use server timestamps and (journey_id,event_type) identity; thirty-minute cohorts count noncompleted attempts. Metric events never authorize payment and contain no raw customer data. D4 owns telemetry controller/events; the V3 migration reserves the table/optional group/order journey fields.

CI records unit/integration discovery counts and fails on zero discovered suites. Clean environment runs migrations/builds/contracts/Playwright. Provider stub tests are not live evidence. A MoMo sandbox smoke and controlled live commissioning are separate gated runs; CI never calls live payments or sends real customer mail. Production adds policy/refund/restock/MFA/privacy/security/load/restore evidence.

The [documentation index](../../README.md#requirement-coverage) maps R01–R24 to tasks and required evidence. The [integration plan](../plans/2026-10-08-integration-delivery.md) defines connected verification, and [team assignments](../../TEAM_TASK_ASSIGNMENTS.md) name owners and reviewers. Completion requires recorded outcomes, not checked boxes copied from this specification.
