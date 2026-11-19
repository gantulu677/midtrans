# Midtrans Sandbox Integration with Supabase

A minimal integration contract for **Midtrans Snap in Sandbox mode only**, using Supabase PostgreSQL and Supabase Edge Functions as the backend.

> **Environment policy:** Midtrans Sandbox only. Production credentials, endpoints, SDK settings, and URLs are out of scope.

## Project Status

This README defines the required backend design. It does not claim that the database schema or Edge Functions are already implemented.

| Component | Status |
|---|---|
| Supabase PostgreSQL order and payment records | Required / planned |
| Edge Function: create Snap transaction | Required / planned |
| Edge Function: receive Midtrans notification | Required / planned |
| Midtrans transaction status verification | Required / planned |
| Frontend Snap.js integration | Required / planned |

## Architecture

- **Frontend:** invokes the authenticated create-payment Edge Function and opens Snap.js with the returned transaction token.
- **Supabase Edge Functions:** communicate with Midtrans, validate requests and notifications, and perform trusted database writes.
- **Supabase PostgreSQL:** stores the application's orders and payment transaction records.
- **Midtrans Sandbox:** processes test payments and sends HTTP notifications to the notification Edge Function.

The frontend must never receive the Midtrans Server Key or decide that an order is paid.

## Payment Flow

1. A signed-in customer selects an existing order and starts checkout.
2. The frontend calls the authenticated `create-snap-transaction` Edge Function.
3. The function verifies that the customer may pay for the order, reads the order and its items from PostgreSQL, and calculates the amount from trusted database data.
4. The function creates a unique Midtrans `order_id`, stores a pending payment attempt, and requests a Snap token from the Sandbox API.
5. The function returns the Snap token to the frontend. The frontend opens the Sandbox Snap payment page.
6. Midtrans sends a notification to `midtrans-notification`.
7. The notification function verifies the signature, retrieves transaction status from Midtrans when needed, and checks the order ID, amount, transaction status, and fraud status against the stored payment/order.
8. The function updates the payment and order in PostgreSQL safely and idempotently.

A browser redirect or Snap.js callback is for user experience only; it is not proof of payment.

## Supabase Database

Use the existing application order table if one already exists. Do not create a duplicate order system.

At minimum, the database must persist:

### Orders

The order record must contain:
- A stable internal order ID.
- The authenticated customer's user ID.
- The authoritative total amount.
- The order status.
- Created and updated timestamps.

### Payment transactions

Use a payment transaction record linked to the order. It must contain:
- A unique internal payment ID.
- The related order ID.
- A unique Midtrans `order_id`.
- The amount submitted to Midtrans.
- The Midtrans transaction ID when available.
- Transaction status and fraud status when available.
- The payment type when available.
- Notification/verification timestamps and created/updated timestamps.

Enforce uniqueness for the Midtrans `order_id`. Store enough information to reconcile notifications and prevent duplicate payment records. Do not store card numbers, CVV, or other sensitive payment credentials.

### Database access

- Enable Row Level Security (RLS) on tables exposed through the Supabase Data API.
- Customers may read only their own permitted order/payment information.
- Do not allow clients to write trusted payment status, transaction IDs, verified amounts, or fraud status.
- Perform payment-state changes only in trusted Edge Function code.
- Use a Supabase secret/service credential only inside Edge Functions when elevated database access is required. Never expose it in frontend code.

The exact SQL migration must match the application's existing schema before implementation; this README does not prescribe destructive schema changes.

## Supabase Edge Functions

Keep the backend to the following required functions.

### 1. `create-snap-transaction`

**Caller:** authenticated frontend.

Responsibilities:
- Require and validate the Supabase user session.
- Accept an internal order ID, not a client-supplied amount.
- Confirm that the order belongs to the caller and is eligible for payment.
- Read the authoritative amount from PostgreSQL.
- Generate a unique Midtrans `order_id`.
- Create or safely reuse a pending payment attempt according to the application's retry policy.
- Call the Sandbox Snap API using the Server Key.
- Persist the returned transaction/token metadata needed by the application.
- Return only the data needed by the frontend, including the Snap token.

Never trust a frontend amount, customer ID, order status, or payment-success flag.

### 2. `midtrans-notification`

**Caller:** Midtrans HTTP notification service.

Responsibilities:
- Accept Midtrans HTTP POST notifications.
- Verify `signature_key` using SHA-512 of the concatenated values:

  ```
  order_id + status_code + gross_amount + MIDTRANS_SERVER_KEY
  ```

- Find the matching stored payment by the exact Midtrans `order_id`.
- Confirm the notification amount matches the stored payment amount and the related order total.
- Verify the transaction with Midtrans Sandbox Get Transaction Status when required; do not trust an unverified payload alone.
- Confirm the returned order identifier and amount match the stored records.
- Treat payment as successful only when `status_code` is `200`, `transaction_status` is `settlement` or `capture`, and, when `fraud_status` is present, it is `accept`.
- Apply valid state transitions and persist transaction details.
- Handle repeated and out-of-order notifications idempotently.
- Return an appropriate HTTP response so Midtrans can determine whether notification delivery succeeded.

