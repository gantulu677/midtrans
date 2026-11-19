# Midtrans Sandbox Integration with Supabase

A minimal project overview for integrating **Midtrans Snap Sandbox only** into the existing Standard Storefront.

> **Environment policy:** Sandbox only. Production credentials, endpoints, configuration, and deployment are out of scope.

## Project Status

This repository currently contains documentation only. The application, database changes, Supabase Edge Functions, and GitHub Pages workflow have not been verified as implemented.

| Area | Target |
|---|---|
| Frontend | React 19 + Vite + TypeScript + Tailwind CSS v4 + Konsta UI React v5 |
| Backend | Supabase Auth, PostgreSQL, RLS, and Edge Functions |
| Payment | Midtrans Snap Sandbox |
| Frontend hosting | GitHub Pages |

The integration adds only a payment step to the existing product, cart, checkout, and order flow. It must not create a replacement storefront or order-management system.

## Architecture at a Glance

- The existing storefront handles product selection and checkout.
- Trusted backend logic creates/retrieves orders and calculates payable totals.
- Supabase Edge Functions create Snap transactions and verify Midtrans notifications.
- Midtrans Sandbox processes test payments.
- The backend is the sole authority for payment status. Browser callbacks and redirects are not proof of payment.

## Documentation

Read these documents before implementation:

| Document | Purpose |
|---|---|
| [requirement.md](./requirement.md) | Product scope, customer journey, frontend behavior, and acceptance criteria |
| [technical-spec.md](./technical-spec.md) | Database requirements, Edge Functions, Midtrans API, security, retry, and payment-state consistency |
| [deployment.md](./deployment.md) | Vite configuration, GitHub Pages, environment variables, Supabase deployment, and Sandbox testing |

The documents define planned requirements; they do not indicate that the application or deployment already exists.

## Official Documentation

- [Midtrans Snap Integration Guide](https://docs.midtrans.com/docs/snap-snap-integration-guide)
- [Midtrans Notifications / Webhooks](https://docs.midtrans.com/docs/https-notification-webhooks)
- [Midtrans Get Transaction Status](https://docs.midtrans.com/reference/get-transaction-status)
- [Midtrans Sandbox Testing](https://docs.midtrans.com/docs/testing-payment-on-sandbox)
- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)
- [Supabase Row Level Security](https://supabase.com/docs/guides/database/secure-data)
- [Tailwind CSS v4 — Vite Installation](https://tailwindcss.com/docs/installation/using-vite)
- [Konsta UI React](https://konstaui.com/react)

---

**Status:** Documentation contract only  
**Payment environment:** Midtrans Sandbox only  
**Production support:** Not supported
