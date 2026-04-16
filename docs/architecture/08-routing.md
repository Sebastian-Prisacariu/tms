# 08 — Routing

> TanStack Router (file-based) with typed params, auth guards, and atom-scoped feature components.

## Overview

We use [TanStack Router](https://tanstack.com/router) with its Vite file-based plugin. Routes are **thin wrappers**: they validate params/search, provide scoped atoms when needed, and delegate all logic to feature components. No business logic lives inside route files.

## Table of Contents

- [Why File-Based Routing (Not a Programmatic Registry)](#why-file-based-routing-not-a-programmatic-registry)
- [Code Splitting with Lazy Routes](#code-splitting-with-lazy-routes)
- [Vite Plugin Config](#vite-plugin-config)
- [Route Tree](#route-tree)
- [Root Layout](#root-layout)
- [Auth Guard Layout](#auth-guard-layout)
- [Thin Route Files](#thin-route-files)
- [Typed Search Params](#typed-search-params)
- [Dynamic Route Params](#dynamic-route-params)
- [Search Params + Atom.searchParam](#search-params--atomsearchparam)
- [Layout Slots (Opt-In Layout Content)](#layout-slots-opt-in-layout-content)
- [Route-Feature Mapping Principle](#route-feature-mapping-principle)

---

## Why File-Based Routing (Not a Programmatic Registry)

A common alternative is a **programmatic route registry** — a single object that defines all routes, their components (via `lazyRouteComponent`), sidebar metadata, and access control:

```ts
// ❌ Programmatic registry pattern — we avoid this
const routeRegistry = {
  dashboard: {
    path: '/$tenantId/dashboard',
    component: lazyRouteComponent(() => import('@/pages/Dashboard')),
    privileges: ['dashboard.view'],
    sidebar: { label: 'Dashboard', icon: LayoutDashboard },
  },
  // ...50 more entries
}

// Then build the route tree at runtime:
const routes = Object.entries(routeRegistry).map(([key, entry]) =>
  createRoute({ path: entry.path, component: entry.component })
)
```

**We chose file-based routing instead.** Here's why:

### What you lose with a programmatic registry

| Concern | File-based | Programmatic registry |
|---|---|---|
| **Type-safe params** | `Route.useParams()` knows `{ id: string }` automatically | Manual type assertions or `as unknown as { id: string }` |
| **Type-safe search params** | `validateSearch` with per-route schema, `useSearch()` is typed | Untyped — search params are `Record<string, unknown>` |
| **Type-safe `Link`** | `<Link to="/bookings/$id" params={{ id }}/>` — compiler checks the path exists and params match | Paths are plain strings — typos compile fine, break at runtime |
| **Code generation** | `routeTree.gen.ts` generated automatically — zero manual wiring | You maintain the tree by hand and wire `getParentRoute` yourself |
| **Loader type inference** | `loader` knows which params/search are available for this specific route | `loader` receives a generic context — no route-specific types |
| **Nested layouts** | Automatic via `_layout.tsx` file convention | Manual parent-child wiring in registry |

For a **large TMS with 50+ routes**, the type safety gap compounds fast. Every `useParams()` call, every `<Link>`, every `loader` becomes a potential runtime bug with the registry approach.

### Where to put sidebar metadata and access control

The registry pattern is appealing because it co-locates routing with sidebar and access control. But these concerns belong in separate, purpose-built structures:

```ts
// src/lib/navigation.ts — sidebar config (NOT routing)
import LayoutDashboard from 'lucide-react/icons/layout-dashboard'
import FileText from 'lucide-react/icons/file-text'

export const sidebarItems: SidebarItem[] = [
  {
    label: 'Dashboard',
    icon: LayoutDashboard,
    to: '/$tenantId/dashboard',   // TanStack Router validates this at build time
    privileges: ['dashboard.view'],
  },
  {
    label: 'Bookings',
    icon: FileText,
    to: '/_authenticated/bookings',
    privileges: ['bookings.view'],
    children: [
      { label: 'All Bookings', to: '/_authenticated/bookings' },
      { label: 'New Booking', to: '/_authenticated/bookings/new' },
    ],
  },
  // ...
]
```

Access control lives in `beforeLoad` on the route itself (see [Auth Guard Layout](#auth-guard-layout)):

```tsx
// src/routes/_authenticated/admin.tsx
export const Route = createFileRoute('/_authenticated/admin')({
  beforeLoad: ({ context }) => {
    if (!context.auth.hasPrivilege('admin.access')) {
      throw redirect({ to: '/403' })
    }
  },
  component: AdminPanel,
})
```

This gives you **separation of concerns**: routes handle routing + access, sidebar config handles navigation UI, and both stay type-safe.

---

## Code Splitting with Lazy Routes

For a large TMS, **lazy loading is essential**. Without it, every page and its dependencies are bundled into one initial chunk.

TanStack Router supports code splitting natively via `.lazy.tsx` files. Each route can split its component (and other heavy exports) into a separate chunk that loads on demand — the same benefit as `lazyRouteComponent` but with full type safety.

### How it works

Split a route into two files — the **critical path** (params, search, loaders, beforeLoad) and the **lazy part** (component, pendingComponent, errorComponent):

```tsx
// src/routes/_authenticated/bookings.tsx — critical path (always loaded)
import { createFileRoute } from '@tanstack/react-router'
import { Schema } from 'effect'

const bookingSearchSchema = Schema.Struct({
  status: Schema.optional(Schema.String),
  search: Schema.optional(Schema.String),
  page: Schema.optional(Schema.NumberFromString),
})

export const Route = createFileRoute('/_authenticated/bookings')({
  validateSearch: (search) => Schema.decodeUnknownSync(bookingSearchSchema)(search),
})
```

```tsx
// src/routes/_authenticated/bookings.lazy.tsx — lazy loaded (separate chunk)
import { createLazyFileRoute } from '@tanstack/react-router'
import { BookingList } from '~/features/booking'

export const Route = createLazyFileRoute('/_authenticated/bookings')({
  component: BookingList,
})
```

### What goes where

| Route export | Critical (`.tsx`) | Lazy (`.lazy.tsx`) |
|---|---|---|
| `validateSearch` | Yes | — |
| `beforeLoad` | Yes | — |
| `loader` | Yes | — |
| `component` | — | Yes |
| `pendingComponent` | — | Yes |
| `errorComponent` | — | Yes |

The critical file is loaded upfront (it's tiny — just validation and guards). The component chunk loads only when the user navigates to that route.

### When to split

- **Always split** routes with heavy feature components (data tables, forms, maps, charts)
- **Don't bother splitting** tiny routes (login, 404, redirects) — the overhead of a separate chunk isn't worth it
- **Rule of thumb:** if the feature component imports more than 2-3 modules from `~/features/`, split it

### Simple routes that don't need splitting

For lightweight routes, keep everything in one file:

```tsx
// src/routes/_authenticated/bookings.tsx — no split needed for simple routes
import { createFileRoute } from '@tanstack/react-router'
import { BookingList } from '~/features/booking'

export const Route = createFileRoute('/_authenticated/bookings')({
  component: BookingList,
})
```

Vite's tree-shaking handles the rest. You only need `.lazy.tsx` when the component itself is heavy.

---

## Vite Plugin Config

The `TanStackRouterVite` plugin scans `src/routes/` and auto-generates `src/routeTree.gen.ts`. Import order matters: the router plugin must run before the React plugin.

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { TanStackRouterVite } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [
    TanStackRouterVite(),
    react(),
  ],
  resolve: {
    alias: { '~': '/src' },
  },
})
```

The generated `routeTree.gen.ts` is committed to source control. Never edit it by hand — it is overwritten on every dev server start or build.

---

## Route Tree

File names map directly to URL segments. TanStack Router's conventions:

| File | URL |
|---|---|
| `__root.tsx` | Root layout (wraps everything) |
| `_authenticated.tsx` | Pathless layout — adds auth guard, no URL segment |
| `_authenticated/bookings.tsx` | `/bookings` |
| `_authenticated/bookings.$id.tsx` | `/bookings/:id` |
| `_authenticated/drivers.tsx` | `/drivers` |
| `_authenticated/vehicles.tsx` | `/vehicles` |
| `_authenticated/route-planning.tsx` | `/route-planning` |
| `login.tsx` | `/login` |
| `index.tsx` | `/` (redirect to `/bookings`) |

```
src/routes/
├── __root.tsx
├── _authenticated.tsx
├── _authenticated/
│   ├── bookings.tsx
│   ├── bookings.$id.tsx
│   ├── drivers.tsx
│   ├── vehicles.tsx
│   └── route-planning.tsx
├── login.tsx
└── index.tsx
```

Pathless layout routes (prefixed with `_`) inject shared UI (sidebar, nav) without adding a URL segment. All protected pages live under `_authenticated/`.

---

## Root Layout

The root route wraps every page. It mounts `AtomProvider` once — nesting it deeper would create a second registry. See [04-state-management.md](./04-state-management.md) for why nested providers are guarded against.

```tsx
// src/routes/__root.tsx
import { createRootRoute, Outlet } from '@tanstack/react-router'
import { AtomProvider } from '~/lib/providers'

export const Route = createRootRoute({
  component: () => (
    <AtomProvider>
      <div className="min-h-screen bg-main-background">
        <Outlet />
      </div>
    </AtomProvider>
  ),
})
```

---

## Auth Guard Layout

`_authenticated.tsx` is a pathless layout that runs `beforeLoad` on every child route. If the user is not authenticated, TanStack Router redirects to `/login` before any component renders — no flicker, no conditional rendering in components.

```tsx
// src/routes/_authenticated.tsx
import { createFileRoute, Outlet, redirect } from '@tanstack/react-router'
import { Sidebar } from '~/components/Sidebar'

export const Route = createFileRoute('/_authenticated')({
  beforeLoad: ({ context }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login' })
    }
  },
  component: () => (
    <div className="flex">
      <Sidebar />
      <main className="flex-1 p-6">
        <Outlet />
      </main>
    </div>
  ),
})
```

`context.auth` is provided by the router context — wire it up when creating the router instance:

```ts
// src/router.ts
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'
import { getAuthContext } from '~/lib/auth'

export const router = createRouter({
  routeTree,
  context: {
    auth: getAuthContext(),
  },
})

declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}
```

---

## Thin Route Files

Routes only render the feature component. All state, data fetching, and business logic live in `features/`.

```tsx
// src/routes/_authenticated/bookings.tsx
import { createFileRoute } from '@tanstack/react-router'
import { BookingList } from '~/features/booking'

export const Route = createFileRoute('/_authenticated/bookings')({
  component: BookingList,
})
```

```tsx
// src/routes/_authenticated/drivers.tsx
import { createFileRoute } from '@tanstack/react-router'
import { DriverList } from '~/features/driver'

export const Route = createFileRoute('/_authenticated/drivers')({
  component: DriverList,
})
```

```tsx
// src/routes/index.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/')({
  beforeLoad: () => {
    throw redirect({ to: '/bookings' })
  },
})
```

---

## Typed Search Params

Use Effect's `Schema` to decode and validate search params. `validateSearch` runs before the component mounts — invalid params are rejected rather than silently ignored.

```tsx
// src/routes/_authenticated/bookings.tsx
import { createFileRoute } from '@tanstack/react-router'
import { Schema } from 'effect'
import { BookingList } from '~/features/booking'

const bookingSearchSchema = Schema.Struct({
  status: Schema.optional(Schema.String),
  search: Schema.optional(Schema.String),
  page: Schema.optional(Schema.NumberFromString),
})

type BookingSearch = Schema.Schema.Type<typeof bookingSearchSchema>

export const Route = createFileRoute('/_authenticated/bookings')({
  validateSearch: (search): BookingSearch =>
    Schema.decodeUnknownSync(bookingSearchSchema)(search),
  component: BookingList,
})
```

Consuming the search params inside the feature component:

```tsx
// src/features/booking/BookingList.tsx
import { useSearch } from '@tanstack/react-router'

export function BookingList() {
  const { status, search, page } = useSearch({ from: '/_authenticated/bookings' })

  // status, search, page are fully typed from bookingSearchSchema
  return <>{/* ... */}</>
}
```

Updating search params without a full navigation:

```tsx
import { useNavigate } from '@tanstack/react-router'

const navigate = useNavigate({ from: '/_authenticated/bookings' })

function setPage(page: number) {
  navigate({ search: (prev) => ({ ...prev, page }) })
}
```

---

## Dynamic Route Params

The `$id` segment becomes a typed param. For detail views, wrap the feature component with the appropriate `ScopedAtom.Provider` so all atoms in the subtree receive the correct booking ID. See [04-state-management.md](./04-state-management.md) for full `ScopedAtom` documentation.

```tsx
// src/routes/_authenticated/bookings.$id.tsx
import { createFileRoute } from '@tanstack/react-router'
import { BookingDetailScope } from '~/features/booking/state/booking.scoped'
import { BookingDetail } from '~/features/booking'

export const Route = createFileRoute('/_authenticated/bookings/$id')({
  component: () => {
    const { id } = Route.useParams()
    return (
      <BookingDetailScope.Provider value={id}>
        <BookingDetail />
      </BookingDetailScope.Provider>
    )
  },
})
```

`BookingDetailScope` is a `ScopedAtom` whose `value` is the booking ID string. All atoms declared inside `BookingDetail` that call `BookingDetailScope.use()` receive this ID automatically, isolating their state from other booking detail instances.

---

## Search Params + Atom.searchParam

`Atom.searchParam` binds an atom's value to a URL search parameter. Use it for state that must be URL-shareable or survive a page reload — filter values, pagination, selected tab, etc.

```tsx
// src/features/booking/state/booking.atoms.ts
import { Atom } from '@effect-atom/atom'
import { Schema } from 'effect'
import { Route } from '~/routes/_authenticated/bookings'

// Reads from and writes to ?page=N in the URL
export const pageAtom = Atom.searchParam({
  route: Route,
  key: 'page',
  schema: Schema.NumberFromString,
  defaultValue: 1,
})

// Reads from and writes to ?status=... in the URL
export const statusFilterAtom = Atom.searchParam({
  route: Route,
  key: 'status',
  schema: Schema.String,
  defaultValue: '',
})
```

Components read and write these atoms with the standard `useAtom` / `useAtomValue` hooks. TanStack Router keeps the URL in sync automatically — no manual `useNavigate` calls needed for these fields.

```tsx
import { useAtom, useAtomValue } from '@effect-atom/atom-react'
import { pageAtom, statusFilterAtom } from './state/booking.atoms'

export function BookingList() {
  const [page, setPage] = useAtom(pageAtom)
  const status = useAtomValue(statusFilterAtom)

  return <>{/* page and status are reflected in the URL */}</>
}
```

**Rule:** Only use `Atom.searchParam` for state that should be in the URL. Internal UI state (drawer open, hover, focus) stays in `useState`.

---

## Layout Slots (Opt-In Layout Content)

Pathless layout routes (`_authenticated.tsx`) give all children the same shell. But sometimes you need **per-page variation within the same shell** — an action bar that only some pages show, or a page header that changes per route.

This is not a layout switch — it's a **slot** that pages opt into.

### The Pattern: Atom-Based Slots

A layout slot is an atom that holds `ReactNode`. The layout renders whatever is in the atom. Pages set the atom's content on mount and clear it on unmount.

```ts
// src/components/layout/layout-slots.atoms.ts
import { Atom } from '@effect-atom/atom'
import type { ReactNode } from 'react'

/** Content shown in the action bar area at the bottom of the main layout. */
export const actionBarAtom = Atom.make<ReactNode>(null)

/** Content shown in the page header area (between the global header and the page content). */
export const pageHeaderAtom = Atom.make<ReactNode>(null)
```

### Rendering Slots in the Layout

The layout renders the slot atoms. If they're `null`, nothing is shown — the slot collapses.

```tsx
// src/routes/_authenticated.tsx
import { createFileRoute, Outlet, redirect } from '@tanstack/react-router'
import { useAtomValue } from '@effect-atom/atom-react'
import { Sidebar } from '~/components/sidebar'
import { Header } from '~/components/header'
import { actionBarAtom, pageHeaderAtom } from '~/components/layout/layout-slots.atoms'

export const Route = createFileRoute('/_authenticated')({
  beforeLoad: ({ context }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login' })
    }
  },
  component: AuthenticatedLayout,
})

function AuthenticatedLayout() {
  const actionBar = useAtomValue(actionBarAtom)
  const pageHeader = useAtomValue(pageHeaderAtom)

  return (
    <div className="flex h-screen">
      <Sidebar />
      <div className="flex flex-1 flex-col">
        <Header />
        {pageHeader && (
          <div className="border-b px-6 py-3">{pageHeader}</div>
        )}
        <main className="flex-1 overflow-auto p-6">
          <Outlet />
        </main>
        {actionBar && (
          <div className="border-t px-6 py-2">{actionBar}</div>
        )}
      </div>
    </div>
  )
}
```

### Pages Opt-In via a Hook

A small hook handles the mount/unmount lifecycle:

```ts
// src/hooks/use-layout-slot.ts
import { useAtomSet } from '@effect-atom/atom-react'
import { type ReactNode, useEffect } from 'react'
import type { Atom } from '@effect-atom/atom'

/**
 * Set content in a layout slot while this component is mounted.
 * Clears the slot on unmount.
 */
export function useLayoutSlot(
  slotAtom: Atom.Writable<ReactNode, ReactNode>,
  content: ReactNode
) {
  const setSlot = useAtomSet(slotAtom)

  useEffect(() => {
    setSlot(content)
    return () => setSlot(null)
  }, [content, setSlot])
}
```

### Feature Pages Using Slots

Pages that want an action bar or page header simply call the hook. Pages that don't — do nothing.

```tsx
// src/features/booking/components/booking-list.tsx
import { useLayoutSlot } from '~/hooks/use-layout-slot'
import { actionBarAtom, pageHeaderAtom } from '~/components/layout/layout-slots.atoms'
import { Button } from '~/components/ui/button'

export function BookingList() {
  // Opt-in: show a page header with title + create button
  useLayoutSlot(
    pageHeaderAtom,
    <div className="flex items-center justify-between">
      <h1 className="text-lg font-semibold">Bookings</h1>
      <Button>New Booking</Button>
    </div>
  )

  // Opt-in: show an action bar with bulk actions
  const { selectedIds } = useBookings()
  useLayoutSlot(
    actionBarAtom,
    selectedIds.length > 0 ? (
      <div className="flex items-center gap-2">
        <span className="text-sm text-muted-foreground">
          {selectedIds.length} selected
        </span>
        <Button variant="outline" size="sm">Assign Driver</Button>
        <Button variant="destructive" size="sm">Cancel Bookings</Button>
      </div>
    ) : null
  )

  return <>{/* ... booking list table ... */}</>
}
```

```tsx
// src/features/driver/components/driver-list.tsx
// This page has NO action bar and NO custom page header — it just renders content.

export function DriverList() {
  return <>{/* ... driver list ... */}</>
}
```

### Slot vs Layout: Decision Guide

| Need | Solution |
|---|---|
| Different structural shell (sidebar vs centered card) | Pathless layout route (`_authenticated.tsx`, `_guest.tsx`) |
| Sub-layout within a shell (admin nav inside sidebar layout) | Nested pathless route (`_authenticated/_admin.tsx`) |
| Optional UI area that some pages fill and others don't | Layout slot atom + `useLayoutSlot` hook |
| Page-specific content in a shared area (page header, action bar, breadcrumbs) | Layout slot atom + `useLayoutSlot` hook |

---

## Route-Feature Mapping Principle

Routes are the thinnest possible boundary between the URL and the feature. A route file does exactly three things:

1. **Validate params and search** — reject bad input before any component renders.
2. **Provide scoped atoms** — wrap the feature component with `ScopedAtom.Provider` when the route has a dynamic segment.
3. **Render the feature component** — one line.

Everything else — data fetching, mutations, derived state, UI structure — belongs in `features/`. This keeps routes stable across refactors: the feature can grow arbitrarily complex without touching the route file.

```
URL → Route file (validate, scope) → Feature component (all logic)
```

Cross-references:

- [01-folder-structure.md](./01-folder-structure.md) — where feature files live and the `features/` directory convention.
- [04-state-management.md](./04-state-management.md) — `ScopedAtom`, `Atom.family`, `Atom.kvs`, and all atom patterns used inside features.
