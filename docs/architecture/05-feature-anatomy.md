# 05 — Feature Anatomy: `features/booking/`

> A complete walkthrough of every file in a single feature, from schema definitions through API service, atoms, hooks, and components.

## Overview

The `booking` feature is the canonical example of the full feature stack. Reading it top-to-bottom shows exactly how data flows from the backend contract definition through to pixels on screen:

```
contract/schemas.ts        ← Shape of data (Effect Schema)
contract/errors.ts         ← Domain error types
api/booking.api.ts         ← HTTP calls (Effect service)
state/booking.atoms.ts     ← Reactive state (atoms + mutations)
hooks/use-bookings.ts      ← React bridge (useAtomValue etc.)
components/booking-list.tsx ← UI (reads via hooks)
components/booking-form.tsx ← UI (writes via hooks)
index.ts                   ← Public barrel export
```

Each layer only imports from the layer below it. Components never touch atoms directly — always through hooks. Hooks never call `fetch` directly — always through atoms. Atoms never import React.

---

## File: `features/booking/contract/schemas.ts`

All Effect Schema definitions for the booking domain. These are the single source of truth for both request and response shapes. TypeScript types are derived from schemas — never hand-written separately.

```ts
// src/features/booking/contract/schemas.ts
import { Schema } from 'effect'

// ---------------------------------------------------------------------------
// Branded ID
// ---------------------------------------------------------------------------

export const BookingId = Schema.String.pipe(Schema.brand('BookingId'))
export type BookingId = Schema.Schema.Type<typeof BookingId>

// ---------------------------------------------------------------------------
// Status
// ---------------------------------------------------------------------------

export const BookingStatus = Schema.Literal(
  'pending',
  'confirmed',
  'in-progress',
  'completed',
  'cancelled',
)
export type BookingStatus = Schema.Schema.Type<typeof BookingStatus>

// ---------------------------------------------------------------------------
// Core entity
// ---------------------------------------------------------------------------

export const Booking = Schema.Struct({
  id: BookingId,
  driverId: Schema.String,
  vehicleId: Schema.String,
  pickupAddress: Schema.String,
  dropoffAddress: Schema.String,
  pickupDate: Schema.DateFromString,   // ISO 8601 string → Date on decode
  status: BookingStatus,
  notes: Schema.optional(Schema.String),
  createdAt: Schema.DateFromString,
  updatedAt: Schema.DateFromString,
})
export type Booking = Schema.Schema.Type<typeof Booking>

// ---------------------------------------------------------------------------
// Create input — subset of fields supplied by the caller
// ---------------------------------------------------------------------------

export const CreateBookingInput = Schema.Struct({
  driverId: Schema.String,
  vehicleId: Schema.String,
  pickupAddress: Schema.String,
  dropoffAddress: Schema.String,
  pickupDate: Schema.String,           // sent as ISO string to the API
  notes: Schema.optional(Schema.String),
})
export type CreateBookingInput = Schema.Schema.Type<typeof CreateBookingInput>

// ---------------------------------------------------------------------------
// Update input — all fields optional (PATCH semantics)
// ---------------------------------------------------------------------------

export const UpdateBookingInput = Schema.partial(
  Schema.Struct({
    driverId: Schema.String,
    vehicleId: Schema.String,
    pickupAddress: Schema.String,
    dropoffAddress: Schema.String,
    pickupDate: Schema.String,
    status: BookingStatus,
    notes: Schema.String,
  }),
)
export type UpdateBookingInput = Schema.Schema.Type<typeof UpdateBookingInput>

// ---------------------------------------------------------------------------
// Query filters
// ---------------------------------------------------------------------------

export const BookingFilters = Schema.Struct({
  status: Schema.optional(BookingStatus),
  search: Schema.optional(Schema.String),
  dateFrom: Schema.optional(Schema.String),
  dateTo: Schema.optional(Schema.String),
  page: Schema.Number,
  pageSize: Schema.Number,
})
export type BookingFilters = Schema.Schema.Type<typeof BookingFilters>

export const defaultBookingFilters: BookingFilters = {
  page: 1,
  pageSize: 20,
}

// ---------------------------------------------------------------------------
// Paginated list response
// ---------------------------------------------------------------------------

export const BookingListResponse = Schema.Struct({
  items: Schema.Array(Booking),
  total: Schema.Number,
  page: Schema.Number,
  pageSize: Schema.Number,
})
export type BookingListResponse = Schema.Schema.Type<typeof BookingListResponse>
```

