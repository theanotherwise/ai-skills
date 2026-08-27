# Vue routing, data access, i18n, and global styling

Read this reference before changing the application bootstrap, root shell, route table, navigation, browser history, API requests, development proxy assumptions, translations, locale dictionaries, or global CSS.

## Bootstrap contract

The HTML entrypoint provides one mount element and loads `/app/main.ts`:

```html
<body>
  <div id="app"></div>
  <script type="module" src="/app/main.ts"></script>
</body>
```

`app/main.ts` creates one Vue application, installs the exported router, imports global CSS once, and mounts to the matching selector:

```ts
import { createApp } from 'vue'

import App from './App.vue'
import { router } from './router'
import './assets/main.css'

createApp(App).use(router).mount('#app')
```

Keep these two sides synchronized. Do not change the mount ID in only one file. Do not mount several application instances for ordinary features. Install a global plugin here only after it has been selected as an application-wide dependency.

## Root shell

For a routed application, `App.vue` exposes the router boundary:

```vue
<template>
  <router-view />
</template>
```

Use the root for UI and providers shared across every route. Keep route-specific elements in views. Avoid inspecting the current path in `App.vue` to manually select pages; that duplicates router responsibility.

If global UI needs route-aware behavior, prefer router metadata and a clear root/layout contract. Do not create a chain of pathname string comparisons in the template.

## Router definition

Create and export one router from `app/router.ts`:

```ts
import { createRouter, createWebHistory } from 'vue-router'

import HomeView from './views/HomeView.vue'

export const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      name: 'home',
      component: HomeView,
    },
  ],
})
```

Every route has a stable path and should have a stable name when code navigates to it or tests identify it. Route components come from `views`. Keep redirects, aliases, props mapping, metadata, and guards explicit in the route definition.

Use eager imports while the application is small and the existing build does so. Use dynamic imports for route-level splitting only when bundle size or loading behavior justifies it:

```ts
{
  path: '/settings',
  name: 'settings',
  component: () => import('./views/SettingsView.vue'),
}
```

Do not change eager routes to lazy routes as an unrelated formatting refactor. Chunking changes loading and failure behavior and deserves intentional validation.

## Browser history and hosting

`createWebHistory(import.meta.env.BASE_URL)` assumes the production server falls back unknown application paths to `index.html`. Preserve the server's SPA fallback when adding nested client routes.

Do not switch to `createWebHashHistory` to work around a missing server fallback without an explicit hosting decision. Do not hardcode `/` as the base when Vite's configured base may differ between deployments.

Use router navigation APIs or `<RouterLink>` for client-side navigation. Do not use a plain location reload for internal navigation unless a full document reload is intentionally required.

## Route parameters and query state

Read route parameters in the route-level view or a route-specific composable and translate them into typed component inputs. Validate required identifiers before using them in API paths.

Keep durable shareable UI state in the URL when the product contract calls for it, such as search, filters, pagination, or selected tabs. Keep ephemeral presentation state local when it does not need history, bookmarking, or cross-navigation persistence.

Do not let ordinary child components parse global route state implicitly when the owning view can pass a clear typed prop. A component that is intentionally router-aware should make that coupling obvious.

## API origin contract

The baseline uses relative paths such as `/api/help/time`. During development, Vite proxies `/api` to the backend. In production, the reverse proxy or ingress is expected to expose frontend and API through the corresponding same-origin path.

Keep components and services independent of development-only container hostnames. Do not hardcode `http://template-api:8080`, localhost ports, cluster service names, or production domains in Vue code when the proxy contract provides `/api`.

When an application intentionally uses a separate API origin, read it through an explicitly documented Vite environment variable, validate its presence, and preserve the target project's CORS and credential contract. Do not introduce a second URL configuration mechanism ad hoc.

## Fetch ownership

Use the browser's `fetch` when it already satisfies the project. Do not add Axios or another transport dependency for ordinary JSON requests without an explicit need and dependency authorization.

A request used by exactly one component may remain local. Extract `services/<resource>.ts` when:

- several views or components call the same resource;
- request construction, headers, response validation, or error translation is repeated;
- the component becomes difficult to read because transport dominates presentation;
- a stable API operation deserves independent focused tests.

Keep a service framework-light. It may expose typed async operations and validate external payloads, but it must not import `.vue` files, manipulate component refs, render notifications directly, or own route navigation unless it is explicitly a navigation service.

## Request and response handling

