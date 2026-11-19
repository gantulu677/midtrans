# 05 — API & Integration Documentation

**Status:** Integration contract; endpoints and functions are planned, not verified as deployed.

## 1. Midtrans environment

Sandbox only.

| Purpose | Sandbox endpoint |
|---|---|
| Create Snap transaction | `https://app.sandbox.midtrans.com/snap/v1/transactions` |
| Snap.js | `https://app.sandbox.midtrans.com/snap/snap.js` |
| Get transaction status | `https://api.sandbox.midtrans.com/v2/{order_id_or_transaction_id}/status` |

Use the Sandbox Client Key in the browser and Sandbox Server Key in backend secrets. No Production endpoints or keys are permitted.

## 2. Frontend-to-backend: create transaction

### Function
`create-snap-transaction`

**Caller:** authenticated frontend.  
**Input:** internal order ID only.

Responsibilities:
- Require valid Supabase user session.
- Validate request and order ID.
- Confirm order belongs to caller and is eligible.
- Read trusted order total and currency.
- Never trust client amount, customer ID, order status, or paid flag.
- Serialize requests per internal order using database transaction/row lock or equivalent atomic claim; do not rely on a frontend lock.
- Reuse the existing active attempt and its Midtrans order_id when one exists.
- Generate a unique Midtrans order_id (maximum 50 characters) only for a new distinct attempt after the previous attempt is conclusively terminal and the order remains eligible.
- Persist the attempt before calling Midtrans; use creating, creation_unknown, token_ready, pending, paid, failed, expired, and cancelled as conceptual attempt states, mapped to actual schema after inspection.
- Enforce a unique database constraint/index on Midtrans order_id.
- Return Snap token and minimal references.
- Never mark an order paid when Snap creation succeeds.

If the Snap create request times out, the response is lost, or processing is interrupted after dispatch, persist/retain creation_unknown and the same Midtrans order_id. A timeout is not proof of failure. Do not issue a new distinct order_id until reconciliation proves the prior attempt terminal and the order remains eligible. The exact JSON response shape must be finalized alongside implementation and should expose no secret or unnecessary internal data.

## 2.1 Snap creation recovery and Get Status 404

Midtrans documents that after a Snap transaction/token is created, Core API Get Status can return 404 Payment not found until the customer selects/proceeds with a payment method. This response is not evidence that Snap token creation failed and is not a terminal payment state.

Recovery rules:
1. Keep the persisted attempt and its original Midtrans order_id.
2. If Get Status returns 404 before a payment method is selected, keep the attempt unresolved/available for the same Snap session; do not create a new identifier.
3. If the token response was lost and Midtrans confirms the identifier remains unused, a new Snap token may be created using the same order_id. Midtrans documents that recreating a token for an unused identifier invalidates the earlier token.
4. Once a payment method has been selected and a transaction status such as pending exists, do not try to create a new Snap transaction with that used order_id. Reopen the existing token if still valid, or reconcile the transaction.
5. If status lookup is unavailable or the state is ambiguous, retain creation_unknown and wait/retry reconciliation with backoff. Never create a new identifier solely because of timeout or 404.

A single Snap order_id can receive multiple payment-method attempts/notifications within Snap's supported retry behavior. Treat these as updates to the same Midtrans attempt, not as separate internal payment attempts. A distinct internal attempt/new Midtrans order_id is considered only after the prior Snap transaction is conclusively terminal and order eligibility is rechecked.

## 2.2 Idempotency and concurrent requests

The v1 safety guarantee comes from local database serialization per order and a unique constraint on Midtrans order_id. Do not assume Midtrans generic Idempotency-Key behavior applies to POST /snap/v1/transactions; only use it if Midtrans confirms support for this exact endpoint. Concurrent Pay Now requests must resolve to the same active attempt or a safe conflict. Active attempt states are creating, creation_unknown, token_ready, and pending.


## 3. Midtrans Snap browser integration

- Load Sandbox Snap.js using the Sandbox Client Key.
- Call `snap.pay(token, ...)` with the token returned by the Edge Function.
- Treat `onSuccess`, `onPending`, `onClose`, and `onError` as UI events only.
- Callbacks may trigger an authorized order/payment status refresh.
- Do not write payment status from the browser.

