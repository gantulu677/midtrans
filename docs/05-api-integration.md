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
- Generate unique Midtrans `order_id` for each distinct attempt within Midtrans constraints and Snap's 50-character maximum.
- Persist or safely reuse the active payment attempt.
- Return Snap token and minimal references.
- Never mark an order paid when Snap creation succeeds.

The exact JSON response shape must be finalized alongside implementation and should expose no secret or unnecessary internal data.

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
2. Compare notification amount and currency, where available, with saved attempt and order.
3. Query Midtrans Sandbox Get Transaction Status API.
4. Confirm status response refers to expected transaction and amount.
5. Treat as paid only when verified response has `status_code` `200` and `transaction_status` is `settlement` or accepted `capture`.
6. If `fraud_status` is present, require `accept` for both settlement and capture.
7. Persist verified details and update payment/order state atomically where practical.
8. Handle repeated and out-of-order notifications idempotently.

A verified paid order must never regress to pending, failed, expired, or cancelled. A failed attempt must not override a separate verified successful attempt.

Deploy this external endpoint without Supabase JWT verification when necessary (e.g. `--no-verify-jwt`), while always enforcing Midtrans signature and status verification internally.

## 5. Transaction status lookup

Use the Sandbox status API only from trusted Edge Function code. Use the identifier supported for the transaction/method (`order_id` or `transaction_id`, as applicable). Persist verified `transaction_id` when available. Never expose Server Key or allow the frontend to query arbitrary Midtrans transactions.

## 6. Authentication for Midtrans API requests

Use HTTP Basic Authentication: Base64-encode `MIDTRANS_SERVER_KEY:` (Server Key as username, empty password).

Calculate `gross_amount` from trusted order data and send an integer in the amount unit expected by Midtrans. For IDR, send rupiah as an integer (e.g. `150000` for Rp150,000), not cents. Keep amount consistent across order, attempt, Snap request, notification, and status verification.

## 7. Official references

- [Midtrans Snap Integration Guide](https://docs.midtrans.com/docs/snap-snap-integration-guide)
- [Midtrans Access Keys](https://docs.midtrans.com/docs/access-keys)
- [Create Snap Transaction](https://docs.midtrans.com/reference/backend-integration)
- [HTTP(S) Notifications / Webhooks](https://docs.midtrans.com/docs/https-notification-webhooks)
- [Handle Notifications](https://docs.midtrans.com/reference/handle-notifications)
- [Get Transaction Status](https://docs.midtrans.com/reference/get-transaction-status)