Set headers intentionally, check `response.ok`, and parse external payloads as `unknown`. Define the minimum validated result the caller needs instead of leaking a broad arbitrary response object throughout the application.

Distinguish aborts, expected HTTP failures, invalid payloads, and unexpected failures when the UI treats them differently. Preserve useful internal cause information for logging or diagnostics without showing implementation details or secrets to users.

For concurrent independent requests, `Promise.all` is appropriate when all results are handled and one rejection should follow the intended aggregate behavior. Catch per-item failures when the interface shows partial results. Do not accidentally discard successful items because one optional endpoint failed.

For requests superseded by a new query or route, use `AbortController`, a request generation token, or another explicit stale-result guard. Do not let a slower old response overwrite state for a newer user selection.

## Polling

Poll only when the feature contract needs refreshed data. Keep the interval visible as a named constant when it carries product meaning. Prevent overlapping requests and clean up on unmount.

Decide how failure affects the last successful value: preserve stale data with an error indicator, replace it with an empty state, or clear it only when the product requires that behavior. Do not silently choose a failure model just because it is easy to implement.

Pause or reduce unnecessary polling when the application has an established visibility policy. Do not add complex backoff or browser-visibility machinery to a simple template unless the task requires it.

## Typed local translations

The baseline locale exports one immutable message object:

```ts
export const messages = {
  endpoint: 'Endpoint',
  response: 'Response',
  timestampUtc: 'Timestamp (UTC)',
} as const
```

The translator derives valid keys from that object:

```ts
import { messages } from './locales/en'

export type MessageKey = keyof typeof messages

export function t(key: MessageKey): string {
  return messages[key]
}
```

This contract intentionally catches misspelled translation keys at type-check time. Component configuration containing a translation key uses `MessageKey`, not an arbitrary `string`.

Keep keys semantic and stable. Use names such as `saveChanges` or `emptyResults`, not the complete English sentence transformed into an identifier. Avoid using a translated string as a route name, storage key, API value, or test selector.

## Adding locales

Do not add locale switching when only one language is required. When multiple locales are requested, keep each dictionary compatible with the canonical key set and define fallback behavior explicitly.

For a small application, a typed record of locale dictionaries and a reactive current locale may be enough. For pluralization, message formatting, locale-aware routing, async locale bundles, or a large translation workflow, evaluate a dedicated i18n library—but do not add or install it without authorization.

Keep user-selected locale persistence and browser-locale detection deterministic. Do not change language during hydration or navigation without a clearly defined source of truth.

## Global CSS and Tailwind

The baseline global entrypoint is intentionally small:

```css
@import "tailwindcss";
```

Import it once from `main.ts`. Do not import the same global stylesheet from several components. Put application-wide design tokens, base typography, and global resets here only when they are genuinely global.

Use Tailwind utilities directly in templates for ordinary layout, spacing, color, typography, and responsive changes. Keep class groups readable and follow the repository's existing formatting. Do not replace the established design language with unrelated component-library defaults.

Use semantic elements and responsive constraints intentionally. Horizontal scrolling for a wide data table belongs on the smallest container that requires it; avoid causing page-wide overflow. Decorative layers must not intercept input or be announced by assistive technology.

## Vite and TypeScript boundary

The Vite configuration lives outside `app`, but changes inside `app` must respect it. The baseline uses `root: 'src'`, Vue and Tailwind plugins, a development `/api` proxy, and an explicit preview host/port.

The TypeScript configuration uses strict mode, bundler resolution, DOM libraries, no emit, and includes the entire app tree. Do not weaken strictness or add `skip` patterns to hide errors introduced by a feature.

When adding an import alias, environment variable, plugin, or nonstandard asset handling, update Vite types and TypeScript resolution together. A type check alone does not prove that Vite can bundle an unresolved runtime alias.

## Routing, data, and i18n review checklist

- `index.html` and `main.ts` agree on the mount selector and entry path.
- `main.ts` installs only intentional application-wide plugins.
- `App.vue` delegates route rendering to `<router-view />`.
- Every route targets a view and preserves stable path/name contracts.
- History mode matches the production SPA fallback and Vite base.
- Internal navigation uses router APIs rather than full reloads.
- API requests use the intended same-origin or environment-configured base.
- External payloads are validated from `unknown`.
- Polling and superseded requests cannot leak or overwrite newer state.
- Shared transport is extracted only when it has multiple consumers or substantial logic.
- Translation keys are typed and locale modules contain data only.
- Global CSS is imported once and remains application-wide.
- Vite and TypeScript configuration stay synchronized when resolution changes.
