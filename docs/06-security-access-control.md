# 06 — Security & Access Control

**Status:** Mandatory security requirements for planned implementation.

## 1. Trust model

The browser is untrusted. Only trusted backend code may determine and persist verified payment status. Snap callbacks, return URLs, redirects, and client-submitted status are not proof of payment.

## 2. Authentication and authorization

- Payment creation requires a valid Supabase user session.
- Backend verifies that the requested order belongs to the authenticated user.
- Backend verifies order payment eligibility before creating/reusing an attempt.
- Customer order/payment reads must be protected by RLS and authorization checks.
- Never authorize order access solely by a public `order_reference`.
- Midtrans notification endpoint is an external service endpoint; it must independently authenticate notifications using Midtrans signature verification.

## 3. Secret management

- Keep `MIDTRANS_SERVER_KEY` in Supabase Edge Function secrets only.
- Keep Supabase service-role/elevated credentials in backend secrets only.
- Frontend may contain only browser-safe configuration such as Supabase URL, anon/publishable key, and Sandbox Midtrans Client Key.
- Never define `VITE_MIDTRANS_SERVER_KEY` or `VITE_SUPABASE_SERVICE_ROLE_KEY`.
- Treat every Vite `VITE_` variable as public after bundling.
- Do not commit credentials or local secret files.

## 4. Payment verification

Before marking paid:
- Verify notification signature using SHA-512 and the Server Key.
- Match exact Midtrans `order_id` to a persisted payment attempt.
- Verify amount and currency against saved records.
- Query Midtrans Sandbox transaction status from backend.
- Require verified `status_code` `200` and `settlement` or accepted `capture`.
- If fraud status is present, require `accept`.
- Reject unknown, mismatched, or invalid notifications.

## 5. Database protection

- Enable RLS on exposed tables.
- Restrict customer reads to their own records.
- Prevent client writes to trusted payment status, verified amount, Midtrans transaction identity, and fraud status.
- Apply state changes only through trusted backend logic.
- Use elevated credentials only inside Edge Functions when required.

## 6. Idempotency and state integrity

- Enforce a database unique constraint/index on every persisted Midtrans order_id.
- Serialize concurrent Pay Now requests per internal order with a database transaction/row lock or equivalent atomic operation; a frontend disabled button is not a security/concurrency control.
- Treat creating, creation_unknown, token_ready, and pending as active attempt states. Reuse the active attempt or return a safe conflict; do not create a second distinct attempt while one is active.
- Create a new distinct attempt only after the prior attempt is conclusively terminal (failed, expired, or cancelled) and the order remains unpaid and eligible.
- Do not assume generic Midtrans Idempotency-Key support applies to Snap Create Transaction unless endpoint-specific support is confirmed.
- Duplicate notifications reconcile the same attempt.
- Verified paid state cannot regress due to stale or out-of-order notifications.
- Failed/expired/cancelled attempt cannot override a different successful attempt.
- Unknown Snap creation outcome must retain creation_unknown and be reconciled using the same Midtrans order_id before another distinct attempt is considered.
- A Core API Get Status 404 Payment not found can be normal after token creation and before payment-method selection; it is not proof of failed creation or a terminal state.
- If a lost token response must be recovered, recreate the token with the same unused order_id only when backend reconciliation confirms that it has not been used. Recreating a token invalidates the previous token. If a payment method has already been selected/status assigned, reuse the existing token if valid or reconcile; do not create another Snap transaction with that used identifier.
- A timeout is not proof that Midtrans failed to create a transaction.

## 6.1 Payment-state integrity and amount contract

Keep payment-attempt state separate from order payment state. A failed/expired/cancelled attempt must never overwrite a different verified successful attempt. Once an order is verified paid, no stale or out-of-order notification may regress it. Version 1 accepts IDR only. Backend derives an exact integer rupiah amount from saved order data; the attempt, Snap request, notification, and Get Status response must match that amount and IDR currency. Parse decimal-formatted Midtrans amount strings exactly; do not compare with binary floating point or round mismatches into acceptance. Reject non-IDR orders until multi-currency behavior is explicitly specified.

## 6.2 Notification acknowledgement safety

Return HTTP 2xx only after a valid notification has been verified and durably persisted, or when it is a confirmed duplicate already durably reconciled. Invalid signature, malformed payload, or unknown/mismatched order_id must not mutate state and should receive a client error. If Midtrans status lookup or database persistence is temporarily unavailable, do not return 200; return a retryable server error (prefer 503) so Midtrans can retry. Keep processing below five seconds where possible. Always query current Midtrans status and apply monotonic state transitions because notifications may be duplicated or out of order.

## 6.3 RLS and trusted database writes

RLS is required for customer-facing access but is not a substitute for backend authorization. Customer clients may read only the minimum order/payment summary for their own authenticated account. Revoke or omit client write privileges for trusted payment fields and order payment status. Elevated/service-role credentials bypass RLS and must remain in Edge Functions only; customer-initiated functions must still validate the authenticated user and order ownership before using elevated access. The unauthenticated notification function must instead verify Midtrans signature and exact persisted attempt identity before any write. Actual policies and grants must be designed only after inspecting the real schema.

## 7. Data minimization

Do not store card numbers, CVV, or other sensitive payment credentials. Do not persist Snap token unless an explicit technical need is established. Persist only payment metadata required for reconciliation, auditability, and customer status display.

## 8. Security references

- [Supabase RLS and database security](https://supabase.com/docs/guides/database/secure-data)
- [Supabase Edge Function authorization](https://supabase.com/docs/guides/functions/auth-headers)
- [Supabase Edge Function secrets](https://supabase.com/docs/guides/functions/secrets)
- [Midtrans notification handling](https://docs.midtrans.com/reference/handle-notifications)
- [Midtrans Snap Advanced Features](https://docs.midtrans.com/docs/snap-advanced-feature)
- [Midtrans Transaction Status Cycle](https://docs.midtrans.com/docs/transaction-status-cycle)
- [Midtrans Best Practices to Handle Notifications](https://docs.midtrans.com/reference/best-practices-to-handle-notification)
