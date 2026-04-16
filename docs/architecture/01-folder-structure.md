# 01 — Folder Structure

> Feature-based organization for a large-scale TMS frontend.

## Overview

Every business domain gets its own folder under `src/features/`. Shared infrastructure lives in top-level directories. Features are **self-contained** — they own their schemas, API calls, state, hooks, and components.

## Full Structure

```
src/
├── features/
│   ├── booking/                    # One feature = one domain
│   │   ├── contract/               # Types, schemas, API contract
│   │   │   ├── schemas.ts          # Effect Schema for API payloads/responses
│   │   │   ├── types.ts            # Shared TypeScript types (derived from schemas)
│   │   │   └── errors.ts           # Domain errors (Schema.TaggedError)
│   │   ├── api/                    # HTTP calls to backend
│   │   │   └── booking.api.ts      # BookingApi service using ApiHttpClient
│   │   ├── state/                  # Atoms
│   │   │   └── booking.atoms.ts    # Atom.make, Atom.readable, runtimeAtom.fn, etc.
│   │   ├── hooks/                  # React hooks
│   │   │   └── use-bookings.ts     # useAtomValue, useAtomSet wrappers
│   │   ├── components/             # Feature-specific UI
│   │   │   ├── booking-list.tsx
│   │   │   └── booking-form.tsx
│   │   └── index.ts                # Barrel export — public API only
│   │
│   ├── driver/                     # Another feature
│   │   ├── contract/
│   │   ├── api/
│   │   ├── state/
│   │   ├── hooks/
│   │   ├── components/
│   │   └── index.ts
│   │
│   ├── route-planning/
│   │   └── ...
│   │
│   └── vehicle/
│       └── ...
│
├── components/
│   └── ui/                         # shadcn/ui base components (installed via CLI)
│       ├── button.tsx
│       ├── input.tsx
│       ├── dialog.tsx
│       └── ...
│
├── hooks/                          # Shared cross-feature hooks
│   └── use-debounce.ts
│
├── lib/                            # Shared utilities & infrastructure
│   ├── http-client.ts              # Configured Effect HttpClient (auth, base URL)
│   ├── providers.tsx               # AtomProvider (RegistryProvider wrapper)
│   ├── cn.ts                       # clsx + tailwind-merge utility
│   └── runtime.ts                  # Atom.runtime setup with AppLayer
│
├── services/                       # Shared Effect services
│   ├── auth.service.ts             # AuthService (token management)
│   └── config.service.ts           # ConfigService (env vars)
│
├── routes/                         # TanStack Router file-based routes
│   ├── __root.tsx                  # Root layout
│   ├── _authenticated.tsx          # Auth guard layout
│   ├── _authenticated/
│   │   ├── bookings.tsx            # /bookings route → renders BookingList
│   │   ├── bookings.$id.tsx        # /bookings/:id route → renders BookingDetail
│   │   ├── drivers.tsx
│   │   └── vehicles.tsx
│   └── login.tsx
│
└── styles/
    └── globals.css                 # Tailwind v4 @theme, CSS custom properties
```

## Directory Purposes

### `features/{name}/contract/`

The **contract** defines the data shapes shared across the feature. Everything here is framework-agnostic — no React, no atoms, no hooks.

- `schemas.ts` — Effect Schema definitions for API request/response payloads. Branded types for IDs (e.g., `BookingId`).
- `types.ts` — Plain TypeScript types derived from schemas via `Schema.Type<typeof MySchema>`. Also enums, constants, and utility types.
- `errors.ts` — Domain-specific errors using `Schema.TaggedError`. These flow through Effect's error channel.

### `features/{name}/api/`

One file per resource. Each file defines an Effect **service** (via `Context.Tag`) that uses `ApiHttpClient` to call the backend. See [03-http-client.md](./03-http-client.md).

### `features/{name}/state/`

Atom definitions grouped by concern. Files are named `{concern}.atoms.ts`. All atoms for a feature are exported as a single namespace object:

```ts
export const Booking = { list, detail, filters, create, update }
```

See [04-state-management.md](./04-state-management.md) for all atom patterns.

### `features/{name}/hooks/`

React hooks that bridge atoms to components. Hooks use `useAtomValue`, `useAtomSet`, `useAtom` from `@effect-atom/atom-react`. They provide a clean API surface for components:

```ts
export function useBookings() {
  const list = useAtomValue(Booking.list)
  const refresh = useAtomRefresh(Booking.list)
  return { list, refresh }
}
```

### `features/{name}/components/`

Feature-specific React components. They import hooks from `../hooks/` and UI primitives from `~/components/ui/`. They never import atoms directly — always go through hooks.

### `features/{name}/index.ts`

Barrel export. Exposes the feature's **public API** — components meant to be embedded by routes or other features, plus re-exports from `contract/`.

```ts
// features/booking/index.ts

// Public components — can be rendered by routes or other features
export { BookingList } from './components/booking-list'
export { BookingForm } from './components/booking-form'
export { BookingStatusBadge } from './components/booking-status-badge'

// Re-export contract — types, schemas, errors are always available
export type { Booking, BookingId, BookingStatus, CreateBookingInput } from './contract/types'
export { BookingSchema, BookingIdSchema } from './contract/schemas'
export { BookingNotFound } from './contract/errors'
```

