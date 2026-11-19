# Midtrans Sandbox Integration — Technical Specification

This document defines **how the trusted payment integration must work**. It is the technical contract for database access, Supabase Edge Functions, Midtrans API calls, security, transaction consistency, retry, and idempotency.

**Environment:** Midtrans Sandbox only.
**Implementation status:** Planned. Inspect the actual application and database before implementing. Do not assume any schema or code exists.

## Architecture and Trust Boundaries

- **Frontend:** React 19 + Vite + TypeScript storefront, styled with Tailwind CSS v4 and using Konsta UI React v5 where appropriate. It requests payment creation and displays backend-reported order/payment status.
- **Supabase PostgreSQL:** source of truth for orders, order totals, and payment attempts.
- **Supabase Edge Functions:** trusted order/payment operations, Snap transaction creation, notification verification, and state updates.
- **Midtrans Sandbox:** payment processing and notification delivery.

The browser must never receive the Midtrans Server Key or Supabase service-role key. The frontend must never determine or write that an order is paid.

## Frontend Technology Constraints

- Use React 19, Vite, and TypeScript.
- Use Tailwind CSS v4 with the official `@tailwindcss/vite` Vite plugin.
- Use Konsta UI React v5 (`konsta`) for suitable reusable UI components. Konsta UI v5 targets React 19.
- Import Konsta's theme stylesheet from the application's main CSS entry point. Choose the Konsta `ios` or `material` theme consistently with the existing storefront; do not redesign unrelated screens.
- Keep payment logic and backend trust boundaries independent of presentation components. Konsta components must not write payment status or bypass the existing authorized data flow.
- If the actual repository's React version or frontend setup differs, inspect it before implementation and resolve compatibility deliberately; do not blindly replace the existing application structure.

## Order and Payment Data Requirements

Reuse the application's existing order schema. Do not create duplicate product or order tables.

### Order data

The existing order record must provide, directly or through related records:
- Stable internal order ID.
- Authenticated customer/user ID.
- Authoritative order total and currency.
- Order status and payment eligibility.
- Created/updated timestamps.

Order items and prices must represent the values agreed at checkout. Backend logic calculates the payable total from trusted order data.

### Payment attempt data

Persist a payment record linked to its order with, at minimum:
- Internal payment ID and related internal order ID.
- Unique Midtrans `order_id` for each distinct Midtrans transaction attempt.
- Amount and currency submitted to Midtrans.
- Midtrans `transaction_id`, when available.
- Midtrans transaction status and fraud status, when available.
- Payment type, when available.
- Created/updated and last verification/notification timestamps.

Enforce uniqueness on Midtrans `order_id`. Persist enough information to match notifications to the exact attempt and safely handle retries. Do not store card numbers, CVV, or other sensitive payment credentials. Do not persist the Snap token unless there is a concrete requirement.

### Database access and RLS

- Enable Row Level Security on tables exposed through the Supabase Data API.
- Customers may read only their own permitted order and payment information.
- Clients must not write trusted payment status, Midtrans transaction identifiers, verified amounts, or fraud status.
- Only trusted backend code may apply payment-state changes.
- Use elevated Supabase credentials only inside Edge Functions when required.

The exact SQL migration is intentionally unspecified. Inspect the actual database schema before proposing changes. Do not create duplicate tables or apply destructive changes based only on this document.

## Required Edge Functions

Two Edge Functions are required. Order creation is not a third mandatory function: reuse the application's existing secure order-creation flow, and add a minimal secure function only if no such mechanism exists.

### 1. `create-snap-transaction`

**Caller:** authenticated frontend.  
**Input:** internal order ID only.

Responsibilities:
- Require a valid Supabase user session.
- Validate request and order ID.
- Confirm the order belongs to the authenticated user and is eligible for payment.
- Read authoritative order total and currency from trusted database records.
- Never accept client-supplied amount, customer ID, order status, or payment-success flag as authoritative.
- Generate a unique Midtrans `order_id` for each distinct transaction attempt, within Midtrans documented character constraints and Snap's 50-character maximum.
- Create or safely reuse a persisted payment attempt.
- Serialize concurrent Pay Now requests using an atomic database operation, transaction, unique constraint, or equivalent concurrency control. At most one active attempt may exist per order. A concurrent request must reuse that attempt or receive a safe conflict response.
- Send the Snap transaction request to the Sandbox API using the Server Key.
- Return the Snap token and only minimal references needed by the frontend.
- Never mark an order paid because the Snap API request succeeded.

#### Retry, timeout, and idempotency rules

- **Duplicate request while creating:** return/reuse the same persisted active attempt; do not create another Midtrans transaction.
- **Snap reopened for the same active transaction:** reuse the active attempt and its valid transaction/token according to persisted state and Midtrans response. Do not issue a new `order_id` merely because Pay Now was clicked again.
- **New transaction after a terminal attempt:** if the prior attempt is terminal and the order remains payable, create a new payment-attempt record with a new unique Midtrans `order_id`.
- **Pending or unknown outcome:** reconcile with Midtrans status before deciding whether another attempt is allowed. Do not create a parallel attempt while the prior one may still be payable.
- **Snap API timeout or lost response:** preserve the existing attempt and its Midtrans `order_id`. The backend must treat the outcome as unknown, query/reconcile Midtrans using that same identifier where supported, and determine whether the transaction already exists before retrying the create request or generating another identifier. A timeout is not proof that Midtrans failed to create the transaction.
- **Retrying a create request:** only retry with the same identifier when the chosen Midtrans API behavior safely supports it; otherwise reconcile first and return a safe pending/unknown result. Never create a second attempt simply because the first HTTP response was not received.
- Enforce uniqueness and concurrency in the database/backend, not only with a frontend button lock.

### 2. `midtrans-notification`

