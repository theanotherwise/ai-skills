# Opsolving Vue 3 application architecture

Read this reference before creating the frontend structure, relocating code, or deciding whether behavior belongs in bootstrap, the root shell, a route view, a component, a composable, a service, a locale, or a global asset.

## Canonical source tree

```text
src/
|-- index.html
|-- tsconfig.json
|-- vite.config.ts
`-- app/
    |-- main.ts
    |-- App.vue
    |-- router.ts
    |-- i18n.ts
    |-- assets/
    |   `-- main.css
    |-- locales/
    |   `-- en.ts
    |-- views/
    |   `-- HomeView.vue
    `-- components/
        `-- ApiResults.vue
```

This tree is the minimal application vocabulary. It is deliberately flat while the application is small. Do not add empty `composables`, `services`, `stores`, `types`, `layouts`, or feature folders as placeholders.

The surrounding build contract matters when editing `src/app`: `src/index.html` loads `/app/main.ts`, Vite uses `src` as its root, and TypeScript includes `app/**/*.ts`, `app/**/*.d.ts`, and `app/**/*.vue`. Imports inside the application therefore begin from the local `app` tree unless the project already defines a synchronized Vite and TypeScript alias.

## Dependency direction

```text
src/index.html
      |
      v
app/main.ts --------------------> assets/main.css
      |
      +--------------+
      v              v
   App.vue        router.ts
      |              |
      +-------+------+
              v
            views
              |
              v
          components
              |
      +-------+--------+
      v                v
  i18n/locales   optional support modules
                 composables/services/types
```

Keep dependencies downward. `main.ts` may import the root component, router, selected global plugins, and global CSS. `router.ts` imports route-level views. Views import components and support modules. Components may import lower-level support modules, but they do not import views, `App.vue`, or `main.ts`.

`i18n.ts`, locale dictionaries, shared types, and pure helpers must not import route views or concrete UI components. A service must not manipulate Vue component instances or DOM elements. A composable may use Vue reactivity and lifecycle APIs, but it must not import its consuming component.

## Placement decision table

| Responsibility | Location | Boundary |
|---|---|---|
| Create and mount the Vue application | `app/main.ts` | Bootstrap only |
| Render the routed application shell | `app/App.vue` | Global layout only |
| Declare browser routes and history | `app/router.ts` | Explicit route table |
| Compose one route-level page | `app/views/<Name>View.vue` | Page layout and feature composition |
| Render one cohesive UI block | `app/components/<Name>.vue` | Typed UI contract and local behavior |
| Expose typed message lookup | `app/i18n.ts` | Translation access contract |
| Store one locale dictionary | `app/locales/<locale>.ts` | Message data only |
| Import Tailwind and application-wide CSS | `app/assets/main.css` | Global presentation entrypoint |
| Reuse reactive behavior across consumers | `app/composables/use<Name>.ts` | Added only when reuse exists |
| Share API/resource operations | `app/services/<resource>.ts` | Added only when transport is shared |
| Share non-reactive TypeScript contracts | `app/types/<domain>.ts` | Added only across multiple owners |
| Share a route shell across several pages | `app/layouts/<Name>Layout.vue` | Added only for real repeated layout |
| Hold application-wide client state | Existing store location | Only when an established store is justified |

## Core file responsibilities

### `app/main.ts`

`main.ts` is the complete bootstrap. It imports `createApp`, the root component, the exported router, and the global CSS entrypoint; then it installs already selected plugins and mounts to `#app`.

Do not put route declarations, page data, API clients, fetch calls, event listeners, timers, authentication decisions, or feature state here. A new feature should almost never make `main.ts` longer unless it introduces an explicitly approved application-wide plugin.

Keep plugin order explicit when order affects behavior. Do not auto-discover or globally register every component. Local imports make component dependencies visible and preserve tree shaking.

### `app/App.vue`

In the baseline routed application, `App.vue` contains only `<router-view />`. It may grow to contain UI shared by every route, such as a global header, footer, notification outlet, modal portal, or a shared layout boundary.

Do not put one route's page markup, API polling, or feature state in `App.vue`. If only some routes share a shell, create a layout component and compose it from the affected views instead of adding conditional route-specific branches to the root.

### `app/router.ts`

The router owns `createRouter`, browser-history selection, base URL handling, and the explicit route array. Every ordinary route targets a component from `views`. Route paths and names are public application contracts; preserve them during unrelated refactors.

Use `createWebHistory(import.meta.env.BASE_URL)` when the hosting server provides SPA fallback and the application base comes from Vite. Do not switch to hash history or hardcode a deployment prefix unless the hosting contract requires it.

