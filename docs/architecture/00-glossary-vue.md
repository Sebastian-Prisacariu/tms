# 00 — Effect + React for Vue Developers

> A quick-reference glossary that maps Effect and React patterns to Vue concepts you already know.

Read this page first if you're coming from Vue (Options API, Composition API, or Pinia). The rest of the architecture docs assume familiarity with these patterns.

---

## Core Concepts

| Effect / React pattern | Vue equivalent |
|---|---|
| `Effect.Effect<A, E>` | A composable that returns data and can fail with a typed error — like a `useFetch` that tracks error types |
| `Effect.gen(function* () { ... })` | The `setup()` function body or `<script setup>` — sequential async steps |
| `yield* BookingApi` | `inject('bookingApi')` — resolving a provided dependency |
| `Context.Tag` | An injection key — `const BookingApiKey = Symbol('BookingApi') as InjectionKey<BookingApi>` |
| `Layer` | `app.provide(BookingApiKey, implementation)` — binding an implementation to an injection key |
| `Layer.provide` | Nested `provide()` calls that wire implementations together |
| `Schema` | Zod/Valibot validation — defines shapes and validates data |
| `Schema.TaggedError` | A custom error class with a discriminant — like a typed error you'd use with `createError()` in Nuxt |
| `.pipe(fn1, fn2, fn3)` | Chaining `.value` transformations — like computed chains |
| Atom (`Atom.make`) | `ref()` in a composable or Pinia `state` — shared reactive state |
| Atom (`Atom.readable`) | `computed()` — derived reactive state |
| Atom (`apiRuntime.atom`) | Pinia action that fetches data + stores it in state — but reactive and auto-updating |
| `useAtomValue(atom)` | `storeToRefs(useBookingStore()).list` — subscribe to reactive state |
| `useAtomSet(atom)` | `useBookingStore().create(input)` — call a store action |

---

## Reading the Syntax

### State Management — Pinia vs Atoms

```ts
// Vue (Pinia store)
export const useBookingStore = defineStore('booking', () => {
  const list = ref<Booking[]>([])
  const loading = ref(false)
  const error = ref<Error | null>(null)

  async function fetchList(filters: BookingFilters) {
    loading.value = true
    try {
      list.value = await api.get('/bookings', { params: filters })
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  async function create(input: CreateBookingInput) {
    const booking = await api.post('/bookings', input)
    await fetchList({}) // refresh
    return booking
  }

  return { list, loading, error, fetchList, create }
})

// Usage in component
const store = useBookingStore()
const { list, loading } = storeToRefs(store)
store.fetchList({ page: 1 })
```

```ts
// React + Effect (Atoms)
import { Atom, Registry, Result } from '@effect-atom/atom'
import { Effect } from 'effect'
import { apiRuntime } from '~/lib/runtime'
import { BookingApi } from '../booking.api'

// Reactive query — auto-fetches, returns Result<BookingListResponse, E>
// (Result has three states: Initial, Success, Failure — no manual loading/error refs)
const list = apiRuntime.atom(
  Effect.gen(function* () {
    const api = yield* BookingApi
    return yield* api.list({})
  }),
)

// Mutation — POST then refresh
const create = apiRuntime.fn(
  Effect.fnUntraced(function* (input: CreateBookingInput) {
    const api = yield* BookingApi
    const booking = yield* api.create(input)
    const registry = yield* Registry.AtomRegistry
    registry.refresh(list)   // like fetchList() after create
    return booking
  }),
)

export const Booking = { list, create }

// Usage in component
const result = useAtomValue(Booking.list)     // like storeToRefs(store).list
const createBooking = useAtomSet(Booking.create)  // like store.create
```

**Key differences:**
- No manual `loading` / `error` refs — atoms return a `Result` with three states (`Initial`, `Success`, `Failure`) and a `.waiting` boolean
- No `try/catch` — errors are tracked in the type system via Effect
- Dependencies come from `yield*` (Effect's DI) instead of importing `api` directly

### Dependency Injection — Vue provide/inject vs Effect

```ts
// Vue — provide/inject
// In main.ts or a parent component:
const bookingApi = new BookingApiService(httpClient)
app.provide('bookingApi', bookingApi)

// In a child component:
const bookingApi = inject('bookingApi')!
```

```ts
// Effect — Context.Tag + Layer

// 1. Injection key (like the provide key)
class BookingApi extends Context.Tag('BookingApi')<
  BookingApi,
  { readonly getById: (id: string) => Effect.Effect<Booking, BookingNotFound> }
>() {}

// 2. Implementation (like the service class)
const BookingApiLive = Layer.effect(
  BookingApi,
  Effect.gen(function* () {
    const client = yield* ApiHttpClient
    return { getById: (id) => /* ... */ }
  }),
)

// 3. Wiring (like app.provide)
const AppLayer = BookingApiLive.pipe(Layer.provide(ApiHttpClientLive))

// 4. Usage (like inject)
const api = yield* BookingApi
```

**Key difference:** Vue's `provide`/`inject` is stringly-typed (or uses `InjectionKey`). Effect's `Context.Tag` is fully type-safe — TypeScript knows exactly what shape the service has, and the compiler catches missing providers at build time (not at runtime).

### Routing — Vue Router vs TanStack Router

```ts
// Vue Router
const routes = [
  {
    path: '/bookings',
    component: () => import('./views/BookingList.vue'),
    meta: { requiresAuth: true },
  },
  {
    path: '/bookings/:id',
    component: () => import('./views/BookingDetail.vue'),
    props: true,
  },
]

// Navigation guard
router.beforeEach((to) => {
  if (to.meta.requiresAuth && !isAuthenticated()) {
    return '/login'
  }
})
```

```tsx
// TanStack Router (file-based)
// src/routes/_authenticated/bookings.tsx — file name = URL path
export const Route = createFileRoute('/_authenticated/bookings')({
  component: BookingList,
})

// src/routes/_authenticated.tsx — pathless layout = auth guard
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: ({ context }) => {
    if (!context.auth.isAuthenticated) throw redirect({ to: '/login' })
  },
  component: () => <><Sidebar /><Outlet /></>,
})
```

**Key difference:** TanStack Router uses file-based routing (file names become URLs) with full type safety on params and search params. Vue Router uses a programmatic route array. Both support lazy loading and navigation guards.

---

## Architecture Mapping

| Vue concept | TMS equivalent | File location |
|---|---|---|
| Pinia store | Atoms namespace (`export const Booking = { list, create }`) | `src/features/*/booking.atoms.ts` |
| Pinia action | `apiRuntime.fn(...)` mutation atom | `src/features/*/booking.atoms.ts` |
| Composable (`use*.ts`) | React hook (`use*.ts`) | `src/features/*/booking.hooks.ts` |
| `provide()` / `inject()` | `Context.Tag` / `yield*` | `src/features/*/booking.api.ts` |
| `app.provide()` | `Layer.provide()` in `runtime.ts` | `src/lib/runtime.ts` |
| Vue Router route config | File-based route (`.tsx` file) | `src/routes/` |
| Navigation guard | `beforeLoad` on route | `src/routes/_authenticated.tsx` |

---

## See Also

- [00-effect-glossary.md](./00-effect-glossary.md) — general Effect glossary (framework-agnostic)
- [02-effect-services.md](./02-effect-services.md) — full service pattern with `Context.Tag`, `Layer`, and tagged errors
- [04-state-management.md](./04-state-management.md) — all atom patterns (the Pinia equivalent)
- [05-feature-anatomy.md](./05-feature-anatomy.md) — complete walkthrough of a feature from schema to component
