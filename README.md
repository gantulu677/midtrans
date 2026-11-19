# Midtrans Sandbox Integration with Supabase

A minimal implementation contract for **Midtrans Snap in Sandbox mode only**, using the application's existing product and order flow with Supabase PostgreSQL and Supabase Edge Functions.

> **Environment policy:** Sandbox only. Production credentials, endpoints, configuration, and deployment are out of scope.

## Project Status

This README defines the target requirements. It does not claim that the database changes, Edge Functions, or frontend integration are already implemented.

| Component | Status |
|---|---|
| Existing product catalogue and customer flow | Reuse existing application |
| Secure order creation | Reuse existing backend; add only if missing |
| Order and payment persistence | Required; adapt existing schema |
| `create-snap-transaction` Edge Function | Required / planned |
| `midtrans-notification` Edge Function | Required / planned |
| Backend transaction verification and status updates | Required / planned |
| Existing Standard Storefront with Midtrans integrated into its checkout | Required / planned |

## Technology Stack

| Layer | Technology | Responsibility |
|---|---|---|
| Frontend | React + Vite + TypeScript | Existing Standard Storefront, checkout UI, and Snap.js integration |
| Backend | Supabase | Authentication, PostgreSQL, Row Level Security, and Edge Functions |
| Payment | Midtrans Snap Sandbox | Customer payment experience and transaction processing |
| Hosting / deployment | GitHub Pages | Static frontend hosting; no backend secrets or payment verification run in the browser |

The frontend is a static React/Vite application deployed to GitHub Pages. Supabase remains the backend for trusted order/payment operations. GitHub Pages must not host or contain Server Keys, service-role keys, or other backend secrets. Deploy Supabase Edge Functions separately to Supabase.

For a Vite project hosted at a GitHub Pages project URL, configure Vite's `base` path to match the repository path (for example, `/<repository-name>/`). Use `/` only when deploying to a user/organization root site or a custom domain configured for the site. The actual repository name and deployment URL must determine the final value.

For this repository's standard GitHub Pages project site, the expected Vite setting is:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  base: "/midtrans/",
});
```

Use `base: "/"` only if the site is deployed at a domain root (for example, a configured custom domain or the account's user/organization site). Recheck the setting if the repository name or Pages URL changes.

### React SPA routing on GitHub Pages

GitHub Pages serves static files and does not provide automatic fallback routing for React routes. A direct visit or browser refresh on a nested `BrowserRouter` URL can return 404.

For the simplest GitHub Pages deployment, use React Router's `HashRouter` so client-side routes remain after the `#` and do not require server-side rewrites. If the existing storefront must keep `BrowserRouter`, document and implement a GitHub Pages-compatible `404.html` redirect/restore strategy, and test direct navigation and refresh on nested routes. Do not assume that configuring Vite `base` alone solves SPA routing.

Frontend build-time environment variables are embedded in the generated static assets and are public. Only put browser-safe configuration there, such as the Midtrans Sandbox Client Key and Supabase project URL/anon key. Never put `MIDTRANS_SERVER_KEY` or the Supabase service-role key in Vite variables, GitHub Pages settings, or frontend source. Store backend secrets in Supabase Edge Function secrets.

For Vite, expose browser configuration through `VITE_`-prefixed variables, for example:

```env
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
VITE_MIDTRANS_CLIENT_KEY=
```

Never define `VITE_MIDTRANS_SERVER_KEY` or `VITE_SUPABASE_SERVICE_ROLE_KEY`. Any `VITE_` value included in a frontend build must be treated as public.

## Scope and Architecture

The integration adds only the secure payment flow to the existing Standard Storefront. Reuse its product catalogue, product detail views (if present), cart, checkout, authentication, and order functionality. Midtrans Snap is a payment step within the existing checkout—not a separate storefront or replacement checkout. Do not build a replacement catalogue, cart, order-management system, order history, or admin dashboard.

- **Frontend:** keeps the existing storefront navigation and components for product browsing, cart, and checkout; invokes the authenticated payment function from the existing checkout; opens Sandbox Snap.js using the returned token; and displays order status through the existing confirmation/status experience.
- **Supabase Edge Functions:** create Snap transactions, verify Midtrans notifications, and update trusted payment/order state.
- **Supabase PostgreSQL:** remains the source of truth for orders, order totals, and payment attempts.
- **Midtrans Sandbox:** processes test payments and sends payment notifications to the notification function.