Configure this webhook function to accept external Midtrans requests without a Supabase user JWT, then perform Midtrans signature verification inside the function. Disabling Supabase JWT verification does **not** mean trusting the request.

### Transaction status verification

Use the Sandbox Get Transaction Status API from trusted Edge Function code for reconciliation or notification verification. Do not expose the Server Key or create a frontend endpoint that lets a caller query arbitrary Midtrans order IDs.

## Sandbox Configuration

Create the Sandbox credentials in the Midtrans Merchant Administration Portal under **Settings → Access Keys**.

Set these values as Supabase Edge Function secrets:

| Secret | Used by | Purpose |
|---|---|---|
| `MIDTRANS_SERVER_KEY` | Edge Functions only | Authenticates backend requests and verifies notification signatures |
| `MIDTRANS_CLIENT_KEY` | Frontend | Opens Midtrans Snap.js in Sandbox mode |

The Server Key must be configured as a Supabase secret, not committed to Git. The Client Key is intended for browser use, but must be the Sandbox key.

Supabase secrets can be configured in the Dashboard or with the CLI:

```bash
supabase secrets set MIDTRANS_SERVER_KEY=your-sandbox-server-key
```

Do not add a Production key or Production URL.

### Midtrans Sandbox endpoints

| Purpose | URL |
|---|---|
| Create Snap transaction | `https://app.sandbox.midtrans.com/snap/v1/transactions` |
| Snap.js | `https://app.sandbox.midtrans.com/snap/snap.js` |
| Get transaction status | `https://api.sandbox.midtrans.com/v2/{order_id}/status` |

Use HTTP Basic Authentication for backend Midtrans API requests: Base64-encode `MIDTRANS_SERVER_KEY:` (Server Key as username and an empty password).

## Frontend Snap.js

Load the Sandbox Snap.js using the Sandbox Client Key:

```html
<script
  src="https://app.sandbox.midtrans.com/snap/snap.js"
  data-client-key="YOUR_SANDBOX_CLIENT_KEY">
</script>
```

The frontend obtains the Snap token from `create-snap-transaction` and uses it to open the payment interface. Do not call the Midtrans create-transaction API directly from the browser.

## Local Development and Deployment

Typical Supabase CLI workflow:

```bash
supabase start
supabase functions serve create-snap-transaction
supabase functions serve midtrans-notification --no-verify-jwt
supabase functions deploy create-snap-transaction
supabase functions deploy midtrans-notification --no-verify-jwt
```

Store local function secrets in `supabase/functions/.env` and do not commit that file. Configure the same Sandbox secret in the Supabase project's Edge Function secrets for the deployed environment.

Configure the Midtrans Sandbox notification URL to:

```
https://<PROJECT_REF>.supabase.co/functions/v1/midtrans-notification
```

Test successful, pending, expired, cancelled, denied, and repeated notification scenarios. Confirm that only verified payments update the order and that repeated notifications do not duplicate payment records or state transitions.

## Security Requirements

- Midtrans Sandbox only.
- Server Key and Supabase elevated credentials stay in Edge Function secrets.
- Authenticated order ownership is checked before creating a payment.
- Amounts are calculated from trusted database records.
- RLS protects customer access to orders and payment data.
- Clients cannot write verified payment state.
- Notification signatures, order identity, amount, transaction status, and fraud status are verified.
- Payment updates are idempotent and reject invalid state transitions.
- Browser redirects and frontend callbacks never mark an order as paid.

## Official Documentation

### Midtrans
- [Snap Integration Guide](https://docs.midtrans.com/docs/snap-snap-integration-guide)
- [Access Keys](https://docs.midtrans.com/docs/access-keys)
- [Backend Integration — Create Snap Transaction](https://docs.midtrans.com/reference/backend-integration)
- [HTTP(S) Notifications / Webhooks](https://docs.midtrans.com/docs/https-notification-webhooks)
- [Handle Notifications](https://docs.midtrans.com/reference/handle-notifications)
- [Get Transaction Status](https://docs.midtrans.com/reference/get-transaction-status)
- [Testing Payments on Sandbox](https://docs.midtrans.com/docs/testing-payment-on-sandbox)

### Supabase
- [Edge Functions](https://supabase.com/docs/guides/functions)
- [Edge Function Secrets](https://supabase.com/docs/guides/functions/secrets)
- [Edge Function Authorization](https://supabase.com/docs/guides/functions/auth-headers)
- [Securing Database Data and RLS](https://supabase.com/docs/guides/database/secure-data)
- [Deploy Edge Functions](https://supabase.com/docs/guides/functions/deploy)

---

**Backend:** Supabase PostgreSQL + Supabase Edge Functions  
**Payment integration:** Midtrans Snap  
**Environment:** Midtrans Sandbox only  
**Production support:** Not supported
