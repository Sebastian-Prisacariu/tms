# 02 — Effect Services

> How services are defined, implemented, and composed using Effect's dependency injection system.

> New to Effect? Start with [00-effect-glossary.md](./00-effect-glossary.md) for a quick mental-model mapping from React/Promise patterns.

## Overview

All data-fetching, side-effectful, and shared infrastructure logic lives in **Effect services**. A service is an interface defined via `Context.Tag` and implemented via a `Layer`. This separates _what a service does_ (the tag/interface) from _how it does it_ (the live layer), enabling testing, mocking, and composition without any framework coupling.

Services never import React. They are pure Effect and can be used in any environment.

---

## 1. Defining a Service

Use `Context.Tag` to declare the interface. The tag is the stable identifier used throughout the app to _depend on_ a service. It is never a concrete implementation.

```ts
// src/features/booking/api/booking.api.ts
import { Context, Effect } from 'effect'
import type { Booking, BookingFilters, CreateBookingInput } from '../contract/types'
import type { BookingNotFound, ValidationError } from '../contract/errors'

class BookingApi extends Context.Tag('BookingApi')<
  BookingApi,
  {
    readonly getById: (id: string) => Effect.Effect<Booking, BookingNotFound>
    readonly list: (filters: BookingFilters) => Effect.Effect<Booking[]>
    readonly create: (input: CreateBookingInput) => Effect.Effect<Booking, ValidationError>
  }
>() {}

export { BookingApi }
```

See [00-effect-glossary.md](./00-effect-glossary.md) for a breakdown of the `Context.Tag` syntax.

**Never export the Live implementation as the default.** Only the tag is public; the layer is wired up at the composition root.

---

## 2. Implementing with `Layer.effect`

Use `Layer.effect` when the implementation needs other services (the common case). The factory function runs in an `Effect.gen`, so you can `yield*` any dependencies.

```ts
// src/features/booking/api/booking.api.ts
import { Context, Effect, Layer } from 'effect'
import { HttpClient, HttpClientRequest } from '@effect/platform'
import { Schema } from 'effect'
import { ApiHttpClient } from '~/lib/http-client'
import { BookingSchema } from '../contract/schemas'
import { BookingNotFound, ValidationError } from '../contract/errors'
import type { Booking, BookingFilters, CreateBookingInput } from '../contract/types'

class BookingApi extends Context.Tag('BookingApi')<
  BookingApi,
  {
    readonly getById: (id: string) => Effect.Effect<Booking, BookingNotFound>
    readonly list: (filters: BookingFilters) => Effect.Effect<Booking[]>
    readonly create: (input: CreateBookingInput) => Effect.Effect<Booking, ValidationError>
  }
>() {}

const BookingApiLive = Layer.effect(
  BookingApi,
  Effect.gen(function* () {
    const client = yield* ApiHttpClient

    return {
      getById: (id) =>
        client.get(`/bookings/${id}`).pipe(
          Effect.flatMap((response) => response.json),
          Effect.flatMap(Schema.decodeUnknown(BookingSchema)),
          Effect.mapError(() => new BookingNotFound({ bookingId: id })),
        ),

      list: (filters) =>
        client.get('/bookings').pipe(
          Effect.flatMap((response) => response.json),
          Effect.flatMap(Schema.decodeUnknown(Schema.Array(BookingSchema))),
        ),

      create: (input) =>
        client
          .post('/bookings', { body: JSON.stringify(input) })
          .pipe(
            Effect.flatMap((response) => response.json),
            Effect.flatMap(Schema.decodeUnknown(BookingSchema)),
            Effect.mapError((e) =>
              new ValidationError({ field: 'unknown', message: String(e) }),
            ),
          ),
    }
  }),
)

export { BookingApi, BookingApiLive }
```

The `yield* ApiHttpClient` call requests the `ApiHttpClient` service from the context. Effect tracks this as a requirement — if `ApiHttpClient` is not provided in the layer graph, the TypeScript type system will surface a compile error.

---

## 3. Configuration with `Config`

For environment variables and app settings, use Effect's built-in `Config` module instead of creating a custom `Context.Tag` service. `Config` provides typed access to configuration values with built-in validation and error messages.

### Reading Config Values

```ts
import { Config, Effect } from 'effect'

// Read a single config value — fails with ConfigError if missing
const apiUrl = Config.string('VITE_API_URL')
const mode = Config.string('MODE')

// With defaults
const authTokenKey = Config.withDefault(Config.string('AUTH_TOKEN_KEY'), 'tms_auth_token')
const pageSize = Config.withDefault(Config.integer('DEFAULT_PAGE_SIZE'), 20)

// Use in an Effect
const program = Effect.gen(function* () {
  const url = yield* apiUrl
  const env = yield* mode
  console.log(`API: ${url}, Mode: ${env}`)
})
```

### Providing Config via ConfigProvider

