# 08 — Testing & Quality Assurance

**Status:** Test plan and acceptance contract. No tests are claimed to exist or have passed.

## 1. Test objectives

Verify that:
- Existing storefront and checkout remain functional.
- Orders and amounts are created from trusted data.
- Only authorized customers can initiate/read their orders.
- Payment success is determined only by backend verification.
- Notifications are secure, idempotent, and safe under concurrency/retry.
- Every Midtrans interaction uses Sandbox.

## 2. Required functional and integration tests

| ID | Test | Expected result |
|---|---|---|
| T-01 | Select product and checkout | Existing storefront flow remains intact. |
| T-02 | Tamper with client amount | Backend ignores/rejects client amount and uses trusted order total. |
| T-03 | Pay another user's order | Request is denied. |
| T-04 | Pay ineligible order | Backend rejects payment creation. |
| T-05 | Valid settlement notification | Matching order becomes paid after verification. |
| T-06 | Accepted capture | Paid only when status and fraud rules pass. |
| T-07 | Pending/denied/expired/cancelled | Correct non-paid state; no false success. |
| T-08 | Invalid signature | Notification rejected. |
| T-09 | Unknown order ID | Notification rejected without changing any order. |
| T-10 | Amount/currency mismatch | Not marked paid. |
| T-11 | Duplicate notification | Same attempt reconciled; no duplicate record. |
| T-12 | Stale/out-of-order notification | Paid state does not regress. |
| T-13 | Concurrent Pay Now | Database serialization and unique order_id constraint ensure one active attempt; concurrent calls reuse it or return a safe conflict. |
| T-14 | Snap closes | UI refreshes backend status; no outcome inferred from close. |
| T-15 | Snap API timeout/lost response | Attempt becomes creation_unknown; same Midtrans order_id is retained and reconciled. No new identifier is generated merely due to timeout. |
| T-18 | Snap Get Status returns 404 before payment-method selection | Treat as expected unresolved Snap state, not creation failure; retain same order_id and do not create a new distinct attempt. |
| T-19 | Lost Snap token response; identifier still unused | Backend may recreate token with the same order_id; prior token invalidation is handled; no duplicate internal attempt is created. |
| T-20 | Status API unavailable during reconciliation | Keep creation_unknown; no new attempt; retry reconciliation with backoff. |
| T-21 | Concurrent Pay Now requests across sessions | Database lock/atomic claim serializes calls; unique order_id constraint prevents duplicate identifiers. |
| T-22 | Failed attempt followed by a new successful attempt | Failed attempt remains failed; successful attempt makes order paid; failed notification cannot regress paid order. |
| T-23 | Out-of-order notification after settlement | Current Midtrans status is queried; paid order remains paid. |
| T-24 | Notification arrives while Get Status API is unavailable | No false state change and no HTTP 200 acknowledgment; transient failure returns retryable 503. |
| T-25 | Notification persistence fails | No 2xx acknowledgment; retry can safely process the event later. |
| T-26 | Duplicate notification after successful persistence | Return 2xx idempotently without duplicate records. |
| T-27 | Invalid signature / unknown order identifier | Reject with client error and no state mutation. |
| T-28 | Amount or currency mismatch, including decimal-formatted amount | Reject payment success; no rounding or floating-point acceptance. |
| T-29 | Non-IDR order | Payment creation is rejected in v1. |
| T-30 | Customer attempts direct payment-field write | RLS/grants deny the write; trusted backend remains the only writer. |
| T-16 | Frontend asset inspection | No Server Key or service-role key in built assets. |
| T-17 | Sandbox isolation | Browser Snap.js and backend API use Sandbox only. |

## 2.1 State-machine and amount test expectations

Test payment-attempt state independently from order payment state. Include valid and invalid transitions, repeated events, a failed attempt followed by a successful distinct attempt, and stale non-paid notifications after the order is paid. Confirm paid is monotonic at the order level. Confirm v1 rejects non-IDR orders and compares amounts exactly using integer rupiah semantics, including Midtrans decimal-string representations such as 150000.00.

## 3. UI state tests

Verify loading, ready-to-pay, creating transaction, pending, paid, failed/expired/cancelled, Snap closed, and error states. Confirm Pay Now is disabled during transaction creation and retry is exposed only when backend eligibility permits it.

## 4. Security tests

- RLS blocks cross-customer reads.
- Client cannot write trusted payment fields.
- Payment creation validates session, ownership, and eligibility.
- Notification signature and status checks reject tampered data.
- Unknown/mismatched transaction identifiers are rejected.
- Secrets are absent from frontend bundles and source control.
- Repeated and stale notifications cannot duplicate or regress state.

## 5. Test environment and evidence

Use Midtrans Sandbox credentials and documented Sandbox test scenarios. Record test date, environment, build/commit identifier, test case ID, result, and relevant sanitized evidence. Never include credentials, card secrets, or unnecessary personal data in test logs.

## 6. Release gate

Do not call the integration production-ready merely because the UI opens Snap. All applicable acceptance criteria in [01 — Business & Product Requirements](./01-business-product-requirements.md) must pass, including timeout reconciliation, access control, and backend-only payment verification.