Keep route metadata and guards here or in router-specific modules when they become substantial. Do not hide route registration in components, views, or a runtime directory scan.

### `app/views`

A view is a router target and page composition boundary. Name it `PascalCaseView.vue`, such as `HomeView.vue`, and let it own the top-level page landmarks, page-wide responsive layout, and assembly of feature components.

Views may read route params or query state, coordinate several components, and select which page-level state to pass downward. A view should not duplicate a reusable widget's internal markup. If a coherent section has its own state, loading behavior, or reusable presentation, extract it to `components`.

Do not import a view into an ordinary component. If two routes share presentation or behavior, extract a component, layout, composable, or service below both views.

### `app/components`

A component is a cohesive visual or interactive block. It owns its template, its typed public props/emits/slots, local state, computed values, and effects whose lifetime matches the component instance.

Keep components flat while the set is easy to navigate. When several files form one feature, group them under `components/<feature>/`. When generic design primitives emerge, use a small explicit group such as `components/base/`; do not label feature-specific components as generic merely because more than one page renders them.

Components may fetch data when the request belongs only to that widget, as in a self-refreshing API-results panel. When endpoint logic is shared or the transport dominates the component, move the transport to a service while the component remains responsible for presentation state.

### `app/i18n.ts` and `app/locales`

The baseline uses a lightweight typed dictionary rather than a framework plugin. `locales/en.ts` exports one `messages` object with `as const`; `i18n.ts` derives `MessageKey` from its keys and exposes `t(key)`.

Keep user-facing strings in the locale dictionary when the application already uses this contract. Keep technical identifiers, API paths, class names, and non-user-facing protocol values out of translations.

Add another locale or runtime switching only when requested. At that point, define how the locale is selected, keep every dictionary key-compatible, and change the translation contract deliberately. Do not install Vue I18n without dependency-change authorization.

### `app/assets`

`assets/main.css` is the single global stylesheet entrypoint. In the baseline it imports Tailwind. It may also contain application-wide tokens, resets, typography defaults, or truly global utility layers.

Do not move a component's one-off layout rules into the global stylesheet. Prefer the project's established Tailwind utilities in templates; use scoped component styles for local CSS that utilities cannot express clearly.

## Conditional scale-up structure

When real reuse or complexity appears, extend the tree narrowly:

```text
app/
|-- components/
|   |-- base/
|   `-- account/
|-- composables/
|   `-- usePolling.ts
|-- layouts/
|   `-- AccountLayout.vue
|-- services/
|   `-- account.ts
|-- types/
|   `-- account.ts
|-- views/
|   |-- HomeView.vue
|   `-- AccountView.vue
|-- locales/
|   `-- en.ts
|-- App.vue
|-- i18n.ts
|-- main.ts
`-- router.ts
```

Use `composables` for reusable reactive state or lifecycle behavior, not as a generic function directory. Use `services` for shared communication with APIs or browser services, not for view rendering. Use `types` only for contracts shared across several modules; otherwise colocate the type with its sole owner.

Add a store directory only when the project has chosen a store library or has an explicit store contract. Do not simulate a store with mutable module globals. Add `layouts` only when several views reuse the same page shell and slots make the contract clear.

## Naming and import rules

Use PascalCase for `.vue` files and component names: `ApiResults.vue`, `HomeView.vue`, `AccountLayout.vue`. Use camelCase for local values and functions, `usePascalCase` for composables, and lowercase domain names for TypeScript service or type files.

Import components and views explicitly. Avoid broad barrel files that obscure ownership or introduce cycles. A small intentional public export is acceptable when a feature package has several internal files and a stable external surface.

Preserve relative imports when that is the project's convention. If an alias is intentionally introduced, configure it identically in Vite and TypeScript and update tests/tooling that resolves modules; do not make TypeScript accept an import that the runtime bundler cannot resolve.

## Structural review checklist

- `main.ts` only assembles and mounts the application.
- `App.vue` contains only application-wide shell behavior.
- `router.ts` owns an explicit route table and targets views.
- Every routed page lives in `views` and uses a `*View.vue` name.
- Views compose components rather than duplicating reusable blocks.
- Components do not import views, `App.vue`, or `main.ts`.
- Shared reactive behavior is a composable only when multiple consumers need it.
- Shared transport is a service only when endpoint logic is reused or obscures UI code.
- Locale modules contain messages only and translation keys remain typed.
- Global CSS contains application-wide rules, not private feature styling.
- No new architectural directory or dependency exists without a real owner and use case.
