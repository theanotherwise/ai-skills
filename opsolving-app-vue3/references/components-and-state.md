# Vue components and state

Read this reference before creating or substantially changing Vue single-file components, component contracts, local state, computed values, lifecycle effects, polling, events, slots, composables, or shared client state.

## Single-file component contract

Use Vue 3 single-file components with a template and `<script setup lang="ts">`. Add a style block only when the component owns CSS that is not better expressed through the project's established utility classes.

The baseline order is template first, then script:

```vue
<template>
  <section aria-labelledby="status-title">
    <h2 id="status-title">{{ title }}</h2>
    <p>{{ status }}</p>
  </section>
</template>

<script setup lang="ts">
defineProps<{
  title: string
  status: string
}>()
</script>
```

Preserve the repository's established SFC block order and formatting. Do not mix Options API and Composition API inside one component without a concrete compatibility reason. Do not use JavaScript-only script blocks in an otherwise strict TypeScript application.

## Component ownership

A component owns one cohesive visual or interactive responsibility. It may own:

- semantic markup and Tailwind classes for that block;
- typed props, emits, and slots;
- local `ref` or `reactive` state;
- `computed` values derived from its inputs and state;
- event handlers for its own interaction;
- API activity used only by that block;
- timers, observers, and subscriptions bounded to its mounted lifetime;
- mapping from technical results to its presentation state.

A component must not own the route table, application mount, unrelated page sections, or another component's internal state. Split a component when independent sections have separate contracts, effects, or reuse. Do not split purely to reduce line count if the result scatters one cohesive behavior across many files.

## View versus component

Create a view when the file is the direct target of a route. Create a component when the file is rendered by a view or another component as a cohesive block.

A view normally owns page landmarks and layout:

```vue
<template>
  <main class="min-h-screen">
    <ApiResults />
  </main>
</template>

<script setup lang="ts">
import ApiResults from '../components/ApiResults.vue'
</script>
```

Keep route-only layout in the view. Keep the results table, refresh behavior, and endpoint presentation in `ApiResults.vue`. If another view later needs the same results block, it can import the component without importing the original view.

## Props

Declare props with TypeScript types in `defineProps`. Use required props when the component cannot render meaningfully without the value. Use optional props only when absence has a defined behavior.

```ts
type StatusTone = 'neutral' | 'success' | 'error'

const props = withDefaults(
  defineProps<{
    label: string
    tone?: StatusTone
  }>(),
  {
    tone: 'neutral',
  },
)
```

Treat props as immutable. Derive local display values with `computed`; do not copy every prop into a `ref` and then synchronize it manually. Create local editable state only when the component intentionally owns a draft distinct from the parent value.

Do not pass a large untyped object when the component needs a small explicit contract. Keep a type next to its sole component owner; move it to `types` only when several modules consume the same domain contract.

## Emits and two-way state

Declare emits explicitly and type their payloads:

```ts
const emit = defineEmits<{
  select: [id: string]
  retry: []
}>()
```

Use events to report user intent or completed component actions upward. Name events for what happened, not for the implementation method. Do not mutate parent-owned state through imported objects or module globals.

Use `defineModel` only when two-way binding is the intentional public contract, such as a form control value. Do not use two-way binding to bypass clear ownership for complex feature state.

## Slots

Use slots when the parent must supply structure or content while the child owns the surrounding layout. Name slots when several insertion points exist and type slot props when the component exposes values to slot content.

Do not add a slot for a fixed string or a one-off element that is simpler as a prop. Do not use slots to expose the component's private implementation state without a stable public contract.

## Local state and derived state

Use `ref` for scalar values and replaceable objects or collections. Use `reactive` when a cohesive object is mutated by property and replacement is not the normal operation. Prefer `computed` for values derived from other reactive state.

```ts
import { computed, ref } from 'vue'

type UserSummary = { name: string }

const query = ref('')
const users = ref<readonly UserSummary[]>([])

const matchingUsers = computed(() => {
  const normalizedQuery = query.value.trim().toLocaleLowerCase()
  if (!normalizedQuery) return users.value

  return users.value.filter((user) =>
    user.name.toLocaleLowerCase().includes(normalizedQuery),
  )
})
```

Do not store a second mutable copy of a value that can be computed. Do not mutate readonly props or exported constants. Use immutable replacement when it makes async state transitions and array identity clearer.

## Lifecycle effects

Start instance-bound effects in the appropriate Vue lifecycle hook and release them when the instance is removed. Common pairs include:

- `onMounted` with `onBeforeUnmount` for browser-only timers and listeners;
- `watch` with its cleanup callback for work tied to reactive inputs;
- `AbortController` with abort during replacement or unmount for cancellable fetches;
- observer `observe` with `disconnect` during cleanup;
- subscription `subscribe` with returned unsubscribe during cleanup.

