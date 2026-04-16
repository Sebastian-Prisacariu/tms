# 06 — Testing

> Mock Layer injection, not module mocking. Test services, atoms, hooks, and components in isolation.

## Overview

| Level | Tool | File naming | What it tests |
|---|---|---|---|
| Unit | Vitest | `*.test.ts` | Effect services, atoms, utility functions |
| Integration | Vitest | `*.test.ts` | Hooks + atoms with React, component rendering |
| E2E | Playwright | `*.e2e.ts` | Full user flows in the browser |

**Key principle:** Test through Effect's DI system. Mock dependencies by providing `Layer.succeed(Tag, mockImpl)` — never mock ES modules. This keeps tests aligned with how the app actually wires things up.

---

## Unit Testing Effect Services

### Testing a Feature API Service

Mock `ApiHttpClient` to test `BookingApi` in isolation:

```ts
// features/booking/api/booking.api.test.ts
import { describe, it, expect } from 'vitest'
import { Effect, Layer } from 'effect'
import { HttpClient, HttpClientResponse } from '@effect/platform'
import { BookingApi, BookingApiLive } from './booking.api'
import { ApiHttpClient } from '~/lib/http-client'

// Create a mock HttpClient that returns canned responses
function mockHttpClient(handler: (req: HttpClientRequest) => unknown) {
  return HttpClient.make((req) =>
    Effect.succeed(
      HttpClientResponse.fromWeb(
        HttpClientRequest.toURL(req),
        new Response(JSON.stringify(handler(req)), {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
        })
      )
    )
  )
}

const mockBookings = [
  {
    id: 'b1',
    driverId: 'd1',
    vehicleId: 'v1',
    pickupAddress: '123 Main St',
    dropoffAddress: '456 Oak Ave',
    pickupDate: '2026-05-01T10:00:00Z',
    status: 'pending',
    createdAt: '2026-04-01T00:00:00Z',
    updatedAt: '2026-04-01T00:00:00Z',
  },
]

// Wire up: mock ApiHttpClient → real BookingApiLive
const TestBookingApiLayer = BookingApiLive.pipe(
  Layer.provide(
    Layer.succeed(
      ApiHttpClient,
      mockHttpClient(() => ({ items: mockBookings, total: 1, page: 1, pageSize: 20 }))
    )
  )
)

describe('BookingApi', () => {
  it('list returns decoded bookings', () =>
    Effect.gen(function* () {
      const api = yield* BookingApi
      const result = yield* api.list({})

      expect(result.items).toHaveLength(1)
      expect(result.items[0].id).toBe('b1')
      expect(result.items[0].status).toBe('pending')
      // pickupDate is decoded from string to Date by Schema
      expect(result.items[0].pickupDate).toBeInstanceOf(Date)
    }).pipe(Effect.provide(TestBookingApiLayer), Effect.runPromise))

  it('getById returns a single booking', () =>
    Effect.gen(function* () {
      const api = yield* BookingApi
      const booking = yield* api.getById('b1')

      expect(booking.id).toBe('b1')
    }).pipe(
      Effect.provide(
        BookingApiLive.pipe(
          Layer.provide(
            Layer.succeed(
              ApiHttpClient,
              mockHttpClient(() => mockBookings[0])
            )
          )
        )
      ),
      Effect.runPromise,
    ))

  it('create sends the correct payload', () =>
    Effect.gen(function* () {
      const api = yield* BookingApi
      const booking = yield* api.create({
        driverId: 'd1',
        vehicleId: 'v1',
        pickupAddress: '789 Elm St',
        dropoffAddress: '321 Pine St',
        pickupDate: '2026-06-01T10:00:00Z',
      })

      expect(booking.id).toBeDefined()
    }).pipe(
      Effect.provide(
        BookingApiLive.pipe(
          Layer.provide(
            Layer.succeed(
              ApiHttpClient,
              mockHttpClient(() => ({
                ...mockBookings[0],
                id: 'b-new',
                pickupAddress: '789 Elm St',
              }))
            )
          )
        )
      ),
      Effect.runPromise,
    ))
})
```

### Testing a Service That Depends on Another Service

Mock `BookingApi` to test higher-level services:

```ts
// Mock the service tag directly — no HTTP involved
const MockBookingApi = Layer.succeed(BookingApi, {
  list: () => Effect.succeed({ items: [], total: 0, page: 1, pageSize: 20 }),
  getById: (id) => Effect.fail(new BookingNotFound({ bookingId: id })),
  create: (input) =>
    Effect.succeed({
      id: 'mock-id',
      ...input,
      status: 'pending' as const,
      createdAt: new Date(),
      updatedAt: new Date(),
    }),
  update: () => Effect.succeed(/* ... */),
  delete: () => Effect.void,
})

// Use in tests
const result = await Effect.gen(function* () {
  const api = yield* BookingApi
  return yield* api.getById('nonexistent')
}).pipe(Effect.provide(MockBookingApi), Effect.runPromiseExit)

expect(Exit.isFailure(result)).toBe(true)
```

