# 00 — Effect + React for PHP Developers

> A quick-reference glossary that maps Effect and React patterns to PHP concepts you already know.

Read this page first if you're coming from a PHP background (Laravel, Symfony, or plain PHP). The rest of the architecture docs assume familiarity with these patterns.

---

## Core Concepts

| Effect / React pattern | PHP equivalent |
|---|---|
| `Effect.Effect<A, E>` | A method return type like `Booking` that can throw a typed exception — but tracked at compile time, not at runtime |
| `Effect.gen(function* () { ... })` | A controller action or service method body — sequential steps that can fail |
| `yield* BookingApi` | `$this->bookingApi` — resolving a dependency from the container |
| `Context.Tag` | An `interface` — e.g., `interface BookingRepositoryInterface` |
| `Layer` | A service provider binding — like `$this->app->bind(BookingRepositoryInterface::class, EloquentBookingRepository::class)` in Laravel, or a Symfony service definition in `services.yaml` |
| `Layer.provide` | Container binding — wiring an implementation to an interface |
| `Schema` | Form Request validation (Laravel) or Symfony's Validator constraints — defines shapes and validates data |
| `Schema.TaggedError` | A custom exception class like `BookingNotFoundException extends Exception` — but the type system tracks which methods can throw it |
| `.pipe(fn1, fn2, fn3)` | Method chaining like `$query->where(...)->orderBy(...)->get()` — transformations flow left to right |
| Atom | A repository or cache that components can subscribe to — like a reactive `Cache::get()` that auto-updates the UI |

---

## Reading the Syntax

### Dependency Injection — PHP vs Effect

```php
// PHP (Laravel) — dependency injection via constructor
class BookingController
{
    public function __construct(
        private BookingRepositoryInterface $bookingRepo,
        private HttpClientInterface $httpClient,
    ) {}

    public function show(string $id): JsonResponse
    {
        $booking = $this->bookingRepo->findOrFail($id);
        return response()->json($booking);
    }
}

// Service provider binding
$this->app->bind(BookingRepositoryInterface::class, EloquentBookingRepository::class);
```

```ts
// Effect — dependency injection via Context.Tag + Layer

// 1. Interface (like BookingRepositoryInterface)
class BookingApi extends Context.Tag('BookingApi')<
  BookingApi,
  { readonly getById: (id: string) => Effect.Effect<Booking, BookingNotFound> }
>() {}

// 2. Implementation (like EloquentBookingRepository)
const BookingApiLive = Layer.effect(
  BookingApi,
  Effect.gen(function* () {
    const client = yield* ApiHttpClient  // ← resolve dependency (like $this->httpClient)
    return {
      getById: (id) => client.get(`/bookings/${id}`).pipe(/* ... */),
    }
  }),
)

// 3. Binding (like the service provider)
const AppLayer = BookingApiLive.pipe(Layer.provide(ApiHttpClientLive))

// 4. Usage (like the controller action)
const program = Effect.gen(function* () {
  const api = yield* BookingApi          // ← $this->bookingRepo
  const booking = yield* api.getById(id) // ← $this->bookingRepo->findOrFail($id)
  return booking
})
```

### Error Handling — PHP vs Effect

```php
// PHP — exceptions
class BookingNotFoundException extends Exception
{
    public function __construct(public readonly string $bookingId) {
        parent::__construct("Booking {$bookingId} not found");
    }
}

// Throwing
throw new BookingNotFoundException($id);

// Catching
try {
    $booking = $this->bookingRepo->findOrFail($id);
} catch (BookingNotFoundException $e) {
    return response()->json(['error' => 'Not found'], 404);
}
```

```ts
// Effect — tagged errors (tracked in the type system)
class BookingNotFound extends Schema.TaggedError<BookingNotFound>()(
  'BookingNotFound',
  { bookingId: Schema.String },
) {}

// "Throwing" — return a failed Effect
Effect.fail(new BookingNotFound({ bookingId: id }))

// "Catching" — pattern match on the tag
api.getById(id).pipe(
  Effect.catchTag('BookingNotFound', (e) =>
    Effect.succeed(null) // handle it
  ),
)
```

**Key difference:** In PHP, any method can throw any exception and the caller might not know. In Effect, the type system tracks exactly which errors a function can produce — `Effect.Effect<Booking, BookingNotFound>` means "returns Booking or fails with BookingNotFound." The compiler ensures you handle it.

### Validation — PHP vs Effect

