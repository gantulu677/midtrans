# Midtrans Sandbox Integration with Supabase

A minimal documentation contract for integrating **Midtrans Snap Sandbox only** into the existing Standard Storefront.

> **Environment policy:** Sandbox only. Production credentials, endpoints, configuration, and deployment are out of scope.

## Project Status

This repository currently contains documentation only. Application source, database changes, Supabase Edge Functions, and the GitHub Pages workflow have not been verified as implemented.

| Area | Target |
|---|---|
| Frontend | React 19 + Vite + TypeScript + Tailwind CSS v4 + Konsta UI React v5 |
| Backend | Supabase Auth, PostgreSQL, RLS, and Edge Functions |
| Payment | Midtrans Snap Sandbox |
| Frontend hosting | GitHub Pages |

The integration adds only a payment step to the existing product, cart, checkout, and order flow. It must not create a replacement storefront or order-management system.

## Documentation Structure

The detailed project documentation is organized into ten focused documents:

| No. | Document | Coverage |
|---|---|---|
| 01 | [Business & Product Requirements](./docs/01-business-product-requirements.md) | Scope, customer journey, functional requirements, acceptance criteria |
| 02 | [UI/UX Documentation](./docs/02-ui-ux-documentation.md) | Checkout behavior, payment states, Snap callbacks, UI constraints |
| 03 | [System Architecture](./docs/03-system-architecture.md) | Components, responsibilities, trust boundaries, data flow |
| 04 | [Database & Supabase Documentation](./docs/04-database-supabase.md) | Existing order data, payment attempts, RLS, Edge Functions |
| 05 | [API & Integration Documentation](./docs/05-api-integration.md) | Snap transaction creation, notifications, status lookup, Sandbox endpoints |
| 06 | [Security & Access Control](./docs/06-security-access-control.md) | Authentication, ownership, secrets, verification, idempotency |
| 07 | [Development Standards](./docs/07-development-standards.md) | React 19, Vite, TypeScript, Tailwind v4, Konsta UI v5 |
| 08 | [Testing & Quality Assurance](./docs/08-testing-quality-assurance.md) | Functional, integration, UI, security, and release tests |
| 09 | [Deployment & DevOps](./docs/09-deployment-devops.md) | GitHub Pages, Vite, Supabase deployment, environment configuration |
| 10 | [User & Administrator Manual](./docs/10-user-administrator-manual.md) | Planned customer and maintainer guidance; no unverified admin feature |

## Core Architecture

- Existing storefront handles product selection, cart, and checkout.
- Trusted backend logic creates/retrieves orders and calculates payable totals.
- Supabase Edge Functions create Snap transactions and verify Midtrans notifications.
- Midtrans Sandbox processes test payments.
- Backend is the sole authority for payment status. Browser callbacks and redirects are not proof of payment.

## Documentation Rules

- Distinguish verified implementation from requirements, examples, and planned work.
- Inspect the actual repository and database before implementation.
- Reuse existing storefront and order functionality; avoid unnecessary systems.
- Do not invent routes, schema, admin features, or deployed resources.
- Keep Midtrans Sandbox only.

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