## 4. Midtrans notification

### Function
`midtrans-notification`

**Caller:** Midtrans notification service via HTTP POST.

Verify `signature_key` using SHA-512 over the concatenated values:

```text
order_id + status_code + gross_amount + MIDTRANS_SERVER_KEY
```

Then:
1. Find payment attempt by exact Midtrans `order_id`; reject unknown/mismatched identifiers.
2. Require the notification amount to match the saved attempt and order. Require currency to be IDR when the field is present; the v1 order and attempt are always IDR.
3. Query Midtrans Sandbox Get Transaction Status API.
4. Confirm status response refers to the expected transaction, exact amount, and IDR currency.
5. Treat as paid only when verified response has `status_code` `200` and `transaction_status` is `settlement` or accepted `capture`.
6. If `fraud_status` is present, require `accept` for both settlement and capture.
7. Persist verified details and update payment/order state atomically where practical.
8. Handle repeated and out-of-order notifications idempotently.

A verified paid order must never regress to pending, failed, expired, or cancelled. A failed attempt must not override a separate verified successful attempt.

Deploy this external endpoint without Supabase JWT verification when necessary (e.g. `--no-verify-jwt`), while always enforcing Midtrans signature and status verification internally.

### Notification HTTP response contract

| Condition | HTTP response | Required behavior |
|---|---:|---|
| Valid notification; status verified and state durably persisted | 200 | Acknowledge. |
| Confirmed duplicate already durably reconciled | 200 | Idempotent no-op; do not duplicate records. |
| Malformed body, invalid signature, unknown/mismatched order_id | 400 | Reject; no state mutation. |
| Midtrans status API unavailable/rate-limited, or database persistence unavailable | 503 | Do not acknowledge; allow provider retry. |
| Unexpected internal processing error | 500 | Do not acknowledge as successful. |

Persist the verified state before returning 2xx. Target a response time below five seconds. Notifications may be duplicated or arrive out of order; re-query status, apply monotonic transitions, and make the handler idempotent. Do not return 200 merely because the HTTP request was received.

## 5. Transaction status lookup

Use the Sandbox status API only from trusted Edge Function code. Use the identifier supported for the transaction/method (`order_id` or `transaction_id`, as applicable). Persist verified `transaction_id` when available. Never expose Server Key or allow the frontend to query arbitrary Midtrans transactions.

## 6. Authentication for Midtrans API requests

Use HTTP Basic Authentication: Base64-encode `MIDTRANS_SERVER_KEY:` (Server Key as username, empty password).

Version 1 is IDR-only. Calculate gross_amount from trusted saved order data and send rupiah as an integer (e.g. 150000 for Rp150,000), not cents. Do not use floating-point arithmetic or silently round. Persist the exact integer amount and IDR currency on the attempt. Compare order, attempt, Snap request, notification, and status response. Notification/status may encode gross_amount as a decimal string (for example 150000.00); parse as exact decimal and compare numerically, not with binary floating point. Reject non-IDR orders and any amount/currency mismatch.

## 7. Official references

- [Midtrans Snap Integration Guide](https://docs.midtrans.com/docs/snap-snap-integration-guide)
- [Midtrans Access Keys](https://docs.midtrans.com/docs/access-keys)
- [Create Snap Transaction](https://docs.midtrans.com/reference/backend-integration)
- [HTTP(S) Notifications / Webhooks](https://docs.midtrans.com/docs/https-notification-webhooks)
- [Handle Notifications](https://docs.midtrans.com/reference/handle-notifications)
- [Get Transaction Status](https://docs.midtrans.com/reference/get-transaction-status)
- [Midtrans Snap Advanced Features — token recreation and multiple payment attempts](https://docs.midtrans.com/docs/snap-advanced-feature)
- [Midtrans Transaction Status Cycle — Snap status 404 behavior](https://docs.midtrans.com/docs/transaction-status-cycle)
- [Midtrans Best Practices to Handle Notifications](https://docs.midtrans.com/reference/best-practices-to-handle-notification)
