# 09 — Overlay Stack (Modals + Drawers)

> Modals and drawers on a single Effect-Atom-backed stack. Typed imperative handles, callable from anywhere, parent state preserved on push.

## Overview

Transport workflows stack overlays — a drawer opens a modal opens a dialog. Per-feature `useState` booleans lose parent state on unmount and can't be opened from outside React.

One stack, five rules:

1. **One render surface** — single `<OverlayRoot />` at the app root.
2. **Push preserves state** — parent stays mounted; forms, scroll, atoms survive.
3. **Typed payloads** — each overlay registers with a Schema, validated at the boundary.
4. **Callable anywhere** — plain function call from React, DOM, Effects, or tests.
5. **URL-restorable** — refresh or share the URL to restore the stack.

## Table of Contents

- [Why a Unified Stack](#why-a-unified-stack)
- [Folder Layout](#folder-layout)
- [Contract: `OverlayEntry`](#contract-overlayentry)
- [The Registry: `registerOverlay`](#the-registry-registeroverlay)
- [The Stack Atom](#the-stack-atom)
- [Imperative Handle — Calling From Outside React](#imperative-handle--calling-from-outside-react)
- [The `<OverlayRoot />` Component](#the-overlayroot--component)
- [URL Sync](#url-sync)
- [State Preservation Mechanics](#state-preservation-mechanics)
- [Example: Drawer Opens Modal, State Preserved](#example-drawer-opens-modal-state-preserved)
- [Example: Open From DOM / Effect / Test](#example-open-from-dom--effect--test)
- [When NOT to Use This](#when-not-to-use-this)
- [See Also](#see-also)

---

## Why a Unified Stack

A common alternative is **per-feature overlays** — every feature manages its own `Dialog open={...}` with local state:

```tsx
// ❌ Per-feature pattern — we avoid this
function BookingList() {
  const [createOpen, setCreateOpen] = useState(false);
  return (
    <>
      <Button onClick={() => setCreateOpen(true)}>New booking</Button>
      <BookingCreateDialog
        open={createOpen}
        onClose={() => setCreateOpen(false)}
      />
    </>
  );
}
```

**What you lose:**

| Concern                          | Unified stack                                     | Per-feature overlays                                             |
| -------------------------------- | ------------------------------------------------- | ---------------------------------------------------------------- |
| Open from keyboard shortcut      | `bookingCreate.open({})` anywhere                 | Requires lifting state + prop-drilling a trigger                 |
| Stack a modal on top of a drawer | Push another entry                                | Nested `Dialog` components fight each other's focus traps        |
| Parent state on push             | Preserved — parent stays mounted                  | Parent may unmount if it was inside the triggering component     |
| Refresh-restore open state       | Automatic via URL sync                            | Each feature writes its own search-param handling                |
| Test "open overlay and assert"   | `bookingCreate.open({}); expect(stack[0].key)...` | Mount the parent, find the trigger, click it, wait for portal... |

The stack is a single ordered array of **overlay entries**. Every entry has a stable id, a registered key (e.g. `'booking-create'`), a kind (`'modal'` or `'drawer'`), and a Schema-validated payload. The top entry is "active" (focusable, backdropped). Everything below stays mounted but inert.

---

## Folder Layout

Per [01-folder-structure.md](./01-folder-structure.md), the overlay system is a normal feature — flat files, barrel-only imports.

```
src/features/overlays/
├── contract/
│   ├── schemas.ts         # OverlayEntry, OverlayKind, OverlaySide, branded ids
│   └── errors.ts          # OverlayNotRegistered, InvalidOverlayPayload
├── overlays.registry.ts   # registerOverlay + module-level factory map
├── overlays.atoms.ts      # stack atom + open/close/closeTo fns
├── overlays.sync.ts       # URL ↔ stack sync atom
├── overlays.hooks.ts      # useOverlays, useIsTop, useOverlayPayload
├── components/
│   ├── overlay-root.tsx   # <OverlayRoot /> — the single renderer
│   └── overlay-layer.tsx  # Renders one entry as modal or drawer
└── index.ts               # Barrel: registerOverlay, OverlayRoot, types
```

**Barrel contents (`index.ts`):**

```ts
export { registerOverlay } from "./overlays.registry";
export { OverlayRoot } from "./components/overlay-root";
export type { OverlayHandle } from "./overlays.registry";
export type { OverlayKind, OverlaySide } from "./contract/schemas";
```

The barrel deliberately **does not** export atoms or hooks — other features get everything they need through `registerOverlay` and `<OverlayRoot />`. See [01-folder-structure.md § Import Rules](./01-folder-structure.md#import-rules).

---

## Contract: `OverlayEntry`

An entry is the on-the-wire shape that lives in the stack atom and serializes to the URL.

```ts
// features/overlays/contract/schemas.ts
import { Schema } from "effect";

export const OverlayKind = Schema.Literal("modal", "drawer");
export type OverlayKind = Schema.Schema.Type<typeof OverlayKind>;

export const OverlaySide = Schema.Literal("left", "right", "top", "bottom");
export type OverlaySide = Schema.Schema.Type<typeof OverlaySide>;

export const OverlayId = Schema.String.pipe(Schema.brand("OverlayId"));
export type OverlayId = Schema.Schema.Type<typeof OverlayId>;

export const OverlayKey = Schema.String.pipe(Schema.brand("OverlayKey"));
export type OverlayKey = Schema.Schema.Type<typeof OverlayKey>;

// Payload shape is opaque at the stack level — the registry resolves the
// concrete schema per-key when decoding. Keeping `Schema.Unknown` here lets
// the stack serialize without knowing every feature's payload up front.
export const OverlayEntry = Schema.Struct({
  id: OverlayId,
  key: OverlayKey,
  kind: OverlayKind,
  side: Schema.optional(OverlaySide),
  payload: Schema.Unknown,
});
export type OverlayEntry = Schema.Schema.Type<typeof OverlayEntry>;

export const OverlayStack = Schema.Array(OverlayEntry);
export type OverlayStack = Schema.Schema.Type<typeof OverlayStack>;
```

```ts
// features/overlays/contract/errors.ts
import { Schema } from "effect";

export class OverlayNotRegistered extends Schema.TaggedError<OverlayNotRegistered>()(
  "OverlayNotRegistered",
  { key: Schema.String },
) {}

export class InvalidOverlayPayload extends Schema.TaggedError<InvalidOverlayPayload>()(
  "InvalidOverlayPayload",
  { key: Schema.String, reason: Schema.String },
) {}
```

**Key points:**

- `OverlayId` is a branded string (`nanoid()` at creation time) — stable across re-renders, which lets React keep layers mounted by `key={entry.id}`.
- `OverlayKey` is the registration identifier (e.g., `'booking-create'`). It's also branded so it can't be confused with arbitrary strings at call sites.
- The payload is `Schema.Unknown` on the stack because the stack is payload-polymorphic. Per-key decoding happens in the registry (see next section).

---

## The Registry: `registerOverlay`

The registry is a **module-level `Map`**, not an atom. Registration is definitional (happens once at import time), not reactive state.

```ts
// features/overlays/overlays.registry.ts
import { Effect, Schema } from "effect";
import type { ComponentType } from "react";
import { appRuntime } from "~/lib/runtime";
import { InvalidOverlayPayload, OverlayNotRegistered } from "./contract/errors";
import {
  OverlayKey,
  type OverlayKind,
  type OverlaySide,
} from "./contract/schemas";
import { Overlays } from "./overlays.atoms";

type OverlayComponentProps<P> = {
  payload: P;
  close: () => void;
  isTop: boolean;
};

type OverlayFactory<P> = {
  key: string;
  kind: OverlayKind;
  side?: OverlaySide;
  payloadSchema: Schema.Schema<P>;
  component: ComponentType<OverlayComponentProps<P>>;
  /** Default true. If false, this overlay is skipped during URL encoding. */
  persistUrl?: boolean;
};

const factories = new Map<string, OverlayFactory<unknown>>();

export function getFactory(key: string): OverlayFactory<unknown> | undefined {
  return factories.get(key);
}

/** Typed handle returned to callers. */
export type OverlayHandle<P> = {
  open: (payload: P) => void;
  close: () => void;
};

export function registerOverlay<P>(
  factory: OverlayFactory<P>,
): OverlayHandle<P> {
  // bug (two features claimed the same key) and we want to fail loud.
  if (factories.has(factory.key)) {
    throw new Error(
      `[overlays] Duplicate registration for key "${factory.key}". ` +
        "Each overlay key must be registered exactly once.",
    );
  }
  factories.set(factory.key, factory as OverlayFactory<unknown>);

  const brandedKey = OverlayKey.make(factory.key);

  return {
    open: (payload) => {
      appRuntime.runPromise(
        Effect.gen(function* () {
          const validated = yield* Schema.decodeUnknown(factory.payloadSchema)(
            payload,
          ).pipe(
            Effect.mapError(
              (e) =>
                new InvalidOverlayPayload({
                  key: factory.key,
                  reason: String(e),
                }),
            ),
          );
          yield* Overlays.open({
            key: brandedKey,
            kind: factory.kind,
            side: factory.side,
            payload: validated,
          });
        }),
      );
    },
    close: () => {
      appRuntime.runPromise(Overlays.closeByKey(brandedKey));
    },
  };
}
```

**Key points:**

- `registerOverlay<P>(...)` returns a typed `OverlayHandle<P>`. The compiler enforces the payload shape at every call site — a rename in the schema breaks every caller at build time.
- Registration is a **side-effectful import** in the feature that owns the overlay. Importing the feature barrel (e.g., `~/features/booking`) registers its overlays by side-effect.
- **Duplicate keys**: throw at registration time — collisions can't ship.
- `appRuntime.runPromise` is how we escape React. `appRuntime` is the `Atom.runtime(AppLayer)` defined in `src/lib/runtime.ts` (see [02-effect-services.md](./02-effect-services.md) for `AppLayer`).
- Payload validation runs **inside** the Effect so a schema decode failure flows as a tagged `InvalidOverlayPayload` error, not a thrown exception.

---

## The Stack Atom

The single source of truth. Every mutation is an `apiRuntime.fn` so it composes with Effect services (logging, analytics, future policies) — see [04-state-management.md § Mutations with runtimeAtom.fn](./04-state-management.md#mutations-with-runtimeatomfn).

```ts
// features/overlays/overlays.atoms.ts
import { Atom, Registry } from "@effect-atom/atom";
import { Effect } from "effect";
import { nanoid } from "nanoid";
import { apiRuntime } from "~/lib/runtime";
import { OverlayNotRegistered } from "./contract/errors";
import {
  type OverlayEntry,
  OverlayId,
  type OverlayKey,
  type OverlayKind,
  type OverlaySide,
  type OverlayStack,
} from "./contract/schemas";
import { getFactory } from "./overlays.registry";

const stack = Atom.make<OverlayStack>([]);

const open = apiRuntime.fn(
  Effect.fnUntraced(function* (input: {
    key: OverlayKey;
    kind: OverlayKind;
    side?: OverlaySide;
    payload: unknown;
  }) {
    if (!getFactory(input.key)) {
      return yield* Effect.fail(new OverlayNotRegistered({ key: input.key }));
    }
    const registry = yield* Registry.AtomRegistry;
    const entry: OverlayEntry = {
      id: OverlayId.make(nanoid()),
      key: input.key,
      kind: input.kind,
      side: input.side,
      payload: input.payload,
    };
    registry.set(stack, [...registry.get(stack), entry]);
    return entry.id;
  }),
);

const closeTop = apiRuntime.fn(
  Effect.fnUntraced(function* () {
    const registry = yield* Registry.AtomRegistry;
    const cur = registry.get(stack);
    if (cur.length === 0) return;
    registry.set(stack, cur.slice(0, -1));
  }),
);

const closeById = apiRuntime.fn(
  Effect.fnUntraced(function* (id: OverlayId) {
    const registry = yield* Registry.AtomRegistry;
    registry.set(
      stack,
      registry.get(stack).filter((e) => e.id !== id),
    );
  }),
);

/** Closes the topmost entry with this key (no-op if not present). */
const closeByKey = apiRuntime.fn(
  Effect.fnUntraced(function* (key: OverlayKey) {
    const registry = yield* Registry.AtomRegistry;
    const cur = registry.get(stack);
    for (let i = cur.length - 1; i >= 0; i--) {
      if (cur[i].key === key) {
        registry.set(stack, [...cur.slice(0, i), ...cur.slice(i + 1)]);
        return;
      }
    }
  }),
);

/** Closes every entry above `id` (exclusive). Leaves `id` as the new top. */
const closeAbove = apiRuntime.fn(
  Effect.fnUntraced(function* (id: OverlayId) {
    const registry = yield* Registry.AtomRegistry;
    const cur = registry.get(stack);
    const idx = cur.findIndex((e) => e.id === id);
    if (idx < 0) return;
    registry.set(stack, cur.slice(0, idx + 1));
  }),
);

const closeAll = apiRuntime.fn(
  Effect.fnUntraced(function* () {
    const registry = yield* Registry.AtomRegistry;
    registry.set(stack, []);
  }),
);

export const Overlays = {
  stack,
  open,
  closeTop,
  closeById,
  closeByKey,
  closeAbove,
  closeAll,
};
```

**Key points:**

- Mutations return the new entry's `OverlayId` where relevant — callers can target a specific instance later (useful for closing a deeply-stacked overlay by id rather than by key).
- `closeByKey` closes the **topmost** matching entry, not all matches. Opening the same key twice is legal (e.g., two separate booking-detail drawers for different IDs).
- `closeAbove(id)` is the primitive for "back out to this layer" — used by breadcrumb navigation and by the URL sync when the URL shrinks.
- The stack is a plain `Atom.make` (not `Atom.kvs`). Persistence comes from URL sync (next section), not from localStorage.

---

## Imperative Handle — Calling From Outside React

The `registerOverlay` return value is the only surface callers touch. It's a plain object with two methods, callable from any context.

```ts
// features/booking/booking.overlays.ts
import { Schema } from "effect";
import { registerOverlay } from "~/features/overlays";
import { BookingCreateDrawer } from "./components/booking-create-drawer";

export const bookingCreate = registerOverlay({
  key: "booking-create",
  kind: "drawer",
  side: "right",
  payloadSchema: Schema.Struct({
    presetDriverId: Schema.optional(Schema.String),
  }),
  component: BookingCreateDrawer,
});
```

Now anywhere in the app:

```ts
bookingCreate.open({ presetDriverId: "d_123" });
bookingCreate.close();
```

**Three consumption contexts:**

### Inside React

```tsx
// components/new-booking-button.tsx
import { bookingCreate } from "~/features/booking";

export function NewBookingButton() {
  return <Button onClick={() => bookingCreate.open({})}>New booking</Button>;
}
```

### Outside React — DOM event listener

```ts
// src/lib/keyboard-shortcuts.ts
import { bookingCreate } from "~/features/booking";

window.addEventListener("keydown", (e) => {
  if (e.key === "n" && e.metaKey) bookingCreate.open({});
});
```

No hook required, no provider context — the handle closes over `appRuntime`, which is a module-level singleton.

### Inside an Effect service

```ts
// Some service method that needs to trigger UI
import { Effect } from "effect";
import { bookingCreate } from "~/features/booking";

const offerBookingForDriver = (driverId: string) =>
  Effect.gen(function* () {
    yield* Effect.sync(() => bookingCreate.open({ presetDriverId: driverId }));
  });
```

The handle is intentionally **sync at the call site**. Inside Effects you wrap with `Effect.sync`. If you need DI-based mocking in tests, define a thin `OverlayService` `Context.Tag` that delegates to the handles — but for most code, calling the handle directly is fine.

---

## The `<OverlayRoot />` Component

Mounted **once** in `src/routes/__root.tsx`. Never inside a route — that would unmount the whole stack on navigation.

```tsx
// src/routes/__root.tsx
import { createRootRoute, Outlet } from "@tanstack/react-router";
import { OverlayRoot } from "~/features/overlays";

export const Route = createRootRoute({
  component: () => (
    <>
      <Outlet />
      <OverlayRoot />
    </>
  ),
});
```

```tsx
// features/overlays/components/overlay-root.tsx
import { useAtomValue } from "@effect-atom/atom-react";
import { Overlays } from "../overlays.atoms";
import { overlayUrlSync } from "../overlays.sync";
import { OverlayLayer } from "./overlay-layer";

export function OverlayRoot() {
  // Mount the URL sync atom — bidirectional subscription
  useAtomValue(overlayUrlSync);

  const stack = useAtomValue(Overlays.stack);
  const topIdx = stack.length - 1;

  return (
    <>
      {stack.map((entry, i) => (
        <OverlayLayer key={entry.id} entry={entry} isTop={i === topIdx} />
      ))}
    </>
  );
}
```

```tsx
// features/overlays/components/overlay-layer.tsx
import { useAtomSet } from "@effect-atom/atom-react";
import { Dialog, DialogContent } from "~/components/ui/dialog";
import { Drawer, DrawerContent, DrawerPortal } from "~/components/ui/drawer";
import type { OverlayEntry } from "../contract/schemas";
import { Overlays } from "../overlays.atoms";
import { getFactory } from "../overlays.registry";

export function OverlayLayer({
  entry,
  isTop,
}: {
  entry: OverlayEntry;
  isTop: boolean;
}) {
  const factory = getFactory(entry.key);
  const closeById = useAtomSet(Overlays.closeById);
  if (!factory) return null;

  const Body = factory.component;
  const close = () => closeById(entry.id);

  // Non-top layers stay mounted (state preserved) but are inert.
  // `inert` removes focus + pointer events; `aria-hidden` hides from AT.
  const inertProps = isTop ? {} : { inert: "" as const, "aria-hidden": true };

  if (entry.kind === "drawer") {
    return (
      <Drawer
        open
        onOpenChange={(o) => !o && close()}
        side={entry.side ?? "right"}
      >
        <DrawerPortal>
          <DrawerContent {...inertProps}>
            <Body
              payload={entry.payload as never}
              close={close}
              isTop={isTop}
            />
          </DrawerContent>
        </DrawerPortal>
      </Drawer>
    );
  }

  return (
    <Dialog open onOpenChange={(o) => !o && close()}>
      <DialogContent {...inertProps}>
        <Body payload={entry.payload as never} close={close} isTop={isTop} />
      </DialogContent>
    </Dialog>
  );
}
```

**Key points:**

- `key={entry.id}` is load-bearing. React uses it to decide whether a layer is "the same" across renders; stable ids mean a push or pop does **not** remount the neighbors.
- `inert=""` on non-top layers keeps focus traps and keyboard interaction on the top only. Focus returns to the new top on close without manual bookkeeping.
- `Body payload={entry.payload as never}` — the cast is the one `as` in the system. It's sound because the factory registered a `Schema.Schema<P>` and `open` decoded through it before the entry was pushed.

---

## URL Sync

Refresh should restore the stack. A user pasting a URL should see the same overlays.

`overlays.sync.ts` is a **reactive sync atom** (pattern from [04-state-management.md § Reactive Sync Atoms](./04-state-management.md#reactive-sync-atoms)) — an invisible atom that subscribes in both directions.

All encoding and decoding flows through **Effect Schema**. The URL codec is a single composed schema — `base64url string ↔ utf-8 JSON string ↔ OverlayStack` — and `Schema.decodeEither` / `Schema.encodeEither` enforce the round-trip.

```ts
// features/overlays/overlays.sync.ts
import { Atom } from "@effect-atom/atom";
import { Either, Schema } from "effect";
import { router } from "~/lib/router";
import { OverlayEntry, OverlayStack } from "./contract/schemas";
import { Overlays } from "./overlays.atoms";
import { getFactory } from "./overlays.registry";

const SEARCH_KEY = "o";
const MAX_URL_BYTES = 2048;

/**
 * Codec: base64url-encoded JSON string  ↔  OverlayStack
 *
 * `Schema.StringFromBase64Url` transforms b64url → utf-8 string.
 * `Schema.parseJson(OverlayStack)` transforms utf-8 JSON → OverlayStack
 * (and validates every entry against the schema in one pass).
 *
 * `Schema.compose` chains them so a single decode/encode does the whole trip.
 */
const UrlStackCodec: Schema.Schema<OverlayStack, string> = Schema.compose(
  Schema.StringFromBase64Url,
  Schema.parseJson(OverlayStack),
);

/** Pre-encode filter: drop entries whose factory opted out of URL persistence. */
const persistable = (stack: OverlayStack): OverlayStack =>
  stack.filter((e) => {
    const f = getFactory(e.key);
    return f !== undefined && f.persistUrl !== false;
  });

/** Post-decode filter: drop entries whose key is no longer registered. */
const knownKeysOnly = (stack: OverlayStack): OverlayStack =>
  stack.filter((e) => getFactory(e.key) !== undefined);

const encode = (stack: OverlayStack): string | undefined => {
  const filtered = persistable(stack);
  if (filtered.length === 0) return undefined;

  const result = Schema.encodeEither(UrlStackCodec)(filtered);
  if (Either.isLeft(result)) {
    console.warn("[overlays] failed to encode stack:", result.left);
    return undefined;
  }
  if (new Blob([result.right]).size > MAX_URL_BYTES) {
    console.warn(
      "[overlays] stack exceeds URL budget (%d bytes); skipping URL persistence. " +
        "Shrink payloads by passing IDs instead of server objects.",
      new Blob([result.right]).size,
    );
    return undefined;
  }
  return result.right;
};

const decode = (raw: string | undefined): OverlayStack => {
  if (!raw) return [];
  return Either.match(Schema.decodeEither(UrlStackCodec)(raw), {
    onLeft: () => [], // malformed / tampered — silently reset
    onRight: knownKeysOnly, // forward-compat: drop unknown keys
  });
};

export const overlayUrlSync = Atom.readable((get) => {
  // URL → stack (on mount + whenever the URL changes)
  const initial = decode(router.state.location.search[SEARCH_KEY]);
  if (initial.length > 0) get.set(Overlays.stack, initial);

  // stack → URL (on every stack change)
  get.subscribe(Overlays.stack, (next) => {
    const encoded = encode(next);
    const current = router.state.location.search[SEARCH_KEY];
    if (encoded === current) return;
    router.navigate({
      to: router.state.location.pathname,
      search: (prev) => ({ ...prev, [SEARCH_KEY]: encoded }),
      replace: true,
    });
  });

  return null;
});
```

**Key points:**

- **One Schema codec for the whole round-trip.** `UrlStackCodec = Schema.compose(Schema.StringFromBase64Url, Schema.parseJson(OverlayStack))` — `Schema.decodeEither` validates the JSON shape and the `OverlayEntry` structure in one pass. There is no hand-rolled parser anywhere in the system.
- `?o=` is the convention — a single search param holds the whole stack. Keeping it in one param means route `validateSearch` schemas elsewhere don't need to know about it.
- `replace: true` — opening an overlay shouldn't pollute the back button with one history entry per push. Back button takes the user to the previous page, not through the overlay history.
- **2KB hard cap.** If the encoded string exceeds the budget we log and fall back to in-memory only. Don't put server data in payloads — put IDs and re-fetch.
- **Forward-compat on decode**: any entry whose key is no longer in the registry is dropped silently by `knownKeysOnly`. Renaming `booking-create` → `booking-new` doesn't crash old shared links.
- **Tamper-safe**: if the URL value isn't valid base64url, isn't valid JSON, or doesn't match `OverlayStack`'s schema, `Schema.decodeEither` returns `Left` and we reset to an empty stack. No partial/corrupt entries ever reach the atom.
- `persistUrl: false` on a factory opts that overlay out of URL encoding (e.g., a confirmation dialog that shouldn't be deep-linkable).

---

## State Preservation Mechanics

Three things together keep parent state alive when a new overlay opens on top.

### 1. Stable keys

`<OverlayLayer key={entry.id} />` — React never remounts a layer just because the stack grew. A drawer at depth 0 stays at depth 0's fiber when depth 1 opens.

### 2. Inert, not unmounted

Non-top layers render with the `inert=""` attribute. The DOM stays, all `useState` / `useRef` / scoped atoms stay, but the layer is hidden from pointer, keyboard, and assistive tech. Modern browsers and BaseUI handle `inert` natively.

### 3. Atoms survive the whole stack

Because overlays are regular React components, any atom they subscribe to follows the normal atom lifecycle. The `defaultIdleTTL = 400` setting on `RegistryProvider` (see [04-state-management.md § Registry Provider Setup](./04-state-management.md#registry-provider-setup)) means even if a layer somehow unsubscribes briefly, the atom value is kept for 400ms.

**What's preserved:**

- Form field values (via component `useState`)
- Scroll position (DOM state)
- Feature-scoped atoms (via `ScopedAtom.Provider` wrapping the overlay body)
- Pending Effect runs (atom mutations continue executing even while peek-behind)

**What's NOT preserved:**

- Focus — focus moves to the new top layer's initial focus target. On close, BaseUI/shadcn restores focus to whatever triggered the top overlay.

---

## Example: Drawer Opens Modal, State Preserved

A driver-assign drawer hosts a half-filled form. The user clicks "Create new driver" which opens a modal on top. On modal submit+close, the drawer's form is still intact.

```tsx
// features/driver/components/driver-assign-drawer.tsx
import { Schema } from "effect";
import { useState } from "react";
import { Button } from "~/components/ui/button";
import { Input } from "~/components/ui/input";
import { registerOverlay } from "~/features/overlays";
import { driverCreate } from "./driver-create-overlay";

type DriverAssignPayload = { bookingId: string };

function DriverAssignDrawerBody({
  payload,
  close,
}: {
  payload: DriverAssignPayload;
  close: () => void;
  isTop: boolean;
}) {
  const [note, setNote] = useState("");

  return (
    <div>
      <h2>Assign driver to booking {payload.bookingId}</h2>
      <Input
        value={note}
        onChange={(e) => setNote(e.target.value)}
        placeholder="Note for driver"
      />
      <Button onClick={() => driverCreate.open({})}>Create new driver</Button>
      <Button onClick={close}>Cancel</Button>
    </div>
  );
}

export const driverAssign = registerOverlay({
  key: "driver-assign",
  kind: "drawer",
  side: "right",
  payloadSchema: Schema.Struct({ bookingId: Schema.String }),
  component: DriverAssignDrawerBody,
});
```

```tsx
// features/driver/components/driver-create-overlay.tsx
import { Schema } from "effect";
import { registerOverlay } from "~/features/overlays";
import { DriverCreateForm } from "./driver-create-form";

export const driverCreate = registerOverlay({
  key: "driver-create",
  kind: "modal",
  payloadSchema: Schema.Struct({}),
  component: ({ close }) => <DriverCreateForm onCreated={close} />,
});
```

**Interaction trace:**

1. User opens the assign drawer. `driverAssign.open({ bookingId: 'b_42' })` pushes `[{ key: 'driver-assign', ... }]`.
2. User types "prefer afternoon" into the note input → React `useState` in `DriverAssignDrawerBody`.
3. User clicks "Create new driver" → `driverCreate.open({})` pushes `[{ key: 'driver-assign', ... }, { key: 'driver-create', ... }]`.
4. The drawer is still mounted at stack index 0 with `inert=""`. Its `useState` value `'prefer afternoon'` is untouched — the fiber is alive.
5. User fills in driver form, submits, modal calls `close()`. Stack drops back to `[{ key: 'driver-assign', ... }]`.
6. Drawer becomes top again (`inert` removed, focus restored). The note input still shows "prefer afternoon".

---

## Example: Open From DOM / Effect / Test

### From a plain DOM listener

```ts
// src/lib/keyboard-shortcuts.ts
import { bookingCreate } from "~/features/booking";
import { driverCreate } from "~/features/driver";

window.addEventListener("keydown", (e) => {
  if (!e.metaKey) return;
  if (e.key === "n") {
    e.preventDefault();
    bookingCreate.open({});
  }
  if (e.key === "d") {
    e.preventDefault();
    driverCreate.open({});
  }
});
```

No React, no hook, no provider context. Importing the handle is enough.

### From inside an Effect

```ts
// features/booking/booking.api.ts (excerpt)
import { Effect } from "effect";
import { bookingConflictDialog } from "./booking-conflict-overlay";

const createWithConflictRecovery = (input: CreateBookingInput) =>
  Effect.gen(function* () {
    const api = yield* BookingApi;
    return yield* api
      .create(input)
      .pipe(
        Effect.catchTag("BookingConflict", (err) =>
          Effect.sync(() =>
            bookingConflictDialog.open({ conflictId: err.bookingId }),
          ).pipe(Effect.andThen(Effect.fail(err))),
        ),
      );
  });
```

### From a unit test

```ts
// features/booking/booking.overlays.test.ts
import { describe, expect, it } from "vitest";
import { Registry } from "@effect-atom/atom";
import { bookingCreate } from "./booking-create-overlay";
// The Overlays atom namespace is private to the feature; tests that need
// to assert on the stack directly import it via a test-only `__test__.ts`
// file in features/overlays/.
import { __test__Overlays } from "~/features/overlays/__test__";

describe("bookingCreate overlay", () => {
  it("pushes an entry when opened", async () => {
    const registry = Registry.make();
    bookingCreate.open({ presetDriverId: "d_1" });
    // open() schedules an Effect on appRuntime; microtask flush
    await new Promise((r) => setTimeout(r, 0));
    const stack = registry.get(__test__Overlays.stack);
    expect(stack).toHaveLength(1);
    expect(stack[0].key).toBe("booking-create");
  });
});
```

See [06-testing.md](./06-testing.md) for the full testing approach, including mock `Layer` injection for overlays that dispatch Effects.

---

## When NOT to Use This

The overlay stack is for **transient, stackable, page-spanning** UI. It's the wrong tool for:

| Case                                            | Use instead                                                                                             |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Tooltips, popovers, hover cards                 | shadcn/BaseUI `Popover`, `Tooltip` — they're positional and don't stack                                 |
| In-page side panels that are part of the layout | A regular component in the route tree                                                                   |
| Small confirmation dialogs that never chain     | A local `useState` dialog is fine; promote to the stack only if it ever needs to spawn further overlays |
| Toasts / notifications                          | Dedicated toast system (sonner / shadcn toaster)                                                        |
| Context menus                                   | BaseUI `ContextMenu`                                                                                    |

**Rule of thumb:** if it's a rectangle with a backdrop that can spawn another rectangle with a backdrop, it belongs in the overlay stack. Otherwise, use a simpler primitive.

---

## See Also

- [01-folder-structure.md](./01-folder-structure.md) — feature layout, barrel-only imports
- [02-effect-services.md](./02-effect-services.md) — `Context.Tag`, `Layer`, tagged errors (used for `InvalidOverlayPayload`)
- [04-state-management.md](./04-state-management.md) — atoms, `Registry.AtomRegistry`, reactive sync atoms
- [05-feature-anatomy.md](./05-feature-anatomy.md) — reference for feature contract style
- [06-testing.md](./06-testing.md) — testing atoms and mutations with mock layers
- [08-routing.md](./08-routing.md) — TanStack Router, search params, `router.navigate`