```php
// PHP (Laravel) — Form Request
class CreateBookingRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'driverId' => 'required|string',
            'pickupAddress' => 'required|string',
            'pickupDate' => 'required|date',
            'status' => 'required|in:pending,confirmed,cancelled',
        ];
    }
}
```

```ts
// Effect — Schema
const CreateBookingInput = Schema.Struct({
  driverId: Schema.String,
  pickupAddress: Schema.String,
  pickupDate: Schema.String,
  status: Schema.Literal('pending', 'confirmed', 'cancelled'),
})

// Schemas also derive TypeScript types (like a DTO class):
type CreateBookingInput = Schema.Schema.Type<typeof CreateBookingInput>
```

**Key difference:** Effect schemas serve triple duty — validation, type generation, and JSON encoding/decoding. A single schema replaces your Form Request, DTO class, and API Resource.

---

## The Effect Type Up Close

`Effect.Effect<A, E, R>` has three type parameters:

| Position | Meaning | PHP analogy |
|---|---|---|
| `A` | Value on **success** | The method's return type (`Booking`, `Collection<Route>`, `void`) |
| `E` | Value on **typed failure** | The exception class(es) the method can throw — but tracked by the compiler |
| `R` | **Requirements** — services that must be provided | Constructor-injected dependencies, but checked at compile time |

```ts
Effect.Effect<Booking, BookingNotFound, BookingApi>
// "Returns a Booking, or fails with BookingNotFound, and needs BookingApi in context"
```

Imagine if PHP let you write `public function show(string $id): Booking throws BookingNotFoundException requires BookingRepository` and the compiler enforced every part of that signature. That's the Effect type.

You'll often see `Effect.Effect<A, E>` (two params) — that's shorthand for `R = never`, meaning "no requirements left, ready to run."

---

## Other Core Types You'll See

### `Option<A>` — a value that may not be there

`Option<A>` replaces `?Booking` / `Booking|null`. Either `Some(value)` or `None`, and the compiler forces you to handle both.

```ts
import { Option } from 'effect'

const maybe: Option.Option<Booking> = Option.some(booking)
const empty: Option.Option<Booking> = Option.none()

Option.match(maybe, {
  onSome: (b) => `Got ${b.id}`,
  onNone: () => 'No booking',
})

Option.getOrElse(maybe, () => defaultBooking)
```

**PHP analogy:** like a strict `?Booking` where calling `$x->id` without a `!== null` check is a compile error, not a runtime `TypeError`. Roughly what `Option` in `hamcrest/hamcrest-php` or `prewk/option` provide, built into the core language of Effect.

### `Either<E, A>` — synchronous success-or-failure

`Either<E, A>` is Effect's synchronous cousin: `Right<A>` (success) or `Left<E>` (failure). No async, no services — just a value-returning alternative to exceptions for pure code.

```ts
import { Either } from 'effect'

const ok: Either.Either<number, string> = Either.right(42)
const bad: Either.Either<number, string> = Either.left('oops')

Either.match(bad, {
  onLeft: (e) => `Failed: ${e}`,
  onRight: (n) => `Got: ${n}`,
})
```

**PHP analogy:** like returning a Result class hierarchy — `abstract class Result { class Success; class Failure; }` — and requiring the caller to pattern match on it instead of catching exceptions. Useful when failure is an ordinary outcome (validation, parsing), not an exceptional event.

**When to use which:**
- `Option<A>` — "is there a value?" (finder methods, nullable lookups)
- `Either<E, A>` — "did this synchronous thing fail?" (parsing, pure validation)
- `Effect<A, E, R>` — anything async, or anything that needs services

Rule of thumb: `Option` / `Either` for pure helpers; `Effect` the moment you touch I/O, HTTP, or the service container.

### `Data.TaggedEnum` — discriminated unions without boilerplate

```ts
import { Data } from 'effect'

type DraftState = Data.TaggedEnum<{
  Local: { readonly data: DraftData }
  Persisted: { readonly id: string; readonly data: DraftData }
}>
const DraftState = Data.taggedEnum<DraftState>()

const s1 = DraftState.Local({ data: empty })
const s2 = DraftState.Persisted({ id: 'abc', data: empty })

// Exhaustive, compiler-checked match
DraftState.$match(s1, {
  Local: () => 'not saved yet',
  Persisted: (p) => `saved as ${p.id}`,
})
```

**PHP analogy:** like an abstract class with `Local` and `Persisted` sealed subclasses plus PHP 8.1 `match` for exhaustiveness — except you get the constructors, the type guards, and the matcher generated for you. Conceptually similar to Rust-style enums or `symfony/console`'s sealed command-result types.

### `Cause<E>` and `Exit<A, E>` — the full failure picture

