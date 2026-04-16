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

Barrel export. Only exposes the feature's **public API** — components meant to be rendered by routes, and any types needed by other parts of the app.

```ts
export { BookingList } from './components/booking-list'
export { BookingForm } from './components/booking-form'
export type { Booking, BookingId } from './contract/types'
```

## Import Rules

| From | Can import from | Cannot import from |
|---|---|---|
| `features/booking/` | `~/lib/*`, `~/components/ui/*`, `~/hooks/*`, `~/services/*` | `~/features/driver/*`, `~/features/vehicle/*` (other features) |
| `routes/` | `~/features/*/index.ts` (barrel only), `~/lib/*`, `~/components/ui/*` | Feature internals (`api/`, `state/`, `hooks/`) |
| `lib/` | `~/services/*`, external packages | `~/features/*` |
| `components/ui/` | `~/lib/cn`, external packages | `~/features/*`, `~/services/*` |

**Key rule:** Features never import from other features. If two features need shared data, extract it to `~/services/` or `~/lib/`.

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
