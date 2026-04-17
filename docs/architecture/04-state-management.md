# 04 — State Management

> All state through `@effect-atom/atom`. No `useState`/`useEffect` for domain data.
> New to Effect? Start with [00-effect-glossary.md](./00-effect-glossary.md) for a quick mental-model mapping.

## Overview

We use [`@effect-atom/atom`](https://github.com/tim-smart/effect-atom) (v0.5.x) for all state management. This library provides reactive atoms that integrate natively with Effect services, streams, and the full Effect ecosystem.

**Rule:** React `useState` is only for ephemeral UI state (modal open/closed, hover, focus). All domain data, server state, and derived computations flow through atoms.

## Table of Contents

### Essentials
- [Registry Provider Setup](#registry-provider-setup)
- [Simple Atoms](#simple-atoms)
- [Derived Atoms (Readable)](#derived-atoms-readable)
- [Runtime Atoms (Service-Backed)](#runtime-atoms-service-backed)
- [Mutations with runtimeAtom.fn](#mutations-with-runtimeatomfn)
- [Result Handling](#result-handling)
- [React Hooks](#react-hooks)
- [Namespace Export Convention](#namespace-export-convention)
- [When to Use React State vs Atoms](#when-to-use-react-state-vs-atoms)

### Advanced Patterns
- [Writable Atoms](#writable-atoms)
- [Registry Access in Mutations](#registry-access-in-mutations)
- [Cross-Atom Dependency API](#cross-atom-dependency-api)
- [Self-Initializing Pattern](#self-initializing-pattern)
- [Reactive Sync Atoms](#reactive-sync-atoms)
- [Persistent Atoms (Atom.kvs)](#persistent-atoms-atomkvs)
- [ScopedAtom](#scopedatom)
- [Atom.family](#atomfamily)

---

## Registry Provider Setup

A single `RegistryProvider` wraps the entire app. We add a nesting guard to prevent silent communication failures between separate registries.

```tsx
// src/lib/providers.tsx
import { RegistryProvider as AtomRegistryProvider } from '@effect-atom/atom-react'
import { type ReactNode, createContext, useContext } from 'react'

const AtomProviderContext = createContext(false)

export function AtomProvider({ children }: { children: ReactNode }) {
  const isNested = useContext(AtomProviderContext)

  if (isNested) {
    throw new Error(
      '[AtomProvider] Nested AtomProvider detected. ' +
        'Only use ONE AtomProvider at the root. ' +
        'Nesting creates separate registries that cannot communicate.'
    )
  }

  return (
    <AtomProviderContext.Provider value={true}>
      <AtomRegistryProvider defaultIdleTTL={400}>
        {children}
      </AtomRegistryProvider>
    </AtomProviderContext.Provider>
  )
}
```

---

## Simple Atoms

The most basic atom — holds a value, can be read and written.

```ts
import { Atom } from '@effect-atom/atom'

// Simple writable atom with initial value
const currentStep = Atom.make<'pickup' | 'dropoff' | 'confirm'>('pickup')

// Boolean flag
const sidebarOpen = Atom.make(false)

// Nullable value
const selectedBookingId = Atom.make<string | null>(null)
```

---

## Derived Atoms (Readable)

Read-only atoms that derive from other atoms. They recompute when dependencies change.

```ts
import { Atom } from '@effect-atom/atom'

const isLoggedIn = Atom.make(false)
const hasOrganization = Atom.make(false)

// Derived: recomputes when isLoggedIn or hasOrganization change
const canCreateBooking = Atom.readable((get) => {
  return get(isLoggedIn) && get(hasOrganization)
})

// Derived from other derived atoms — dependency chains work naturally
const stepList = Atom.readable((get) =>
  buildStepList({
    isLoggedIn: get(isLoggedIn),
    hasOrganization: get(hasOrganization),
  })
)

const currentStepIndex = Atom.readable((get) => {
  const steps = get(stepList)
  const current = get(currentStep)
  return steps.indexOf(current)
})

const progress = Atom.readable((get) => {
  const steps = get(stepList)
  const index = get(currentStepIndex)
  if (steps.length <= 1) return 100
  return Math.round((index / (steps.length - 1)) * 100)
})
```

---

## Runtime Atoms (Service-Backed)

`Atom.runtime(Layer)` creates an `AtomRuntime` that can `yield*` Effect services. This is how we bridge the Effect service world into atoms.

```ts
// src/lib/runtime.ts
import { Atom } from '@effect-atom/atom'
import { AppApiLayer } from './http-client'

// Create a runtime with all API services available
export const apiRuntime = Atom.runtime(AppApiLayer)
```

```ts
// features/booking/state/booking.atoms.ts
import { Atom, Result } from '@effect-atom/atom'
import { Effect } from 'effect'
import { apiRuntime } from '~/lib/runtime'
import { BookingApi } from '../api/booking.api'

// Atom that fetches data from a service — returns Atom<Result<Booking[], E>>
const list = apiRuntime.atom(
  Effect.gen(function* () {
    const api = yield* BookingApi
    return yield* api.list({})
  })
)

// Atom with parameters — use a function that returns an Effect
const byId = (id: string) =>
  apiRuntime.atom(
    Effect.gen(function* () {
      const api = yield* BookingApi
      return yield* api.getById(id)
    })
  )
```

---

## Mutations with runtimeAtom.fn

Side-effectful operations (create, update, delete) use `runtimeAtom.fn`. These return writable atoms — call `useAtomSet` to get a trigger function.

```ts
// features/booking/state/booking.atoms.ts
import { Registry } from '@effect-atom/atom'
import { Effect } from 'effect'
import { apiRuntime } from '~/lib/runtime'
import { BookingApi } from '../api/booking.api'
import type { CreateBookingInput } from '../contract/schemas'

const create = apiRuntime.fn(
  Effect.fnUntraced(function* (input: CreateBookingInput) {
    const api = yield* BookingApi
    const booking = yield* api.create(input)

    // Refresh the list atom after creating
    const registry = yield* Registry.AtomRegistry
    registry.refresh(list)

    return booking
  })
)

const remove = apiRuntime.fn(
  Effect.fnUntraced(function* (bookingId: string) {
    const api = yield* BookingApi
    yield* api.delete(bookingId)

    const registry = yield* Registry.AtomRegistry
    registry.refresh(list)
  })
)
```

**Usage in components:**

```tsx
import { useAtomSet, useAtomValue } from '@effect-atom/atom-react'

function CreateBookingButton() {
  const createBooking = useAtomSet(Booking.create)

  const handleCreate = () => {
    createBooking({
      pickupAddress: '123 Main St',
      dropoffAddress: '456 Oak Ave',
      date: '2026-05-01',
    })
  }

  return <Button onClick={handleCreate}>Create Booking</Button>
}
```

---

## Result Handling

Effectful atoms (created via `Atom.make(Effect.gen(...))` or `runtimeAtom.atom(...)`) return `Result<A, E>` with three states:

```ts
import { Result } from '@effect-atom/atom'

// Result states:
// - Initial    (._tag === 'Initial')  — atom hasn't run yet
// - Success    (._tag === 'Success')  — has .value
// - Failure    (._tag === 'Failure')  — has .cause

// All states have:
// - .waiting: boolean — true while a re-fetch is in flight

// Reading values
const value = Result.value(result)        // Option<A> — Some if Success
const isLoading = result.waiting          // boolean

// In components — pattern matching
function BookingList() {
  const result = useAtomValue(Booking.list)

  if (result._tag === 'Initial' || result.waiting) return <Spinner />
  if (result._tag === 'Failure') return <ErrorDisplay cause={result.cause} />
  return <List items={result.value} />
}

// Or using the builder pattern
function BookingList() {
  const result = useAtomValue(Booking.list)

  return Result.builder(result)
    .onInitial(() => <Spinner />)
    .onFailure((cause) => <ErrorDisplay cause={cause} />)
    .onSuccess((bookings, { waiting }) => (
      <div>
        {waiting && <RefreshIndicator />}
        <List items={bookings} />
      </div>
    ))
    .render()
}
```

---

## React Hooks

All hooks from `@effect-atom/atom-react`:

```tsx
import {
  useAtomValue,    // Read atom value (subscribes to updates)
  useAtomSet,      // Get setter function
  useAtom,         // Read + write: [value, setter]
  useAtomSuspense, // Throws Promise while loading (React Suspense)
  useAtomRefresh,  // Returns () => void to force re-fetch
  useAtomMount,    // Mount atom without reading (keeps alive)
  useAtomSubscribe // Subscribe to changes imperatively
} from '@effect-atom/atom-react'

// useAtomValue — read-only subscription
const bookings = useAtomValue(Booking.list)
const step = useAtomValue(WizardFlow.currentStep)

// useAtomValue with inline selector
const count = useAtomValue(Booking.list, (result) =>
  Result.value(result).pipe(Option.map(arr => arr.length), Option.getOrElse(() => 0))
)

// useAtomSet — write-only (also mounts the atom)
const setStep = useAtomSet(WizardFlow.currentStep)
const createBooking = useAtomSet(Booking.create)

// useAtom — read + write together
const [filters, setFilters] = useAtom(Booking.filters)
const [generateResult, setGenerate] = useAtom(RouteState.generate)

// useAtomRefresh — manual re-fetch
const refresh = useAtomRefresh(Booking.list)
// Later: refresh()

// useAtomSuspense — for React Suspense boundaries
function BookingListSuspense() {
  const result = useAtomSuspense(Booking.list)
  // result is guaranteed to be Success here
  return <List items={result.value} />
}
```

---

## Namespace Export Convention

All atoms for a feature are exported as a single namespace object. This keeps imports clean and makes the feature's state API discoverable.

```ts
// features/booking/state/booking.atoms.ts

const list = apiRuntime.atom(/* ... */)
const filters = Atom.writable<BookingFilters, Partial<BookingFilters>>(/* ... */)
const create = apiRuntime.fn(/* ... */)
const update = apiRuntime.fn(/* ... */)
const remove = apiRuntime.fn(/* ... */)

export const Booking = {
  list,
  filters,
  create,
  update,
  remove,
}
```

**Usage in hooks/components:**

```ts
import { Booking } from '../state/booking.atoms'

const result = useAtomValue(Booking.list)
const createBooking = useAtomSet(Booking.create)
const [filters, setFilters] = useAtom(Booking.filters)
```

---

## When to Use React State vs Atoms

| Concern | Use | Example |
|---|---|---|
| Server/domain data | Atoms (`apiRuntime.atom`) | Booking list, driver details |
| Derived computations | Atoms (`Atom.readable`) | Filtered bookings, step progress |
| Cross-component shared state | Atoms (`Atom.make`) | Current step, selected ID |
| Persistent user preferences | Atoms (`Atom.kvs`) | Sidebar state, view mode |
| Ephemeral UI state | React `useState` | Modal open/closed, input focus, hover |
| Form field being typed | React `useState` | Input value before submit |
| Animation/transition state | React `useState` | Accordion expanded |

**Why atoms over useState?** Atoms are framework-agnostic, testable without React, compose with Effect services, and support derived state without `useEffect` → `setState` chains.

---

# Advanced Patterns

> The patterns below are used in specific scenarios. Start with the essentials above — you can come back here when you need them.

---

## Writable Atoms

Custom read + write logic. The reader function receives `get`, the writer receives `(ctx, value)`.

```ts
import { Atom } from '@effect-atom/atom'

// Atom with custom write logic
const filters = Atom.writable<BookingFilters, Partial<BookingFilters>>(
  // Reader: return current value
  (get) => {
    const self = get.self<BookingFilters>()
    if (Option.isSome(self)) return self.value
    return defaultFilters
  },
  // Writer: merge partial updates into current value
  (ctx, partial) => {
    const current = ctx.self<BookingFilters>()
    ctx.setSelf({
      ...(Option.isSome(current) ? current.value : defaultFilters),
      ...partial,
    })
  }
)
```

---

## Registry Access in Mutations

Inside `runtimeAtom.fn`, `yield* Registry.AtomRegistry` gives you imperative read/write access to any atom. This is how mutations update state optimistically.

```ts
import { Atom, Registry } from '@effect-atom/atom'
import { Data, Effect, Option, Schedule, pipe } from 'effect'

// State atom
type DraftState = Data.TaggedEnum<{
  Local: { readonly data: DraftData }
  Persisted: { readonly id: string; readonly data: DraftData }
}>
const DraftState = Data.taggedEnum<DraftState>()

const state = Atom.writable<DraftState, DraftState>(
  (get) => DraftState.Local({ data: emptyDraft }),
  (ctx, value) => ctx.setSelf(value)
)

// Mutation that reads and writes atoms via registry
const update = apiRuntime.fn(
  Effect.fnUntraced(function* (partial: Partial<DraftData>) {
    const registry = yield* Registry.AtomRegistry

    // Read current state
    const current = registry.get(state)
    const newData = { ...current.data, ...partial }

    // Write updated state immediately (optimistic)
    const newState = DraftState.$match(current, {
      Local: () => DraftState.Local({ data: newData }),
      Persisted: (p) => DraftState.Persisted({ ...p, data: newData }),
    })
    registry.set(state, newState)

    // Persist to server if saved
    if (current._tag === 'Persisted') {
      const api = yield* BookingApi
      yield* pipe(
        api.update(current.id, newData),
        Effect.retry(Schedule.once.pipe(Schedule.addDelay(() => '1 second')))
      )
    }
  })
)
```

---

## Cross-Atom Dependency API

The `get` context inside atom definitions has a rich API for reading and subscribing to other atoms:

| Method | Purpose | Tracking |
|---|---|---|
| `get(atom)` | Read value — re-runs this atom when it changes | **Tracked** |
| `get.once(atom)` | Read value once — no re-computation on change | Untracked |
| `get.self<T>()` | Previous value of _this_ atom (returns `Option`) | — |
| `get.setSelf(value)` | Imperatively write to _this_ atom | — |
| `get.set(atom, value)` | Imperatively write to another atom | — |
| `get.subscribe(atom, cb)` | React to changes without re-rendering | — |
| `get.refresh(atom)` | Force re-computation of another atom | — |
| `get.addFinalizer(fn)` | Run cleanup when atom is rebuilt or disposed | — |

---

## Self-Initializing Pattern

A common pattern: an atom initializes from a "source" atom once (untracked), then becomes independently editable. Uses `get.self()` to check if it already has a value.

```ts
import { Atom, Result } from '@effect-atom/atom'
import { Option, pipe } from 'effect'

// Draft.data is the source atom with server data
// This atom initializes from it once, then is editable independently

const pickupAddress = Atom.writable<string, string>(
  (get) => {
    // If we already have a local value, keep it
    const self = get.self<string>()
    if (Option.isSome(self)) return self.value

    // Otherwise, initialize from the source (untracked — no re-computation)
    return pipe(
      get.once(Draft.data).pickupAddress,
      Option.getOrElse(() => '')
    )
  },
  (ctx, v) => ctx.setSelf(v)
)
```

**With reactive sync from mutations:**

```ts
const pickupAddress = Atom.writable<string, string>(
  (get) => {
    // Subscribe to AI edit results — when edit completes, update this atom
    get.subscribe(editAddress, (result) => {
      const edited = Option.getOrNull(Result.value(result))
      if (edited) get.setSelf(edited)
    })

    // Self-initializing from source
    const self = get.self<string>()
    if (Option.isSome(self)) return self.value
    return pipe(
      get.once(Draft.data).pickupAddress,
      Option.getOrElse(() => '')
    )
  },
  (ctx, v) => ctx.setSelf(v)
)
```

---

## Reactive Sync Atoms

"Invisible" atoms that bridge mutation results into state atoms **without React renders**. They use `get.subscribe()` to imperatively update other atoms when a mutation emits results.

Mount them in a component with `useAtomValue` to activate the subscription.

```ts
// Bridges: when `generate` stream emits → update `routes` atom
const routeGenerateSync = Atom.readable((get) => {
  get.subscribe(generate, (result) => {
    const routes = Option.getOrNull(Result.value(result))
    if (!routes) return

    get.set(
      routeList,
      routes.map((r, i) => ({
        id: r.id ?? `__partial_${i}`,
        origin: r.origin ?? '',
        destination: r.destination ?? '',
        isGenerating: !r.id,
      }))
    )
  })
})

// Bridges: when `editRoute` stream emits → update the specific route
const routeEditSync = Atom.readable((get) => {
  get.subscribe(editRoute, (result) => {
    const partial = Option.getOrNull(Result.value(result))
    const targetId = get.once(targetRouteId)
    if (!partial || !targetId) return

    get.set(
      routeList,
      get.once(routeList).map((r) =>
        r.id === targetId
          ? { ...r, origin: partial.origin ?? '', destination: partial.destination ?? '' }
          : r
      )
    )
  })
})
```

**Mounting in components:**

```tsx
function RouteEditor() {
  // Mount sync atoms — activates the subscriptions
  useAtomValue(RouteState.routeGenerateSync)
  useAtomValue(RouteState.routeEditSync)

  // These atoms are now kept in sync with mutation results
  const routes = useAtomValue(RouteState.routeList)
  return <RouteList routes={routes} />
}
```

---

## Persistent Atoms (Atom.kvs)

`Atom.kvs` creates atoms backed by localStorage with automatic Schema validation on hydration.

```ts
import { Atom } from '@effect-atom/atom'
import { BrowserKeyValueStore } from '@effect/platform-browser'
import { Schema } from 'effect'

const kvsRuntime = Atom.runtime(BrowserKeyValueStore.layerLocalStorage)

// Boolean preference — persisted to localStorage
const sidebarHidden = Atom.kvs({
  runtime: kvsRuntime,
  key: 'tms-sidebar-hidden',
  schema: Schema.Boolean,
  defaultValue: () => false,
})

// Complex object — Schema validates on hydration, falls back on invalid data
const savedFilters = Atom.kvs({
  runtime: kvsRuntime,
  key: 'tms-booking-filters',
  schema: Schema.Struct({
    status: Schema.optional(Schema.String),
    dateRange: Schema.optional(Schema.String),
    sortBy: Schema.optional(Schema.String),
  }),
  defaultValue: () => ({}),
})

// Optional value
const localDraft = Atom.kvs({
  runtime: kvsRuntime,
  key: 'tms-route-draft',
  schema: Schema.OptionFromNullOr(RouteDraftSchema),
  defaultValue: () => Option.none(),
})
```

---

## ScopedAtom

`ScopedAtom.make` creates atoms that are **scoped to a React subtree**. Each `Provider` gets its own atom instance. Use this for feature-level state that shouldn't be global.

```ts
// features/booking/state/booking.scoped.ts
import { Atom, ScopedAtom } from '@effect-atom/atom-react'
import { Effect } from 'effect'
import { apiRuntime } from '~/lib/runtime'
import { BookingApi } from '../api/booking.api'

// Scoped atom without input — same instance for all children
const BookingListScope = ScopedAtom.make(() =>
  apiRuntime.atom(
    Effect.gen(function* () {
      const api = yield* BookingApi
      return yield* api.list({})
    })
  )
)

// Scoped atom WITH input — atom factory parameterized by value prop
const BookingDetailScope = ScopedAtom.make((bookingId: string) =>
  apiRuntime.atom(
    Effect.gen(function* () {
      const api = yield* BookingApi
      return yield* api.getById(bookingId)
    })
  )
)
```

**Usage:**

```tsx
// Route component provides the scope
function BookingsPage() {
  return (
    <BookingListScope.Provider>
      <BookingListContent />
    </BookingListScope.Provider>
  )
}

function BookingDetailPage({ bookingId }: { bookingId: string }) {
  return (
    <BookingDetailScope.Provider value={bookingId}>
      <BookingDetailContent />
    </BookingDetailScope.Provider>
  )
}

// Child components consume the scoped atom
function BookingListContent() {
  const atom = BookingListScope.use()      // gets atom from context
  const result = useAtomValue(atom)        // subscribe to its value
  // ...
}

function BookingDetailContent() {
  const atom = BookingDetailScope.use()
  const result = useAtomValue(atom)
  // ...
}
```

---

## Atom.family

Stable per-key atom factory. Returns the same atom instance for the same key.

```ts
import { Atom } from '@effect-atom/atom'

// Create one atom per booking ID — stable reference across renders
const bookingExpanded = Atom.family((bookingId: string) =>
  Atom.make(false)
)

// Usage in component
function BookingRow({ id }: { id: string }) {
  const [expanded, setExpanded] = useAtom(bookingExpanded(id))
  return (
    <div>
      <button onClick={() => setExpanded(!expanded)}>Toggle</button>
      {expanded && <BookingDetails id={id} />}
    </div>
  )
}
```
