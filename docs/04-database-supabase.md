# 04 — Database & Supabase Documentation

**Status:** Data requirements only. No schema, migration, or deployed function is asserted to exist.

## 1. Database principles

- Reuse the application's existing order and order-item schema.
- Do not create duplicate product or order tables.
- Inspect the real database schema before proposing migrations.
- Preserve the existing order lifecycle where possible.
- The exact SQL schema and migration are intentionally unspecified until inspection.

## 2. Required order data

Existing order records must provide, directly or through related records:
- Stable internal order ID.
- Authenticated customer/user ID.
- Authoritative order total and currency.
- Order status and payment eligibility.
- Created/updated timestamps.

Order items and prices must represent values agreed at checkout. Trusted backend logic calculates the payable total.

## 3. Payment attempt data

Persist a payment record linked to its order with at minimum:
- Internal payment ID and related internal order ID.
- Unique Midtrans `order_id` per distinct transaction attempt.
- Amount and currency submitted to Midtrans.
- Midtrans `transaction_id`, when available.
- Midtrans transaction status and fraud status, when available.
- Payment type, when available.
- Created/updated and last verification/notification timestamps.

Enforce uniqueness on Midtrans `order_id`. Persist enough data to match notifications to the exact attempt and safely handle retries. Do not store card numbers, CVV, or other sensitive payment credentials. Do not persist Snap token without a concrete requirement.

## 4. Row Level Security and access

- Enable RLS on tables exposed through the Supabase Data API.
- Customers may read only their own permitted order and payment information.
- Clients must not write trusted payment status, Midtrans transaction identifiers, verified amounts, or fraud status.
- Only trusted backend code may apply payment-state changes.
- Use elevated Supabase credentials only inside Edge Functions when required.

Actual policy expressions must be based on the inspected schema and user/order relationship. Do not invent table or column names.

## 3.1 Attempt uniqueness, locking, and state semantics

These are logical requirements, not a prescribed SQL schema:
- Enforce a unique database constraint/index for each Midtrans order_id.
- Serialize payment-attempt creation per internal order using a database transaction with row-level locking or an equivalent atomic compare-and-set mechanism. A frontend button lock is only a UX aid and is not a concurrency guarantee.
- Treat creating, creation_unknown, token_ready, and pending as active attempt states. A second concurrent request must reuse the active attempt or return a safe conflict.
- A new distinct attempt is eligible only after the previous attempt is conclusively terminal (failed, expired, or cancelled) and the order remains unpaid and eligible. A 404 Payment not found from Get Status after Snap token creation is not a terminal state.
- Keep attempt status separate from order payment status. An attempt's failure cannot override another attempt's verified success. Once the order payment state is paid, it is immutable to non-paid transitions.
- Conceptual state labels must be mapped to the actual schema only after inspection; do not invent table, column, enum, or policy names.

Do not rely on generic Midtrans Idempotency-Key support for Snap Create Transaction unless endpoint-specific support is confirmed. Local atomicity and unique identifiers are mandatory regardless.

## 3.2 Amount and currency contract

Version 1 is IDR-only. Trusted backend logic derives the payable total from saved order/item data, using an exact integer rupiah representation; do not use floating-point arithmetic. Persist the exact amount and IDR currency on the payment attempt. Match amount and currency against the order, Snap request, notification, and Get Status response before accepting a paid state. Reject a mismatch or non-IDR order; never silently round. Midtrans may represent notification/status gross_amount as a decimal string such as 150000.00; parse it as an exact decimal and compare numerically to the expected integer rupiah amount, without binary floating-point comparison.

## 4.1 Notification persistence and acknowledgement

The notification handler must durably persist the verified attempt/order transition before returning HTTP 2xx. A duplicate notification that is already durably reconciled may return 2xx without creating duplicate records. Invalid JSON, invalid signature, or an unknown/mismatched order_id must not change payment data and should receive a client-error response. If Midtrans Get Status, the database, or another required dependency is unavailable, do not mark the event handled and do not return 200; return a retryable server error (503 is preferred for temporary dependency failure). Keep the handler fast, targeting less than five seconds, and make processing idempotent because notifications can be retried or arrive out of order.

## 4.2 Credential and client access boundary

Customer-facing clients may read only the minimum order/payment summary needed for their own authorized account, through RLS and/or a backend authorization check. They must never write trusted attempt state, order payment state, amount/currency verification, Midtrans transaction identity, signature result, or fraud status. Elevated credentials may bypass RLS only inside trusted Edge Functions; every such function must still validate ownership for customer-initiated actions. The external notification function has no customer session, so it must authenticate Midtrans using signature verification and then resolve the exact stored attempt. Do not expose service-role credentials or assume that merely enabling RLS makes an elevated backend action safe.

## 5. Supabase Edge Functions

### `create-snap-transaction`
- Requires a valid Supabase user session.
- Validates request and internal order ID.
- Confirms order ownership and payment eligibility.
- Reads trusted order total and currency.
- Does not accept client amount, customer ID, status, or paid flag as authoritative.
- Atomically claims or reuses the persisted active attempt under a per-order lock.
- On timeout/lost response, persists creation_unknown and retains the same Midtrans order_id; never creates a second identifier until reconciliation.
- Enforces concurrency so at most one active attempt exists per order.
- Calls Midtrans Sandbox using the Server Key.
- Returns Snap token and minimal references only.
- Never marks order paid merely because transaction creation succeeded.

### `midtrans-notification`
- Accepts Midtrans HTTP POST notifications.
- Does not require a Supabase user JWT because Midtrans does not send one.
- Must still verify Midtrans signature and transaction details itself.
- Finds the exact attempt by Midtrans `order_id`.
- Verifies status with Midtrans Sandbox status API.
- Queries status and validates exact amount/currency before applying state changes.
- Persists the state transition durably before returning HTTP 2xx.
- Returns 2xx for safely processed/confirmed duplicate notifications; invalid signature/unknown identifier receives a client error; transient status/database failures receive a retryable server error (prefer 503), not 200.
- Updates attempt/order idempotently and prevents invalid state regression.

Example deployment flag for the notification endpoint: `--no-verify-jwt`. This disables Supabase JWT verification only; it does not disable Midtrans signature verification.

## 6. Credentials and secrets

| Name | Location | Classification |
|---|---|---|
| `MIDTRANS_SERVER_KEY` | Supabase Edge Function secret | Backend secret |
| `MIDTRANS_CLIENT_KEY` | Frontend build configuration | Public Sandbox key |
| Supabase URL | Frontend build configuration | Public |
| Supabase anon/publishable key | Frontend build configuration | Public; protected by RLS |

Never expose Midtrans Server Key or Supabase service-role key to the browser.

## 7. Migration and implementation safeguards

Before writing migrations:
1. Inspect existing tables, columns, foreign keys, indexes, and RLS policies.
2. Identify existing order ownership and status conventions.
3. Determine whether a payment table or equivalent already exists.
4. Propose the smallest additive schema change.
5. Review migration safety and rollback implications before applying it.

No database migration is part of the current documentation-only change.
