# 05 — Feature Anatomy: `features/booking/`

> A walkthrough of a complete feature — how data flows from schema definitions through API, atoms, hooks, and components.

## Overview

The `booking` feature demonstrates the full stack. Each layer imports only from the layer below it:

```
contract/schemas.ts        ← Shape of data (Effect Schema)
contract/errors.ts         ← Domain error types
booking.api.ts             ← HTTP calls (Effect service)
booking.atoms.ts           ← Reactive state (atoms + mutations)
booking.hooks.ts           ← React bridge
components/booking-list.tsx ← UI (reads via hooks)
components/booking-form.tsx ← UI (writes via hooks)
index.ts                   ← Public barrel export
```

> For the full patterns referenced below, see:
> [02-effect-services.md](./02-effect-services.md) (services, layers, errors) |
> [03-http-client.md](./03-http-client.md) (HTTP client, schema validation) |
> [04-state-management.md](./04-state-management.md) (atoms, hooks, Result handling)

---

## Layer 1: Contract — `contract/schemas.ts`

Schemas are the single source of truth. TypeScript types are derived from them — never hand-written separately.

```ts
import { Schema } from 'effect'

export const BookingId = Schema.String.pipe(Schema.brand('BookingId'))
export type BookingId = Schema.Schema.Type<typeof BookingId>

export const Booking = Schema.Struct({
  id: BookingId,
  driverId: Schema.String,
  vehicleId: Schema.String,
  pickupAddress: Schema.String,
  dropoffAddress: Schema.String,
  pickupDate: Schema.DateFromString,   // ISO string → Date on decode
  status: Schema.Literal('pending', 'confirmed', 'in-progress', 'completed', 'cancelled'),
  notes: Schema.optional(Schema.String),
  createdAt: Schema.DateFromString,
  updatedAt: Schema.DateFromString,
})
export type Booking = Schema.Schema.Type<typeof Booking>

export const CreateBookingInput = Schema.Struct({
  driverId: Schema.String,
  vehicleId: Schema.String,
  pickupAddress: Schema.String,
  dropoffAddress: Schema.String,
  pickupDate: Schema.String,           // sent as ISO string to the API
  notes: Schema.optional(Schema.String),
})
export type CreateBookingInput = Schema.Schema.Type<typeof CreateBookingInput>
```

**Key points:**
- `Schema.DateFromString` auto-converts ISO strings to `Date` during response decoding — no manual `new Date()` calls.
- `Schema.brand('BookingId')` prevents accidentally passing a plain string where a `BookingId` is expected.
- `Schema.partial(...)` produces a partial version (for PATCH endpoints).

---

## Layer 1: Contract — `contract/errors.ts`

