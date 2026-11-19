# Midtrans Sandbox Integration with Supabase

A minimal integration contract for **Midtrans Snap in Sandbox mode only**, using Supabase PostgreSQL and Supabase Edge Functions as the backend.

> **Environment policy:** Midtrans Sandbox only. Production credentials, endpoints, SDK settings, and URLs are out of scope.

## Project Status

This README defines the required backend design. It does not claim that the database schema or Edge Functions are already implemented.

| Component | Status |
|---|---|
| Existing Supabase order and payment records | Required / planned |
| Secure order creation (reuse existing application flow; add only if missing) | Required when no existing path |\n| Edge Function: create Snap transaction | Required / planned |
| Edge Function: receive Midtrans notification | Required / planned |
| Midtrans transaction status verification | Required / planned |
| Frontend Snap.js integration | Required / planned |

## Architecture

- **Frontend:** displays products, manages a temporary cart, submits product identifiers and quantities for order creation, reviews the persisted order, invokes the authenticated payment Edge Function, and opens Snap.js with the returned transaction token.
- **Supabase Edge Functions:** communicate with Midtrans, validate requests and notifications, and perform trusted database writes.
- **Supabase PostgreSQL:** stores the application's orders and payment transaction records; reuse existing product and order tables.
- **Midtrans Sandbox:** processes test payments and sends HTTP notifications to the notification Edge Function.

The frontend must never receive the Midtrans Server Key or decide that an order is paid.

## Required Purchase and Payment Flow

Keep the customer flow limited to the steps needed to create an order and pay through **Midtrans Sandbox**. Reuse the application's existing product catalogue, cart, authentication, and order functionality where available. Do not build a new catalogue or complex cart system as part of this integration.

1. **Select product** — The customer selects an available product and quantity using the existing frontend.
2. **Checkout** — The customer reviews the selected product(s) and proceeds to checkout.
3. **Create or retrieve order** — The application creates an order using trusted backend logic. Product identifiers and quantities are validated against Supabase data; prices and the total are calculated by the backend, never trusted from the browser. Reuse an existing secure order-creation flow. Add a minimal `create-order` function only if no such flow exists.
4. **Review order** — The frontend displays the saved order reference, items, and authoritative total from the backend.
5. **Create Snap transaction** — The authenticated frontend calls `create-snap-transaction` with the internal order ID only. The function checks order ownership and eligibility, reads the saved total, and requests a Snap token from the Midtrans Sandbox API.
6. **Open payment page** — The frontend opens Midtrans Sandbox Snap.js using the returned token and Sandbox Client Key.
7. **Complete payment** — The customer selects a payment method and completes or exits the Sandbox payment flow.
8. **Verify notification** — Midtrans sends a notification to `midtrans-notification`. The function verifies the signature, order ID, amount, transaction status, and fraud status against the saved records.
9. **Show final status** — The frontend reads the order/payment status from the application backend. Show success only after the verified backend state is paid.

### Flow summary

```text
Select Product and Quantity
          |
       Checkout
          |
 Create / Retrieve Order
 (trusted backend validation)
          |
    Review Order Total
          |
       Pay Now
          |
create-snap-transaction
          |
 Midtrans Sandbox Snap
          |
   Customer Payment
          |
 midtrans-notification
          |
 Verify and Update Database
          |
 Display Verified Order Status
```

### Scope

- Product browsing, cart behavior, and order creation are existing application responsibilities, except for the minimal secure order-creation path needed if one does not already exist.
- This integration adds only the Midtrans Sandbox payment flow and the required secure connection to the application's order records.
- Do not add coupons, shipping workflows, product administration, customer dashboards, or other unrelated checkout features.
- Browser callbacks and redirects are informational only. They must never mark an order as paid.

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

Keep the backend minimal. Reuse an existing secure order-creation path if the application already has one; otherwise add only the small order-creation function described below, alongside the two payment functions.

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



## Simple Frontend Checkout Page

The frontend is a single, minimal checkout page for paying an existing order with Midtrans Snap. It is a customer-facing payment page, not a shopping cart, order-management dashboard, or payment administration system.

### Page requirements

The page must show only the information needed to review and pay for an order:

- Order reference or order number.
- A concise order summary (items may be shown if already available to the frontend).
- The order total, displayed using the authoritative amount returned by the application backend.
- A **Pay Now** button.
- A clear payment state or error message.

Do not add product catalogue management, cart features, coupons, shipping workflows, customer administration, or other features that are not required for this payment flow.

### Checkout interaction

1. The customer opens the checkout page for an existing order.
2. The frontend loads and displays that order's permitted summary and total from the application backend. It must not calculate or submit the payable amount as trusted input.
3. The customer selects **Pay Now**.
4. The frontend disables the button while the request is in progress to prevent accidental duplicate submissions.
5. The frontend calls the authenticated `create-snap-transaction` Edge Function with the internal order ID.
6. If the function returns a valid Snap token, the frontend opens the Midtrans Sandbox Snap interface using `snap.pay(token, ...)`.
7. The frontend may show a temporary message after Snap closes or redirects, but must not treat the browser callback as proof of payment.
8. The page obtains the current order/payment state from the application's trusted backend or database read path. The verified notification handler is responsible for applying the final payment state.

### Required UI states

| State | Expected behavior |
|---|---|
| Loading order | Show a loading indicator; do not enable payment yet. |
| Ready to pay | Show the order summary and total with **Pay Now** enabled. |
| Creating transaction | Disable **Pay Now** and show progress. |
| Snap opened | Let the customer complete or leave the Sandbox payment flow. |
| Pending / awaiting confirmation | Explain that payment confirmation is being processed; allow status refresh through the trusted application read path. |
| Paid | Show success only when the backend/database reports the verified paid state. |
| Failed / expired / cancelled | Show the state reported by the backend and provide a retry action only if the order remains eligible for payment. |
| Error | Show a concise message and allow a safe retry when appropriate. |

### Frontend integration contract

**Request to `create-snap-transaction`:**
- Send the authenticated user's request and the internal order ID.
- Do not send an amount, customer ID, Midtrans Server Key, or a client-selected payment status as authoritative values.

**Successful response:**
- Must provide the Snap token required by `snap.pay`.
- May provide a payment/order reference needed for subsequent status display.
- Must not expose the Server Key or Supabase service-role credentials.

**Snap.js behavior:**
- Load only the Sandbox Snap.js script and use the Sandbox Client Key.
- Use the token returned by the Edge Function.
- Treat `onSuccess`, `onPending`, `onError`, and `onClose` callbacks as UI events only. They may trigger a status refresh, but must never write a paid status or override the backend's verified state.

### Frontend acceptance criteria

- [ ] A customer can view only an order they are authorized to access.
- [ ] The page displays the order total from trusted application data.
- [ ] The Pay Now action calls `create-snap-transaction` with the order ID, not a client-calculated amount.
- [ ] Repeated clicks are prevented while transaction creation is in progress.
- [ ] The Snap interface opens with the returned token and Sandbox configuration.
- [ ] No Server Key or elevated Supabase credential is present in frontend code or browser requests.
- [ ] Snap callbacks and browser redirects cannot mark an order as paid.
- [ ] The paid state is displayed only after the backend reports a verified payment.
- [ ] Pending, failed, expired, cancelled, and error states are presented clearly.
- [ ] A retry is allowed only when the backend confirms that the order is eligible for another payment attempt.

### Frontend scope

The initial implementation should remain one simple responsive checkout page. Use the repository's existing frontend framework and styling conventions if present. Do not introduce a new framework, state-management library, routing system, or UI component library solely for this page.

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
