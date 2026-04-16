# 07 — Styling & Component Guide

**Stack:** Tailwind CSS v4 + shadcn/ui + BaseUI
**App type:** Vite SPA (React, no SSR)

---

## 1. Tailwind v4 CSS-First Configuration

Tailwind v4 moves all configuration into CSS. There is no `tailwind.config.ts`. Design tokens and theme mappings live in `src/styles/globals.css`.

```css
/* src/styles/globals.css */
@import "tailwindcss";

/* ─── Design Tokens ──────────────────────────────────────────────────────── */

:root {
  --main-background: oklch(95% 0 0);
  --background: oklch(100% 0 0);
  --foreground: oklch(14% 0 290);
  --card: oklch(100% 0 0);
  --card-foreground: oklch(14% 0 290);
  --popover: oklch(100% 0 0);
  --popover-foreground: oklch(14% 0 290);
  --primary: oklch(21% 0 290);
  --primary-foreground: oklch(98% 0 0);
  --secondary: oklch(97% 0 290);
  --secondary-foreground: oklch(21% 0 290);
  --muted: oklch(97% 0 290);
  --muted-foreground: oklch(55% 0 290);
  --accent: oklch(49% 0.2 260);
  --accent-foreground: oklch(98% 0 0);
  --success: oklch(65% 0.2 130);
  --success-foreground: oklch(99% 0 120);
  --destructive: oklch(64% 0.2 25);
  --destructive-foreground: oklch(98% 0 0);
  --warning: oklch(0.68 0.14 76);
  --warning-foreground: oklch(0.99 0.03 102);
  --border: oklch(92% 0 290);
  --input: oklch(92% 0 290);
  --ring: oklch(21% 0 290);
  --sidebar-background: oklch(98% 0 0);
  --sidebar-foreground: oklch(37% 0 290);
  --sidebar-primary: oklch(21% 0 290);
  --sidebar-primary-foreground: oklch(98% 0 0);
  --sidebar-accent: oklch(97% 0 290);
  --sidebar-accent-foreground: oklch(21% 0 290);
  --sidebar-border: oklch(93% 0 260);
  --sidebar-ring: oklch(62% 0.2 260);
  --radius: 0.5rem;
}

.dark {
  --main-background: oklch(24% 0 290);
  --background: oklch(18% 0 290);
  --foreground: oklch(98% 0 0);
  --card: oklch(24% 0 290);
  --card-foreground: oklch(98% 0 0);
  --popover: oklch(24% 0 290);
  --popover-foreground: oklch(98% 0 0);
  --primary: oklch(98% 0 0);
  --primary-foreground: oklch(21% 0 290);
  --secondary: oklch(27% 0 290);
  --secondary-foreground: oklch(98% 0 0);
  --muted: oklch(27% 0 290);
  --muted-foreground: oklch(71% 0 290);
  --accent: oklch(49% 0.2 260);
  --accent-foreground: oklch(98% 0 0);
  --success: oklch(85% 0.2 130);
  --success-foreground: oklch(33% 0.1 150);
  --destructive: oklch(51% 0.2 28);
  --destructive-foreground: oklch(98% 0 0);
  --border: oklch(37% 0 290);
  --input: oklch(37% 0 290);
  --ring: oklch(62% 0.2 260);
  --sidebar-background: oklch(23% 0 290);
  --sidebar-foreground: oklch(97% 0 290);
  --sidebar-primary: oklch(49% 0.2 260);
  --sidebar-primary-foreground: oklch(100% 0 0);
  --sidebar-accent: oklch(27% 0 290);
  --sidebar-accent-foreground: oklch(97% 0 290);
  --sidebar-border: oklch(29% 0 290);
  --sidebar-ring: oklch(62% 0.2 260);
}

/* ─── Tailwind Theme Mapping ─────────────────────────────────────────────── */

@theme {
  --radius-lg: var(--radius);
  --radius-md: calc(var(--radius) - 2px);
  --radius-sm: calc(var(--radius) - 4px);
  --radius-xs: calc(var(--radius) - 6px);
  --font-sans: 'Geist Sans', ui-sans-serif, system-ui, sans-serif;
}
```

All color tokens use the OKLCH color space, which provides perceptually uniform lightness and works correctly with Tailwind's opacity modifier syntax (e.g. `bg-primary/80`).

The `@theme` block exposes radius and font values as Tailwind utility classes (`rounded-lg`, `rounded-md`, etc.) by mapping CSS variables into Tailwind's design system.

---

## 2. The `cn()` Utility

All conditional class merging goes through `cn()`. This combines `clsx` (conditional logic) with `tailwind-merge` (conflict resolution so later classes win cleanly).

```ts
// src/lib/cn.ts
import { type ClassValue, clsx } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

Install the dependencies:

```bash
npm install clsx tailwind-merge
```

Usage:

```tsx
import { cn } from '~/lib/cn'