In a Vite app, environment variables come from `import.meta.env`. Wire them up once via `ConfigProvider`:

```ts
// src/lib/config.ts
import { ConfigProvider, Layer } from 'effect'

/**
 * Provides all import.meta.env values to Effect's Config system.
 * Place this at the bottom of the layer graph so all services can
 * read config via Config.string('VITE_API_URL') etc.
 */
export const AppConfigLive = Layer.setConfigProvider(
  ConfigProvider.fromJson(import.meta.env)
)
```

### Using Config in Services

Services read config values directly via `Config` — no need for a `ConfigService` tag:

```ts
// src/lib/http-client.ts
import { HttpClient, HttpClientRequest } from '@effect/platform'
import { Config, Effect, Layer, Schedule } from 'effect'

export const ApiHttpClientLive = Layer.effect(
  ApiHttpClient,
  Effect.gen(function* () {
    const apiUrl = yield* Config.string('VITE_API_URL')
    const baseClient = yield* HttpClient.HttpClient

    return baseClient.pipe(
      HttpClient.mapRequest((req) =>
        req.pipe(
          HttpClientRequest.prependUrl(apiUrl),
          HttpClientRequest.acceptJson,
        )
      ),
      HttpClient.filterStatusOk,
      HttpClient.retryTransient({
        times: 3,
        schedule: Schedule.exponential('200 millis'),
      }),
    )
  })
).pipe(Layer.provide(BrowserHttpClient.layer))
```

### When to Use `Config` vs `Context.Tag`

| Concern | Use `Config` | Use `Context.Tag` |
|---|---|---|
| Environment variables | `Config.string('VITE_API_URL')` | — |
| Static app settings | `Config.withDefault(Config.integer('PAGE_SIZE'), 20)` | — |
| Service with methods | — | `class BookingApi extends Context.Tag(...)` |
| Stateful service (token mgmt, caching) | — | `class AuthService extends Context.Tag(...)` |

**Rule:** If it's a value you read, use `Config`. If it's a service with behavior (methods, state), use `Context.Tag`.

### Implementing with `Layer.succeed`

`Layer.succeed` is still useful for services with no dependencies — simple adapters or in-memory implementations:

```ts
// Test mock — no deps, no config, just a static implementation
const MockBookingApi = Layer.succeed(BookingApi, {
  list: () => Effect.succeed({ items: [], total: 0, page: 1, pageSize: 20 }),
  getById: (id) => Effect.fail(new BookingNotFound({ bookingId: id })),
  create: (input) => Effect.die('not implemented'),
  update: () => Effect.die('not implemented'),
  delete: () => Effect.die('not implemented'),
})
```

---

## 4. Composing Layers

Layers form a **dependency graph**. Each layer declares what it needs (via `yield*` in `Layer.effect`) and what it provides (its tag). Compose layers with `Layer.provide`, `Layer.merge`, and `Layer.mergeAll`.

### Dependency Chain

```
BrowserHttpClient.layer          ← platform primitive (@effect/platform-browser)
         ↓
  ApiHttpClientLive              ← adds base URL + auth header (src/lib/http-client.ts)
         ↓
  BookingApiLive                 ← BookingApi service
  DriverApiLive                  ← DriverApi service
  VehicleApiLive                 ← VehicleApi service
         ↓
     AppApiLayer                 ← merged layer provided to the atom runtime
```

### Wiring it up

```ts
// src/lib/runtime.ts
import { Layer } from 'effect'
import { BrowserHttpClient } from '@effect/platform-browser'
import { ApiHttpClientLive } from '~/lib/http-client'
import { AppConfigLive } from '~/lib/config'
import { BookingApiLive } from '~/features/booking/api/booking.api'
import { DriverApiLive } from '~/features/driver/api/driver.api'
import { VehicleApiLive } from '~/features/vehicle/api/vehicle.api'

// Bottom of the graph: platform HTTP + config provider
const BaseLayer = Layer.mergeAll(
  BrowserHttpClient.layer,
  AppConfigLive,
)

// ApiHttpClientLive reads Config.string('VITE_API_URL') and BrowserHttpClient
const HttpLayer = ApiHttpClientLive.pipe(Layer.provide(BaseLayer))

// Feature API layers all need ApiHttpClient
const AppApiLayer = Layer.mergeAll(
  BookingApiLive,
  DriverApiLive,
  VehicleApiLive,
).pipe(Layer.provide(HttpLayer))

// Full app layer consumed by the atom runtime
export const AppLayer = AppApiLayer.pipe(Layer.provide(BaseLayer))
```

### Key composition operators

| Operator | What it does |
|---|---|
| `Layer.provide(dependencies)` | Satisfies a layer's requirements by providing another layer beneath it |
| `Layer.merge(a, b)` | Combines two layers that share no dependencies into one |
| `Layer.mergeAll(a, b, c, ...)` | Same as `merge` but for many layers |

