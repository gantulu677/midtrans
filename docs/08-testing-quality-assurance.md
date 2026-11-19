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
| T-13 | Concurrent Pay Now | At most one active attempt per order. |
| T-14 | Snap closes | UI refreshes backend status; no outcome inferred from close. |
| T-15 | Snap API timeout/lost response | Existing attempt/order ID reconciled before retry/new attempt. |
| T-16 | Frontend asset inspection | No Server Key or service-role key in built assets. |
| T-17 | Sandbox isolation | Browser Snap.js and backend API use Sandbox only. |

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