The Midtrans Server Key and Supabase elevated credentials must never be exposed to the browser. The frontend must never decide or write that an order is paid.

## Required Purchase and Payment Flow

1. **Browse and select products** — The customer browses the existing product catalogue and selects products and quantities using the Standard Storefront.
2. **Shopping cart and checkout** — The customer reviews the cart and proceeds through the existing checkout page and flow. Do not create a separate Midtrans-only checkout page.
3. **Create or retrieve order** — The application creates or retrieves an order through trusted backend logic. Validate product identifiers and quantities against database records. Calculate prices and totals on the backend; never trust prices or totals submitted by the browser. Reuse the existing secure order-creation mechanism. Add a minimal secure order-creation function only if no such mechanism exists.
4. **Review checkout** — The existing checkout displays the saved order reference, item summary, and authoritative total returned from the backend before payment begins.
5. **Start payment** — The authenticated frontend calls `create-snap-transaction` with the internal order ID only.
6. **Open Snap** — The function validates ownership and payment eligibility, reads the saved order total, creates a unique Midtrans `order_id`, requests a Snap token from Sandbox, and returns the token. The frontend opens Sandbox Snap.js.
7. **Complete payment** — The customer chooses a payment method and completes, abandons, or closes the Sandbox payment flow.
8. **Verify payment** — Midtrans sends a notification to `midtrans-notification`. The function verifies the signature and reconciles the transaction with Midtrans Sandbox status data and the saved order/payment records.
9. **Show order result** — Return the customer to the existing order confirmation/status experience. The frontend reads the application's payment/order status and shows success only after the backend has verified the payment and persisted the paid state.

```text
Select Product
     |
   Checkout
     |
Create / Retrieve Order (trusted backend)
     |
  Review Order
     |
    Pay Now
     |
create-snap-transaction
     |
Sandbox Snap.js
     |
Customer Payment
     |
midtrans-notification
     |
Verify and Update Database
     |
Show Backend-Verified Status
```

**Important:** Snap.js callbacks, browser redirects, and return URLs are user-interface signals only. They are never proof of payment and must not update payment status.

## Supabase Data Requirements

Reuse the application's existing order schema wherever possible. Do not create duplicate product or order tables. Adapt the schema only as needed to persist and protect the payment lifecycle.

### Order data

The existing order record must provide, directly or through related records:

- Stable internal order ID.
- Authenticated customer/user ID.
- Authoritative order total and currency.
- Order status and payment eligibility.
- Created/updated timestamps.

Order items and their prices must represent the values agreed at checkout. The backend must calculate the payable total from trusted order data.

### Payment attempt data

Persist a payment record linked to its order with, at minimum:

- Internal payment ID and related internal order ID.
- Unique Midtrans `order_id` for each distinct Midtrans transaction attempt.
- Amount and currency submitted to Midtrans.
- Midtrans `transaction_id`, when available.
- Midtrans transaction status and fraud status, when available.
- Payment type, when available.
- Created/updated and last verification/notification timestamps.

Enforce uniqueness on the Midtrans `order_id`. Persist enough information to match notifications to the exact payment attempt and safely handle retries. Do not store card numbers, CVV, or other sensitive payment credentials. Do not persist the Snap token unless the application has a concrete requirement for doing so.

### Database access

- Enable Row Level Security on tables exposed through the Supabase Data API.
- Customers may read only their own permitted order and payment information.
- Clients must not write trusted payment status, Midtrans transaction identifiers, verified amounts, or fraud status.
- Only trusted backend code may apply payment-state changes.
- Use elevated Supabase credentials only inside Edge Functions when required. Never expose them in frontend code.

The exact SQL migration is intentionally not specified here. Any future schema change must first be checked against the application's actual database schema; do not create duplicate tables or apply destructive changes based only on this README.

## Supabase Edge Functions

The payment integration requires two Edge Functions. Order creation is not a third mandatory function: reuse the application's existing secure order-creation flow, and add a minimal function only if that flow does not exist.

### 1. `create-snap-transaction`

**Caller:** authenticated frontend.

**Input:** internal order ID only.

Responsibilities:

