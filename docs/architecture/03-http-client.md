# 03 — HTTP Client

> `@effect/platform` HttpClient configured as an injectable Effect service — base URL, auth headers, schema validation, and feature API services.

## Overview

The frontend communicates with the headless backend exclusively through Effect services built on `@effect/platform`'s `HttpClient`. There is no raw `fetch` anywhere in application code.

The stack has two layers:

1. **`ApiHttpClient`** — a pre-configured `HttpClient` wired with the base URL, default headers, and auth token injection. Lives in `src/lib/http-client.ts`.
2. **Feature API services** — thin Effect services (e.g., `BookingApi`) that use `ApiHttpClient` to make typed, schema-validated requests. Live in `src/features/{name}/api/`.

See [02-effect-services.md](./02-effect-services.md) for the general service pattern and [04-state-management.md](./04-state-management.md) for connecting API services to atoms.

---

## 1. ApiHttpClient Service

`ApiHttpClient` is a `Context.Tag` that wraps a pre-configured `HttpClient.HttpClient`. Consumers yield it from context and get a client that already has the base URL prepended and default headers set.

**`src/lib/http-client.ts`**

```ts
import { HttpClient, HttpClientRequest } from '@effect/platform'
import { BrowserHttpClient } from '@effect/platform-browser'
import { Config, Context, Effect, Layer, Schedule } from 'effect'

// Tag — identifies this service in the Effect context
export class ApiHttpClient extends Context.Tag('ApiHttpClient')<
  ApiHttpClient,
  HttpClient.HttpClient
>() {}

// Live implementation — wraps the platform HttpClient with project-wide config
export const ApiHttpClientLive = Layer.effect(
  ApiHttpClient,
  Effect.gen(function* () {
    // Read base URL from Config (provided by AppConfigLive → import.meta.env)
    const apiUrl = yield* Config.string('VITE_API_URL')
    const baseClient = yield* HttpClient.HttpClient

    return baseClient.pipe(
      HttpClient.mapRequest((req) =>
        req.pipe(
          HttpClientRequest.prependUrl(apiUrl),
          // Sets Accept: application/json header
          // (Content-Type is set automatically by schemaBodyJson when sending a body)
          HttpClientRequest.acceptJson,
        )
      ),
      // Turns any non-2xx response into a ResponseError in the error channel
      HttpClient.filterStatusOk,
      // Retry on transient errors: network failures, timeouts, and status >= 429
      HttpClient.retryTransient({
        times: 3,
        schedule: Schedule.exponential('200 millis'),
      }),
    )
  })
).pipe(Layer.provide(BrowserHttpClient.layer))
```

`BrowserHttpClient.layer` provides the underlying `fetch`-based `HttpClient.HttpClient`. `ApiHttpClientLive` depends on it and is self-contained — callers just provide `ApiHttpClientLive` to their layer graph.

---

## 2. Dynamic Auth Headers

When auth tokens must be read at request time (not at layer construction time), use `HttpClient.mapRequestEffect`. This allows yielding from `AuthService` inside the request transformation, so a fresh token is retrieved for each request.

**`src/lib/http-client.ts`** (auth-aware variant)