function StatusBadge({ active }: { active: boolean }) {
  return (
    <span
      className={cn(
        'rounded-sm px-2 py-0.5 text-xs font-medium',
        active
          ? 'bg-success/15 text-success'
          : 'bg-muted text-muted-foreground',
      )}
    >
      {active ? 'Active' : 'Inactive'}
    </span>
  )
}
```

---

## 3. shadcn/ui — Direct Installation

### Why shadcn/ui

shadcn/ui components are copied into your source tree. Each component ships with CVA (class-variance-authority) variants already defined. You get full access to the source and can extend it, but you do not need to write CVA yourself.

**Do not manually define `buttonVariants` or any other CVA variant object.** The shadcn CLI writes that code for you when you add the component.

### `components.json`

Place this at the project root before running the CLI:

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/styles/globals.css",
    "baseColor": "neutral",
    "cssVariables": true,
    "prefix": ""
  },
  "iconLibrary": "lucide",
  "aliases": {
    "components": "src/components",
    "utils": "src/lib/cn"
  }
}
```

Key decisions:

| Field | Value | Reason |
|---|---|---|
| `rsc` | `false` | Vite SPA — no React Server Components |
| `tailwind.config` | `""` | Tailwind v4 has no config file |
| `tailwind.css` | `src/styles/globals.css` | Points to the CSS-first config |
| `aliases.utils` | `src/lib/cn` | CLI will import `cn` from here |

### Installing Components

```bash
npx shadcn@latest add button input table badge dialog
```

The CLI copies component files into `src/components/ui/`. After running the command you will see:

```
src/components/ui/button.tsx
src/components/ui/input.tsx
src/components/ui/table.tsx
src/components/ui/badge.tsx
src/components/ui/dialog.tsx
```

Each file is yours to read and modify. The `button.tsx` that is generated already contains the full CVA setup:

```tsx
// src/components/ui/button.tsx  (generated — do not rewrite by hand)
import { Slot } from '@radix-ui/react-slot'
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '~/lib/cn'

const buttonVariants = cva(
  'inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground shadow hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground shadow-sm hover:bg-destructive/90',
        outline: 'border border-input bg-background shadow-sm hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground shadow-sm hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-9 px-4 py-2',
        sm: 'h-8 rounded-md px-3 text-xs',
        lg: 'h-10 rounded-md px-8',
        icon: 'h-9 w-9',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  },
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

export function Button({ className, variant, size, asChild = false, ...props }: ButtonProps) {
  const Comp = asChild ? Slot : 'button'
  return <Comp className={cn(buttonVariants({ variant, size, className }))} {...props} />
}
```

This is shown for reference only. The CLI writes this — do not copy-paste it manually.

### Using the Generated Components

```tsx
import { Button } from '~/components/ui/button'
import { Badge } from '~/components/ui/badge'

export function BookingActions({ onConfirm, onCancel }: BookingActionsProps) {
  return (
    <div className="flex items-center gap-2">
      <Badge variant="outline">Pending</Badge>
      <Button variant="default" onClick={onConfirm}>
        Confirm Booking
      </Button>
      <Button variant="ghost" size="sm" onClick={onCancel}>
        Cancel
      </Button>
    </div>
  )
}
```

---

## 4. BaseUI — When to Use

BaseUI provides unstyled, accessible primitives for interactions that are too complex for shadcn or where you need full control over the rendering tree.

### Use shadcn/ui for

- Button, Input, Textarea, Select (simple)
- Dialog, Sheet, Alert Dialog
- Table, Badge, Card, Tooltip
- Dropdown Menu (basic)

### Use BaseUI for

- Combobox with async search and multi-select
- Data grid with virtual scrolling
- Menu with complex keyboard navigation trees
- Number Input with custom formatting
- Any component where shadcn's rendered HTML structure is too constrained

### BaseUI Installation

```bash
npm install @base-ui-components/react
```

BaseUI components render with no built-in styles. Apply Tailwind classes directly:

```tsx
import * as Select from '@base-ui-components/react/select'
import { cn } from '~/lib/cn'

interface DriverSelectProps {
  value: string
  onChange: (value: string) => void
  drivers: { id: string; name: string }[]
}

export function DriverSelect({ value, onChange, drivers }: DriverSelectProps) {
  return (
    <Select.Root value={value} onValueChange={onChange}>
      <Select.Trigger
        className={cn(
          'flex h-9 w-full items-center justify-between rounded-md border border-input',
          'bg-background px-3 py-2 text-sm shadow-sm',
          'focus:outline-none focus:ring-1 focus:ring-ring',
          'disabled:cursor-not-allowed disabled:opacity-50',
        )}
      >
        <Select.Value placeholder="Select driver" />
        <Select.Icon className="text-muted-foreground" />
      </Select.Trigger>

      <Select.Portal>
        <Select.Positioner>
          <Select.Popup
            className={cn(
              'z-50 min-w-[8rem] overflow-hidden rounded-md border border-border',
              'bg-popover text-popover-foreground shadow-md',
            )}
          >
            <Select.Listbox className="p-1">
              {drivers.map((driver) => (
                <Select.Item
                  key={driver.id}
                  value={driver.id}
                  className={cn(
                    'relative flex cursor-default select-none items-center rounded-sm px-2 py-1.5 text-sm',
                    'outline-none data-[highlighted]:bg-accent data-[highlighted]:text-accent-foreground',
                    'data-[disabled]:pointer-events-none data-[disabled]:opacity-50',
                  )}
                >
                  <Select.ItemText>{driver.name}</Select.ItemText>
                </Select.Item>
              ))}
            </Select.Listbox>
          </Select.Popup>
        </Select.Positioner>
      </Select.Portal>
    </Select.Root>
  )
}
```

