# 00 — Effect for React Developers

> A quick-reference glossary that maps Effect patterns to concepts you already know.

Read this page first if you're new to Effect. The rest of the architecture docs assume familiarity with these patterns.

---

## Core Concepts

| Effect pattern | Think of it as... |
|---|---|
| `Effect.Effect<A, E>` | A `Promise<A>` that can fail with typed error `E` (instead of untyped `throw`) |
| `Effect.gen(function* () { ... })` | An `async function() { ... }` — generators instead of promises |
| `yield* something` | `await something` — unwraps the Effect and gives you the value |
| `.pipe(fn1, fn2, fn3)` | Method chaining, like `.then(fn1).then(fn2)` — transformations flow left to right |
| `Context.Tag` | A service interface — declares *what* a service does, without *how* |
| `Layer` | A provider that supplies the implementation — like Angular's `providers: []` or React's Context value |
| `Schema` | A Zod-like validation library built into Effect — defines shapes and validates data |
| `Schema.TaggedError` | A typed error class with a `_tag` field for pattern matching (like a discriminated union) |

---

## Reading the Syntax

### `Context.Tag` — defining a service interface

```ts
class BookingApi extends Context.Tag('BookingApi')<
  BookingApi,      // ← the unique key (the class itself)
  {                // ← the shape: what callers can do
    readonly getById: (id: string) => Effect.Effect<Booking, BookingNotFound>
  }
>() {}
```

**Why the unusual syntax?** `Context.Tag` uses a class pattern so TypeScript can use the class as both a type and a runtime value. The string `'BookingApi'` is a unique identifier (like a React context's display name). You'll write this boilerplate once per service and then never think about it again.

### `Layer.effect` — providing an implementation

```ts
const BookingApiLive = Layer.effect(
  BookingApi,                           // ← which service this implements
  Effect.gen(function* () {
    const client = yield* ApiHttpClient // ← request a dependency (like useContext)
    return {
      getById: (id) => client.get(`/bookings/${id}`).pipe(/* ... */),
    }
  }),
)
```

**Mental model:** A Layer is a factory function that receives its dependencies via `yield*` and returns the service implementation. Think of it as a React context provider that can depend on other providers.

### `Effect.gen` — the async/await equivalent

```ts
// JavaScript async/await:
async function getBooking(id: string) {
  const response = await fetch(`/bookings/${id}`)
  const data = await response.json()
  return data
}

// Effect equivalent:
const getBooking = (id: string) =>
  Effect.gen(function* () {
    const api = yield* BookingApi           // "await" the service
    const booking = yield* api.getById(id)  // "await" the call
    return booking
  })
```

The `function*` / `yield*` syntax is how Effect tracks dependencies and errors at the type level. You read it exactly like `async` / `await`.

### `.pipe()` — chaining transformations

```ts
// Promise style:
fetch('/bookings')
  .then(res => res.json())
  .then(data => validate(data))
  .catch(err => handleError(err))

// Effect pipe style:
client.get('/bookings').pipe(
  Effect.flatMap(res => res.json),       // .then()
  Effect.flatMap(Schema.decode(Booking)), // .then()
  Effect.mapError(toBookingNotFound),     // .catch()
)
```

`Effect.flatMap` is `.then()`. `Effect.mapError` is `.catch()`. The pipe reads top-to-bottom, just like a promise chain.

### `Schema.TaggedError` — typed errors

```ts
class BookingNotFound extends Schema.TaggedError<BookingNotFound>()(
  'BookingNotFound',         // ← the _tag value (used for pattern matching)
  { bookingId: Schema.String }, // ← the error's data
) {}

// Handling:
Effect.catchTag('BookingNotFound', (e) => {
  console.log(`Booking ${e.bookingId} not found`)
  return Effect.succeed(null)
})
```

**Why not just `throw new Error()`?** Tagged errors are tracked in the type system. TypeScript tells you exactly which errors a function can produce, and the compiler ensures you handle them.

---

## The Dependency Injection Flow

```
1. Define interface    →  class BookingApi extends Context.Tag(...)
2. Implement it        →  const BookingApiLive = Layer.effect(BookingApi, ...)
3. Wire dependencies   →  AppLayer = BookingApiLive.pipe(Layer.provide(HttpLayer))
4. Use the service     →  const api = yield* BookingApi  (inside Effect.gen)
```

Steps 1-3 happen once. Step 4 happens everywhere you need the service. The Layer graph (step 3) is assembled in `src/lib/runtime.ts` — individual features never wire their own dependencies.

---

## See Also

- [02-effect-services.md](./02-effect-services.md) — full service pattern with `Context.Tag`, `Layer`, and tagged errors
- [03-http-client.md](./03-http-client.md) — HTTP client service built on these patterns
- [04-state-management.md](./04-state-management.md) — how services connect to reactive state via atoms