**Key points:**

- `BookingId` is a branded string — it is structurally `string` but TypeScript won't let you pass a plain `string` where a `BookingId` is expected without an explicit cast.
- `Schema.DateFromString` handles the ISO 8601 string → `Date` conversion automatically during `schemaBodyJson` decoding. No manual `new Date(...)` calls anywhere.
- `CreateBookingInput` sends `pickupDate` as a `string` — the encoding direction does not need `DateFromString` because we are constructing the value, not parsing it.
- `UpdateBookingInput` is produced with `Schema.partial` — identical to `Partial<T>` in TypeScript but understood by the schema runtime.

---

## File: `features/booking/contract/errors.ts`

Domain errors for the booking feature. Using `Schema.TaggedError` means each error has a `_tag` discriminant for `Effect.catchTag`, plus full schema encoding if errors are ever serialised.

```ts
// src/features/booking/contract/errors.ts
import { Schema } from 'effect'

export class BookingNotFound extends Schema.TaggedError<BookingNotFound>()(
  'BookingNotFound',
  {
    bookingId: Schema.String,
  },
) {}

export class BookingConflict extends Schema.TaggedError<BookingConflict>()(
  'BookingConflict',
  {
    bookingId: Schema.String,
  },
) {}
```

**Usage example:**

```ts
import { Effect } from 'effect'
import { BookingApi } from '../api/booking.api'
import { BookingNotFound } from '../contract/errors'

const program = Effect.gen(function* () {
  const api = yield* BookingApi
  return yield* api.getById('b-123').pipe(
    Effect.catchTag('BookingNotFound', (e) =>
      // handle missing booking — show fallback, log, etc.
      Effect.succeed(null),
    ),
  )
})
```

---

## File: `features/booking/api/booking.api.ts`

The HTTP service for the booking domain. This is a `Context.Tag` (interface) plus a `Layer.effect` (implementation) that yields `ApiHttpClient` from context. No direct `fetch` calls — everything goes through the platform `HttpClient`.

