# 01 — Folder Structure

> Feature-based organization for a large-scale TMS frontend.

## Overview

Every business domain gets its own folder under `src/features/`. Shared infrastructure lives in top-level directories. Features are **self-contained** — they own their schemas, API calls, state, hooks, and components.

## Full Structure

```
src/
├── features/
│   ├── booking/                    # One feature = one domain
│   │   ├── contract/               # Types, schemas, errors
│   │   │   ├── schemas.ts          # Effect Schema for API payloads/responses
│   │   │   ├── types.ts            # TypeScript types (derived from schemas)
│   │   │   └── errors.ts           # Domain errors (Schema.TaggedError)
│   │   ├── booking.api.ts          # BookingApi service (Context.Tag + Layer)
│   │   ├── booking.atoms.ts        # Atom state (Atom.make, runtimeAtom.fn, etc.)
│   │   ├── booking.hooks.ts        # React hooks (useAtomValue, useAtomSet wrappers)
│   │   ├── components/             # Feature-specific UI
│   │   │   ├── booking-list.tsx
│   │   │   └── booking-form.tsx
│   │   └── index.ts                # Barrel export — public API only
│   │
│   ├── driver/                     # Another feature
│   │   ├── contract/
│   │   ├── driver.api.ts
│   │   ├── driver.atoms.ts
│   │   ├── driver.hooks.ts
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
│   ├── config.ts                   # AppConfigLive — ConfigProvider from import.meta.env
│   ├── http-client.ts              # Configured Effect HttpClient (auth, base URL)
│   ├── providers.tsx               # AtomProvider (RegistryProvider wrapper)
│   ├── cn.ts                       # clsx + tailwind-merge utility
│   └── runtime.ts                  # Atom.runtime setup with AppLayer
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

### `features/{name}/{name}.api.ts`

The Effect **service** for this feature — a `Context.Tag` (interface) + `Layer.effect` (implementation) that uses `ApiHttpClient` to call the backend. See [03-http-client.md](./03-http-client.md).

### `features/{name}/{name}.atoms.ts`

Atom definitions for the feature. All atoms are exported as a single namespace object:

```ts
export const Booking = { list, detail, filters, create, update }
```

See [04-state-management.md](./04-state-management.md) for all atom patterns.

### `features/{name}/{name}.hooks.ts`

React hooks that bridge atoms to components. Hooks use `useAtomValue`, `useAtomSet`, `useAtom` from `@effect-atom/atom-react`:

```ts
export function useBookings() {
  const list = useAtomValue(Booking.list)
  const refresh = useAtomRefresh(Booking.list)
  return { list, refresh }
}
```

### `features/{name}/components/`

Feature-specific React components. They import hooks from `../{name}.hooks` and UI primitives from `~/components/ui/`. They never import atoms directly — always go through hooks.

### Growing into directories

Start flat. When a concern has 3+ files, create a directory for it:

```
# Before: one atoms file → flat
booking/
  booking.atoms.ts

# After: multiple atoms files → directory
booking/
  state/
    booking-list.atoms.ts
    booking-detail.atoms.ts
    booking-mutations.atoms.ts
```

Same applies to API (multiple endpoints), hooks (multiple hooks), etc.

### `features/{name}/index.ts`

The barrel export is the feature's **public API**. Other features, routes, and shared code import from here — never from internal files. The barrel can export anything the feature wants to make public: types, schemas, errors, service Tags, components.

```ts
// features/booking/index.ts

// Types and schemas
export type { Booking, BookingId, BookingStatus, CreateBookingInput } from './contract/types'
export { BookingSchema, BookingIdSchema } from './contract/schemas'

// Errors
export { BookingNotFound, BookingConflict } from './contract/errors'

// Service Tags (interface only — NOT the Layer implementation)
export { BookingApi } from './api/booking.api'