- Require a valid Supabase user session.
- Validate the request and order ID.
- Confirm the order belongs to the authenticated user and is eligible for payment.
- Read the authoritative order total and currency from trusted database records.
- Never accept a client-supplied amount, customer ID, order status, or payment-success flag as authoritative.
- Generate a unique Midtrans `order_id` for the payment attempt. Keep it within Midtrans's documented length and character constraints.
- Create or safely reuse a payment attempt so repeated requests do not create unintended duplicate attempts.
- Serialize concurrent Pay Now requests using an atomic database operation, transaction, unique constraint, or equivalent concurrency control. The policy must allow at most one active payment attempt per order at a time; a concurrent request must reuse the active attempt or receive a safe conflict response, not create another one.
- A genuinely new Midtrans transaction attempt must receive a new unique Midtrans `order_id`; never reuse an identifier from a previous attempt.
- Send the Snap transaction request to the **Sandbox** API using the Server Key.
- Return the Snap token and only the minimal references needed by the frontend.
- Handle API errors without marking the order as paid.

For a new payment attempt, the Midtrans `order_id` must be unique. Do not reuse an identifier already used for a different Midtrans transaction. A retry must follow the persisted order/payment state and retry policy. Concurrent requests must be serialized so that only one active attempt exists for an order at a time; enforce this with an atomic claim/transaction and database uniqueness or an equivalent mechanism, not only a frontend button lock.

Apply these retry rules consistently:

- **Duplicate request while creating:** return/reuse the same persisted active attempt; do not create another Midtrans transaction.
- **Snap reopened for the same active transaction:** reuse that attempt and its valid transaction/token according to the backend's persisted state and Midtrans response. Do not issue a new `order_id` merely because the customer clicked Pay Now again.
- **New transaction after the previous attempt is terminal and the order remains payable:** create a new payment-attempt record and a new unique Midtrans `order_id`.
- **Unknown outcome or pending transaction:** reconcile with Midtrans status before deciding whether another attempt is allowed. Do not create a parallel attempt while the previous one may still be payable.
- Enforce Midtrans's documented `order_id` length and character constraints when generating identifiers (including the 50-character maximum for Snap).

### 2. `midtrans-notification`

**Caller:** Midtrans HTTP notification service.

Responsibilities:

- Accept Midtrans HTTP POST notifications.
- Verify `signature_key` using SHA-512 over the concatenated values:

  ```text
  order_id + status_code + gross_amount + MIDTRANS_SERVER_KEY
  ```

- Find the payment attempt by the exact Midtrans `order_id`; reject unknown or mismatched identifiers.
- Compare the notification amount and currency, where available, with the saved payment attempt and related order.
- Verify transaction status with the Midtrans Sandbox Get Transaction Status API as part of trusted reconciliation. Do not rely on an unverified browser callback or notification payload alone.
- Confirm the status response refers to the expected transaction and matches the saved order/payment amount.
- Treat a transaction as paid only when the verified Midtrans status response has `status_code` `200` and `transaction_status` is `settlement` or an accepted `capture`. Whenever `fraud_status` is present, require it to be `accept` for both `settlement` and `capture`; do not mark a transaction paid when the fraud status is `challenge` or another non-accepted value.
- Persist the verified Midtrans transaction details and update the order/payment state atomically where practical.
- Handle repeated and out-of-order notifications idempotently. A stale notification must not downgrade an already verified paid order.
- Apply explicit, monotonic order/payment state rules: a verified paid order must never transition back to pending, failed, expired, or cancelled because of a duplicate or stale notification. A non-paid notification must not overwrite a verified successful payment attempt.
- Update the order only from the payment attempt matched to that order and verified Midtrans transaction. If an order has multiple attempts, a successful attempt may mark the order paid; a failed/expired/cancelled attempt must not override a different verified successful attempt.
- Make payment-attempt updates idempotent using the unique Midtrans `order_id` and persisted transaction identity. Repeated notifications update/reconcile the existing attempt rather than inserting a duplicate.
- Keep notification processing efficient and return an appropriate HTTP response in under 5 seconds where practical, so Midtrans can determine promptly whether delivery succeeded. Avoid unnecessary blocking or long-running work in the notification request.

Configure the notification function to accept external Midtrans requests without requiring a Supabase user JWT (for example, deploy with `--no-verify-jwt`). The function must still verify the Midtrans signature and transaction details itself. Disabling Supabase JWT verification does **not** mean trusting the request.