## Import Rules

In a TMS, domains are interconnected — a booking references drivers and vehicles, route planning references all three. Features **can** import from each other, but only through controlled boundaries.

### Two-Tier Export System

Each feature has two public boundaries:

| Tier | Import path | What's exposed | When to use |
|---|---|---|---|
| **Contract** | `~/features/driver/contract/schemas` | Types, schemas, errors, constants | You need a type or schema from another feature (e.g., `DriverId` in a booking form) |
| **Barrel** | `~/features/driver` (index.ts) | Contract re-exports + public components | You need to render another feature's component (e.g., `DriverSelect` in a booking form) |

### What is allowed

```ts
// features/booking/contract/schemas.ts
// ✅ Import types/schemas from another feature's contract
import { DriverId } from '~/features/driver/contract/schemas'
import { VehicleId } from '~/features/vehicle/contract/schemas'

export const BookingSchema = Schema.Struct({
  id: BookingId,
  driverId: Schema.NullOr(DriverId),   // references driver domain
  vehicleId: Schema.NullOr(VehicleId), // references vehicle domain
  // ...
})
```

```tsx
// features/booking/components/booking-form.tsx
// ✅ Import a public component from another feature's barrel
import { DriverSelect } from '~/features/driver'

function BookingForm() {
  return (
    <form>
      {/* DriverSelect is a public component exported from the driver feature */}
      <DriverSelect onSelect={handleDriverSelect} />
      {/* ... */}
    </form>
  )
}
```

### What is NOT allowed

```ts
// ❌ Importing another feature's API service
import { DriverApi } from '~/features/driver/api/driver.api'

// ❌ Importing another feature's atoms
import { Driver } from '~/features/driver/state/driver.atoms'

// ❌ Importing another feature's hooks
import { useDrivers } from '~/features/driver/hooks/use-drivers'

// ❌ Importing an internal component not exported from index.ts
import { DriverRow } from '~/features/driver/components/driver-row'
```

### Why this boundary?

| Layer | Cross-feature import? | Reason |
|---|---|---|
| `contract/` (types, schemas, errors) | **Yes** | Types are stable, have no runtime behavior, and no dependencies. `DriverId` is just a branded string — it doesn't pull in the driver feature's React components or API calls. |
| `index.ts` (public components) | **Yes, curated** | A `DriverSelect` dropdown is a reusable widget. But only explicitly exported components — internal components like `DriverRow` stay private. |
| `api/` (Effect services) | **No** | If booking needs driver data, it should call its own backend endpoint (e.g., `GET /bookings/:id` returns driver info) or use a shared service in `~/services/`. Features shouldn't call each other's APIs — that creates hidden coupling. |
| `state/` (atoms) | **No** | Atoms are internal reactivity. If two features need shared reactive state, use a shared service or lift the atom to `~/lib/`. |
| `hooks/` (React hooks) | **No** | Hooks depend on feature-internal atoms. Cross-feature hook usage creates deep coupling to another feature's state implementation. |

### Full Import Matrix

| From | Can import from | Cannot import from |
|---|---|---|
| `features/booking/contract/` | Other features' `contract/` dirs, `~/lib/*`, `~/services/*` | Other features' `api/`, `state/`, `hooks/`, `components/` |
| `features/booking/api/` | Own `contract/`, other features' `contract/`, `~/lib/*`, `~/services/*` | Other features' `api/`, `state/`, `hooks/` |
| `features/booking/state/` | Own `contract/`, own `api/`, `~/lib/*`, `~/services/*` | Other features (any dir) |
| `features/booking/hooks/` | Own `contract/`, own `state/`, `~/lib/*`, `~/hooks/*` | Other features (any dir) |
| `features/booking/components/` | Own `hooks/`, own `contract/`, other features' `index.ts` (barrel), `~/components/ui/*`, `~/hooks/*`, `~/lib/*` | Other features' internals |
| `routes/` | Any feature's `index.ts` (barrel only), `~/lib/*`, `~/components/ui/*` | Feature internals |
| `lib/` | `~/services/*`, external packages | `~/features/*` |
| `components/ui/` | `~/lib/cn`, external packages | `~/features/*`, `~/services/*` |

### When to extract to shared

If you find yourself importing the same type from `features/driver/contract/` in 4+ other features, consider whether it belongs in `~/services/` or `~/lib/` instead. Common candidates:

- `~/lib/types.ts` — shared ID types used everywhere (`TenantId`, `UserId`)
- `~/services/address.service.ts` — address lookup used by booking, driver, vehicle, route-planning

## File Naming Conventions

| Type | Convention | Example |
|---|---|---|
| Components | `kebab-case.tsx` | `booking-list.tsx` |
| Hooks | `use-{name}.ts` | `use-bookings.ts` |
| Atoms | `{concern}.atoms.ts` | `booking.atoms.ts` |
| API services | `{resource}.api.ts` | `booking.api.ts` |
| Schemas | `schemas.ts` | `schemas.ts` |
| Effect services | `{name}.service.ts` | `auth.service.ts` |
| Tests (unit) | `{name}.test.ts` | `booking.api.test.ts` |
| Tests (e2e) | `{name}.e2e.ts` | `booking-flow.e2e.ts` |