```ts
// src/features/booking/api/booking.api.ts
import { HttpClientRequest, HttpClientResponse } from '@effect/platform'
import { Context, Effect, Layer } from 'effect'
import { ApiHttpClient } from '~/lib/http-client'
import {
  Booking,
  BookingFilters,
  BookingListResponse,
  CreateBookingInput,
  UpdateBookingInput,
} from '../contract/schemas'
import { BookingNotFound, BookingConflict } from '../contract/errors'

// ---------------------------------------------------------------------------
// Service interface
// ---------------------------------------------------------------------------

interface BookingApiShape {
  readonly list: (
    filters: BookingFilters,
  ) => Effect.Effect<BookingListResponse, BookingNotFound>

  readonly getById: (
    id: string,
  ) => Effect.Effect<Booking, BookingNotFound>

  readonly create: (
    input: CreateBookingInput,
  ) => Effect.Effect<Booking, BookingConflict>

  readonly update: (
    id: string,
    input: UpdateBookingInput,
  ) => Effect.Effect<Booking, BookingNotFound>

  readonly delete: (id: string) => Effect.Effect<void, BookingNotFound>
}

// ---------------------------------------------------------------------------
// Tag
// ---------------------------------------------------------------------------

export class BookingApi extends Context.Tag('BookingApi')<
  BookingApi,
  BookingApiShape
>() {}

// ---------------------------------------------------------------------------
// Live implementation
// ---------------------------------------------------------------------------

export const BookingApiLive = Layer.effect(
  BookingApi,
  Effect.gen(function* () {
    const client = yield* ApiHttpClient

    return {
      // GET /bookings?page=1&pageSize=20&...
      list: (filters) => {
        const params = new URLSearchParams()
        params.set('page', String(filters.page))
        params.set('pageSize', String(filters.pageSize))
        if (filters.status !== undefined) params.set('status', filters.status)
        if (filters.search !== undefined) params.set('search', filters.search)
        if (filters.dateFrom !== undefined) params.set('dateFrom', filters.dateFrom)
        if (filters.dateTo !== undefined) params.set('dateTo', filters.dateTo)

        return client.get(`/bookings?${params.toString()}`).pipe(
          Effect.flatMap(HttpClientResponse.schemaBodyJson(BookingListResponse)),
          Effect.mapError(() => new BookingNotFound({ bookingId: 'all' })),
        )
      },

      // GET /bookings/:id
      getById: (id) =>
        client.get(`/bookings/${id}`).pipe(
          Effect.flatMap(HttpClientResponse.schemaBodyJson(Booking)),
          Effect.mapError(() => new BookingNotFound({ bookingId: id })),
        ),

      // POST /bookings
      create: (input) =>
        HttpClientRequest.post('/bookings').pipe(
          HttpClientRequest.schemaBodyJson(CreateBookingInput)(input),
          Effect.flatMap(client.execute),
          Effect.flatMap(HttpClientResponse.schemaBodyJson(Booking)),
          Effect.mapError(() => new BookingConflict({ bookingId: 'new' })),
        ),

      // PATCH /bookings/:id
      update: (id, input) =>
        HttpClientRequest.patch(`/bookings/${id}`).pipe(
          HttpClientRequest.schemaBodyJson(UpdateBookingInput)(input),
          Effect.flatMap(client.execute),
          Effect.flatMap(HttpClientResponse.schemaBodyJson(Booking)),
          Effect.mapError(() => new BookingNotFound({ bookingId: id })),
        ),

      // DELETE /bookings/:id
      delete: (id) =>
        client.del(`/bookings/${id}`).pipe(
          Effect.asVoid,
          Effect.mapError(() => new BookingNotFound({ bookingId: id })),
        ),
    }
  }),
)
```

**Key points:**

- `HttpClientResponse.schemaBodyJson(Schema)` reads the response body as JSON and validates it. If the body is malformed or the shape is wrong, the effect fails.
- `HttpClientRequest.schemaBodyJson(Schema)(value)` encodes and attaches a request body. The two-argument curried form is intentional — the first call produces a function, the second applies it to the value.
- `Effect.mapError` converts every network/parse error to a typed domain error. Callers pattern-match on those domain errors, never on `ResponseError` directly.
- The tag (`BookingApi`) and live layer (`BookingApiLive`) are both exported. Only `src/lib/runtime.ts` imports `BookingApiLive`.

---

## File: `features/booking/state/booking.atoms.ts`

All reactive state for the booking feature. One atom per concern, exported as a single namespace object. The atoms use the `apiRuntime` (created in `~/lib/runtime`) which makes Effect services available inside atom computations.