### Transaction status lookup

Use the Sandbox Get Transaction Status API only from trusted Edge Function code. Use the identifier supported for the relevant transaction/method (Midtrans `order_id` or `transaction_id` as applicable). Persist `transaction_id` from verified Midtrans data when available. Never expose the Server Key or provide a frontend endpoint that can query arbitrary Midtrans transactions.

## Sandbox Configuration

Create Sandbox credentials in the Midtrans Merchant Administration Portal under **Settings → Access Keys**.

| Configuration | Location | Purpose |
|---|---|---|
| `MIDTRANS_SERVER_KEY` | Supabase Edge Function secret | Backend API authentication and notification signature verification |
| `MIDTRANS_CLIENT_KEY` | Frontend environment/configuration | Loads Sandbox Snap.js in the browser |

Only the Server Key is a backend secret. The Client Key is intended for browser use, but it must be the Sandbox Client Key. Do not commit either key to the repository.

Set the Server Key as a Supabase secret:

```bash
supabase secrets set MIDTRANS_SERVER_KEY=your-sandbox-server-key
```

Configure the Sandbox Client Key using the frontend framework's environment-variable mechanism. Only expose the public Client Key; never expose the Server Key or Supabase service-role key.

### Sandbox endpoints

| Purpose | Sandbox URL |
|---|---|
| Create Snap transaction | `https://app.sandbox.midtrans.com/snap/v1/transactions` |
| Snap.js | `https://app.sandbox.midtrans.com/snap/snap.js` |
| Get transaction status | `https://api.sandbox.midtrans.com/v2/{order_id_or_transaction_id}/status` |

Use HTTP Basic Authentication for backend Midtrans API requests: Base64-encode `MIDTRANS_SERVER_KEY:` (Server Key as username and an empty password). The status endpoint accepts the documented `order_id` or `transaction_id` identifier; use `transaction_id` for BI-SNAP and DANA status lookups, as required by Midtrans.

For Snap requests, calculate `gross_amount` from trusted order data and send an integer in the amount unit expected by Midtrans for the selected currency. For IDR, send the rupiah amount as an integer (for example, `150000` for Rp150,000); do not multiply it by 100 or convert it to cents. Keep the amount consistent across the order, payment attempt, Snap request, notification, and status verification.

## Frontend Standard Storefront Integration

Integrate Midtrans into the application's **existing Standard Storefront**. Preserve the current frontend framework, routing, page structure, styling, and component conventions. Do not introduce a second storefront, a standalone payment application, or a replacement checkout page.

The existing customer journey remains:

1. **Product Catalog** — browse and select existing products.
2. **Shopping Cart** — review selected products and quantities.
3. **Existing Checkout** — review customer/order details and the authoritative order total.
4. **Midtrans Snap Sandbox** — launch Snap as the payment step from the existing checkout after the backend creates a transaction.
5. **Existing Order Confirmation / Status** — show the order result and payment status read from the authorized backend path.

Only integrate the payment action and the minimum payment-state display required by the existing checkout/confirmation experience. Reuse existing order detail, confirmation, and status routes/components where available. Do not add a new cart, catalogue, order history, customer dashboard, or payment administration UI for this integration.

### Checkout content and payment action

Within the existing checkout, show the order reference, concise item summary, authoritative order total, available payment action, and relevant payment state/error. Enable **Pay Now** only when the backend indicates that the order is eligible for payment.

1. Load the order through the application's existing authorized read path and display only orders the signed-in customer may access.
2. When the customer chooses **Pay Now**, call the authenticated `create-snap-transaction` function with the internal order ID only. Disable the action while the request is running.
3. Load Sandbox Snap.js with the Sandbox Client Key and call `snap.pay(token, ...)` using the token returned by the Edge Function.
4. Treat `onSuccess`, `onPending`, `onError`, and `onClose` as UI events only. They may trigger a refresh of authorized order/payment status, but must never write a paid status or be treated as proof of payment.
5. After Snap closes or redirects, continue through the existing checkout/confirmation navigation. Read the latest status through the application's authorized backend/database read path. The notification function is responsible for applying verified payment state.

### Snap callback behavior