```ts
import { HttpClient, HttpClientRequest } from '@effect/platform'
import { BrowserHttpClient } from '@effect/platform-browser'
import { Config, Context, Effect, Layer, Schedule } from 'effect'
import { AuthService } from '~/services/auth.service'

export class ApiHttpClient extends Context.Tag('ApiHttpClient')<
  ApiHttpClient,
  HttpClient.HttpClient
>() {}

export const ApiHttpClientLive = Layer.effect(
  ApiHttpClient,
  Effect.gen(function* () {
    const apiUrl = yield* Config.string('VITE_API_URL')
    const auth = yield* AuthService
    const baseClient = yield* HttpClient.HttpClient

    return baseClient.pipe(
      HttpClient.mapRequestEffect((req) =>
        Effect.map(auth.getToken, (token) =>
          req.pipe(
            HttpClientRequest.prependUrl(apiUrl),
            HttpClientRequest.acceptJson,
            HttpClientRequest.bearerToken(token),
          )
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

The key difference:

|                      | `mapRequest`                                | `mapRequestEffect`                         |
| -------------------- | ------------------------------------------- | ------------------------------------------ |
| Callback return type | `HttpClientRequest`                         | `Effect<HttpClientRequest>`                |
| Can yield Effects    | No                                          | Yes                                        |
| Use for              | Static transforms (base URL, fixed headers) | Dynamic transforms (token lookup, signing) |

---

## 3. Feature API Service Pattern

Each feature owns its API service in `src/features/{name}/api/{name}.api.ts`. The service is a `Context.Tag` whose implementation yields `ApiHttpClient` and exposes typed methods.

### Complete BookingApi Example

**`src/features/booking/api/booking.api.ts`**

```ts
import {
  HttpClient,
  HttpClientRequest,
  HttpClientResponse,
} from "@effect/platform";
import { Context, Effect, Layer } from "effect";
import { Schema } from "effect";
import { ApiHttpClient } from "~/lib/http-client";
import {
  BookingRequest,
  BookingResponse,
  BookingListResponse,
  BookingUpdateRequest,
  BookingPatchRequest,
} from "../contract/schemas";
import {
  BookingNotFound,
  BookingValidationError,
  ApiRequestFailed,
} from "../contract/errors";

// --- Service interface ---

interface BookingApiShape {
  readonly getAll: Effect.Effect<
    Schema.Schema.Type<typeof BookingListResponse>,
    ApiRequestFailed
  >;
  readonly getById: (
    id: string,
  ) => Effect.Effect<
    Schema.Schema.Type<typeof BookingResponse>,
    BookingNotFound | ApiRequestFailed
  >;
  readonly create: (
    body: Schema.Schema.Type<typeof BookingRequest>,
  ) => Effect.Effect<
    Schema.Schema.Type<typeof BookingResponse>,
    BookingValidationError | ApiRequestFailed
  >;
  readonly update: (
    id: string,
    body: Schema.Schema.Type<typeof BookingUpdateRequest>,
  ) => Effect.Effect<
    Schema.Schema.Type<typeof BookingResponse>,
    BookingNotFound | ApiRequestFailed
  >;
  readonly patch: (
    id: string,
    body: Schema.Schema.Type<typeof BookingPatchRequest>,
  ) => Effect.Effect<
    Schema.Schema.Type<typeof BookingResponse>,
    BookingNotFound | ApiRequestFailed
  >;
  readonly remove: (
    id: string,
  ) => Effect.Effect<void, BookingNotFound | ApiRequestFailed>;
}

// --- Tag ---

export class BookingApi extends Context.Tag("BookingApi")<
  BookingApi,
  BookingApiShape
>() {}

// --- Live implementation ---