```ts
// src/features/booking/state/booking.atoms.ts
import { Atom, Registry, Result } from '@effect-atom/atom'
import { Effect, Option } from 'effect'
import { apiRuntime } from '~/lib/runtime'
import { BookingApi } from '../api/booking.api'
import { BookingFilters, defaultBookingFilters } from '../contract/schemas'
import type { BookingListResponse } from '../contract/schemas'

// ---------------------------------------------------------------------------
// list — fetches paginated bookings from the API
// Returns Result<BookingListResponse, BookingNotFound>
// ---------------------------------------------------------------------------

const list = apiRuntime.atom(
  Effect.gen(function* () {
    const api = yield* BookingApi
    const filtersValue = yield* Atom.get(filters)
    return yield* api.list(filtersValue)
  }),
)

// ---------------------------------------------------------------------------
// filters — writable atom with self-initializing pattern
// Reader accepts full BookingFilters, writer accepts Partial<BookingFilters>
// ---------------------------------------------------------------------------

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
      // Reset to page 1 whenever any filter (other than page itself) changes
      page: partial.page ?? 1,
    })
  },
)

// ---------------------------------------------------------------------------
// create — mutation: POST /bookings, then refresh list
// ---------------------------------------------------------------------------

const create = apiRuntime.fn(
  Effect.fnUntraced(function* (input: Parameters<BookingApi['Service']['create']>[0]) {
    const api = yield* BookingApi
    const booking = yield* api.create(input)

    const registry = yield* Registry.AtomRegistry
    registry.refresh(list)

    return booking
  }),
)

// ---------------------------------------------------------------------------
// update — mutation: PATCH /bookings/:id with optimistic update
// ---------------------------------------------------------------------------

const update = apiRuntime.fn(
  Effect.fnUntraced(function* ({
    id,
    input,
  }: {
    id: string
    input: Parameters<BookingApi['Service']['update']>[1]
  }) {
    const api = yield* BookingApi
    const registry = yield* Registry.AtomRegistry

    // Optimistic: patch the cached list immediately
    const currentResult = registry.get(list)
    const currentData = Option.getOrNull(Result.value(currentResult))

    if (currentData !== null) {
      const optimistic: BookingListResponse = {
        ...currentData,
        items: currentData.items.map((b) =>
          b.id === id ? { ...b, ...input } : b,
        ),
      }
      registry.set(list, Result.success(optimistic))
    }

    // Persist to server
    const updated = yield* api.update(id, input)

    // Reconcile with server truth
    const afterResult = registry.get(list)
    const afterData = Option.getOrNull(Result.value(afterResult))

    if (afterData !== null) {
      registry.set(
        list,
        Result.success({
          ...afterData,
          items: afterData.items.map((b) => (b.id === id ? updated : b)),
        }),
      )
    }

    return updated
  }),
)

// ---------------------------------------------------------------------------
// remove — mutation: DELETE /bookings/:id, then refresh list
// ---------------------------------------------------------------------------

const remove = apiRuntime.fn(
  Effect.fnUntraced(function* (id: string) {
    const api = yield* BookingApi
    yield* api.delete(id)

    const registry = yield* Registry.AtomRegistry
    registry.refresh(list)
  }),
)

// ---------------------------------------------------------------------------
// listSync — reactive sync atom
// Bridges create/update/remove results into the list atom without extra
// renders. Mount via useAtomValue(Booking.listSync) in the root component.
// ---------------------------------------------------------------------------

const listSync = Atom.readable((get) => {
  // When create completes successfully, refresh the list
  get.subscribe(create, (result) => {
    const created = Option.getOrNull(Result.value(result))
    if (created) get.refresh(list)
  })

  // When update completes, the update fn already patches the cache inline.
  // Subscribe here only to trigger any derived atoms that depend on list.
  get.subscribe(update, (result) => {
    if (result._tag === 'Success') get.refresh(list)
  })

  // When remove completes, the remove fn already refreshed list.
  // This subscription is a safety net for derived consumers.
  get.subscribe(remove, (result) => {
    if (result._tag === 'Success') get.refresh(list)
  })
})

// ---------------------------------------------------------------------------
// Namespace export
// ---------------------------------------------------------------------------

export const Booking = {
  list,
  filters,
  create,
  update,
  remove,
  listSync,
}
```

**Key points:**

- `list` depends on `filters` via `Atom.get(filters)` inside the Effect — when `filters` changes, `list` automatically re-fetches.
- `filters` uses the self-initializing pattern: `get.self()` returns `Option.none()` on first read, so it falls back to `defaultBookingFilters`. After the first write, it returns `Option.some(currentValue)`.
- `update` does an optimistic write (`registry.set`) before the network call, then reconciles with the server response. If the server rejects the change, the atom will be refreshed on the next read.
- `listSync` is an "invisible" atom — it returns no value callers care about. Its purpose is to activate `get.subscribe` wiring. It must be mounted with `useAtomValue` in a component that lives for the feature's lifetime.