---

## Testing Atoms

### Testing Simple and Derived Atoms

Use `Registry.make()` to create an isolated registry for tests:

```ts
// features/booking/state/booking.atoms.test.ts
import { describe, it, expect } from 'vitest'
import { Registry } from '@effect-atom/atom'
import { Booking } from './booking.atoms'

describe('Booking atoms', () => {
  it('filters atom initializes with defaults', () => {
    const registry = Registry.make()
    const filters = registry.get(Booking.filters)

    expect(filters.page).toBe(1)
    expect(filters.pageSize).toBe(20)
    expect(filters.status).toBeUndefined()
  })

  it('filters atom accepts partial updates', () => {
    const registry = Registry.make()

    registry.set(Booking.filters, { status: 'pending' })
    const filters = registry.get(Booking.filters)

    expect(filters.status).toBe('pending')
    expect(filters.page).toBe(1) // other fields preserved
  })
})
```

### Testing Runtime Atoms (Service-Backed)

For atoms that use `apiRuntime.atom(...)`, provide mock layers through the registry:

```ts
import { describe, it, expect } from 'vitest'
import { Registry, Result } from '@effect-atom/atom'
import { Effect, Layer } from 'effect'
import { Booking } from './booking.atoms'
import { BookingApi } from '../api/booking.api'

describe('Booking list atom', () => {
  it('fetches and returns bookings', async () => {
    const mockData = {
      items: [{ id: 'b1', status: 'pending' /* ... */ }],
      total: 1,
      page: 1,
      pageSize: 20,
    }

    const MockBookingApi = Layer.succeed(BookingApi, {
      list: () => Effect.succeed(mockData),
      getById: () => Effect.fail(new BookingNotFound({ bookingId: 'x' })),
      create: () => Effect.die('not implemented'),
      update: () => Effect.die('not implemented'),
      delete: () => Effect.die('not implemented'),
    })

    // Create registry and wait for the atom to resolve
    const registry = Registry.make({
      // Provide the mock layer to the runtime
      layers: [MockBookingApi],
    })

    // Read the result — it will be Result<BookingListResponse, E>
    await new Promise((resolve) => {
      registry.subscribe(Booking.list, (result) => {
        if (result._tag === 'Success') {
          expect(result.value.items).toHaveLength(1)
          resolve(undefined)
        }
      })
    })
  })
})
```

---

## Testing React Hooks

Use `renderHook` with `RegistryProvider` to test hooks that use atoms:

```ts
// features/booking/hooks/use-bookings.test.ts
import { describe, it, expect } from 'vitest'
import { renderHook, waitFor } from '@testing-library/react'
import { RegistryProvider } from '@effect-atom/atom-react'
import { type ReactNode } from 'react'
import { useBookings } from './use-bookings'

function createWrapper() {
  return function Wrapper({ children }: { children: ReactNode }) {
    return (
      <RegistryProvider defaultIdleTTL={400}>
        {children}
      </RegistryProvider>
    )
  }
}

describe('useBookings', () => {
  it('returns initial loading state', () => {
    const { result } = renderHook(() => useBookings(), {
      wrapper: createWrapper(),
    })

    // Initially, result should be in Initial state
    expect(result.current.result._tag).toBe('Initial')
  })

  it('provides filter setter', () => {
    const { result } = renderHook(() => useBookings(), {
      wrapper: createWrapper(),
    })

    // Set filters
    result.current.setFilters({ status: 'confirmed' })

    expect(result.current.filters.status).toBe('confirmed')
  })
})
```

---

## Testing Components

Render components with the necessary providers and scoped atoms:

```ts
// features/booking/components/booking-list.test.tsx
import { describe, it, expect } from 'vitest'
import { render, screen } from '@testing-library/react'
import { RegistryProvider } from '@effect-atom/atom-react'
import { BookingList } from './booking-list'

function renderWithProviders(ui: React.ReactElement) {
  return render(
    <RegistryProvider defaultIdleTTL={400}>
      {ui}
    </RegistryProvider>
  )
}

describe('BookingList', () => {
  it('renders loading state initially', () => {
    renderWithProviders(<BookingList />)
    expect(screen.getByRole('status')).toBeInTheDocument() // spinner
  })
})
```

**Testing with ScopedAtom:**

```tsx
import { BookingDetailScope } from '../state/booking.scoped'
import { BookingDetail } from './booking-detail'

it('renders booking detail for given ID', () => {
  render(
    <RegistryProvider defaultIdleTTL={400}>
      <BookingDetailScope.Provider value="booking-123">
        <BookingDetail />
      </BookingDetailScope.Provider>
    </RegistryProvider>
  )

  // Assert based on mock data provided through layers
})
```

---

## Playwright E2E Tests