**Caller:** Midtrans HTTP notification service.

Responsibilities:
- Accept Midtrans HTTP POST notifications.
- Verify `signature_key` using SHA-512 over the concatenated values:

  ```text
  order_id + status_code + gross_amount + MIDTRANS_SERVER_KEY
  ```

- Find the payment attempt by the exact Midtrans `order_id`; reject unknown or mismatched identifiers.
- Compare notification amount and currency, where available, with the saved payment attempt and related order.
- Verify transaction status with the Midtrans Sandbox Get Transaction Status API. Do not rely on browser callbacks or notification payload alone.
- Confirm the status response refers to the expected transaction and matches the saved order/payment amount.
- Treat a transaction as paid only when the verified status response has `status_code` `200` and `transaction_status` is `settlement` or an accepted `capture`.
- Whenever `fraud_status` is present, require `accept` for both `settlement` and `capture`. Do not mark paid when fraud status is `challenge` or another non-accepted value.
- Persist verified Midtrans transaction details and update order/payment state atomically where practical.
- Handle repeated and out-of-order notifications idempotently.
- A verified paid order must never transition back to pending, failed, expired, or cancelled because of a duplicate or stale notification.
- A non-paid notification must not overwrite a verified successful payment attempt.
- Update the order only from the payment attempt matched to that order and verified Midtrans transaction. If an order has multiple attempts, a successful attempt may mark the order paid; a failed/expired/cancelled attempt must not override a different verified successful attempt.
- Make payment-attempt updates idempotent using unique Midtrans `order_id` and persisted transaction identity. Repeated notifications reconcile the existing attempt rather than inserting duplicates.
- Keep processing efficient and return an appropriate HTTP response in under 5 seconds where practical.

Configure this function to accept external Midtrans requests without requiring a Supabase user JWT (for example, deploy with `--no-verify-jwt`). The function must still verify the Midtrans signature and transaction details itself. Disabling Supabase JWT verification does not mean trusting the request.

### Transaction status lookup

Use the Sandbox Get Transaction Status API only from trusted Edge Function code. Use the identifier supported for the relevant transaction/method (`order_id` or `transaction_id`, as applicable). Persist `transaction_id` from verified Midtrans data when available. Never expose the Server Key or provide a frontend endpoint that can query arbitrary Midtrans transactions.

## Midtrans Sandbox API Contract

### Credentials

| Configuration | Location | Purpose |
|---|---|---|
| `MIDTRANS_SERVER_KEY` | Supabase Edge Function secret | Backend API authentication and notification signature verification |
| `MIDTRANS_CLIENT_KEY` | Frontend environment/configuration | Loads Sandbox Snap.js in browser |

The Client Key is intended for browser use, but it must be the Sandbox Client Key. Do not commit either key.

### Endpoints

| Purpose | Sandbox URL |
|---|---|
| Create Snap transaction | `https://app.sandbox.midtrans.com/snap/v1/transactions` |
| Snap.js | `https://app.sandbox.midtrans.com/snap/snap.js` |
| Get transaction status | `https://api.sandbox.midtrans.com/v2/{order_id_or_transaction_id}/status` |

Use HTTP Basic Authentication for backend Midtrans API requests: Base64-encode `MIDTRANS_SERVER_KEY:` (Server Key as username and an empty password). The status endpoint accepts the documented `order_id` or `transaction_id` identifier; use `transaction_id` for BI-SNAP and DANA status lookups as required by Midtrans.

Calculate `gross_amount` from trusted order data and send an integer in the amount unit expected by Midtrans for the selected currency. For IDR, send rupiah as an integer (for example, `150000` for Rp150,000); do not multiply by 100 or convert to cents. Keep the amount consistent across order, payment attempt, Snap request, notification, and status verification.

## Payment State Consistency

- A verified paid order is terminal for payment purposes and cannot regress to pending, failed, expired, or cancelled.
- Only a payment attempt matched to the order and verified Midtrans transaction may update that order.
- A successful verified attempt may mark the order paid.
- A failed/expired/cancelled attempt must not override another verified successful attempt.
- Duplicate notifications update/reconcile the same payment attempt.
- Unknown or pending attempts must be reconciled before another attempt is created.

## Security Checklist

- Midtrans Sandbox only; no Production configuration.
- Keep Server Key and Supabase elevated credentials in backend secrets only.
- Authenticate payment creation and verify order ownership.
- Calculate payable amounts from trusted database records.
- Protect customer order/payment reads with RLS and authorization checks.
- Prevent clients from writing verified payment state.
- Verify notification signature, transaction identity, amount, status, and fraud status as applicable.
- Use unique Midtrans order identifiers for distinct attempts.
- Apply payment updates idempotently and reject invalid or stale state transitions.
- Never treat Snap.js callbacks, browser redirects, or frontend-submitted status as proof of payment.

## Official Technical References

### Midtrans
- [Snap Integration Guide](https://docs.midtrans.com/docs/snap-snap-integration-guide)
- [Access Keys](https://docs.midtrans.com/docs/access-keys)
- [Backend Integration — Create Snap Transaction](https://docs.midtrans.com/reference/backend-integration)
- [HTTP(S) Notifications / Webhooks](https://docs.midtrans.com/docs/https-notification-webhooks)
- [Handle Notifications](https://docs.midtrans.com/reference/handle-notifications)
- [Get Transaction Status](https://docs.midtrans.com/reference/get-transaction-status)

### Supabase
- [Edge Functions](https://supabase.com/docs/guides/functions)
- [Edge Function Secrets](https://supabase.com/docs/guides/functions/secrets)
- [Edge Function Authorization](https://supabase.com/docs/guides/functions/auth-headers)
- [Securing Database Data and RLS](https://supabase.com/docs/guides/database/secure-data)
