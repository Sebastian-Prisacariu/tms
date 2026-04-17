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

## The Effect Type Up Close

`Effect.Effect<A, E, R>` has three type parameters:

| Position | Meaning | Example |
|---|---|---|
| `A` | Value on **success** | `Booking`, `Array<Route>`, `void` |
| `E` | Value on **typed failure** | `BookingNotFound`, `NetworkError` |
| `R` | **Requirements** — services that must be provided | `BookingApi`, `HttpClient` |

```ts
Effect.Effect<Booking, BookingNotFound, BookingApi>
// "Returns a Booking, or fails with BookingNotFound, and needs BookingApi in context"
```

A `Promise<T>` only knows its success type — errors are untyped and thrown. Effect tracks all three: success, the exact error set, and required services. The third slot is how dependency injection gets compiler-checked: an Effect that requires `BookingApi` won't run until a `Layer` supplies one.

You'll often see `Effect.Effect<A, E>` (two params) — that's shorthand for `R = never`, meaning "no requirements left, ready to run."

---

## Other Core Types You'll See

### `Option<A>` — a value that may not be there

Effect's replacement for `T | null` / `T | undefined`. Either `Some(value)` or `None` — no quiet nullish slips.

```ts
import { Option } from 'effect'

const maybeBooking: Option.Option<Booking> = Option.some(booking)
const empty: Option.Option<Booking> = Option.none()

// Safe unwrap — compiler forces you to handle the None case
Option.match(maybeBooking, {
  onSome: (b) => `Got ${b.id}`,
  onNone: () => 'No booking',
})

// Or unwrap with a fallback
Option.getOrElse(maybeBooking, () => defaultBooking)
```

**React analogy:** like a typed `undefined | T` where the compiler refuses to let you read properties before you check. Replaces ad-hoc `if (x)` guards with a value you can pipe through.

### `Either<E, A>` — synchronous success-or-failure

Effect's synchronous, dependency-free cousin: `Right<A>` (success) or `Left<E>` (failure). No async, no services.

```ts
import { Either } from 'effect'

const ok: Either.Either<number, string> = Either.right(42)
const bad: Either.Either<number, string> = Either.left('oops')

Either.match(bad, {
  onLeft: (e) => `Failed: ${e}`,
  onRight: (n) => `Got: ${n}`,
})
```

**When to use which:**
- `Option<A>` — "is there a value?" (lookup, find, maybe-present)
- `Either<E, A>` — "did this synchronous thing fail?" (parse, validate, pure computation)
- `Effect<A, E, R>` — anything async, or anything that needs services

Rule of thumb: reach for `Option` / `Either` in pure helpers; reach for `Effect` the moment you touch I/O or dependency injection.

### `Data.TaggedEnum` — discriminated unions without boilerplate

```ts
import { Data } from 'effect'

type DraftState = Data.TaggedEnum<{
  Local: { readonly data: DraftData }
  Persisted: { readonly id: string; readonly data: DraftData }
}>
const DraftState = Data.taggedEnum<DraftState>()

// Constructors are free
const s1 = DraftState.Local({ data: empty })
const s2 = DraftState.Persisted({ id: 'abc', data: empty })

// Exhaustive, compiler-checked match
DraftState.$match(s1, {
  Local: () => 'not saved yet',
  Persisted: (p) => `saved as ${p.id}`,
})
```

**React analogy:** like a `useReducer` action union (`{ type: 'Local', ... } | { type: 'Persisted', ... }`), but you get free constructors, free type guards, and exhaustive matchers.

### `Cause<E>` and `Exit<A, E>` — the full failure picture

Every Effect ends with an `Exit<A, E>`:
- `Exit.Success(a)` — normal path
- `Exit.Failure(cause)` — something went wrong; `cause` is a `Cause<E>`

A `Cause<E>` distinguishes:
- **Expected failures** (`Fail<E>`) — your tagged errors, what callers handle
- **Defects** — unexpected `throw`s, real bugs
- **Interruptions** — a fiber was cancelled

You rarely inspect `Cause` by hand — `Effect.catchTag`, `Effect.catchAll`, and `Effect.catchAllCause` do it for you. But the split is why Effect treats "expected error path" and "something exploded" as genuinely different things.

---

## Key Combinators — One-Line Reference

| Operator | What it does | Promise equivalent |
|---|---|---|
| `Effect.succeed(a)` | Lift a value into Effect | `Promise.resolve(a)` |
| `Effect.fail(e)` | Produce a typed failure | `Promise.reject(e)` (but typed) |
| `Effect.sync(() => x)` | Wrap a synchronous side-effect | `new Promise(r => r(x))` |
| `Effect.tryPromise(...)` | Bring a Promise into Effect | — |
| `Effect.map(eff, fn)` | Transform the success value | `.then(fn)` (sync fn) |
| `Effect.flatMap(eff, fn)` | Chain another Effect | `.then(fn)` (fn returns Promise) |
| `Effect.all([a, b, c])` | Run in parallel, collect results | `Promise.all([a, b, c])` |
| `Effect.catchTag('Tag', fn)` | Recover from one tagged error | `catch (e) { if (e._tag === 'Tag') ... }` |
| `Effect.catchAll(fn)` | Recover from any typed error | `.catch(fn)` |
| `Effect.mapError(fn)` | Transform the error channel | — |
| `Effect.tap(fn)` | Run a side-effect, keep the value | `.then(v => { log(v); return v })` |
| `Effect.orElse(() => fallback)` | Fall back on failure | `.catch(() => fallback)` |

Inside `.pipe(...)`, these stack. A long pipeline is just these operators composed top-to-bottom.

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