---

## File: `features/booking/hooks/use-bookings.ts`

React hooks that bridge atoms to components. Each hook is a thin wrapper — no logic lives here that should live in atoms, and no UI logic lives here that should live in components.

```ts
// src/features/booking/hooks/use-bookings.ts
import {
  useAtom,
  useAtomRefresh,
  useAtomSet,
  useAtomValue,
} from '@effect-atom/atom-react'
import { Result } from '@effect-atom/atom'
import { Cause, Option } from 'effect'
import { Booking } from '../state/booking.atoms'
import { BookingNotFound } from '../contract/errors'
import type { BookingFilters, CreateBookingInput, UpdateBookingInput } from '../contract/schemas'

// ---------------------------------------------------------------------------
// useBookings — main listing hook
// ---------------------------------------------------------------------------

export function useBookings() {
  // Mount the sync atom — activates the create/update/remove subscriptions
  useAtomValue(Booking.listSync)

  const result = useAtomValue(Booking.list)
  const refresh = useAtomRefresh(Booking.list)
  const [filters, setFilters] = useAtom(Booking.filters)

  return { result, refresh, filters, setFilters }
}

// ---------------------------------------------------------------------------
// useBooking — single booking by ID (derived from list cache)
// ---------------------------------------------------------------------------

export function useBooking(id: string) {
  const result = useAtomValue(Booking.list)

  if (result._tag !== 'Success') return { result }

  const booking = result.value.items.find((b) => b.id === id)
  const singleResult = booking
    ? Result.success(booking)
    : Result.failure(Cause.fail(new BookingNotFound({ bookingId: id })))

  return { result: singleResult }
}

// ---------------------------------------------------------------------------
// useCreateBooking — mutation hook
// ---------------------------------------------------------------------------

export function useCreateBooking() {
  const [result, create] = useAtom(Booking.create)
  return { result, create }
}

// ---------------------------------------------------------------------------
// useUpdateBooking — mutation hook
// ---------------------------------------------------------------------------

export function useUpdateBooking() {
  const [result, update] = useAtom(Booking.update)
  return { result, update }
}

// ---------------------------------------------------------------------------
// useRemoveBooking — mutation hook
// ---------------------------------------------------------------------------

export function useRemoveBooking() {
  const [result, remove] = useAtom(Booking.remove)
  return { result, remove }
}
```

**Key points:**

- `useAtomValue(Booking.listSync)` in `useBookings` mounts the sync atom. This call is side-effect only — the return value is ignored. It must be called in a component that is always mounted while the booking feature is in use.
- `useBooking(id)` derives from the list cache rather than making a separate atom per ID. For a detail page that needs data not in the list response, replace this with a dedicated `apiRuntime.atom` that calls `api.getById`.
- `useAtom` on a mutation atom returns `[result, trigger]` — `result` is the `Result` of the last invocation, `trigger` is the function to call.

---

## File: `features/booking/components/booking-list.tsx`

The main list component. It reads data through hooks and renders using shadcn/ui primitives. Result pattern matching follows the three-state `Initial / Failure / Success` pattern from [04-state-management.md](./04-state-management.md).

