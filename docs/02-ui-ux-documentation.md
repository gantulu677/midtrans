# 02 — UI/UX Documentation

**Status:** Behavioral requirements only. Actual screens, routes, design tokens, and components have not been verified.

## 1. UI integration principle

Add payment interaction to the existing checkout and order experience. Preserve current page structure, routes, visual conventions, and component patterns. Do not redesign unrelated storefront screens.

Use Tailwind CSS v4 for styling and Konsta UI React v5 components where they fit the existing interface. Avoid introducing a second component system or broad redesign solely to adopt Konsta.

## 2. Required checkout information

The existing checkout/order view must display:
- Saved order reference.
- Concise item summary.
- Authoritative order total.
- Payment action only when backend reports the order is eligible.
- Relevant payment status or concise error.

Do not display a client-calculated amount as the authoritative payable total.

## 3. Required interaction states

| State | Expected behavior |
|---|---|
| Loading | Show progress; payment unavailable until eligibility is known. |
| Ready to pay | Show order summary and total; enable Pay Now only when eligible. |
| Creating transaction | Disable Pay Now and show progress. |
| Payment pending | Explain confirmation is pending; allow authorized status refresh. |
| Paid | Show success only after backend reports verified paid status. |
| Failed / expired / cancelled | Show backend-reported state; offer retry only if order remains eligible. |
| Snap closed | Keep customer in existing checkout/order experience; do not infer outcome. |
| Error | Show concise error; refresh backend status; allow retry only when backend confirms it is safe. |
| Creation unknown | Show reconciliation progress; do not create a second attempt or infer failure from a status API 404. |

## 3.1 Unknown Snap creation outcome

If the backend reports that Snap creation is unresolved, show a neutral “Checking payment session” state. Do not show payment failure and do not enable a new payment attempt. A status API 404 Payment not found may occur before the customer selects a payment method; it is not evidence that token creation failed. Keep the customer on the existing order experience while the backend reconciles the same attempt. Reopen or recreate a Snap token only when the backend explicitly returns a safe action for that same Midtrans order_id.

## 4. Snap callback behavior

| Snap event | Frontend behavior |
|---|---|
| `onSuccess` | Refresh order/payment status. Show success only if backend reports verified paid status. |
| `onPending` | Explain confirmation is pending and refresh authorized status. |
| `onClose` | Stay in or return to existing checkout/order experience. Do not infer failed, cancelled, or paid. |
| `onError` | Show concise error, refresh backend status, and allow retry only if eligible. |

Callbacks may trigger a refresh; they must never write paid status.

## 5. Component and accessibility expectations

- Use existing application components where available.
- Use Konsta UI v5 for suitable reusable controls, with a single consistent `ios` or `material` theme selected after reviewing the existing storefront.
- Keep button labels and error messages clear.
- Provide visible loading/disabled states to prevent accidental duplicate requests.
- Ensure payment controls remain usable on mobile and desktop layouts.
- Preserve keyboard access, focus visibility, and semantic labels.

## 6. Not specified yet

Do not invent route names, page inventory, wireframes, brand palette, typography, or administrator screens. Record these after inspecting the actual frontend. This document defines payment UX behavior, not an approved visual redesign.
