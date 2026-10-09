# Electronics Store documentation

**Active revision:** 3.0, 2026-10-08. Documentation is reconciled; application implementation and its verification have not started.

The user selected a larger application with live payments/production operations, Vietnam/VND, and MoMo first (VNPAY later). The twelve-week student project is an integrated core milestone; it is not the production-launch deadline.

## Read in this order

1. [PRD](ELECTRONICS_STORE_PRD.md): purpose, user journeys, R01–R24 acceptance requirements and core/launch priorities.
2. [TRD](ELECTRONICS_STORE_TRD.md): engineering invariants, trust boundaries, concurrency, payment recovery, security and operations.
3. [Design specification](superpowers/specs/2026-10-06-electronics-store-design.md): canonical schema, field names, APIs, state machines, algorithm and launch contracts.
4. [Integration/delivery plan](superpowers/plans/2026-10-08-integration-delivery.md): sequencing, connected verification and core evidence.
5. Domain plans listed below, then the [production launch plan](superpowers/plans/2026-10-08-production-launch.md).
6. [Team task assignments](TEAM_TASK_ASSIGNMENTS.md): ownership for Phat, Bảo and Huy, reviewers, handoffs and first working session.
7. [Individual task guides](tasks/README.md): detailed business rules, prerequisites, outputs, acceptance scenarios and review evidence for each member.

Product intent comes from the PRD; exact implementation contracts come from the specification; task plans implement those contracts. If they conflict, resolve the documents together before coding. This folder contains the current requirements, specification, implementation plans and team assignments.

## Implementation plans and dependencies

| Plan | Task IDs | Owns |
| --- | --- | --- |
| [Backend foundation](superpowers/plans/2026-10-07-backend-foundation.md) | F1–F7 | Identity, exact money, catalog/cache, grouped cart, admin, quotas; V1 migration |
| [Builder/optimizer](superpowers/plans/2026-10-07-pc-builder-optimizer.md) | B1–B6 | Rules, draft history, Python search, Java boundary, optional explanation; V2 migration |
| [Checkout/recovery](superpowers/plans/2026-10-07-checkout-pipeline.md) | C1–C7 | Quotes, atomic holds/idempotency, MoMo, closing, retries, receipts/compensation; V3 migration |
| [Frontend](superpowers/plans/2026-10-07-frontend.md) | U1–U6 | Session client, catalog, builder, cart/quote, checkout/history, admin/accessibility |
| [Integration/delivery](superpowers/plans/2026-10-08-integration-delivery.md) | D1–D4 | Compose/seeds, contracts/E2E, concurrency/restart/sandbox, CI/evaluation |
| [Production launch](superpowers/plans/2026-10-08-production-launch.md) | P1–P7 | Merchant readiness, email/MFA, commerce, returns/refunds, governance, operations/launch; V4–V7 migrations |

Execute F1–F4 first; B1 then unblocks F5/F6. F7 can follow F3/F4. Checkout requires these foundation tasks and B1. Python search/drafts and UI can advance against shared schemas while checkout is implemented. D1 follows service scaffolds; D2–D4 prove integrated behavior. B6 is optional and requires an AI provider/model decision before live enablement. P1 can gather merchant/operational inputs early; no live enablement occurs before P2–P7 evidence.

Suggested core sequence for three students:

| Weeks | Milestone |
| --- | --- |
| 1–2 | Runtime/schema/contracts, accounts/catalog, UI/Python scaffolds; obtain sandbox access and designate migration coordinator |
| 3 | Supported rules, grouped cart, admin validation/stock audit, request protection |
| 4–5 | Saved drafts/search, quotes/atomic allocation, catalog/builder/cart UI |
| 6–7 | MoMo durable jobs/authorization/capture/closing/retry, optimizer revalidation, checkout UI |
| 8 | Receipts/compensation/fulfillment/admin recovery views |
| 9–10 | Full-stack contract/E2E/concurrency/restart checks and MoMo sandbox smoke |
| 11–12 | Fix remaining core failures, record CI/load/usability/algorithm evidence; review production estimate |
| After core, estimated separately | P1–P7 commerce/security/operations and controlled live commissioning |

This is sequencing guidance, not an achieved schedule. Early merchant or data-quality problems can change it; reduce enhancements before weakening checkout correctness.

## Decisions and assumptions