```tsx
// src/features/booking/components/booking-list.tsx
import { Result } from '@effect-atom/atom'
import { Badge } from '~/components/ui/badge'
import { Button } from '~/components/ui/button'
import { Skeleton } from '~/components/ui/skeleton'
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '~/components/ui/table'
import { cn } from '~/lib/cn'
import { useBookings, useRemoveBooking } from '../hooks/use-bookings'
import type { Booking, BookingStatus } from '../contract/schemas'

// ---------------------------------------------------------------------------
// Status badge colour map
// ---------------------------------------------------------------------------

const statusVariant: Record<BookingStatus, 'default' | 'secondary' | 'destructive' | 'outline'> =
  {
    pending: 'outline',
    confirmed: 'secondary',
    'in-progress': 'default',
    completed: 'secondary',
    cancelled: 'destructive',
  }

// ---------------------------------------------------------------------------
// BookingList
// ---------------------------------------------------------------------------

export function BookingList() {
  const { result, refresh, filters, setFilters } = useBookings()
  const { remove, result: removeResult } = useRemoveBooking()

  // ---- Loading / error states ----

  if (result._tag === 'Initial' || (result._tag === 'Success' && result.waiting)) {
    return <BookingListSkeleton />
  }

  if (result._tag === 'Failure') {
    return (
      <div className="flex flex-col items-center gap-4 py-12 text-center">
        <p className="text-sm text-muted-foreground">Failed to load bookings.</p>
        <Button variant="outline" size="sm" onClick={refresh}>
          Retry
        </Button>
      </div>
    )
  }

  // ---- Success ----

  const { items, total, page, pageSize } = result.value

  return (
    <div className="flex flex-col gap-4">
      {/* Refresh indicator while re-fetching */}
      {result.waiting && (
        <p className="text-xs text-muted-foreground animate-pulse">Refreshing…</p>
      )}

      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>Pickup</TableHead>
            <TableHead>Dropoff</TableHead>
            <TableHead>Date</TableHead>
            <TableHead>Status</TableHead>
            <TableHead className="text-right">Actions</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {items.length === 0 && (
            <TableRow>
              <TableCell colSpan={5} className="text-center text-muted-foreground py-8">
                No bookings found.
              </TableCell>
            </TableRow>
          )}
          {items.map((booking) => (
            <BookingRow
              key={booking.id}
              booking={booking}
              onRemove={() => remove(booking.id)}
              removing={
                removeResult._tag === 'Success' && removeResult.waiting
              }
            />
          ))}
        </TableBody>
      </Table>

      {/* Pagination */}
      <div className="flex items-center justify-between text-sm text-muted-foreground">
        <span>
          {(page - 1) * pageSize + 1}–{Math.min(page * pageSize, total)} of {total}
        </span>
        <div className="flex gap-2">
          <Button
            variant="outline"
            size="sm"
            disabled={page <= 1}
            onClick={() => setFilters({ page: page - 1 })}
          >
            Previous
          </Button>
          <Button
            variant="outline"
            size="sm"
            disabled={page * pageSize >= total}
            onClick={() => setFilters({ page: page + 1 })}
          >
            Next
          </Button>
        </div>
      </div>
    </div>
  )
}

// ---------------------------------------------------------------------------
// BookingRow
// ---------------------------------------------------------------------------

function BookingRow({
  booking,
  onRemove,
  removing,
}: {
  booking: Booking
  onRemove: () => void
  removing: boolean
}) {
  return (
    <TableRow>
      <TableCell className="font-medium">{booking.pickupAddress}</TableCell>
      <TableCell>{booking.dropoffAddress}</TableCell>
      <TableCell>
        {booking.pickupDate.toLocaleDateString('en-GB', {
          day: '2-digit',
          month: 'short',
          year: 'numeric',
        })}
      </TableCell>
      <TableCell>
        <Badge
          variant={statusVariant[booking.status]}
          className={cn(booking.status === 'in-progress' && 'animate-pulse')}
        >
          {booking.status}
        </Badge>
      </TableCell>
      <TableCell className="text-right">
        <Button
          variant="ghost"
          size="sm"
          disabled={removing}
          onClick={onRemove}
          className="text-destructive hover:text-destructive"
        >
          Remove
        </Button>
      </TableCell>
    </TableRow>
  )
}

// ---------------------------------------------------------------------------
// Skeleton
// ---------------------------------------------------------------------------

function BookingListSkeleton() {
  return (
    <div className="flex flex-col gap-2">
      {Array.from({ length: 5 }).map((_, i) => (
        <Skeleton key={i} className="h-12 w-full rounded-md" />
      ))}
    </div>
  )
}
```