export const BookingApiLive = Layer.effect(
  BookingApi,
  Effect.gen(function* () {
    const client = yield* ApiHttpClient;

    // Schema for matching ResponseError shape from @effect/platform
    const ResponseError = Schema.Struct({
      _tag: Schema.Literal('ResponseError'),
      response: Schema.Struct({ status: Schema.Number }),
      message: Schema.String,
    })
    const isResponseError = Schema.is(ResponseError)

    // Helper: map HTTP/parse errors to domain errors based on status
    const mapHttpError = (method: string, path: string) => (error: unknown) => {
      if (isResponseError(error)) {
        if (error.response.status === 404) {
          return new BookingNotFound({ bookingId: path })
        }
        return new ApiRequestFailed({
          method, path,
          status: error.response.status,
          message: error.message,
        })
      }
      return new ApiRequestFailed({
        method, path,
        status: 0,
        message: String(error),
      })
    }

    return {
      // GET /bookings — decode response with schema
      getAll: client
        .get("/bookings")
        .pipe(
          Effect.flatMap(
            HttpClientResponse.schemaBodyJson(BookingListResponse),
          ),
          Effect.mapError(mapHttpError("GET", "/bookings")),
        ),

      // GET /bookings/:id
      getById: (id) =>
        client
          .get(`/bookings/${id}`)
          .pipe(
            Effect.flatMap(HttpClientResponse.schemaBodyJson(BookingResponse)),
            Effect.mapError(mapHttpError("GET", `/bookings/${id}`)),
          ),

      // POST /bookings — encode request body with schema
      create: (body) =>
        HttpClientRequest.post("/bookings").pipe(
          HttpClientRequest.schemaBodyJson(BookingRequest)(body),
          Effect.flatMap(client.execute),
          Effect.flatMap(HttpClientResponse.schemaBodyJson(BookingResponse)),
          Effect.mapError(mapHttpError("POST", "/bookings")),
        ),

      // PUT /bookings/:id — full replacement
      update: (id, body) =>
        HttpClientRequest.put(`/bookings/${id}`).pipe(
          HttpClientRequest.schemaBodyJson(BookingUpdateRequest)(body),
          Effect.flatMap(client.execute),
          Effect.flatMap(HttpClientResponse.schemaBodyJson(BookingResponse)),
          Effect.mapError(mapHttpError("PUT", `/bookings/${id}`)),
        ),

      // PATCH /bookings/:id — partial update
      patch: (id, body) =>
        HttpClientRequest.patch(`/bookings/${id}`).pipe(
          HttpClientRequest.schemaBodyJson(BookingPatchRequest)(body),
          Effect.flatMap(client.execute),
          Effect.flatMap(HttpClientResponse.schemaBodyJson(BookingResponse)),
          Effect.mapError(mapHttpError("PATCH", `/bookings/${id}`)),
        ),

      // DELETE /bookings/:id — no response body
      remove: (id) =>
        client
          .del(`/bookings/${id}`)
          .pipe(
            Effect.asVoid,
            Effect.mapError(mapHttpError("DELETE", `/bookings/${id}`)),
          ),
    };
  }),
);
```

### Request Method Helpers

`HttpClient` exposes convenience methods for common verbs:

```ts
client.get("/path"); // GET
client.post("/path"); // POST (no body — use HttpClientRequest.post for body)
client.put("/path"); // PUT
client.patch("/path"); // PATCH
client.del("/path"); // DELETE
client.execute(request); // Execute a pre-built HttpClientRequest
```

---

### Requests with Payloads (POST, PUT, PATCH)

When a request needs a body, you build the request with `HttpClientRequest.*`, attach the body, then execute via `client.execute`:

```ts
// The pipe chain:
HttpClientRequest.post("/bookings").pipe(
  // 1. Create request
  HttpClientRequest.schemaBodyJson(MySchema)(body), // 2. Validate + encode body
  Effect.flatMap(client.execute), // 3. Send request
  Effect.flatMap(HttpClientResponse.schemaBodyJson(ResponseSchema)), // 4. Decode response
);
```

**Step by step:**

1. `HttpClientRequest.post('/bookings')` — creates an `HttpClientRequest` with method POST and path `/bookings`. No body yet.
2. `HttpClientRequest.schemaBodyJson(CreateBookingInput)(body)` — this is where **input validation happens**. The function:
   - Validates `body` against the `CreateBookingInput` schema
   - If validation **fails** (wrong types, missing fields, invalid values) → short-circuits with a `BodyError` — the request is **never sent**
   - If validation **passes** → encodes the value to JSON, sets `Content-Type: application/json`, and attaches the body to the request
   - Returns `Effect<HttpClientRequest, BodyError>` — the pipe chain transitions from a plain `HttpClientRequest` to an `Effect`
3. `Effect.flatMap(client.execute)` — sends the validated request
4. `Effect.flatMap(HttpClientResponse.schemaBodyJson(ResponseSchema))` — validates the response

**The key insight: `schemaBodyJson` is an input validation gate.** If you pass `{ status: 123 }` where the schema expects `Schema.String`, the Effect fails before any HTTP request is made.

#### Alternative: `Effect.gen` style (more readable for complex requests)

```ts
create: (input) =>
  Effect.gen(function* () {
    const req = yield* HttpClientRequest.schemaBodyJson(CreateBookingInput)(
      HttpClientRequest.post('/bookings'),
      input,
    )
    const res = yield* client.execute(req)
    return yield* HttpClientResponse.schemaBodyJson(BookingResponse)(res)
  }),