### Page Object Pattern

```ts
// e2e/pages/booking.page.ts
import { type Page, type Locator } from '@playwright/test'

export class BookingPage {
  readonly page: Page
  readonly bookingTable: Locator
  readonly createButton: Locator
  readonly searchInput: Locator
  readonly statusFilter: Locator

  constructor(page: Page) {
    this.page = page
    this.bookingTable = page.getByRole('table')
    this.createButton = page.getByRole('button', { name: 'Create Booking' })
    this.searchInput = page.getByPlaceholder('Search bookings...')
    this.statusFilter = page.getByRole('combobox', { name: 'Status' })
  }

  async goto() {
    await this.page.goto('/bookings')
    await this.bookingTable.waitFor()
  }

  async createBooking(data: {
    pickupAddress: string
    dropoffAddress: string
    date: string
  }) {
    await this.createButton.click()
    await this.page.getByLabel('Pickup Address').fill(data.pickupAddress)
    await this.page.getByLabel('Dropoff Address').fill(data.dropoffAddress)
    await this.page.getByLabel('Pickup Date').fill(data.date)
    await this.page.getByRole('button', { name: 'Submit' }).click()
  }

  async filterByStatus(status: string) {
    await this.statusFilter.click()
    await this.page.getByRole('option', { name: status }).click()
  }

  async search(query: string) {
    await this.searchInput.fill(query)
  }

  async getRowCount() {
    return this.bookingTable.getByRole('row').count() - 1 // minus header
  }
}
```

### E2E Test File

```ts
// e2e/booking-flow.e2e.ts
import { test, expect } from '@playwright/test'
import { BookingPage } from './pages/booking.page'

test.describe('Booking Management', () => {
  let bookingPage: BookingPage

  test.beforeEach(async ({ page }) => {
    // Login (use a test fixture or API call)
    await page.goto('/login')
    await page.getByLabel('Email').fill('test@example.com')
    await page.getByLabel('Password').fill('password')
    await page.getByRole('button', { name: 'Sign in' }).click()

    bookingPage = new BookingPage(page)
    await bookingPage.goto()
  })

  test('displays booking list', async () => {
    const rowCount = await bookingPage.getRowCount()
    expect(rowCount).toBeGreaterThan(0)
  })

  test('creates a new booking', async () => {
    const initialCount = await bookingPage.getRowCount()

    await bookingPage.createBooking({
      pickupAddress: '100 Test St',
      dropoffAddress: '200 Mock Ave',
      date: '2026-06-15',
    })

    // Wait for the list to refresh
    await expect(async () => {
      const newCount = await bookingPage.getRowCount()
      expect(newCount).toBe(initialCount + 1)
    }).toPass({ timeout: 5000 })
  })

  test('filters by status', async () => {
    await bookingPage.filterByStatus('Pending')

    // All visible rows should show "Pending" badge
    const rows = bookingPage.bookingTable.getByRole('row')
    const count = await rows.count()

    for (let i = 1; i < count; i++) {
      await expect(rows.nth(i).getByText('Pending')).toBeVisible()
    }
  })

  test('search filters results', async () => {
    await bookingPage.search('Main St')

    // Results should contain the search term
    const firstRow = bookingPage.bookingTable.getByRole('row').nth(1)
    await expect(firstRow).toContainText('Main St')
  })
})
```

### Playwright Config

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  testMatch: '**/*.e2e.ts',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',

  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
  ],

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
})
```

---

## Vitest Configuration

### Workspace Setup

```ts
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config'

export default defineWorkspace([
  {
    test: {
      name: 'unit',
      include: ['src/**/*.test.ts', 'src/**/*.test.tsx'],
      environment: 'jsdom',
      setupFiles: ['./src/test/setup.ts'],
    },
  },
])
```

### Test Setup

```ts
// src/test/setup.ts
import '@testing-library/jest-dom/vitest'
```

### Package.json Scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
}
```

---

## Testing Cheat Sheet

| What to test | How to mock | Example |
|---|---|---|
| `BookingApi` service | `Layer.succeed(ApiHttpClient, mock)` | Provide fake HTTP responses |
| Higher-level service | `Layer.succeed(BookingApi, mock)` | Provide fake service methods |
| Simple atom | `Registry.make()` + `registry.get/set` | Direct atom manipulation |
| Runtime atom | Provide mock layers to registry | Service-level mocking |
| React hook | `renderHook` + `RegistryProvider` | Wrap in provider |
| Component | `render` + `RegistryProvider` | Wrap in provider |
| Scoped component | `render` + `ScopedAtom.Provider` | Provide scope value |
| E2E flow | Real app + Playwright | Page object pattern |

**Never do:**
- `vi.mock('~/features/booking/api/booking.api')` — module mocking breaks DI
- `jest.spyOn(window, 'fetch')` — mock at the Effect layer, not at fetch
- Test implementation details — test behavior through the service interface