**Key points:**

- The component checks `result._tag` explicitly rather than using the builder API — both are valid. The explicit check is clearer for multi-branch logic that also reads `result.value` sub-fields.
- `result.waiting` is checked _inside_ the Success branch to show a refresh indicator without hiding the existing data.
- All layout primitives (`Table`, `Badge`, `Button`) come from `~/components/ui/` — never from npm package paths directly.
- `cn()` from `~/lib/cn` (clsx + tailwind-merge) is used for conditional class concatenation.

---

## File: `features/booking/components/booking-form.tsx`

A create form. Local React `useState` manages the ephemeral field values while the user types. The atom mutation fires only on submit.

```tsx
// src/features/booking/components/booking-form.tsx
import { Result } from '@effect-atom/atom'
import { type FormEvent, useState } from 'react'
import { Button } from '~/components/ui/button'
import { Input } from '~/components/ui/input'
import { Label } from '~/components/ui/label'
import { useCreateBooking } from '../hooks/use-bookings'

// ---------------------------------------------------------------------------
// BookingForm
// ---------------------------------------------------------------------------

export function BookingForm({ onSuccess }: { onSuccess?: () => void }) {
  const { create, result } = useCreateBooking()

  // Ephemeral UI state — local React state is correct here (see 04-state-management.md)
  const [driverId, setDriverId] = useState('')
  const [vehicleId, setVehicleId] = useState('')
  const [pickupAddress, setPickupAddress] = useState('')
  const [dropoffAddress, setDropoffAddress] = useState('')
  const [pickupDate, setPickupDate] = useState('')
  const [notes, setNotes] = useState('')

  const isSubmitting = result._tag !== 'Initial' && result.waiting
  const error =
    result._tag === 'Failure'
      ? 'Failed to create booking. Please try again.'
      : null

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault()

    create({
      driverId,
      vehicleId,
      pickupAddress,
      dropoffAddress,
      pickupDate,
      notes: notes.trim() === '' ? undefined : notes.trim(),
    })

    // Reset local state after triggering the mutation
    setDriverId('')
    setVehicleId('')
    setPickupAddress('')
    setDropoffAddress('')
    setPickupDate('')
    setNotes('')
  }

  // Call onSuccess when the mutation transitions to a non-waiting Success
  if (result._tag === 'Success' && !result.waiting && onSuccess) {
    onSuccess()
  }

  return (
    <form onSubmit={handleSubmit} className="flex flex-col gap-4">
      <div className="grid grid-cols-2 gap-4">
        <div className="flex flex-col gap-1.5">
          <Label htmlFor="driverId">Driver ID</Label>
          <Input
            id="driverId"
            value={driverId}
            onChange={(e) => setDriverId(e.target.value)}
            placeholder="drv_…"
            required
            disabled={isSubmitting}
          />
        </div>

        <div className="flex flex-col gap-1.5">
          <Label htmlFor="vehicleId">Vehicle ID</Label>
          <Input
            id="vehicleId"
            value={vehicleId}
            onChange={(e) => setVehicleId(e.target.value)}
            placeholder="veh_…"
            required
            disabled={isSubmitting}
          />
        </div>
      </div>

      <div className="flex flex-col gap-1.5">
        <Label htmlFor="pickupAddress">Pickup address</Label>
        <Input
          id="pickupAddress"
          value={pickupAddress}
          onChange={(e) => setPickupAddress(e.target.value)}
          placeholder="123 Main St, London"
          required
          disabled={isSubmitting}
        />
      </div>

      <div className="flex flex-col gap-1.5">
        <Label htmlFor="dropoffAddress">Dropoff address</Label>
        <Input
          id="dropoffAddress"
          value={dropoffAddress}
          onChange={(e) => setDropoffAddress(e.target.value)}
          placeholder="456 Oak Ave, Manchester"
          required
          disabled={isSubmitting}
        />
      </div>

      <div className="flex flex-col gap-1.5">
        <Label htmlFor="pickupDate">Pickup date</Label>
        <Input
          id="pickupDate"
          type="datetime-local"
          value={pickupDate}
          onChange={(e) => setPickupDate(e.target.value)}
          required
          disabled={isSubmitting}
        />
      </div>

      <div className="flex flex-col gap-1.5">
        <Label htmlFor="notes">Notes (optional)</Label>
        <Input
          id="notes"
          value={notes}
          onChange={(e) => setNotes(e.target.value)}
          placeholder="Any special instructions…"
          disabled={isSubmitting}
        />
      </div>

      {error && (
        <p role="alert" className="text-sm text-destructive">
          {error}
        </p>
      )}

      <Button type="submit" disabled={isSubmitting} className="self-end">
        {isSubmitting ? 'Creating…' : 'Create booking'}
      </Button>
    </form>
  )
}
```