Every Effect ends with an `Exit<A, E>`:
- `Exit.Success(a)` — normal path
- `Exit.Failure(cause)` — something went wrong; `cause` is a `Cause<E>`

A `Cause<E>` distinguishes:
- **Expected failures** (`Fail<E>`) — your tagged errors, what callers handle (like catching `BookingNotFoundException`)
- **Defects** — unexpected `throw`s, real bugs (like an uncaught `TypeError` or `Error`)
- **Interruptions** — a fiber was cancelled (closest PHP analogy: a request timing out mid-handler)

You rarely inspect `Cause` by hand — `Effect.catchTag`, `Effect.catchAll`, and `Effect.catchAllCause` do it for you. But the split is why Effect treats "expected business failure" and "something exploded" as genuinely different things, instead of both being `\Throwable`.

---

## Key Combinators — One-Line Reference

| Operator | What it does | PHP/Laravel analogy |
|---|---|---|
| `Effect.succeed(a)` | Lift a value into Effect | `return $value` |
| `Effect.fail(e)` | Produce a typed failure | `throw new DomainException(...)` — but typed |
| `Effect.sync(() => x)` | Wrap a synchronous side-effect | Any pure statement you want lazily evaluated |
| `Effect.tryPromise(...)` | Bring a Promise into Effect | Wrapping an external async SDK call |
| `Effect.map(eff, fn)` | Transform the success value | `->map(fn)` on a Laravel Collection |
| `Effect.flatMap(eff, fn)` | Chain another Effect | `->then($next)` or nested `await` |
| `Effect.all([a, b, c])` | Run in parallel, collect results | Like `Http::pool(...)` in Laravel |
| `Effect.catchTag('Tag', fn)` | Recover from one tagged error | `catch (BookingNotFoundException $e) { ... }` |
| `Effect.catchAll(fn)` | Recover from any typed error | `catch (\Throwable $e) { ... }` (but typed) |
| `Effect.mapError(fn)` | Transform the error channel | Rewrapping an exception before re-throwing |
| `Effect.tap(fn)` | Run a side-effect, keep the value | `->tap($fn)` on a Laravel Collection |
| `Effect.orElse(() => fallback)` | Fall back on failure | `try { ... } catch { return $fallback; }` |

Inside `.pipe(...)`, these stack. A long pipeline is just these operators composed top-to-bottom — conceptually very similar to chaining Laravel Collection or Query Builder methods.

---

## Architecture Mapping

| PHP concept | TMS equivalent | File location |
|---|---|---|
| Controller | Route file (thin) + Feature component | `src/routes/`, `src/features/*/components/` |
| Service / Repository | Effect service (`Context.Tag` + `Layer`) | `src/features/*/booking.api.ts` |
| Model / Entity | Effect Schema | `src/features/*/contract/schemas.ts` |
| Custom Exception | `Schema.TaggedError` | `src/features/*/contract/errors.ts` |
| Service Provider | Layer composition in `runtime.ts` | `src/lib/runtime.ts` |
| Form Request | Effect Schema (same as model) | `src/features/*/contract/schemas.ts` |
| Middleware | TanStack Router `beforeLoad` guard | `src/routes/_authenticated.tsx` |
| `config/app.php` | `Config.string('VITE_API_URL')` | Read via Effect's `Config` module |
| Cache/Session | Atoms (`Atom.make`, `Atom.kvs`) | `src/features/*/booking.atoms.ts` |

---

## The Request Lifecycle — PHP vs TMS

```
PHP:
  Browser → Route → Middleware → Controller → Service → Repository → DB
                                                                    ↓
  Browser ← JSON/View ← Controller ← Service ← Repository ← Query Result

TMS:
  Browser → Route file → Auth guard → Feature component → Hook → Atom
                                                                   ↓
                                                            Effect service (BookingApi)
                                                                   ↓
                                                            ApiHttpClient → Backend API
                                                                   ↓
  Browser ← React re-render ← Atom update ← Schema-validated response
```

The main mental shift: in PHP, the server does everything and sends HTML/JSON. In this TMS frontend, the browser runs the app and calls the backend API via Effect services. The data flows through atoms (reactive state) which automatically update the UI.

---

## See Also

- [00-effect-glossary.md](./00-effect-glossary.md) — general Effect glossary (framework-agnostic)
- [02-effect-services.md](./02-effect-services.md) — full service pattern with `Context.Tag`, `Layer`, and tagged errors
- [05-feature-anatomy.md](./05-feature-anatomy.md) — complete walkthrough of a feature from schema to component
