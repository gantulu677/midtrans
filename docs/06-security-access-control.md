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

- Enforce unique Midtrans `order_id` for each distinct attempt.
- Serialize concurrent Pay Now requests; at most one active attempt per order.
- Duplicate notifications reconcile the same attempt.
- Verified paid state cannot regress due to stale or out-of-order notifications.
- Failed/expired/cancelled attempt cannot override a different successful attempt.
- Unknown Snap creation outcome must be reconciled using the same Midtrans order identifier before a new attempt is considered.
- A timeout is not proof that Midtrans failed to create a transaction.

## 7. Data minimization

Do not store card numbers, CVV, or other sensitive payment credentials. Do not persist Snap token unless an explicit technical need is established. Persist only payment metadata required for reconciliation, auditability, and customer status display.

## 8. Security references

- [Supabase RLS and database security](https://supabase.com/docs/guides/database/secure-data)
- [Supabase Edge Function authorization](https://supabase.com/docs/guides/functions/auth-headers)
- [Supabase Edge Function secrets](https://supabase.com/docs/guides/functions/secrets)
- [Midtrans notification handling](https://docs.midtrans.com/reference/handle-notifications)