```

Both styles are equivalent. Use `Effect.gen` when you need intermediate logic (logging, conditional headers, etc.) between building the request and executing it.

---

### Requests with Query Parameters (GET with filters)

For GET requests that need query/search parameters, use `HttpClientRequest.setUrlParams`. It accepts a `Record` where:
- **Values are coerced automatically** — numbers, booleans, and strings all work (no `String()` wrapping)
- **`undefined` values are stripped** — pass optional fields directly, no conditional spreading needed

```ts
// Inside BookingApiLive:
list: (filters: BookingFilters) =>
  Effect.gen(function* () {
    const req = HttpClientRequest.get('/bookings').pipe(
      // undefined values are automatically stripped — no conditional spread needed
      HttpClientRequest.setUrlParams({
        page: filters.page ?? 1,
        pageSize: filters.pageSize ?? 20,
        status: filters.status,       // undefined → stripped
        search: filters.search,       // undefined → stripped
        dateFrom: filters.dateFrom,   // undefined → stripped
        dateTo: filters.dateTo,       // undefined → stripped
      }),
    )
    const res = yield* client.execute(req)
    return yield* HttpClientResponse.schemaBodyJson(BookingListResponse)(res)
  }),
```

You can also add params one at a time with `appendUrlParam`:

```ts
HttpClientRequest.get('/bookings').pipe(
  HttpClientRequest.appendUrlParam('page', '1'),
  HttpClientRequest.appendUrlParam('status', 'pending'),
)
```

#### Schema-Validated Query Parameters

Validate filter input with Effect Schema, then pass the decoded values directly to `setUrlParams`:

```ts
import { Schema } from 'effect'

const BookingFilters = Schema.Struct({
  status: Schema.optional(Schema.Literal('pending', 'confirmed', 'cancelled', 'in-progress', 'completed')),
  search: Schema.optional(Schema.String),
  dateFrom: Schema.optional(Schema.String),
  dateTo: Schema.optional(Schema.String),
  page: Schema.optional(Schema.Number).pipe(Schema.withDefault(() => 1)),
  pageSize: Schema.optional(Schema.Number).pipe(Schema.withDefault(() => 20)),
})
type BookingFilters = Schema.Schema.Type<typeof BookingFilters>

// In the service:
list: (filters: unknown) =>
  Effect.gen(function* () {
    // Validate input — fails with ParseError if filters are invalid
    const validated = yield* Schema.decodeUnknown(BookingFilters)(filters)

    // Pass directly — numbers are coerced, undefined is stripped
    const req = HttpClientRequest.get('/bookings').pipe(
      HttpClientRequest.setUrlParams(validated),
    )
    const res = yield* client.execute(req)
    return yield* HttpClientResponse.schemaBodyJson(BookingListResponse)(res)
  }),
```

This gives you **input validation on both sides**: `Schema.decodeUnknown` validates the filters before the request is built, and `schemaBodyJson` validates the response after it arrives.

---

### Per-Request Headers

When a specific request needs extra headers (not set globally on `ApiHttpClient`):

```ts
import { HttpClientRequest } from '@effect/platform'

