# Midtrans Sandbox Integration — Deployment Guide

This document covers local setup and deployment configuration for the planned React/Vite frontend and Supabase backend.

**Environment:** Midtrans Sandbox only. No Production credentials, endpoints, configuration, or deployment are supported.

**Current repository status:** Documentation only. The commands and workflow below are examples for implementation; they are not currently runnable in this repository because application source, package manifests, lockfiles, and workflow files have not been verified as present.

## Deployment Architecture

- **Frontend:** static React + Vite + TypeScript build hosted on GitHub Pages.
- **Backend:** Supabase Auth, PostgreSQL, and Edge Functions deployed separately to Supabase.
- **Payment:** Midtrans Snap Sandbox.

GitHub Pages must never contain the Midtrans Server Key, Supabase service-role key, or other backend secrets. Payment verification runs only in trusted Supabase Edge Functions.

## Frontend UI Dependencies

The target frontend stack is React 19, Vite, TypeScript, Tailwind CSS v4, and Konsta UI React v5. Konsta UI v5 targets React 19.

Install the Tailwind Vite integration and Konsta UI in the existing React application:

```bash
npm install tailwindcss @tailwindcss/vite konsta
```

If the application does not already have the React Vite plugin and React dependencies, inspect its existing package manifest before adding or changing them. Do not scaffold a second application or overwrite existing dependencies just to match this example.

Add the Tailwind plugin to the existing Vite configuration while preserving the React plugin and the GitHub Pages `base` setting:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  base: "/midtrans/",
});
```

In the application's main CSS entry point, import Tailwind and the Konsta theme:

```css
@import "tailwindcss";
@import "konsta/react/theme.css";
```

Import Konsta components from `konsta/react`. Wrap the relevant application tree with Konsta's `App` component and select one consistent theme (`ios` or `material`) appropriate to the existing storefront. Example:

```tsx
import { App, Page } from "konsta/react";

export function StorefrontApp() {
  return (
    <App theme="ios">
      <Page>
        {/* Existing storefront routes and screens */}
      </Page>
    </App>
  );
}
```

Treat the snippet as an integration example, not a replacement for the existing app root or router. Preserve the current routing and application shell. If the actual app is not on React 19, resolve Konsta v5 compatibility before adopting it; do not assume the dependency can be added without checking the installed React version.

## Vite Configuration

For a GitHub Pages project site, set Vite's `base` to the repository path. For this repository, the expected value is `/midtrans/`:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  base: "/midtrans/",
});
```

Use `base: "/"` only for a user/organization root site or a custom domain configured at the domain root. Recheck the value if the repository name or Pages URL changes.

## React SPA Routing

GitHub Pages does not automatically fall back to an application's route handler. Direct visits or refreshes on nested `BrowserRouter` URLs may return 404.

For the simplest deployment, use React Router's `HashRouter`. If the existing storefront must retain `BrowserRouter`, implement and test a GitHub Pages-compatible `404.html` redirect/restore strategy. Vite's `base` setting alone does not solve SPA routing.

## Frontend Environment Variables

Vite build-time variables are embedded in generated static assets and are public. Use only browser-safe values:

```env
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
VITE_MIDTRANS_CLIENT_KEY=
```

Never define `VITE_MIDTRANS_SERVER_KEY` or `VITE_SUPABASE_SERVICE_ROLE_KEY`. Treat every `VITE_` variable as public after bundling.

Configure the Sandbox Server Key separately as a Supabase Edge Function secret:

```bash
supabase secrets set MIDTRANS_SERVER_KEY=your-sandbox-server-key
```

Do not commit credentials or local secret files.

## Frontend Build

After the React/Vite application and its lockfile exist, the expected build commands are:

```bash
npm ci
npm run build
```

These commands are not currently executable in the documentation-only repository.

## GitHub Pages Workflow

The following is an example workflow to be created at `.github/workflows/deploy.yml` when the frontend implementation is ready. It is not an existing repository workflow.

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

When implementing the workflow:
1. Set **Settings → Pages → Build and deployment → Source** to **GitHub Actions**.
2. Add `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, and `VITE_MIDTRANS_CLIENT_KEY` as repository Actions variables or another safe public build-time configuration.
3. Remember that these values are public after bundling.
4. Do not place Server Key or service-role credentials in frontend build variables or assets.
5. Configure Supabase backend secrets separately in Supabase.

## Supabase Local Development and Deployment

Typical CLI commands after the Supabase project and functions have been created:

```bash
supabase start
supabase functions serve create-snap-transaction
supabase functions serve midtrans-notification --no-verify-jwt
supabase functions deploy create-snap-transaction
supabase functions deploy midtrans-notification --no-verify-jwt
```

Store local function secrets in `supabase/functions/.env` and ensure that file is not committed. Configure the Sandbox Server Key in the deployed Supabase project's Edge Function secrets.

Set the Midtrans Sandbox notification URL to:

```text
https://<PROJECT_REF>.supabase.co/functions/v1/midtrans-notification
```

The notification function may be deployed without Supabase JWT verification because Midtrans does not send a Supabase user JWT. It must still verify the Midtrans signature and transaction details itself.

## Sandbox Setup and Test Checklist

1. Create Sandbox credentials in the Midtrans Merchant Administration Portal under **Settings → Access Keys**.
2. Configure the Sandbox Client Key in the frontend's public build configuration.
3. Configure `MIDTRANS_SERVER_KEY` as a Supabase Edge Function secret.
4. Configure the notification URL in the Midtrans Sandbox settings.
5. Test:
   - Successful settlement.
   - Successful capture with accepted fraud status.
   - Pending payment.
   - Denied, expired, and cancelled payment.
   - Invalid signature and unknown `order_id`.
   - Amount/order mismatch.
   - Repeated and out-of-order notifications.
   - Duplicate Pay Now requests and eligible retry behavior.
   - Snap transaction creation timeout/unknown response and reconciliation using the same Midtrans order identifier.
6. Confirm that only backend-verified payments update the order and that repeated notifications do not create duplicate payment records or invalid state transitions.

## Official Deployment References

- [Vite Deploying a Static Site](https://vite.dev/guide/static-deploy.html)
- [GitHub Pages Documentation](https://docs.github.com/en/pages)
- [GitHub Pages with GitHub Actions](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages)
- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)
- [Supabase Edge Function Secrets](https://supabase.com/docs/guides/functions/secrets)
- [Supabase Deploy Edge Functions](https://supabase.com/docs/guides/functions/deploy)
- [Midtrans Sandbox Testing](https://docs.midtrans.com/docs/testing-payment-on-sandbox)