**Key points:**

- Local `useState` for every field is intentional — these are ephemeral values that only matter while the user is typing. They do not need to survive re-mounts, be shared across components, or be derived from server state.
- The atom mutation (`create`) is only called on form submit — this keeps the atom history clean (one entry per actual submission, not per keystroke).
- `isSubmitting` is derived from `result.waiting` while the mutation is in a non-Initial state — this disables inputs and changes the button label without any separate loading boolean.
- `onSuccess` is called synchronously inside the render function when the result transitions. This pattern avoids a `useEffect` for the simple case of closing a modal or navigating after a mutation.

---

## File: `features/booking/index.ts`

The barrel export — the feature's public API. Only components and types that other parts of the app (routes, other features) legitimately need to import are exported here. Internal modules (`api/`, `state/`, `hooks/`) are never re-exported.

```ts
// src/features/booking/index.ts

// Components — consumed by route files
export { BookingList } from './components/booking-list'
export { BookingForm } from './components/booking-form'

// Types — may be needed by routes or shared utilities
export type { Booking, BookingId, BookingStatus, BookingFilters } from './contract/schemas'
```

**What is NOT exported:**

| Module | Reason |
|---|---|
| `BookingApi`, `BookingApiLive` | Wired in `src/lib/runtime.ts` only |
| `Booking` (atoms namespace) | Used only by hooks inside this feature |
| `useBookings`, `useCreateBooking`, etc. | Components inside the feature import hooks directly |
| `BookingNotFound`, `BookingConflict` | Domain errors flow through Effect error channels, not component props |

Route files import like this:

```ts
// src/routes/_authenticated/bookings.tsx
import { BookingList, BookingForm } from '~/features/booking'

export function Component() {
  return (
    <div className="p-8 flex flex-col gap-6">
      <h1 className="text-2xl font-semibold">Bookings</h1>
      <BookingForm />
      <BookingList />
    </div>
  )
}
```

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
       │    ├─ HttpClientRequest.schemaBodyJson encodes body
       │    ├─ ApiHttpClient prepends base URL + auth header
       │    └─ HttpClientResponse.schemaBodyJson validates response
       │
       ├─ registry.refresh(list)        ← triggers list re-fetch
       │
       └─ returns Booking               ← Result<Booking, BookingConflict>
              │
              ▼
       BookingForm reads result._tag === 'Success'
              │
              ▼
       onSuccess() called → modal closes / navigation occurs
              │
              ▼
       BookingList re-renders with refreshed list data
```

---

## See Also

- [01-folder-structure.md](./01-folder-structure.md) — directory layout and import rules
- [02-effect-services.md](./02-effect-services.md) — `Context.Tag`, `Layer.effect`, tagged errors
- [03-http-client.md](./03-http-client.md) — `ApiHttpClient`, `schemaBodyJson`, `filterStatusOk`
- [04-state-management.md](./04-state-management.md) — all atom patterns: `Atom.readable`, `Atom.writable`, `apiRuntime.fn`, registry access, reactive sync
