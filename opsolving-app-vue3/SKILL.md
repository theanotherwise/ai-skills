---
name: opsolving-app-vue3
description: Create and refactor Vue 3 applications using the Opsolving frontend layout with a minimal bootstrap, router-driven views, typed single-file components, explicit local state and lifecycle, lightweight i18n, and global assets. Use for Vue 3 application structure and UI feature work; do not use for Vue 2, Nuxt, React, backend services, or deployment infrastructure.
---

# Build structured Vue 3 applications

## Goal

Build a Vue 3 and TypeScript application under `src/app` whose startup files remain minimal and whose UI is divided by responsibility. Keep application bootstrapping in `main.ts`, the root render boundary in `App.vue`, route declarations in `router.ts`, route-level composition in `views`, cohesive UI and interactive behavior in `components`, shared translated messages in `locales` through `i18n.ts`, and global CSS entrypoints in `assets`.

The canonical composition path is:

```text
index.html -> app/main.ts -> App.vue + router.ts -> views -> components
```

Shared support code may be used from the owning view or component:

```text
views/components -> i18n + locales
views/components -> assets
views/components -> optional composables/services/types when reuse requires them
```

Do not put page rendering, API requests, timers, feature state, or route-specific logic in `main.ts`. Do not turn `App.vue` into a catch-all page component when the router is the application navigation boundary.

## Required workflow

1. Read the governing project instructions and inspect `package.json`, the package manager and lockfile, Vue/Vite/TypeScript versions, the Vite root, `index.html`, `tsconfig`, router setup, global styles, existing source structure, and tests.
2. Identify the requested routes, route-level views, reusable or feature-local components, props and events, local and shared state, API interactions, lifecycle resources, translation keys, styling scope, and accessibility behavior.
3. Read [references/architecture.md](references/architecture.md) before creating a source layout, moving application files, or deciding whether behavior belongs in a view, component, composable, service, locale, or global asset.
4. Read [references/components-and-state.md](references/components-and-state.md) before adding or substantially changing Vue single-file components, props, emits, slots, local state, lifecycle effects, timers, subscriptions, or shared reactive logic.
5. Read [references/routing-data-and-i18n.md](references/routing-data-and-i18n.md) before changing `main.ts`, `App.vue`, routes, navigation, browser history, API fetching, Vite proxy assumptions, translations, or global CSS.
6. Preserve existing public routes, route names, component contracts, translated keys, API paths, responsive behavior, and visual language unless the user explicitly requests a behavioral or design change.
7. Validate the narrowest relevant type, build, lint, formatting, and test surface without installing dependencies or starting Vite, preview servers, containers, or browsers unless explicitly authorized.

## Core source contract

Use these ownership rules:

- `src/app/main.ts` creates the Vue application, installs the router and already selected global plugins, imports the one global stylesheet entrypoint, and mounts to the `index.html` root.
- `src/app/App.vue` is the root shell. In a routed application it normally renders `<router-view />` and only adds layout shared by every route.
- `src/app/router.ts` creates and exports the router, selects history behavior, and declares the explicit route table whose components come from `views`.
- `src/app/views/` contains route targets. A view owns page-level layout and composes components for one route; it is not a generic reusable widget.
- `src/app/components/` contains cohesive UI blocks. Components own their template, typed component contract, local reactive state, and component-scoped effects.
- `src/app/i18n.ts` exposes the application's translation access contract and typed message keys.
- `src/app/locales/` contains locale dictionaries only. Locale modules do not contain component logic or HTML.
- `src/app/assets/` contains global CSS entrypoints and truly application-wide static assets. Feature-specific presentation remains with the feature component.

Add `composables`, `services`, `types`, `layouts`, or nested feature directories only when real shared responsibility appears. Do not create an enterprise folder tree for a small application. Do not add Pinia, Axios, Vue I18n, a UI framework, or another dependency merely to satisfy a directory name; use the dependencies already selected by the project unless the user explicitly requests a change.