// Add a custom header to a single request
getReport: (bookingId, format) =>
  HttpClientRequest.get(`/bookings/${bookingId}/report`).pipe(
    HttpClientRequest.setHeader('Accept', format === 'pdf' ? 'application/pdf' : 'text/csv'),
    Effect.succeed,
    Effect.flatMap(client.execute),
    // ... handle response
  ),
```

---

### What `filterStatusOk` Does

`HttpClient.filterStatusOk` is applied at the `ApiHttpClient` layer. It converts any response with a status outside 2xx into a `ResponseError` in the Effect error channel — automatically, before any feature service sees the response. This means:

- A `404` from `GET /bookings/xyz` becomes a `ResponseError`, not a decoded response.
- Feature services map that `ResponseError` to a domain error (e.g., `BookingNotFound`) using `Effect.mapError`.
- Feature service methods never need to inspect the HTTP status themselves.

---

## 4. Schema Validation on Responses

`HttpClientResponse.schemaBodyJson(Schema)` reads the response body as JSON, parses it, and validates it against the given `Schema`. If the body is missing, malformed JSON, or does not match the schema, the effect fails with a `ResponseError` or `ParseError`.

### Defining Response Schemas

**`src/features/booking/contract/schemas.ts`**

```ts
import { Schema } from "effect";

// Branded ID type
export const BookingId = Schema.String.pipe(Schema.brand("BookingId"));
export type BookingId = Schema.Schema.Type<typeof BookingId>;

// Single booking response
export const BookingResponse = Schema.Struct({
  id: BookingId,
  status: Schema.Literal("pending", "confirmed", "cancelled"),
  pickupDate: Schema.DateFromString, // parses ISO string → Date
  dropoffDate: Schema.DateFromString,
  driverId: Schema.NullOr(Schema.String),
  vehicleId: Schema.NullOr(Schema.String),
  origin: Schema.String,
  destination: Schema.String,
});

// List response — array of bookings
export const BookingListResponse = Schema.Array(BookingResponse);

// POST request body
export const BookingRequest = Schema.Struct({
  status: Schema.Literal("pending", "confirmed", "cancelled"),
  pickupDate: Schema.String,
  dropoffDate: Schema.String,
  origin: Schema.String,
  destination: Schema.String,
});

// PUT request body — full replacement (same shape, no id)
export const BookingUpdateRequest = BookingRequest;

// PATCH request body — partial
export const BookingPatchRequest = Schema.partial(BookingRequest);
```

### Using schemaBodyJson in a Service

```ts
import { HttpClientResponse } from '@effect/platform'
import { Effect } from 'effect'
import { BookingResponse } from '../contract/schemas'
import { BookingNotFound } from '../contract/errors'

// Inside BookingApiLive:
getById: (id) =>
  client.get(`/bookings/${id}`).pipe(
    // Decodes and validates — fails with ParseError if shape is wrong
    Effect.flatMap(HttpClientResponse.schemaBodyJson(BookingResponse)),
    // Map any error (ResponseError or ParseError) to a domain error
    Effect.mapError(() => new BookingNotFound({ bookingId: id })),
  ),
