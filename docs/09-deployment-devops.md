# 09 — Deployment & DevOps

**Status:** Deployment plan and examples only. No workflow or deployment is asserted to exist.

## 1. Deployment architecture

- Frontend: static React/Vite/TypeScript build on GitHub Pages.
- Backend: Supabase Auth, PostgreSQL, and Edge Functions deployed separately.
- Payment: Midtrans Snap Sandbox only.

GitHub Pages must never contain Midtrans Server Key, Supabase service-role key, or other backend secrets.

## 2. GitHub Pages Vite base

For this repository's project site, expected Vite base is `/midtrans/`:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  base: "/midtrans/",
});
```

Use `/` only for a user/organization root site or custom domain at the domain root. Recheck if repository name or Pages URL changes.

## 3. SPA routing

GitHub Pages does not automatically route nested URLs to the SPA. Prefer React Router `HashRouter` for the simplest deployment. If retaining `BrowserRouter`, implement and test a Pages-compatible `404.html` redirect/restore strategy. Vite `base` alone does not solve SPA routing.

## 4. Frontend environment variables

Vite build-time variables are public in generated assets. Allowed browser-safe configuration:

```env
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
VITE_MIDTRANS_CLIENT_KEY=
```

Never define `VITE_MIDTRANS_SERVER_KEY` or `VITE_SUPABASE_SERVICE_ROLE_KEY`. Treat every `VITE_` variable as public.

## 5. Supabase secrets

Configure Sandbox Server Key as a Supabase Edge Function secret:

```bash
supabase secrets set MIDTRANS_SERVER_KEY=your-sandbox-server-key
```

Do not commit credentials or local secret files. Store local function secrets in `supabase/functions/.env` and ensure it is ignored by Git.

## 6. Build commands

After application source and lockfile exist:

```bash
npm ci
npm run build
```

These commands are not currently executable in this documentation-only repository until the app and package files exist.

## 7. GitHub Actions workflow example

The following is a proposed workflow for `.github/workflows/deploy.yml`, not an existing workflow:

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
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
        env:
          VITE_SUPABASE_URL: ${{ vars.VITE_SUPABASE_URL }}
          VITE_SUPABASE_ANON_KEY: ${{ vars.VITE_SUPABASE_ANON_KEY }}
          VITE_MIDTRANS_CLIENT_KEY: ${{ vars.VITE_MIDTRANS_CLIENT_KEY }}
      - uses: actions/upload-pages-artifact@v3
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

When implemented:
1. Set **Settings → Pages → Build and deployment → Source** to **GitHub Actions**.
2. Add the three public `VITE_` values as repository Actions variables.
3. Keep Server Key and service-role credentials out of build variables.
4. Configure backend secrets in Supabase separately.

## 8. Supabase CLI examples

After the Supabase project and functions exist:

```bash
supabase start
supabase functions serve create-snap-transaction
supabase functions serve midtrans-notification --no-verify-jwt
supabase functions deploy create-snap-transaction
supabase functions deploy midtrans-notification --no-verify-jwt
```

Set Midtrans Sandbox notification URL to:

```text
https://<PROJECT_REF>.supabase.co/functions/v1/midtrans-notification
```

The notification function may disable Supabase JWT verification because Midtrans does not send a user JWT. It must still verify Midtrans signature and transaction details.

## 9. Sandbox deployment checklist

1. Create Sandbox credentials in Midtrans Merchant Administration Portal under Settings → Access Keys.
2. Configure Sandbox Client Key in frontend public build configuration.
3. Configure `MIDTRANS_SERVER_KEY` in Supabase Edge Function secrets.
4. Configure notification URL in Midtrans Sandbox.
5. Run the test plan in [08 — Testing & Quality Assurance](./08-testing-quality-assurance.md).
6. Confirm only backend-verified payments update orders.

## 10. Official references

- [Vite static deployment](https://vite.dev/guide/static-deploy.html)
- [GitHub Pages](https://docs.github.com/en/pages)
- [GitHub Pages custom workflows](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages)
- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)
- [Supabase Edge Function secrets](https://supabase.com/docs/guides/functions/secrets)
- [Supabase deploy functions](https://supabase.com/docs/guides/functions/deploy)
- [Midtrans Sandbox testing](https://docs.midtrans.com/docs/testing-payment-on-sandbox)