// Public components
export { BookingList } from './components/booking-list'
export { BookingForm } from './components/booking-form'
export { BookingStatusBadge } from './components/booking-status-badge'
```

**What goes in the barrel:**
- Types, schemas, errors, constants — always safe
- Service Tags (`Context.Tag`) — just an interface, no runtime coupling
- Reusable components — widgets other features might embed

**What stays out:**
- Layer implementations (`BookingApiLive`) — wired up in `runtime.ts`, not imported by features
- Atoms — internal reactivity
- Hooks — depend on internal atoms
- Internal components — not meant for reuse

## Import Rules

One rule: **features import from each other's `index.ts` barrel. Never from internal files.**

### What is allowed

```ts
// features/booking/contract/schemas.ts
// ✅ Import types from another feature's barrel
import { DriverId } from '~/features/driver'
import { VehicleId } from '~/features/vehicle'

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
      <DriverSelect onSelect={handleDriverSelect} />
      {/* ... */}
    </form>
  )
}
```

```ts
// features/booking/booking.api.ts
// ✅ Import a service Tag from another feature's barrel (it's just an interface)
import { AuthService } from '~/features/auth'

const BookingApiLive = Layer.effect(
  BookingApi,
  Effect.gen(function* () {
    const auth = yield* AuthService   // use the Tag — implementation is wired elsewhere
    const client = yield* ApiHttpClient
    // ...
  })
)
```

### What is NOT allowed

```ts
// ❌ Importing from internal files — always go through the barrel
import { DriverApi } from '~/features/driver/driver.api'
import { Driver } from '~/features/driver/driver.atoms'
import { useDrivers } from '~/features/driver/driver.hooks'
import { DriverRow } from '~/features/driver/components/driver-row'
```

The rule is simple: if it's not in `index.ts`, it's private. No exceptions, no per-directory rules to remember.

### Why barrel-only?

The barrel export acts as a **deliberate public API**. A feature author decides what to expose. Everything else is an implementation detail that can change without breaking other features.

| Exported from barrel | Safe to import? | Why |
|---|---|---|
| Types, schemas, errors | Yes | Pure data shapes — zero runtime behavior |
| Service Tags (`Context.Tag`) | Yes | Just an interface identifier — no implementation pulled in. The Layer is wired separately in `runtime.ts` |
| Public components | Yes | Reusable widgets — self-contained with their own hooks/state |
| Layer implementations | **Never export** | Creates hard coupling to another feature's dependency graph |
| Atoms | **Never export** | Creates shared mutable state across features — impossible to reason about |
| Hooks | **Never export** | Depend on internal atoms — coupling to state implementation |

### Enforcing the rule

Use `eslint-plugin-boundaries` to enforce barrel-only imports at CI time:

```json
// .eslintrc.json (simplified)
{
  "rules": {
    "boundaries/element-types": [2, {
      "default": "disallow",
      "rules": [
        {
          "from": "features/*",
          "allow": ["features/*/index.ts", "lib/*", "components/ui/*", "hooks/*"]
        }
      ]
    }]
  }
}
```

### Import Matrix

| From | Can import from | Cannot import from |
|---|---|---|
| Any feature | Other features' `index.ts` (barrel), `~/lib/*`, `~/components/ui/*`, `~/hooks/*` | Other features' internal files |
| `routes/` | Any feature's `index.ts`, `~/lib/*`, `~/components/ui/*` | Feature internals |
| `lib/` | External packages | `~/features/*` |
| `components/ui/` | `~/lib/cn`, external packages | `~/features/*` |

### When to extract to `~/lib/`

If a type or utility is used by nearly every feature, consider moving it to `~/lib/` instead of keeping it in a feature barrel:

- `~/lib/types.ts` — shared ID types used everywhere (`TenantId`, `UserId`)
- `~/lib/address.ts` — address utilities used by booking, driver, vehicle, route-planning

Everything else — including auth, config, and other services — is a feature with its own barrel export. Auth has a login page, profile settings, and session management; config might have an admin panel. They're features that happen to also export service Tags.

## File Naming Conventions

| Type | Convention | Example |
|---|---|---|
| API service | `{feature}.api.ts` | `booking.api.ts` |
| Atoms | `{feature}.atoms.ts` | `booking.atoms.ts` |
| Hooks | `{feature}.hooks.ts` | `booking.hooks.ts` |
| Components | `kebab-case.tsx` | `booking-list.tsx` |
| Schemas | `schemas.ts` | `contract/schemas.ts` |
| Errors | `errors.ts` | `contract/errors.ts` |
| Types | `types.ts` | `contract/types.ts` |
| Tests (unit) | `{name}.test.ts` | `booking.api.test.ts` |
| Tests (e2e) | `{name}.e2e.ts` | `booking-flow.e2e.ts` |