Domain errors use `Schema.TaggedError` for type-safe error handling (see [02-effect-services.md](./02-effect-services.md#5-tagged-errors)):

```ts
import { Schema } from 'effect'

export class BookingNotFound extends Schema.TaggedError<BookingNotFound>()(
  'BookingNotFound',
  { bookingId: Schema.String },
) {}

export class BookingConflict extends Schema.TaggedError<BookingConflict>()(
  'BookingConflict',
  { bookingId: Schema.String },
) {}
```

Callers handle these with `Effect.catchTag('BookingNotFound', (e) => ...)`.

---

## Layer 2: API Service — `booking.api.ts`

A `Context.Tag` + `Layer.effect` that uses `ApiHttpClient` for typed HTTP calls (see [02-effect-services.md](./02-effect-services.md) for the service pattern, [03-http-client.md](./03-http-client.md) for HTTP specifics):

```ts
// Interface
export class BookingApi extends Context.Tag('BookingApi')<
  BookingApi, BookingApiShape
>() {}

// Implementation (condensed — showing the key patterns)
export const BookingApiLive = Layer.effect(
  BookingApi,
  Effect.gen(function* () {
    const client = yield* ApiHttpClient

    return {
      list: (filters) =>
        client.get(`/bookings?${toParams(filters)}`).pipe(
          Effect.flatMap(HttpClientResponse.schemaBodyJson(BookingListResponse)),
          Effect.mapError(mapHttpError('GET', '/bookings')),
        ),

      create: (input) =>
        HttpClientRequest.post('/bookings').pipe(
          HttpClientRequest.schemaBodyJson(CreateBookingInput)(input),
          Effect.flatMap(client.execute),
          Effect.flatMap(HttpClientResponse.schemaBodyJson(Booking)),
          Effect.mapError(mapHttpError('POST', '/bookings')),
        ),

      // getById, update, delete follow the same patterns
    }
  }),
)
```

**Key points:**
- `schemaBodyJson` on the request side validates and encodes the body before sending — invalid data never hits the network.
- `schemaBodyJson` on the response side decodes and validates the response — `DateFromString` fields become `Date` objects automatically.
- `mapError` converts platform `ResponseError` to domain-specific tagged errors (e.g., 404 → `BookingNotFound`).

---

## Layer 3: Atoms — `booking.atoms.ts`

Atoms bridge Effect services into reactive state (see [04-state-management.md](./04-state-management.md)):

```ts
import { Atom, Registry, Result } from '@effect-atom/atom'
import { Effect, Option } from 'effect'
import { apiRuntime } from '~/lib/runtime'
import { BookingApi } from '../booking.api'

// Fetches paginated bookings — re-fetches when filters change
const list = apiRuntime.atom(
  Effect.gen(function* () {
    const api = yield* BookingApi
    const filtersValue = yield* Atom.get(filters)
    return yield* api.list(filtersValue)
  }),
)

// Writable atom with merge-on-write — resets to page 1 on filter changes
const filters = Atom.writable<BookingFilters, Partial<BookingFilters>>(
  (get) => {
    const self = get.self<BookingFilters>()
    if (Option.isSome(self)) return self.value
    return defaultBookingFilters
  },
  (ctx, partial) => {
    const current = ctx.self<BookingFilters>()
    ctx.setSelf({
      ...(Option.isSome(current) ? current.value : defaultBookingFilters),
      ...partial,
      page: partial.page ?? 1,
    })
  },
)

// Mutation: POST then refresh the list
const create = apiRuntime.fn(
  Effect.fnUntraced(function* (input: CreateBookingInput) {
    const api = yield* BookingApi
    const booking = yield* api.create(input)
    const registry = yield* Registry.AtomRegistry
    registry.refresh(list)
    return booking
  }),
)

// Namespace export — one object per feature
export const Booking = { list, filters, create, /* update, remove */ }
```

**Key points:**
- `apiRuntime.atom(Effect)` creates an atom that can `yield*` Effect services. It returns `Result<A, E>` with three states: Initial, Success, Failure.
- `apiRuntime.fn(Effect.fnUntraced(...))` creates a mutation atom — call it via `useAtomSet` in components.
- `registry.refresh(list)` triggers a re-fetch after mutations.

---

## Layer 4: Hooks — `booking.hooks.ts`

Thin wrappers that bridge atoms to React. No logic here that should live in atoms.

```ts
import { useAtom, useAtomRefresh, useAtomSet, useAtomValue } from '@effect-atom/atom-react'
import { Booking } from '../booking.atoms'

export function useBookings() {
  const result = useAtomValue(Booking.list)
  const refresh = useAtomRefresh(Booking.list)
  const [filters, setFilters] = useAtom(Booking.filters)
  return { result, refresh, filters, setFilters }
}

export function useCreateBooking() {
  const [result, create] = useAtom(Booking.create)
  return { result, create }
}
```

---

## Layer 5: Components

### Reading data — `booking-list.tsx`

Components read through hooks and pattern-match on `Result`:

```tsx
import { useBookings, useRemoveBooking } from '../booking.hooks'

export function BookingList() {
  const { result, refresh, filters, setFilters } = useBookings()

  if (result._tag === 'Initial' || (result._tag === 'Success' && result.waiting)) {
    return <BookingListSkeleton />
  }

  if (result._tag === 'Failure') {
    return <ErrorWithRetry onRetry={refresh} />
  }

  const { items, total, page, pageSize } = result.value
  return (
    <>
      {result.waiting && <RefreshIndicator />}
      <Table>
        {items.map((booking) => <BookingRow key={booking.id} booking={booking} />)}
      </Table>
      <Pagination page={page} pageSize={pageSize} total={total} onChange={setFilters} />
    </>
  )
}
```

### Writing data — `booking-form.tsx`

Forms use React `useState` for ephemeral field values. The atom mutation fires only on submit:

```tsx
import { useCreateBooking } from '../booking.hooks'

export function BookingForm({ onSuccess }: { onSuccess?: () => void }) {
  const { create, result } = useCreateBooking()
  const [pickupAddress, setPickupAddress] = useState('')
  // ... other field state

  const isSubmitting = result._tag !== 'Initial' && result.waiting

  function handleSubmit(e: FormEvent) {
    e.preventDefault()
    create({ pickupAddress, /* ... */ })
  }

  if (result._tag === 'Success' && !result.waiting && onSuccess) onSuccess()

  return <form onSubmit={handleSubmit}>{/* fields */}</form>
}
```

**Key points:**
- `useState` for form fields is intentional — ephemeral values that don't need to survive re-mounts. See [04-state-management.md](./04-state-management.md#when-to-use-react-state-vs-atoms).
- `result.waiting` drives the loading state — no separate `isLoading` boolean.

---

## Layer 6: Barrel Export — `index.ts`

The feature's public API. Other features and routes import from here only:

```ts
export { BookingList } from './components/booking-list'
export { BookingForm } from './components/booking-form'
export type { Booking, BookingId, BookingStatus } from './contract/schemas'
```

**Not exported:** `BookingApi` / `BookingApiLive` (wired in `runtime.ts`), atoms (internal), hooks (internal). See [01-folder-structure.md](./01-folder-structure.md#import-rules).

---

## Data Flow Summary

```
User submits form
       │
       ▼
BookingForm.handleSubmit
       │
       ▼
useCreateBooking().create(input)        ← hook calls atom
       │
       ▼
Booking.create atom (apiRuntime.fn)     ← Effect runs in runtime context
       │
       ├─ yield* BookingApi             ← DI resolves BookingApiLive
       │
       ├─ api.create(input)             ← POST /bookings via ApiHttpClient
       │    │
       │    ├─ schemaBodyJson encodes + validates request body
       │    ├─ ApiHttpClient prepends base URL + auth header
       │    └─ schemaBodyJson decodes + validates response
       │
       ├─ registry.refresh(list)        ← triggers list re-fetch
       │
       └─ returns Booking               ← Result<Booking, BookingConflict>
              │
              ▼
       BookingForm reads result._tag === 'Success'
              │
              ▼
       onSuccess() → modal closes / navigation
              │
              ▼
       BookingList re-renders with refreshed data
```

---

## See Also

- [01-folder-structure.md](./01-folder-structure.md) — directory layout and import rules
- [02-effect-services.md](./02-effect-services.md) — `Context.Tag`, `Layer.effect`, tagged errors
- [03-http-client.md](./03-http-client.md) — `ApiHttpClient`, `schemaBodyJson`, retries
- [04-state-management.md](./04-state-management.md) — atom patterns, Result handling, React hooks
