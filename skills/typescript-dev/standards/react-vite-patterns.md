# Standard: React + Vite Patterns

> **Audience:** AI agents writing, reviewing, or refactoring React + Vite TypeScript applications.
> **Prerequisite:** Read `skills/typescript-dev/standards/idiomatic-typescript.md` for general TypeScript rules.
> **Context:** Typically the frontend in an Aspire-orchestrated stack. Aspire wires service discovery and proxying.

## Core Principle

React components are **pure functions of props**. Keep state minimal, effects explicit, and business logic outside components. Vite provides the build toolchain — don't fight it.

---

## 1. Project Structure

**Rule:** Feature-based folder structure. No `components/`, `hooks/`, `utils/` grab-bags.

```
src/
├── app/                  # App shell, routing, providers
│   ├── App.tsx
│   ├── routes.tsx
│   └── providers.tsx
├── features/
│   ├── orders/           # Feature module
│   │   ├── OrderList.tsx
│   │   ├── OrderDetail.tsx
│   │   ├── useOrders.ts  # Feature-specific hook
│   │   ├── orderApi.ts   # API client
│   │   └── types.ts      # Feature-specific types
│   └── dashboard/
│       └── ...
├── shared/               # Truly shared utilities
│   ├── ui/               # Design system components
│   ├── hooks/            # Generic hooks (useDebounce, etc.)
│   └── api/              # API client setup
└── main.tsx
```

### Guidelines

- **Feature folders** — everything for a feature lives together. Easy to find, easy to delete.
- **`shared/`** only for genuinely reused code. If it's used by one feature, it belongs in that feature.
- **Barrel exports** (`index.ts`) only at feature boundaries, not everywhere.

---

## 2. Components

**Rule:** Components are functions. Use `readonly` props interfaces. No `React.FC` — use plain function declarations.

### Bad

```tsx
const OrderList: React.FC<{ orders: Order[] }> = ({ orders }) => {
    return <div>{orders.map(o => <div key={o.id}>{o.name}</div>)}</div>;
};
```

### Good

```tsx
interface OrderListProps {
    readonly orders: readonly Order[];
    readonly onSelect: (id: string) => void;
}

function OrderList({ orders, onSelect }: OrderListProps) {
    return (
        <ul>
            {orders.map(order => (
                <li key={order.id} onClick={() => onSelect(order.id)}>
                    {order.name}
                </li>
            ))}
        </ul>
    );
}
```

### Guidelines

- **No `React.FC`** — it adds `children` implicitly and has historical baggage.
- **`readonly` on all props** — enforces immutability at the type level.
- **No default exports** — named exports are greppable and refactorable.
- **One component per file** — exception: tiny co-located helpers used only by the parent.

---

## 3. State Management

**Rule:** Start with React's built-in state. Escalate only when needed.

| Scale | Tool |
|-------|------|
| Component-local | `useState`, `useReducer` |
| Shared within feature | Lift state to parent, or context scoped to feature |
| App-wide (auth, theme) | React Context + `useReducer` |
| Server state (API data) | TanStack Query (`@tanstack/react-query`) |

### Guidelines

- **TanStack Query for server state** — handles caching, deduplication, background refresh, and error/loading states. Don't reinvent this.
- **No global state for server data** — let TanStack Query be the cache.
- **`useReducer`** over `useState` when state has more than 2–3 fields or complex transitions.
- **Context is not state management** — it's dependency injection. Keep context values stable (memoize).

---

## 4. API Client

**Rule:** Use a typed API client. Keep HTTP concerns out of components.

```typescript
// src/shared/api/client.ts
const API_BASE = import.meta.env.VITE_API_URL ?? "/api";

async function fetchJson<T>(path: string, init?: RequestInit): Promise<T> {
    const response = await fetch(`${API_BASE}${path}`, {
        ...init,
        headers: { "Content-Type": "application/json", ...init?.headers },
    });
    if (!response.ok) {
        throw new ApiError(response.status, await response.text());
    }
    return response.json() as Promise<T>;
}

// src/features/orders/orderApi.ts
export function getOrders(): Promise<readonly Order[]> {
    return fetchJson("/orders");
}

export function submitOrder(order: NewOrder): Promise<Order> {
    return fetchJson("/orders", {
        method: "POST",
        body: JSON.stringify(order),
    });
}
```

### Guidelines

- **`VITE_API_URL`** — Aspire sets this via service discovery. In dev, Vite proxies to the backend.
- **Typed return values** — no `any` leaking from API calls.
- **Error types** — define an `ApiError` class so hooks can pattern-match on status codes.

---

## 5. Vite Configuration

**Rule:** Minimal `vite.config.ts`. Use the proxy for backend API in development.

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
    plugins: [react()],
    server: {
        port: parseInt(process.env.PORT ?? "5173"),
        proxy: {
            "/api": {
                target: process.env.services__api__http__0 ?? "http://localhost:5000",
                changeOrigin: true,
            },
        },
    },
});
```

### Guidelines

- **`services__api__http__0`** — Aspire injects this environment variable for service discovery. Use it as the proxy target.
- **`PORT`** — Aspire sets this for the Vite dev server. Respect it.
- **No ejecting** — if you need custom Webpack config, you've gone wrong somewhere.

---

## 6. Testing

**Rule:** Vitest for unit tests. React Testing Library for component tests. Test behaviour, not implementation.

```typescript
// OrderList.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { OrderList } from "./OrderList";

test("calls onSelect when order is clicked", async () => {
    const onSelect = vi.fn();
    render(<OrderList orders={[{ id: "1", name: "Test" }]} onSelect={onSelect} />);

    await userEvent.click(screen.getByText("Test"));

    expect(onSelect).toHaveBeenCalledWith("1");
});
```

### Guidelines

- **`screen.getByText` / `getByRole`** — query by what users see, not by test IDs or class names.
- **`userEvent` over `fireEvent`** — more realistic interaction simulation.
- **No snapshot tests** — they break on every render change and provide no insight.

---

## Self-Check Table

| # | Smell | Fix |
|---|-------|-----|
| 1 | `React.FC` type annotation | Plain function with typed props interface |
| 2 | `components/` / `hooks/` / `utils/` at root | Feature-based folder structure |
| 3 | `useState` for server data | TanStack Query |
| 4 | Hardcoded API URLs | `import.meta.env.VITE_API_URL` + Vite proxy |
| 5 | Default exports on components | Named exports only |
| 6 | `any` in API responses | Typed `fetchJson<T>()` wrapper |
| 7 | Snapshot tests | Behaviour tests with Testing Library |
| 8 | Global state store for API cache | TanStack Query replaces it |