```

`Schema.DateFromString` is particularly useful — the backend sends ISO 8601 strings and `schemaBodyJson` automatically converts them to `Date` objects as part of validation. No manual `new Date(...)` calls in application code.

---

## 5. Retries and Error Handling

### Transient Retry at the Client Level

`HttpClient.retryTransient` is applied on `ApiHttpClient` so **all** feature API services get automatic retries on transient failures. It retries on:

- **Network failures** — `RequestError` with `reason: "Transport"` (DNS, connection refused, etc.)
- **Timeouts** — `TimeoutException`
- **Server errors** — any response with `status >= 429` (429 Too Many Requests, 500, 502, 503, 504, etc.)

```ts
// Applied in ApiHttpClientLive (see section 1):
HttpClient.retryTransient({
  times: 3, // max 3 retries
  schedule: Schedule.exponential("200 millis"), // 200ms → 400ms → 800ms
});
```

The `mode` option controls what gets retried:

| Mode               | Retries errors | Retries 429/5xx responses |
| ------------------ | -------------- | ------------------------- |
| `"both"` (default) | Yes            | Yes                       |
| `"errors-only"`    | Yes            | No                        |
| `"response-only"`  | No             | Yes                       |

### Per-Method Retry with `HttpClient.retry`

For retry logic that applies to a specific request (not all requests), use `HttpClient.retry` or `Effect.retry` directly:

```ts
// Retry only this specific call, with a custom schedule
getById: (id) =>
  client.get(`/bookings/${id}`).pipe(
    Effect.retry({
      times: 2,
      schedule: Schedule.exponential('500 millis'),
    }),
    Effect.flatMap(HttpClientResponse.schemaBodyJson(BookingResponse)),
    Effect.mapError(() => new BookingNotFound({ bookingId: id })),
  ),
```

### Domain Errors with Schema.TaggedError

All domain errors extend `Schema.TaggedError` — never use plain `new Error(...)`. Tagged errors are:

- **Type-safe** — TypeScript tracks them in the error channel
- **Serializable** — Effect Schema can encode/decode them
- **Pattern-matchable** — use `._tag` for exhaustive matching

```ts
// src/features/booking/contract/errors.ts
import { Schema } from "effect";

export class BookingNotFound extends Schema.TaggedError<BookingNotFound>()(
  "BookingNotFound",
  { bookingId: Schema.String },
) {}

export class BookingConflict extends Schema.TaggedError<BookingConflict>()(
  "BookingConflict",
  {
    bookingId: Schema.String,
    message: Schema.String,
  },
) {}

export class BookingValidationError extends Schema.TaggedError<BookingValidationError>()(
  "BookingValidationError",
  {
    field: Schema.String,
    message: Schema.String,
  },
) {}

export class ApiRequestFailed extends Schema.TaggedError<ApiRequestFailed>()(
  "ApiRequestFailed",
  {
    method: Schema.String,
    path: Schema.String,
    status: Schema.Number,
    message: Schema.String,
  },
) {}
```

### Mapping HTTP Errors to Domain Errors

Feature services map the generic `ResponseError` (from `filterStatusOk`) to domain-specific tagged errors using `Effect.mapError`. Use `Schema.is` to check the error shape cleanly:

```ts
import { Schema } from 'effect'

// Define once, reuse across all service methods
const ResponseError = Schema.Struct({
  _tag: Schema.Literal('ResponseError'),
  response: Schema.Struct({ status: Schema.Number }),
  message: Schema.String,
})
const isResponseError = Schema.is(ResponseError)

getById: (id) =>
  client.get(`/bookings/${id}`).pipe(
    Effect.flatMap(HttpClientResponse.schemaBodyJson(BookingResponse)),
    Effect.mapError((error) => {
      if (isResponseError(error)) {
        if (error.response.status === 404) {
          return new BookingNotFound({ bookingId: id })
        }
        if (error.response.status === 409) {
          return new BookingConflict({ bookingId: id, message: 'Booking already exists' })
        }
        return new ApiRequestFailed({
          method: 'GET',
          path: `/bookings/${id}`,
          status: error.response.status,
          message: error.message,
        })
      }
      // ParseError or other — wrap as generic
      return new ApiRequestFailed({
        method: 'GET',
        path: `/bookings/${id}`,
        status: 0,
        message: String(error),
      })
    }),
  ),