**Rule:** Never call `Layer.provide` inside a feature file. Layer wiring belongs in `src/lib/runtime.ts` (or a dedicated composition file). Feature files export the tag and the Live layer — nothing more.

---

## 5. Tagged Errors

Each service defines its own error types in `contract/errors.ts`. Use `Schema.TaggedError` so errors are both serialisable and pattern-matchable via `Effect.catchTag` / `Effect.catchTags`.

```ts
// src/features/booking/contract/errors.ts
import { Schema } from 'effect'

export class BookingNotFound extends Schema.TaggedError<BookingNotFound>()(
  'BookingNotFound',
  {
    bookingId: Schema.String,
  },
) {}

export class ValidationError extends Schema.TaggedError<ValidationError>()(
  'ValidationError',
  {
    field: Schema.String,
    message: Schema.String,
  },
) {}

export class BookingConflict extends Schema.TaggedError<BookingConflict>()(
  'BookingConflict',
  {
    bookingId: Schema.String,
    reason: Schema.String,
  },
) {}
```

Callers use `Effect.catchTag` to handle specific errors without losing type safety:

```ts
import { Effect } from 'effect'
import { BookingApi } from '~/features/booking/api/booking.api'
import { BookingNotFound } from '~/features/booking/contract/errors'

const program = Effect.gen(function* () {
  const api = yield* BookingApi
  const booking = yield* api.getById('b-123').pipe(
    Effect.catchTag('BookingNotFound', (e) =>
      // handle missing booking — redirect, show fallback, etc.
      Effect.succeed(null),
    ),
  )
  return booking
})
```

`Schema.TaggedError` gives each error a `_tag` discriminant field that Effect uses for matching, plus full schema encoding/decoding for free.

---

## 6. Service Design Rules

### Services never import React

Services are pure Effect. If you find yourself importing `useState`, `useEffect`, or any React hook inside a service file, it belongs in a hook (`src/features/{name}/hooks/`) instead.

```ts
// WRONG — services must not touch React
import { useState } from 'react'
import { Context } from 'effect'

// RIGHT — services are pure Effect
import { Context, Effect, Layer } from 'effect'
```

### Error types live in `contract/errors.ts`

Each feature owns its error types. Errors are not defined inline inside the service implementation — they live next to the schemas that describe the domain.

```
features/booking/
├── contract/
│   ├── schemas.ts   ← Effect Schema definitions
│   ├── types.ts     ← TypeScript types derived from schemas
│   └── errors.ts    ← BookingNotFound, ValidationError, etc.
└── api/
    └── booking.api.ts  ← imports errors from ../contract/errors
```

### Depend on tags, not implementations

Services declare dependencies on other `Context.Tag` classes — never on the Live layer directly. This preserves the ability to swap implementations (e.g., mocks in tests).

```ts
// WRONG — importing the live implementation creates a hard coupling
import { ApiHttpClientLive } from '~/lib/http-client'

// RIGHT — depend on the tag; the layer graph provides the implementation
import { ApiHttpClient } from '~/lib/http-client'

const BookingApiLive = Layer.effect(
  BookingApi,
  Effect.gen(function* () {
    const client = yield* ApiHttpClient  // ← tag, not Live
    // ...
  }),
)
```

### The Tag defines the interface; the Layer wires the dependencies

The tag file exports two things: the tag class (interface) and the Live layer (implementation). The tag travels everywhere. The Live layer only appears in `src/lib/runtime.ts`.

```ts
// What a feature file exports
export { BookingApi }        // ← used by atoms, hooks, other services
export { BookingApiLive }    // ← used only by src/lib/runtime.ts

// What callers import
import { BookingApi } from '~/features/booking/api/booking.api'
// Never: import { BookingApiLive } from '...'  (outside of runtime.ts)
```

---

## Quick Reference

```ts
// 1. Define the interface
class MyService extends Context.Tag('MyService')<
  MyService,
  { readonly doThing: (x: string) => Effect.Effect<Result, MyError> }
>() {}

// 2a. Implement with dependencies
const MyServiceLive = Layer.effect(
  MyService,
  Effect.gen(function* () {
    const dep = yield* SomeDependency
    return { doThing: (x) => /* use dep */ }
  }),
)

// 2b. Implement without dependencies
const MyServiceLive = Layer.succeed(MyService, {
  doThing: (x) => Effect.succeed({ result: x }),
})

// 3. Compose
const AppLayer = MyServiceLive.pipe(Layer.provide(SomeDependencyLive))

// 4. Use in an Effect
const program = Effect.gen(function* () {
  const svc = yield* MyService
  return yield* svc.doThing('hello')
})
```

---

## See Also

- [03-http-client.md](./03-http-client.md) — `ApiHttpClient` service: base URL, auth headers, and the `BrowserHttpClient` platform layer
- [04-state-management.md](./04-state-management.md) — how services connect to atoms via `Atom.runtime` and `runtimeAtom.fn`
