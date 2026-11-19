# Midtrans Sandbox Integration — Requirements

This document defines **what the application must do**. It is the product and functional requirements contract for adding Midtrans Snap Sandbox to the existing Standard Storefront.

**Environment:** Midtrans Sandbox only. Production support is out of scope.
**Implementation status:** Planned. This document does not imply that application features have been implemented.

## Scope

Add a secure payment step to the existing Standard Storefront. Reuse its product catalogue, product details (if present), cart, checkout, authentication, and order functionality.

Do not build a replacement storefront, catalogue, cart, order history, customer dashboard, order-management system, or admin/payment dashboard. Do not create a separate Midtrans-only checkout page.

## Technology Context

| Layer | Target |
|---|---|
| Frontend | React 19 + Vite + TypeScript + Tailwind CSS v4 + Konsta UI React v5 |
| Backend | Supabase Auth, PostgreSQL, RLS, and Edge Functions |
| Payment | Midtrans Snap Sandbox |
| Frontend hosting | GitHub Pages |

The frontend is a static application. Use Tailwind CSS v4 for utility styling and Konsta UI React v5 for reusable interface components. Keep the existing storefront's page structure, routes, and visual conventions; do not redesign or replace it solely to adopt the UI library. Konsta UI v5 targets React 19, so the implementation must use a compatible React version. Supabase handles trusted order and payment operations. Frontend hosting must never contain backend secrets or perform payment verification.

See [technical-spec.md](./technical-spec.md) for backend, database, security, and Midtrans integration rules. See [deployment.md](./deployment.md) for environment and deployment instructions.

## Customer Journey

1. **Browse products:** Customer selects products and quantities in the existing catalogue.
2. **Cart:** Customer reviews selected products and quantities in the existing cart.
3. **Checkout:** Customer uses the existing checkout flow.
4. **Create/retrieve order:** Trusted backend logic validates product IDs and quantities, calculates prices and totals from trusted records, and creates or retrieves the order. Client-submitted prices and totals are never authoritative.
5. **Review order:** Existing checkout displays the saved order reference, item summary, and authoritative total.
6. **Start payment:** Authenticated frontend requests a Snap transaction using the internal order ID only.
7. **Pay:** Customer completes, abandons, or closes Midtrans Snap Sandbox.
8. **Verify:** Backend verifies the Midtrans notification and reconciles it with Midtrans Sandbox transaction status and saved records.
9. **Show result:** Existing order confirmation/status experience displays success only after the backend has verified and persisted the paid state.

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

**Payment authority:** Snap.js callbacks, browser redirects, and return URLs are UI signals only. They must never be treated as proof of payment or directly update payment status.

## Frontend Requirements

Integrate the payment action into the existing checkout and preserve the application's current routing, page structure, and component conventions. Use Tailwind CSS v4 for styling and Konsta UI React v5 components where they fit the existing interface; avoid introducing a second component system or broad UI redesign.

The existing checkout must show:
- Order reference.
- Concise item summary.
- Authoritative order total.
- Payment action when the backend says the order is eligible.
- Relevant payment status or error.

The frontend must:
1. Load orders through the existing authorized read path and show only records the signed-in customer may access.
2. Call the authenticated `create-snap-transaction` function with the internal order ID only.
3. Disable Pay Now while transaction creation is in progress.
4. Load Sandbox Snap.js with the Sandbox Client Key and call `snap.pay(token, ...)` with the token returned by the Edge Function.
5. Treat Snap callbacks as UI events only. They may trigger a refresh of authorized order/payment status, but may not write a paid status.
6. Continue through the existing checkout/confirmation experience after Snap closes or redirects.
7. Read the latest status through the authorized backend/database path.

### Snap callback behavior

| Snap event | Required frontend behavior |
|---|---|
| `onSuccess` | Refresh order/payment status. Show success only if backend reports verified paid status. |
| `onPending` | Explain that confirmation is pending and refresh authorized status. |
| `onClose` | Remain in or return to existing checkout/order experience. Do not infer failed, cancelled, or paid. Retry only if backend says eligible. |
| `onError` | Show a concise error, refresh backend status, and allow retry only if eligible. |

### Required UI states

| State | Expected behavior |
|---|---|
| Loading | Show progress; payment is unavailable until eligibility is known. |
| Ready to pay | Show checkout summary and total; enable Pay Now only when eligible. |
| Creating transaction | Disable Pay Now and show progress. |
| Payment pending | Explain that confirmation is pending; allow status refresh. |
| Paid | Show success only after backend reports verified paid status. |
| Failed / expired / cancelled | Show backend-reported state; offer retry only if order remains eligible. |
| Snap closed | Keep customer in existing checkout/order experience; do not infer payment outcome. |
| Error | Show concise error and allow safe retry when appropriate. |

## Out of Scope

- Midtrans Production credentials, endpoints, configuration, and deployment.
- New storefront, cart, catalogue, order history, or admin/payment management system.
- Browser-side payment verification or browser-side writes to trusted payment status.
- Storing card numbers, CVV, or other sensitive payment credentials.

## Acceptance Criteria

| ID | Scenario | Expected result |
|---|---|---|
| AC-01 | Valid settlement | Backend verifies Midtrans status and marks the matching payment/order as paid. |
| AC-02 | Valid capture | Payment is marked paid only when verified status is accepted and any supplied fraud status is `accept`. |
| AC-03 | Pending transaction | Order remains unpaid/pending; no success state is shown. |
| AC-04 | Denied, expired, or cancelled transaction | Attempt is recorded accurately; retry is allowed only when backend eligibility rules permit it. |
| AC-05 | Invalid signature | Notification is rejected and cannot mark an order paid. |
| AC-06 | Unknown or mismatched `order_id` | Notification is rejected; no unrelated order/payment is changed. |
| AC-07 | Amount or currency mismatch | Payment is not marked paid. |
| AC-08 | Duplicate notification | Existing payment attempt is reconciled idempotently; no duplicate payment record is created. |
| AC-09 | Out-of-order/stale notification | A verified paid order never regresses to a non-paid state. |
| AC-10 | Duplicate/concurrent Pay Now requests | At most one active attempt exists per order; requests safely reuse it or receive a conflict response. |
| AC-11 | Snap closes or browser redirects | UI refreshes authorized backend status; callback/redirect alone never determines outcome. |
| AC-12 | Customer reads order/payment | Only records authorized for that authenticated customer are returned. |
| AC-13 | Frontend bundle/configuration | No Midtrans Server Key or Supabase service-role key is present in frontend assets or build variables. |
| AC-14 | Sandbox isolation | All Midtrans requests and Snap.js use Sandbox endpoints and credentials only. |
| AC-15 | Unknown Snap creation outcome | Backend reconciles the existing attempt using the same Midtrans order identifier before considering a new attempt. |
