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

## 5. Supabase Edge Functions

### `create-snap-transaction`
- Requires a valid Supabase user session.
- Validates request and internal order ID.
- Confirms order ownership and payment eligibility.
- Reads trusted order total and currency.
- Does not accept client amount, customer ID, status, or paid flag as authoritative.
- Creates/reuses a persisted payment attempt.
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