```

### Error Handling in Components

Tagged errors flow through atoms and can be pattern-matched in React:

```tsx
function BookingDetail() {
  const result = useAtomValue(Booking.detail);

  if (result._tag === "Failure") {
    const error = Cause.failureOption(result.cause);
    if (Option.isSome(error)) {
      switch (error.value._tag) {
        case "BookingNotFound":
          return (
            <NotFound message={`Booking ${error.value.bookingId} not found`} />
          );
        case "BookingConflict":
          return <Alert variant="warning">{error.value.message}</Alert>;
        case "ApiRequestFailed":
          return (
            <ErrorDisplay
              status={error.value.status}
              message={error.value.message}
            />
          );
      }
    }
    return <ErrorDisplay message="An unexpected error occurred" />;
  }

  // ... render success
}
```

---

## 6. Full Layer Graph

The dependency graph flows from platform primitives up through shared infrastructure to feature services, and finally into `AppApiLayer` which is provided to `Atom.runtime`.

```
BrowserHttpClient.layer          (platform fetch impl — no dependencies)
       │
       ▼
AuthServiceLive                  (token management — reads BrowserHttpClient for token refresh)
       │
       ▼
ApiHttpClientLive                (base URL + auth header + filterStatusOk)
       │
       ├──────────────────────────────────────┐
       ▼                                      ▼
BookingApiLive                        DriverApiLive
       │                                      │
       └──────────────┬───────────────────────┘
                      ▼
               AppApiLayer               (Layer.mergeAll — provided to Atom.runtime)
```

**`src/lib/runtime.ts`**

```ts
import { Layer } from "effect";
import { ApiHttpClientLive } from "~/lib/http-client";
import { AuthServiceLive } from "~/services/auth.service";
import { BookingApiLive } from "~/features/booking/api/booking.api";
import { DriverApiLive } from "~/features/driver/api/driver.api";

// Merge all feature API layers
const AppApiLayer = Layer.mergeAll(
  BookingApiLive,
  DriverApiLive,
  // add more feature API layers here
);

// Full application layer — ApiHttpClientLive is shared across all feature services
export const AppLayer = AppApiLayer.pipe(
  Layer.provide(ApiHttpClientLive),
  Layer.provide(AuthServiceLive),
);
```

Each feature API layer (e.g., `BookingApiLive`) declares `ApiHttpClient` as a dependency. `Layer.provide(ApiHttpClientLive)` satisfies that dependency for all of them at once — there is a single shared `ApiHttpClient` instance.

---

## Putting It Together: End-to-End Request Flow

1. An atom calls `BookingApi.getAll` inside an `Atom.fn` or `Atom.readable` computation.
2. `BookingApi.getAll` yields `ApiHttpClient` from context and calls `client.get('/bookings')`.
3. `ApiHttpClient` prepends `VITE_API_URL`, sets `Accept: application/json`, and attaches the bearer token (via `mapRequestEffect` + `AuthService`).
4. The request is executed. If it fails with a transient error (network, timeout, 429/5xx), `retryTransient` retries up to 3 times with exponential backoff.
5. `filterStatusOk` rejects non-2xx responses as `ResponseError`.
6. `HttpClientResponse.schemaBodyJson(BookingListResponse)` decodes and validates the JSON body.
7. If any step fails, `mapError` converts it to a domain-specific `Schema.TaggedError` (e.g., `BookingNotFound`, `ApiRequestFailed`).
8. The decoded `BookingResponse[]` value flows back to the atom, which stores it in state.

All of this is fully typed through the Effect type system. TypeScript knows the success type, the error type, and the required services at every step.

---

## File Locations Reference

| File                                       | Purpose                                               |
| ------------------------------------------ | ----------------------------------------------------- |
| `src/lib/http-client.ts`                   | `ApiHttpClient` tag + `ApiHttpClientLive` layer       |
| `src/lib/runtime.ts`                       | `AppLayer` composition                                |
| `src/services/auth.service.ts`             | `AuthService` — provides `getToken`                   |
| `src/features/booking/api/booking.api.ts`  | `BookingApi` tag + `BookingApiLive` layer             |
| `src/features/booking/contract/schemas.ts` | Effect Schema definitions for request/response shapes |
| `src/features/booking/contract/errors.ts`  | Domain errors (`BookingNotFound`, etc.)               |
