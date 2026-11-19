# Midtrans Sandbox Integration with Supabase

A minimal project overview for integrating **Midtrans Snap Sandbox only** into the existing Standard Storefront.

> **Environment policy:** Sandbox only. Production credentials, endpoints, configuration, and deployment are out of scope.

## Project Status

This repository currently contains documentation only. The application, database changes, Supabase Edge Functions, and GitHub Pages workflow have not been verified as implemented.

| Area | Target |
|---|---|
| Frontend | React + Vite + TypeScript |
| Backend | Supabase Auth, PostgreSQL, RLS, and Edge Functions |
| Payment | Midtrans Snap Sandbox |
| Frontend hosting | GitHub Pages |

The integration is intended to add a payment step to the existing product, cart, checkout, and order flow. It must not create a replacement storefront, cart, order history, or administration system.

## Architecture at a Glance

- The existing storefront handles product selection and checkout.
- Trusted backend logic creates/retrieves orders and calculates payable totals.
- Supabase Edge Functions create Snap transactions and verify Midtrans notifications.
- Midtrans Sandbox processes test payments.
- The backend is the sole authority for payment status. Browser callbacks and redirects are not proof of payment.

## Configuration Overview

- Use the repository path as Vite's `base` for a GitHub Pages project site (currently expected to be `/midtrans/`).
- Use `HashRouter` for straightforward SPA routing on GitHub Pages, unless a tested `BrowserRouter` fallback strategy is implemented.
- Frontend build variables may contain only public values such as Supabase URL, Supabase anon key, and Midtrans Sandbox Client Key.
- Store `MIDTRANS_SERVER_KEY` and Supabase service-role credentials only in Supabase backend secrets.

## Implementation Requirements

The complete implementation contract—including checkout flow, database requirements, Edge Function behavior, retry/idempotency rules, Sandbox endpoints, frontend states, deployment guidance, and acceptance criteria—is maintained in **[REQUIREMENTS.md](./REQUIREMENTS.md)**.

Implementers must read and follow that document before changing application code or database schema.

## Official Documentation

- [Midtrans Snap Integration Guide](https://docs.midtrans.com/docs/snap-snap-integration-guide)
- [Midtrans Notifications / Webhooks](https://docs.midtrans.com/docs/https-notification-webhooks)
- [Midtrans Get Transaction Status](https://docs.midtrans.com/reference/get-transaction-status)
- [Midtrans Sandbox Testing](https://docs.midtrans.com/docs/testing-payment-on-sandbox)
- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)
- [Supabase Row Level Security](https://supabase.com/docs/guides/database/secure-data)

---

**Status:** Documentation contract only  
**Payment environment:** Midtrans Sandbox only  
**Production support:** Not supported
