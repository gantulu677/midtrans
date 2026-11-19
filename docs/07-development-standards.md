# 07 — Development Standards

**Status:** Target standards. Verify the existing repository before applying them.

## 1. Approved frontend stack

| Concern | Standard |
|---|---|
| UI framework | React 19 |
| Language | TypeScript |
| Build/dev server | Vite |
| CSS | Tailwind CSS v4 |
| Component library | Konsta UI React v5 |
| Backend SDK/platform | Supabase |

Konsta UI v5 targets React 19. Resolve version compatibility before adding dependencies; do not blindly upgrade or replace the existing app.

## 2. Dependency setup

In the existing React application, install:

```bash
npm install tailwindcss @tailwindcss/vite konsta
```

If React, Vite, or the React Vite plugin are not already configured, inspect the current package manifest and project setup before changing dependencies. Do not scaffold a second app.

## 3. Vite configuration

Add the Tailwind plugin while preserving the existing React plugin and deployment base:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  base: "/midtrans/",
});
```

The `/midtrans/` base is the expected GitHub Pages project-site path for this repository. Revisit it if the repository or hosting URL changes.

## 4. Main CSS entry

```css
@import "tailwindcss";
@import "konsta/react/theme.css";
```

Import Konsta components from `konsta/react`. Use one consistent `ios` or `material` theme after reviewing the existing storefront. Integrate Konsta into the current app shell; do not replace existing routes or application structure.

## 5. Example component integration

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

This is an integration illustration only, not a claim that these components or files currently exist.

## 6. Code and integration conventions

- Preserve existing folder structure, naming, routing, and component patterns where reasonable.
- Keep payment business logic out of presentation components.
- Keep Midtrans Server Key and elevated Supabase credentials server-side.
- Use typed request/response contracts for Edge Function calls.
- Validate all untrusted input at backend boundaries.
- Do not duplicate existing cart, checkout, order, or authentication logic.
- Keep changes focused on the approved payment scope.

## 7. Repository-first implementation rule

Before changing code, inspect the actual source tree, package manifest/lockfile, router, Supabase client, order flow, schema, and deployment workflow. Where current implementation differs from this target, document the gap and propose the smallest compatible change. Do not state that planned commands or code are implemented until verified.