For polling, keep the timer handle typed for the browser, prevent overlapping requests, and always clear the interval:

```ts
import { onBeforeUnmount, onMounted } from 'vue'

let refreshTimer: number | undefined
let requestInFlight = false

async function refresh() {
  if (requestInFlight) return
  requestInFlight = true

  try {
    await loadCurrentState()
  } finally {
    requestInFlight = false
  }
}

onMounted(() => {
  void refresh()
  refreshTimer = window.setInterval(() => void refresh(), 1_000)
})

onBeforeUnmount(() => {
  if (refreshTimer !== undefined) window.clearInterval(refreshTimer)
})
```

Use `try/finally` so an unexpected failure cannot permanently lock an in-flight guard. Do not start a timer at module import time. Do not leave promises unhandled; use `void` only when the asynchronous action deliberately has internal error handling.

## External data boundaries

Treat parsed JSON as `unknown`. Validate the shape before accessing fields or asserting a domain type:

```ts
type ApiPayload = Record<string, unknown>

function isApiPayload(value: unknown): value is ApiPayload {
  return value !== null && typeof value === 'object' && !Array.isArray(value)
}

async function getApiPayload(path: string): Promise<ApiPayload> {
  const response = await fetch(path, {
    headers: { Accept: 'application/json' },
  })

  if (!response.ok) {
    throw new Error(`API request failed with status ${response.status}`)
  }

  const payload: unknown = await response.json()
  if (!isApiPayload(payload)) {
    throw new Error('API returned an invalid JSON object')
  }

  return payload
}
```

Validate every field the component depends on. A successful HTTP status does not prove the response shape. Do not use `as SomeType` on unvalidated external data merely to satisfy TypeScript.

Model the UI states required by the product: initial/loading, available data, empty data, recoverable failure, and stale data when relevant. Do not silently render an empty success state for a failure unless that is the explicit interface contract.

## Lists and keyed rendering

Use a stable identity in `v-for`:

```vue
<li v-for="endpoint in endpoints" :key="endpoint.path">
  {{ endpoint.label }}
</li>
```

Do not use the array index when items can be inserted, removed, reordered, or independently updated. Keep a static configuration list `readonly` when component code must not mutate it.

Move expensive filtering or sorting out of the template and into `computed`. Keep template expressions short enough to reveal the rendered structure without reverse-engineering business logic.

## Composables

Extract `composables/use<Name>.ts` when reactive behavior is shared by multiple consumers or when one component's effect orchestration becomes independently testable and reusable. A composable may own refs, computed state, watchers, and lifecycle hooks.

Make inputs and outputs explicit. Return readonly state when consumers should not mutate it. Keep DOM assumptions, route assumptions, and API endpoints visible in the composable contract instead of reaching through unrelated global state.

Do not extract a composable merely because a component has several lines of script. A one-off effect that is easiest to understand beside its template should remain in the component.

## Shared application state

Lift state to the nearest common owning view when sibling components coordinate. Use props downward and events upward. Extract a composable when several branches reuse reactive logic but do not need a global singleton.

Introduce a store only for durable application-wide state with multiple unrelated consumers, such as authenticated identity or a shared cart, and only when the project has selected a store solution. Do not install Pinia or create mutable module-level pseudo-stores as a default structural step.

Keep server state separate from local form and presentation state. Do not mirror every API response into a global store when one component or route owns it.

## Accessibility and semantics

Use native landmarks and elements first: `main`, `nav`, `section`, headings, `button`, `label`, lists, tables, and `time`. Use buttons for actions and links for navigation. Do not attach click behavior to a non-interactive `div` without complete keyboard semantics.

Label controls and regions. Use `aria-live="polite"` for non-disruptive asynchronous status when users need updates announced. Mark decorative overlays with `aria-hidden="true"`. Preserve focus visibility and do not remove an outline without an accessible replacement.

For dynamic failures and loading states, provide meaningful visible text when the interface requires it; color alone is not sufficient. Keep heading levels and landmark nesting coherent at the view level.

## Component review checklist

- The file is correctly classified as a route view or ordinary component.
- The SFC uses the project's Vue 3 TypeScript convention consistently.
- Props are typed, minimal, and not mutated.
- Emits describe intentional upward events with typed payloads.
- Derived values use `computed` instead of duplicated mutable state.
- External data starts as `unknown` and is validated before use.
- Loading, empty, failure, and success states follow the requested UI contract.
- Timers, listeners, observers, subscriptions, and requests have cleanup.
- Polling cannot overlap indefinitely or remain locked after failure.
- `v-for` uses stable semantic keys.
- Markup uses native semantics and supports keyboard and assistive technology.
- Shared state is lifted or extracted only as far as its actual consumers require.
