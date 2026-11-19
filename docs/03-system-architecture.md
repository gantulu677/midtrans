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

## 6. Actual architecture verification

Before implementation, inspect the repository's actual entry point, router, Supabase client, auth flow, checkout, order persistence, and deployment setup. Update this diagram if the real structure differs. Do not create duplicate application layers to match a conceptual diagram.