| Topic | Decision/source |
| --- | --- |
| Product ambition | User: larger application, live payments and production operations |
| Market/currency | User: Vietnam/VND; one whole đồng per application minor unit |
| First gateway | User: MoMo first; VNPAY afterward |
| Inventory concurrency | User: fresh backend version, JPA @Version, whole-order allocation and bounded retries |
| Payment hold | User: ten minutes, retain on recoverable error, safe compensation rather than long DB locks |
| Provider-specific design | Two-step MoMo authorization/capture, subject to merchant capability validation. A failed terminal provider attempt can be replaced within the same retained order/window |
| Hardware scope | Initial design: seven slots, discrete GPU, one RAM kit/NVMe drive, vetted bundled cooling and platform evidence; broader hardware is a later enhancement |
| Purchase identity | Design: account-required persistent cart/checkout, user-scoped UUID idempotency and server snapshots; guest checkout later |
| Shipping | Design: manually configured supported zones/flat rates; actual zones/rates need business approval |
| Taxes/returns/privacy | Required production inputs and reviewed reference cases/policies; no fabricated rate or legal approval |
| Operations | One-region/single-warehouse start; owners, hosting budget and accepted service/restore objectives need launch record |
| AI | Optional, disabled until provider/model/cost/data decision; no impact on purchase correctness |

MoMo uses the stock accounting/version/hold invariants defined in the [TRD](ELECTRONICS_STORE_TRD.md) and [specification](superpowers/specs/2026-10-06-electronics-store-design.md). Pending authorization does not imply the confirm API can cancel a waiting wallet transaction. A closing order can retain stock past ten minutes while provider state is uncertain.

## Requirement coverage

| Requirement | Implementation tasks | Required evidence |
| --- | --- | --- |
| R01 Discovery | F4, U2, D2 | Live stock/search/filter/page/detail journeys |
| R02 Accounts | F3, U1, D2 | Concurrent register/counters/rotation/logout/CSRF |
| R03 Cart groups | F2, F5, U4, D2 | Atomic seven-slot groups, shared products, concurrent adds |
| R04 Manual builder | B1, U3, D2 | Partial/complete states, authoritative add, mobile/keyboard |
| R05 Drafts | B2, U3, D2 | Saved price baseline, delist notices, ownership/revisions |
| R06 Compatibility | B1, F6, B4, D2 | Boundary/invalid/valid fixtures and cross-language parity |
| R07 Optimizer | B3–B5, U3, D4 | Exhaustive oracle, constraints, deadlines/status/proof checks |
| R08 Optional explanation | B6, U3 | Deadline/error/hash/PII controls; separate actual-provider test if enabled |
| R09 Price review | C1, U4, P3, D2 | Quote expiry/repricing, exact totals/consent |
| R10 Versioned stock | F2, C2, C4/C5, D3 | Last-unit/ample-stock/version/whole-order tests |
| R11 Idempotency | C2, C6, U5, D2/D3 | Replay/reload/concurrent keys and uncertain commit |
| R12 Recovery | C3–C7, U5, D3 | Create/authorize/capture/closing/refund restart/race tests |
| R13 Orders | C7, U5/U6, D2 | Owned immutable receipts, cleanup, fulfillment |
| R14 Admin | F6, B1, C7, U6, D2 | Publication preview, audited deltas, recovery visibility |
| R15 Security | F3/F7, B3/B5, C3/C4, U1, P2/P6/P7 | Ownership/CSRF/quotas/internal auth/signature/security review |
| R16 UX | U2–U6, D2 | Accessibility/mobile/recoverable error journeys |
| R17 Delivery | F1, B3, U1, D1–D4 | Fresh Compose/CI/build/schema/E2E/sandbox evidence |
| R18 Evaluation | B4, D3/D4 | Algorithm oracle, safety plus success metrics, bounded pilot study |
| R19 Live identity/payment | P1/P2/P7 | Merchant two-step approval, verified account/recovery/MFA, live commissioning |
| R20 Commerce | P3/P7 | Approved Vietnamese address/zones/tax cases and quote versioning |
| R21 Shipment/returns/refunds | P4/P5/P7 | Shipment/partial refunds/inspection/restock/support |
| R22 Operations | P6/P7 | Capacity/alerts/backup restore/rollback/incidents/owners |
| R23 Trust/governance | P2/P5/P6/P7 | Policies/retention/PII/MFA/secrets/security/customer data requests |
| R24 Launch | P7 | Complete evidence ledger, no critical findings, controlled rollout |

Each row is designed/planned, not implemented or verified. The [production launch plan](superpowers/plans/2026-10-08-production-launch.md) defines the required business inputs, operational evidence and launch gates.