| Snap event | Frontend behavior |
|---|---|
| `onSuccess` | Refresh order/payment status. Show paid/success only if the backend reports verified paid status. |
| `onPending` | Show that confirmation is pending and refresh status through the authorized read path. |
| `onClose` | Return to or remain on the existing checkout/order experience. Do not assume failed, cancelled, or paid; show the latest backend status. Allow retry only if the backend says the order remains eligible. |
| `onError` | Show a concise error, refresh backend status, and allow retry only if the order remains eligible. |

### Required UI states

| State | Expected behavior |
|---|---|
| Loading | Show progress; payment is not yet available |
| Ready to pay | Show existing checkout summary and total; enable Pay Now when eligible |
| Creating transaction | Disable Pay Now and show progress |
| Payment pending | Explain that confirmation is pending; allow status refresh |
| Paid | Show success only after the backend reports verified paid status |
| Failed / expired / cancelled | Show the backend-reported state; offer retry only if the order remains eligible |
| Snap closed | Keep the customer in the existing checkout/order experience; do not infer a payment outcome from closing Snap |
| Error | Show a concise error and allow a safe retry when appropriate |

## Local Development, Deployment, and Testing

The React/Vite frontend and Supabase backend are deployed separately.

### Frontend — GitHub Pages

Build the Vite application and publish the generated `dist/` directory using the repository's GitHub Pages deployment workflow. The workflow should install dependencies, run the production build, and publish the build output. Configure the Vite `base` path for the actual GitHub Pages URL as described in [Technology Stack](#technology-stack).

Example local build commands:

```bash
npm ci
npm run build
```

Configure only public frontend values (for example, the Supabase project URL, Supabase anon key, and Midtrans Sandbox Client Key) as Vite build-time variables. These values are visible to users in the published JavaScript bundle. Do not configure backend secrets in GitHub Actions frontend build variables.

A minimal GitHub Actions workflow can use the official Pages artifact/deployment actions. Save it as `.github/workflows/deploy.yml`:

```yaml
name: Deploy frontend to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - name: Install dependencies
        run: npm ci
      - name: Build
        run: npm run build
        env:
          VITE_SUPABASE_URL: ${{ vars.VITE_SUPABASE_URL }}
          VITE_SUPABASE_ANON_KEY: ${{ vars.VITE_SUPABASE_ANON_KEY }}
          VITE_MIDTRANS_CLIENT_KEY: ${{ vars.VITE_MIDTRANS_CLIENT_KEY }}
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

In repository settings, set **Pages → Build and deployment → Source** to **GitHub Actions**. Add the three public `VITE_` values as repository Actions variables (or provide them through another safe build-time configuration). They are public after bundling; do not store Server Key or service-role credentials in Actions variables or secrets for the frontend build. Supabase backend secrets are configured separately in Supabase.

Typical Supabase CLI commands:

```bash
supabase start
supabase functions serve create-snap-transaction
supabase functions serve midtrans-notification --no-verify-jwt
supabase functions deploy create-snap-transaction
supabase functions deploy midtrans-notification --no-verify-jwt
```

Store local function secrets in `supabase/functions/.env` and do not commit that file. Configure the Sandbox Server Key in the deployed Supabase project's Edge Function secrets.

Set the Midtrans Sandbox notification URL to:

```text
https://<PROJECT_REF>.supabase.co/functions/v1/midtrans-notification
```

Test at least:

- Successful settlement.
- Successful capture with accepted fraud status.
- Pending payment.
- Denied, expired, and cancelled payment.
- Invalid signature and unknown `order_id`.
- Amount/order mismatch.
- Repeated and out-of-order notifications.
- Duplicate Pay Now requests and eligible retry behavior.

Confirm that only backend-verified payments update the order and that repeated notifications do not create duplicate payment records or invalid state transitions.

## Security Requirements

- Midtrans Sandbox only; no Production configuration.
- Keep the Server Key and Supabase elevated credentials in backend secrets only.
- Authenticate payment creation and verify order ownership.
- Calculate payable amounts from trusted database records.
- Protect customer order/payment reads with RLS and authorization checks.
- Prevent clients from writing verified payment state.
- Verify notification signatures, transaction identity, amount, transaction status, and fraud status as applicable.
- Use unique Midtrans order identifiers for distinct payment attempts.
- Apply payment updates idempotently and reject invalid or stale state transitions.
- Never treat Snap.js callbacks, browser redirects, or frontend-submitted status as proof of payment.

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
