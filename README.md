# Midtrans Sandbox Integration with Supabase

A minimal integration contract for **Midtrans Snap in Sandbox mode only**, using Supabase PostgreSQL and Supabase Edge Functions as the backend.

> **Environment policy:** Midtrans Sandbox only. Production credentials, endpoints, SDK settings, and URLs are out of scope.

## Project Status

This README defines the required backend design. It does not claim that the database schema or Edge Functions are already implemented.

| Component | Status |
|---|---|
| Supabase PostgreSQL product, order, order-item, and payment records | Required / planned |
| Trusted order-creation backend path (existing secure path or small Edge Function) | Required / planned |\n| Edge Function: create Snap transaction | Required / planned |
| Edge Function: receive Midtrans notification | Required / planned |
| Midtrans transaction status verification | Required / planned |
| Frontend Snap.js integration | Required / planned |

## Architecture

- **Frontend:** displays products, manages a temporary cart, submits product identifiers and quantities for order creation, reviews the persisted order, invokes the authenticated payment Edge Function, and opens Snap.js with the returned transaction token.
- **Supabase Edge Functions:** communicate with Midtrans, validate requests and notifications, and perform trusted database writes.
- **Supabase PostgreSQL:** stores the application's orders and payment transaction records.
- **Midtrans Sandbox:** processes test payments and sends HTTP notifications to the notification Edge Function.

The frontend must never receive the Midtrans Server Key or decide that an order is paid.

## Customer Purchase Flow: Product Selection to Checkout

This section describes the complete customer journey before and during payment. Keep the experience linear and minimal: product list, product selection, cart/review, checkout, and payment. The product catalogue and order data are application data in Supabase; Midtrans is used only after an order has been created.

### Step-by-step flow

1. **Open the product page**
   - The frontend loads the available products from the application's product catalogue in Supabase.
   - Show only products that are available for purchase.
   - Product name, description, image, displayed price, and availability come from the application backend/database.

2. **Select a product**
   - The customer opens a product and chooses any required option or variant.
   - The customer selects a quantity.
   - The frontend may calculate a provisional line subtotal for immediate display, but this value is only a preview and is not trusted by the backend.

3. **Add to cart**
   - The selected product/variant and quantity are added to a simple cart held by the frontend.
   - The customer can change quantities or remove items before checkout.
   - The cart displays item names, selected options, quantities, and provisional subtotals.
   - No payment transaction is created at this stage.

4. **Review the cart**
   - The customer checks the selected items and quantities.
   - The frontend displays a provisional subtotal/total based on the latest product data it has.
   - The customer proceeds to checkout. The cart contents are still untrusted input until validated by the backend.

5. **Create the order at checkout**
   - The authenticated frontend submits only product/variant identifiers and quantities to a trusted order-creation backend path.
   - The backend verifies product availability, valid variants, quantity limits, and current prices using Supabase data.
   - The backend calculates the authoritative item subtotals and order total. It must ignore any client-supplied price, subtotal, discount, or total.
   - The backend creates the order and its order-item records in PostgreSQL in one database transaction, then returns the internal order ID and a safe order summary.
   - If the repository does not already have a secure order-creation path, implement a small authenticated `create-order` Edge Function for this purpose. Do not allow the browser to create trusted order totals directly.

6. **Show the checkout review**
   - The frontend displays the persisted order reference, items, quantities, and authoritative total returned by the backend.
   - The customer confirms that the order is correct and selects **Pay Now**.
   - If the order is no longer eligible or its data is stale, the backend must reject payment creation and the frontend should refresh the order.

7. **Create the Midtrans Snap transaction**
   - The frontend calls the authenticated `create-snap-transaction` Edge Function with the internal order ID only.
   - The function verifies ownership and order eligibility, reads the persisted order total, creates a unique Midtrans `order_id`, and requests a Snap token from the Midtrans Sandbox API.
   - The frontend opens Snap.js using the returned token.

8. **Customer completes or leaves payment**
   - The customer selects a payment method and follows the Midtrans Sandbox payment instructions.
   - Snap.js callbacks may show temporary user-interface messages, but do not update the trusted payment or order status.

9. **Receive and verify payment notification**
   - Midtrans sends the notification to `midtrans-notification`.
   - The Edge Function verifies the signature and transaction status, and matches the Midtrans order ID and amount against the stored payment and application order.
   - Only verified, accepted successful transactions may move the order to the paid state.

10. **Display the final order status**
    - The frontend refreshes the order/payment state through the application's authorized read path.
    - Show success only when the database reports the verified paid state.
    - Otherwise show pending, failed, expired, or cancelled as reported by the backend. Offer retry only when the order remains eligible for another payment attempt.

### Flow at a glance

```text
Product List
    |
Select Product / Variant / Quantity
    |
Add to Cart
    |
Review Cart
    |
Checkout
    |
Trusted Backend Validates Products and Prices
    |
Create Order + Order Items in Supabase
    |
Display Persisted Order Summary
    |
Pay Now
    |
create-snap-transaction
    |
Midtrans Sandbox Snap
    |
Customer Completes Payment
    |
midtrans-notification
    |
Verify Signature + Status + Amount + Order
    |
Update Payment and Order in Supabase
    |
Frontend Reads Verified Order Status
```

### Responsibilities by layer

| Layer | Responsibility |
|---|---|
| Frontend | Display products, manage temporary cart selection, collect quantities/options, show checkout summary, open Snap.js, and display backend-reported status. |
| Supabase PostgreSQL | Store product catalogue, orders, order items, and payment transactions; enforce ownership and data access rules. |
| Trusted order-creation backend | Validate product identifiers, availability, variants, and prices; calculate totals and create the order atomically. |
| `create-snap-transaction` | Create a Midtrans Sandbox Snap transaction for an existing eligible order using the persisted total. |
| `midtrans-notification` | Verify Midtrans notification and update payment/order state safely. |
| Midtrans Sandbox | Present payment methods, process test payments, and send transaction notifications. |

### Minimum order data

An order must have related order-item records so that the purchased products and quantities can be reviewed later. Each order item should preserve a purchase-time snapshot of at least the product name/description or display name, selected variant when applicable, quantity, unit price, and line subtotal. Historical order totals must not change when the current product catalogue price is edited.

### Purchase-flow acceptance criteria

- [ ] Customers can browse available products and select a valid variant and quantity when applicable.
- [ ] Customers can add, update, and remove items in the cart before checkout.
- [ ] Cart totals shown in the browser are treated as previews only.
- [ ] The backend revalidates product availability, variants, quantities, and current prices at order creation.
- [ ] The backend calculates the authoritative total; client-supplied prices and totals are ignored.
- [ ] The order and its item snapshots are created consistently before payment begins.
- [ ] The checkout page displays the persisted order summary and total.
- [ ] Midtrans transaction creation uses the persisted order total, not cart data from the browser.
- [ ] Final payment state is based on verified Midtrans notification processing, not browser callbacks.

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

### 1. `create-order` (only if no secure order-creation path exists)

**Caller:** authenticated frontend.

Responsibilities:
- Require and validate the Supabase user session.
- Accept product/variant identifiers and quantities only.
- Validate that each product and variant exists and is available for purchase.
- Read current prices from trusted database records; ignore client-supplied prices and totals.
- Validate quantity limits and calculate line subtotals and the authoritative order total.
- Create the order and its order-item purchase snapshots atomically.
- Return the internal order ID and a safe order summary.

Do not create a payment transaction in this function. Payment creation remains the responsibility of `create-snap-transaction`.

### 2. `create-snap-transaction`

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

### 3. `midtrans-notification`

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
