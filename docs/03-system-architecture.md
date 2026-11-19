# 03 — System Architecture

**Status:** Target architecture; actual source code and deployed resources have not been verified.

## 1. Technology architecture

| Layer | Target technology | Responsibility |
|---|---|---|
| Browser frontend | React 19 + TypeScript | Existing storefront, checkout, Snap UI, authorized status display |
| Build tooling | Vite | Development and static production build |
| Styling | Tailwind CSS v4 | Utility styling |
| UI components | Konsta UI React v5 | Suitable reusable UI components |
| Backend platform | Supabase | Auth, PostgreSQL, Edge Functions |
| Payment provider | Midtrans Snap Sandbox | Test payment processing and notifications |
| Static hosting | GitHub Pages | Serve built frontend assets |

## 2. Trust boundaries and data flow

```text
Customer Browser
  ├─ Existing Storefront / Checkout
  ├─ Supabase Auth session
  ├─ Authenticated request: create-snap-transaction
  └─ Snap.js (Sandbox Client Key)
            |
            v
Supabase Edge Functions
  ├─ Validate user and order ownership
  ├─ Read trusted order total
  ├─ Create/reuse payment attempt
  ├─ Call Midtrans Sandbox API
  └─ Verify Midtrans notification/status
            |
            v
Supabase PostgreSQL
  ├─ Existing orders and order items
  └─ Payment attempt records
            ^
            |
Midtrans Sandbox ── HTTP notification ──> midtrans-notification
```

## 3. Responsibility boundaries

### Frontend
- Uses existing storefront and checkout.
- Requests payment creation for an internal order ID.
- Opens Snap with the returned token.
- Refreshes authorized order/payment status.
- Never calculates the authoritative payable total or determines paid status.

### Supabase PostgreSQL
- Stores existing orders and authoritative totals.
- Stores payment attempts linked to orders.
- Enforces uniqueness and access policies required for safe payment handling.

### Supabase Edge Functions
- Authenticate payment creation requests and check ownership/eligibility.
- Communicate with Midtrans using backend-only credentials.
- Verify notifications and query transaction status.
- Persist payment state safely and idempotently.

### Midtrans Sandbox
- Processes test payments.
- Sends server-to-server notifications.
- Is the external payment processor; the backend verifies its reports before updating local state.

## 4. Required functions

1. `create-snap-transaction`: authenticated frontend endpoint; accepts internal order ID only.
2. `midtrans-notification`: external Midtrans notification endpoint; validates signature and transaction status.

Reuse the existing secure order-creation flow. A third order function is not required unless inspection shows no secure existing mechanism.

## 5. Architectural constraints

- Backend is the sole authority for payment status.
- Browser callbacks and redirects are not proof of payment.
- Server Key and Supabase service-role key remain server-side.
- Payment attempts are linked to exact internal orders and unique Midtrans order identifiers.
- At most one active attempt exists per order.
- Unknown transaction outcomes are reconciled before creating another attempt.
- Verified paid state cannot regress because of stale notifications.
- No production payment configuration is included.

## 5.1 Payment attempt lifecycle and concurrency

The payment attempt lifecycle is distinct from the order payment state. The following are conceptual states; actual schema names must follow database inspection:
- creating → token_ready or creation_unknown.
- creation_unknown remains unresolved until reconciled; timeout or lost response is not a failure result.
- token_ready → pending when Midtrans assigns a transaction status; it may then become paid, failed, expired, or cancelled after verified status.
- A new distinct attempt may be considered only after the previous attempt is conclusively terminal and the order remains unpaid and eligible.
- A paid attempt makes the order payment state paid. No later event may regress it. A failed attempt cannot override a separate paid attempt.

Serialize Pay Now requests in trusted backend/database logic, not with a frontend disabled button alone. Use a database transaction/row lock or equivalent atomic claim for the order, plus a unique constraint on each persisted Midtrans order_id. Define the active-attempt set as creating, creation_unknown, token_ready, and pending. Concurrent requests must reuse the same eligible attempt or return a safe conflict. Do not create another attempt while any attempt is active.

Midtrans Snap supports re-creating a token for an unused Snap order_id before the customer chooses a payment method; doing so invalidates the previous token. Once a payment method is selected and the transaction has a status (for example pending), the same order_id cannot be used to create a new Snap transaction. Reopen the existing token where still valid, or reconcile the transaction; do not assume a new token can be created for a used identifier.

Do not assume generic Midtrans Idempotency-Key support applies to the Snap Create Transaction endpoint. The v1 contract relies on local database serialization, unique order_id, and reconciliation unless Midtrans explicitly confirms endpoint-specific support.

## 5.2 Snap status 404 and recovery

A Core API Get Status 404 Payment not found may be expected after Snap token creation and before the customer selects a payment method. Treat it as “not yet assigned a payment status”, not as evidence that the create request failed. Keep the same attempt and Midtrans order_id. If the token response was lost, backend may recreate a token with that same unused identifier in accordance with Snap documentation; if status lookup is unavailable or the identifier may already be in use, retain creation_unknown and defer retry until reconciled. Never generate a new identifier solely due to timeout or 404.

## 5.3 Notification acknowledgement contract

The notification handler must validate signature and identify the persisted attempt before applying changes. Persist verified state durably before acknowledging success. Return HTTP 2xx only when the notification has been safely handled or is a confirmed duplicate. Return a client-error response for malformed/invalid-signature/unknown identifiers; return a retryable server error (prefer 503) when the handler cannot verify or durably persist because a dependency is unavailable. Do not acknowledge an unprocessed notification with 200. Keep processing fast (target under five seconds) and rely on Midtrans retry behavior plus later reconciliation for transient failures.

## 6. Actual architecture verification

Before implementation, inspect the repository's actual entry point, router, Supabase client, auth flow, checkout, order persistence, and deployment setup. Update this diagram if the real structure differs. Do not create duplicate application layers to match a conceptual diagram.
