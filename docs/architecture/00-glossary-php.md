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
