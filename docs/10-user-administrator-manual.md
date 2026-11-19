# 10 — User & Administrator Manual

**Status:** Planned documentation outline. No customer-facing or administrator interface has been verified.

## 1. Purpose

This document will explain how to use the existing storefront payment flow and, only if applicable, how authorized maintainers configure the Sandbox integration.

It does not define or imply a new admin dashboard.

## 2. Customer guide — planned

Once the frontend is implemented and inspected, document:
1. How to select products in the existing storefront.
2. How to review the cart and continue to checkout.
3. How to review the saved order and total.
4. How to start payment using the Sandbox payment action.
5. What pending, successful, failed, expired, and closed payment states mean.
6. How to return to the existing order experience and refresh status.

Important customer guidance:
- A Snap success/close screen or browser redirect is not final proof of payment.
- The order status shown by the application is authoritative because it is updated after backend verification.
- If status is pending, wait for backend confirmation and refresh the authorized order view.

## 3. Maintainer guide — planned

After deployment configuration exists, document:
- Where Sandbox Client Key is configured for the frontend.
- Where Sandbox Server Key is configured as a Supabase Edge Function secret.
- How the Midtrans Sandbox notification URL is configured.
- How to deploy and verify the two Edge Functions.
- How to run the Sandbox test checklist.
- How to inspect sanitized logs and troubleshoot failed notifications or unknown transaction outcomes.

Never include actual credentials, secret values, or service-role keys in this manual.

## 4. Administrator scope limitation

No administrator role, admin panel, manual payment override, refund workflow, reconciliation dashboard, or user-management feature is specified by the current requirements. Do not invent operational procedures for features that do not exist.

If an actual administrator workflow is later confirmed, document its roles, permissions, audit trail, and safe operating steps separately.

## 5. Completion condition

Replace this outline with verified, step-by-step instructions only after the corresponding user and maintainer flows have been implemented and tested. Clearly label any remaining planned procedure.