BaseUI uses `data-*` attributes for state (`data-highlighted`, `data-disabled`, `data-open`). Target these with Tailwind's `data-[]` variant syntax.

---

## 5. Component Conventions

### File Naming

Use kebab-case for all component files.

```
src/components/
  ui/                        # shadcn-generated, do not create files here manually
    button.tsx
    input.tsx
    table.tsx
  bookings/
    booking-list.tsx
    booking-card.tsx
    booking-status-badge.tsx
  drivers/
    driver-select.tsx
    driver-avatar.tsx
  shared/
    data-table.tsx
    page-header.tsx
    empty-state.tsx
```

### Props Interface

Every component exports a named props interface. Use the component name as the prefix.

```tsx
// src/components/bookings/booking-card.tsx
import { cn } from '~/lib/cn'
import { Badge } from '~/components/ui/badge'
import { Button } from '~/components/ui/button'
import { Card, CardContent, CardHeader, CardTitle } from '~/components/ui/card'

export interface BookingCardProps {
  id: string
  origin: string
  destination: string
  scheduledAt: Date
  status: 'pending' | 'confirmed' | 'in-transit' | 'delivered' | 'cancelled'
  driverName?: string
  onViewDetails: (id: string) => void
}

const statusVariant: Record<BookingCardProps['status'], 'default' | 'secondary' | 'destructive' | 'outline'> = {
  pending: 'outline',
  confirmed: 'secondary',
  'in-transit': 'default',
  delivered: 'secondary',
  cancelled: 'destructive',
}

export function BookingCard({
  id,
  origin,
  destination,
  scheduledAt,
  status,
  driverName,
  onViewDetails,
}: BookingCardProps) {
  return (
    <Card className="hover:shadow-md transition-shadow">
      <CardHeader className="flex flex-row items-center justify-between pb-2">
        <CardTitle className="text-sm font-medium">#{id}</CardTitle>
        <Badge variant={statusVariant[status]}>{status}</Badge>
      </CardHeader>
      <CardContent>
        <div className="space-y-1 text-sm">
          <p className="text-muted-foreground">
            <span className="font-medium text-foreground">{origin}</span>
            {' → '}
            <span className="font-medium text-foreground">{destination}</span>
          </p>
          <p className="text-muted-foreground">
            {scheduledAt.toLocaleDateString('en-GB', {
              day: '2-digit',
              month: 'short',
              year: 'numeric',
              hour: '2-digit',
              minute: '2-digit',
            })}
          </p>
          {driverName && (
            <p className="text-muted-foreground">
              Driver: <span className="text-foreground">{driverName}</span>
            </p>
          )}
        </div>
        <Button
          variant="ghost"
          size="sm"
          className="mt-3 w-full"
          onClick={() => onViewDetails(id)}
        >
          View Details
        </Button>
      </CardContent>
    </Card>
  )
}
```

### Using `cn()` for Conditional Classes

Pass `className` through to the root element whenever the component needs to be composable:

```tsx
import { cn } from '~/lib/cn'

interface PageHeaderProps {
  title: string
  description?: string
  className?: string
  children?: React.ReactNode
}

export function PageHeader({ title, description, className, children }: PageHeaderProps) {
  return (
    <div className={cn('flex items-center justify-between border-b border-border pb-4', className)}>
      <div>
        <h1 className="text-2xl font-semibold tracking-tight">{title}</h1>
        {description && (
          <p className="mt-1 text-sm text-muted-foreground">{description}</p>
        )}
      </div>
      {children && <div className="flex items-center gap-2">{children}</div>}
    </div>
  )
}
```

### Import Paths

Use the `~` alias (configured in `vite.config.ts` and `tsconfig.json`) for all internal imports. Never use relative paths that traverse more than one directory level.

```tsx
// Good
import { Button } from '~/components/ui/button'
import { cn } from '~/lib/cn'
import { useBookings } from '~/hooks/use-bookings'

// Avoid
import { Button } from '../../../components/ui/button'
```

### Avoid Prop Drilling Beyond Two Levels

If a value needs to pass through more than two component layers, use a context or a state management store instead of threading it through props.

---

## Quick Reference

| Need | Use |
|---|---|
| Button, Input, Dialog, Table, Badge, Card | `shadcn/ui` — install with CLI |
| Complex combobox, data grid, nested menus | `BaseUI` — unstyled primitives |
| Conditional class merging | `cn()` from `~/lib/cn` |
| Custom CVA variants | Only extend the generated component — do not create new `cva()` calls for components shadcn already covers |
| Dark mode | Add `.dark` class to `<html>` — tokens switch automatically |
| New design token | Add to both `:root` and `.dark` in `globals.css` |
