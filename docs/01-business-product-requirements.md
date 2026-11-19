# 01 — Business & Product Requirements

**Status:** Planned requirements; not evidence of implementation.  
**Payment environment:** Midtrans Snap Sandbox only.

## 1. Product purpose

Add a secure payment step to the existing Standard Storefront. Reuse its existing product catalogue, product details (if present), cart, checkout, authentication, and order functionality.

The goal is a minimal payment integration, not a replacement commerce platform.

## 2. Scope

### In scope
- Customer selects products and quantities using the existing storefront.
- Existing checkout creates or retrieves an order through trusted backend logic.
- Customer reviews the saved order reference, item summary, and authoritative total.
- Authenticated customer starts a Midtrans Snap Sandbox transaction for an eligible order.
- Backend verifies notifications and reconciles payment status.
- Existing order/confirmation experience displays backend-verified status.

### Out of scope
- Midtrans Production credentials, endpoints, configuration, or deployment.
- Replacement storefront, catalogue, cart, order history, customer dashboard, or order-management system.
- Separate Midtrans-only checkout page.
- New admin/payment dashboard unless a real existing operational need is established.
- Browser-side payment verification or client writes to trusted payment status.
- Storing card numbers, CVV, or other sensitive payment credentials.

## 3. Target technology context

| Layer | Target |
|---|---|
| Frontend | React 19, Vite, TypeScript |
| Styling | Tailwind CSS v4 |
| UI components | Konsta UI React v5 |
| Backend | Supabase Auth, PostgreSQL, RLS, Edge Functions |
| Payment | Midtrans Snap Sandbox |
| Static hosting | GitHub Pages |

Konsta UI v5 targets React 19. Inspect the actual app and dependencies before implementation; do not scaffold a replacement app or blindly upgrade dependencies.

## 4. Customer journey

1. Browse products in the existing catalogue.
2. Review products and quantities in the existing cart.
3. Continue through the existing checkout.
4. Trusted backend validates product IDs and quantities, reads trusted prices, calculates totals, and creates or retrieves the order.
5. Existing checkout displays saved order reference, item summary, and authoritative total.
6. Authenticated frontend requests a Snap transaction using the internal order ID only.
7. Customer completes, abandons, or closes Snap Sandbox.
8. Backend verifies the notification and reconciles it with Midtrans status and saved records.
9. Existing order experience shows success only after verified paid status is persisted.

```text
Existing Storefront → Cart → Checkout → Trusted Order
    → Review Order → Pay Now → Snap Sandbox
    → Midtrans Notification → Backend Verification
    → Persisted Payment Status → Existing Order Experience
```

Snap callbacks, browser redirects, and return URLs are UI signals only; they are never proof of payment.

## 5. Functional requirements

- Reuse existing catalogue, cart, checkout, authentication, and order flow.
- Backend validates product IDs and quantities and calculates totals from trusted records.
- Client-supplied prices, totals, customer IDs, or payment statuses are not authoritative.
- Payment creation accepts only an internal order ID from an authenticated customer.
- Customer can pay only an order they own and that backend marks eligible.
- Backend alone verifies and persists payment status.
- Customer can see only orders/payment details authorized for their account.
- Repeated payment requests and notifications are handled safely and idempotently.

## 6. Acceptance criteria

| ID | Scenario | Expected result |
|---|---|---|
| AC-01 | Valid settlement | Backend verifies Midtrans status and marks the matching payment/order paid. |
| AC-02 | Valid capture | Mark paid only for accepted verified capture; if fraud status is supplied, it must be `accept`. |
| AC-03 | Pending transaction | Order remains unpaid/pending; no success state is shown. |
| AC-04 | Denied, expired, or cancelled | Attempt is recorded accurately; retry only when backend says eligible. |
| AC-05 | Invalid signature | Notification is rejected and cannot mark an order paid. |
| AC-06 | Unknown/mismatched `order_id` | Reject; do not change an unrelated order/payment. |
| AC-07 | Amount/currency mismatch | Payment is not marked paid. |
| AC-08 | Duplicate notification | Reconcile existing attempt; do not create a duplicate payment record. |
| AC-09 | Stale/out-of-order notification | Verified paid order never regresses to a non-paid state. |
| AC-10 | Concurrent Pay Now requests | Database-backed serialization allows only one active attempt; concurrent callers receive the same reusable attempt or a safe conflict. |
| AC-11 | Snap closes/redirects | Refresh authorized backend status; callback/redirect alone never determines outcome. |
| AC-12 | Customer reads order/payment | Return only records authorized for that authenticated customer. |
| AC-13 | Frontend bundle | No Server Key or service-role key in frontend assets/build variables. |
| AC-14 | Sandbox isolation | All Midtrans requests and Snap.js use Sandbox endpoints and credentials only. |
| AC-15 | Unknown Snap creation outcome | Preserve the attempt and its Midtrans order_id; reconcile it before retrying. A Core API 404 Payment not found after Snap token creation is not proof that creation failed. |

## 6.1 Payment state and currency rules

Version 1 supports IDR only. Trusted backend logic derives the payable total from the saved order and saved item/price data. The exact integer rupiah amount and IDR currency must match across the order, payment attempt, Snap request, notification, and verified Get Status response. Do not use floating-point arithmetic or accept client totals. Reject non-IDR orders; multi-currency support is out of scope.

Keep two separate state concepts:
- Payment-attempt state: creating, creation_unknown, token_ready, pending, paid, failed, expired, or cancelled.
- Order payment state: unpaid, pending, or paid. Keep any existing order fulfillment/lifecycle status separate.

These are conceptual contract values, not prescribed database enum or column names. A paid attempt may make its order paid. A failed/expired/cancelled attempt does not make the order paid and cannot override another paid attempt. Order payment state is monotonic: once verified paid, it never returns to a non-paid state. A new distinct attempt may be considered only after the previous attempt is conclusively terminal and the order remains eligible.

## 6.2 Snap creation uncertainty

A timeout, lost HTTP response, or process interruption after sending the Snap create request leaves the attempt in creation_unknown; it does not prove failure. Preserve its existing Midtrans order_id. A Core API Get Status 404 Payment not found can be normal after Snap token creation while the customer has not selected a payment method. Do not generate a new Midtrans order_id because of that 404. Reconcile using the same identifier and, where appropriate, recreate the Snap token using that same order_id in accordance with Midtrans Snap behavior. If status cannot be established, keep the attempt unresolved and do not create another distinct attempt.

## 7. Open implementation dependencies

The actual storefront routes, checkout UX, order schema, authentication setup, and existing status display must be inspected before implementation. Requirements must be mapped onto those actual structures rather than assuming they exist.
