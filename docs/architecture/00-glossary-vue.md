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

## The Effect Type Up Close

`Effect.Effect<A, E, R>` has three type parameters:

| Position | Meaning | Vue analogy |
|---|---|---|
| `A` | Value on **success** | What a composable returns (`Booking`, `Route[]`, `void`) |
| `E` | Value on **typed failure** | The error shape a composable can produce — tracked by the compiler |
| `R` | **Requirements** — services that must be provided | `inject()` keys needed to run — checked at compile time |

```ts
Effect.Effect<Booking, BookingNotFound, BookingApi>
// "Returns a Booking, or fails with BookingNotFound, and needs BookingApi in context"
```

Imagine if a Vue composable's signature encoded "this calls `inject(BookingApiKey)` and returns `Booking | BookingNotFound`" — and the compiler enforced that you provided the injection key at the top and handled the error at the call site. That's the Effect type.

You'll often see `Effect.Effect<A, E>` (two params) — that's shorthand for `R = never`, meaning "no requirements left, ready to run."

---

## Other Core Types You'll See

### `Option<A>` — a value that may not be there

`Option<A>` replaces `T | null` / `T | undefined`. Either `Some(value)` or `None`, and the compiler forces you to handle both.

```ts
import { Option } from 'effect'

const maybe: Option.Option<Booking> = Option.some(booking)
const empty: Option.Option<Booking> = Option.none()

Option.match(maybe, {
  onSome: (b) => `Got ${b.id}`,
  onNone: () => 'No booking',
})

Option.getOrElse(maybe, () => defaultBooking)
```

**Vue analogy:** like a `Ref<T | null>` where the compiler refuses to let you touch `.value.foo` before you null-check. Replaces the `v-if="data"` + `if (!data.value) return` patterns with a value you can transform in place.

### `Either<E, A>` — synchronous success-or-failure

`Either<E, A>` is Effect's synchronous cousin: `Right<A>` (success) or `Left<E>` (failure). No async, no services — just a value-returning alternative to exceptions for pure code.

```ts
import { Either } from 'effect'

const ok: Either.Either<number, string> = Either.right(42)
const bad: Either.Either<number, string> = Either.left('oops')

Either.match(bad, {
  onLeft: (e) => `Failed: ${e}`,
  onRight: (n) => `Got: ${n}`,
})
```

**Vue analogy:** like returning a discriminated union `{ ok: true, data: T } | { ok: false, error: E }` from a function, which you then narrow with an `if (result.ok)` check — except `Either` comes with `match`, `map`, and composition helpers built in.

**When to use which:**
- `Option<A>` — "is there a value?" (lookups, find methods, maybe-present data)
- `Either<E, A>` — "did this synchronous thing fail?" (parsing, pure validation)
- `Effect<A, E, R>` — anything async, or anything that needs services

Rule of thumb: `Option` / `Either` in pure helpers; `Effect` the moment you touch a fetch, a store mutation, or `inject()`.

### `Data.TaggedEnum` — discriminated unions without boilerplate

```ts
import { Data } from 'effect'

type DraftState = Data.TaggedEnum<{
  Local: { readonly data: DraftData }
  Persisted: { readonly id: string; readonly data: DraftData }
}>
const DraftState = Data.taggedEnum<DraftState>()

const s1 = DraftState.Local({ data: empty })
const s2 = DraftState.Persisted({ id: 'abc', data: empty })

// Exhaustive, compiler-checked match
DraftState.$match(s1, {
  Local: () => 'not saved yet',
  Persisted: (p) => `saved as ${p.id}`,
})
```

**Vue analogy:** the same pattern you'd model in a Pinia store as `type State = { status: 'Local', data: ... } | { status: 'Persisted', id: string, data: ... }` — except constructors, type guards, and the exhaustive matcher are generated for you. No more forgetting to handle the new variant you just added.

### `Cause<E>` and `Exit<A, E>` — the full failure picture

Every Effect ends with an `Exit<A, E>`:
- `Exit.Success(a)` — normal path
- `Exit.Failure(cause)` — something went wrong; `cause` is a `Cause<E>`

A `Cause<E>` distinguishes:
- **Expected failures** (`Fail<E>`) — your tagged errors, what callers handle
- **Defects** — unexpected `throw`s, real bugs
- **Interruptions** — a fiber was cancelled (closest Vue analogy: component unmounting before an `await` resolves)

You rarely inspect `Cause` by hand — `Effect.catchTag`, `Effect.catchAll`, and `Effect.catchAllCause` do it for you. But the split is why Effect treats "expected error path" and "something exploded" as genuinely different things, instead of both being a raw `Error` caught in a single `try/catch`.

---

## Key Combinators — One-Line Reference

| Operator | What it does | Promise/Vue analogy |
|---|---|---|
| `Effect.succeed(a)` | Lift a value into Effect | `Promise.resolve(a)` |
| `Effect.fail(e)` | Produce a typed failure | `Promise.reject(e)` (but typed) |
| `Effect.sync(() => x)` | Wrap a synchronous side-effect | A setter you want lazily evaluated |
| `Effect.tryPromise(...)` | Bring a Promise into Effect | Wrapping `fetch`, `axios`, or any async SDK |
| `Effect.map(eff, fn)` | Transform the success value | `.then(fn)` (sync fn) — or `computed()` over a value |
| `Effect.flatMap(eff, fn)` | Chain another Effect | `.then(fn)` (fn returns a Promise) |
| `Effect.all([a, b, c])` | Run in parallel, collect results | `Promise.all([a, b, c])` |
| `Effect.catchTag('Tag', fn)` | Recover from one tagged error | `catch (e) { if (e._tag === 'Tag') ... }` |
| `Effect.catchAll(fn)` | Recover from any typed error | `.catch(fn)` |
| `Effect.mapError(fn)` | Transform the error channel | Rewrapping an error before re-throwing |
| `Effect.tap(fn)` | Run a side-effect, keep the value | `.then(v => { log(v); return v })` |
| `Effect.orElse(() => fallback)` | Fall back on failure | `.catch(() => fallback)` |

Inside `.pipe(...)`, these stack. A long pipeline is just these operators composed top-to-bottom — structurally similar to chaining `.value` transformations through a series of `computed()` refs.

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