## Minimal bootstrap and root

Keep `src/app/main.ts` limited to application assembly:

```ts
import { createApp } from 'vue'

import App from './App.vue'
import { router } from './router'
import './assets/main.css'

createApp(App).use(router).mount('#app')
```

Keep the routed root minimal:

```vue
<template>
  <router-view />
</template>
```

Register additional routes in `router.ts`, not in `main.ts` or `App.vue`. Put page content in a view. Add a shared header, footer, notification host, or layout shell to `App.vue` only when it truly applies across the routed application.

## Views and components

Create one PascalCase `*View.vue` file for each route-level page, for example `HomeView.vue`. A view composes child components and owns route-level layout, but it should delegate cohesive interactive blocks to components. The router imports views; ordinary components must not import views or the router table.

Create PascalCase `.vue` components such as `ApiResults.vue`. Use Vue 3 Composition API with `<script setup lang="ts">` for new code unless the existing project deliberately uses another consistent style. Keep the template, typed script, and optional component-specific style in the same single-file component.

Use typed `defineProps`, `defineEmits`, and slots for parent-child contracts. Keep mutable state local with `ref` or `reactive` until more than one owner genuinely needs it. Extract a `use<Name>` composable for shared reactive behavior; introduce a store only when application-wide state and the existing dependency policy justify one.

## Effects and data

Keep a one-off API request local to the component that owns the result. Extract shared transport or resource operations to `services/<resource>.ts` only when several consumers use the same endpoint contract or when transport concerns obscure the component. Use relative API paths such as `/api/...` when the development proxy and production reverse proxy provide a same-origin contract.

Treat external JSON as `unknown` and validate its shape before assigning typed state. Model loading, success, empty, and failure states according to the UI contract. Prevent overlapping polling requests, cancel or ignore stale work when necessary, and never use `any` as a shortcut around an external payload boundary.

Every timer, event listener, observer, subscription, or abortable request created for a mounted component must have a corresponding cleanup in `onBeforeUnmount` or the owning composable's disposal hook. Do not leave effects at module scope where they outlive component instances.

## Template and presentation rules

Use semantic HTML before adding ARIA. Preserve visible focus, keyboard operation, useful labels, and live-region behavior for asynchronously refreshed information. Use stable semantic keys in `v-for`; do not use an array index when the item has a durable identity.

Keep global framework imports and design tokens in `assets/main.css`. With the template's Tailwind setup, compose ordinary presentation with utility classes in Vue templates. Use component-scoped CSS for behavior or styling that cannot be expressed clearly through the established utility system; do not move one component's private styles into the global file.

Use translated keys for user-facing copy when the application already exposes `t`. Keep locale keys stable and typed. Do not hardcode a second translation mechanism inside a component or install a full i18n library when the current typed dictionary is sufficient.

## Existing applications

Do not reorganize an existing frontend solely because this skill was selected. Apply the layout when the user asks to create an application, add a feature within this structure, or refactor toward it. During a refactor, move one complete route or component behavior at a time and update imports, route registration, translation keys, styles, tests, and API assumptions together.

Preserve the project's chosen import style. The baseline uses relative imports and no path alias; do not invent an alias without updating TypeScript and Vite resolution together. Preserve eager or lazy route loading unless bundle behavior is intentionally being changed.

## Validation

Run the project's existing focused type checks, tests, lint, formatting checks, and production build when their dependencies are already available and the governing instructions permit them. At minimum, inspect every changed SFC and TypeScript file, check router reachability and import direction, run the available Vue TypeScript check, run `git diff --check`, and review repository status.

Do not run `pnpm install`, `npm install`, or another dependency fetch without authorization. Do not start `vite`, `pnpm dev`, preview servers, Docker Compose, local HTTP services, or browser verification unless explicitly requested. If dependencies are unavailable, report the exact skipped command instead of presenting structural inspection as runtime proof.
