---
name: react-vite-admin-ts-shadcn-best-practice-skill
description: >
  Coding conventions for Vite + React + TypeScript projects using shadcn/ui.
  Use this skill whenever creating, scaffolding, or refactoring a Vite+React+TS
  frontend project — especially admin panels, dashboards, or internal tools.
  Covers directory structure, type naming (VO/QO), service layer, hooks,
  page component patterns, auth, and reusable component conventions.
---

# Vite + React + TypeScript + shadcn/ui Conventions

When building or refactoring a Vite + React + TypeScript frontend project, follow these conventions throughout. These are framework-level patterns — no business domain assumptions.

## Directory Structure

```
src/
├── app/                          # App-level wiring
│   ├── providers.tsx             # Composed context providers
│   └── router.tsx                # createBrowserRouter, lazy-loaded pages
├── types/
│   └── index.ts                  # Shared types: Result<T>, PageVO<T>, BasePageQO
├── services/                     # API service layer
│   ├── http.ts                   # HTTP client wrapper (fetch-based)
│   ├── <entity>.ts               # Service object per domain
│   └── <entity>.typings.d.ts    # VO/QO type definitions
├── hooks/                        # Custom hooks
│   └── use-<entity>.ts
├── pages/                        # Page components
│   └── <feature>/
│       ├── index.tsx             # Main page component
│       └── <sub-component>.tsx   # Extracted sub-components
├── components/                   # Reusable components
│   ├── ui/                       # shadcn/ui primitives
│   └── <component>.tsx           # App-level reusable components
├── constants/                    # Constants (navigation, config)
├── utils/                        # Utility functions
├── lib/                          # Core utilities (e.g., cn() for Tailwind)
├── layouts/                      # Layout shell components
└── main.tsx                      # Entry point
```

Rules:
- `app/` only contains providers and router — no business logic
- `types/index.ts` holds shared types only; domain types live in `services/<entity>.typings.d.ts`
- `components/ui/` is for shadcn primitives; higher-level reusable components go directly in `components/`
- Never create empty directories — only add directories when they have content

## Type Naming Conventions

Use these suffixes consistently:

| Suffix | Meaning | Example |
|--------|---------|---------|
| `VO` | View Object — API response shape | `ListItemVO`, `DetailVO` |
| `QO` | Query Object — query/filter params | `ListQueryQO`, `SearchQueryQO` |
| `DTO` | Data Transfer Object — mutation request body | `CreateDTO`, `UpdateDTO` |

Rules:
- **String literal unions, NOT enums** — e.g., `type Status = "ACTIVE" | "DISABLED"`
- Response types in `services/<entity>.typings.d.ts` as exported interfaces
- Query types (extending `BasePageQO`) exported from the service file itself
- Shared types (`Result<T>`, `PageVO<T>`, `BasePageQO`) in `types/index.ts`

Shared types (always include these in `types/index.ts`):

```typescript
export type Result<T> =
  | { success: true; data: T }
  | { success: false; code: string; msg: string };

export interface PageVO<T> {
  pageNum: number;
  pageSize: number;
  totalPage: number;
  totalRow: number;
  hasPrevious: boolean;
  hasNext: boolean;
  data: T[];
}

export interface BasePageQO {
  pageNum: number;
  pageSize: number;
}
```

Domain types go in their own typings file:

```typescript
// services/<entity>.typings.d.ts
export interface EntityListItemVO {
  id: number;
  name: string;
  status: "ACTIVE" | "DISABLED";
  createTime?: string;
}

export interface EntityDetailVO extends EntityListItemVO {
  description?: string;
}
```

## HTTP Client

Use a thin fetch wrapper — no axios needed for simple admin APIs.

```typescript
// services/http.ts
export const http = {
  get<T>(path: string, params?: unknown, token?: string): Promise<T>,
  post<T>(path: string, body?: unknown, token?: string): Promise<T>,
};
```

Key behaviors:
- Build full URL from `VITE_API_BASE` env var (default `/api`)
- Use `URLSearchParams` to serialize query params — never manually concatenate
- Inject `Authorization` header when `token` is provided
- Parse every response as `Result<T>` envelope
- Throw `HttpError` on non-200 or `success: false`
- Register an `unauthorizedHandler` callback for 401 interception

## Service Layer

One plain exported object per domain. No classes.

```typescript
// services/<entity>.ts
export interface EntityQueryQO extends BasePageQO {
  keyword?: string;
  status?: string;
}

export const entityService = {
  list(params: EntityQueryQO, token?: string) {
    return http.get<PageVO<EntityListItemVO>>("/api/entities", params, token);
  },
  detail(id: number, token?: string) {
    return http.get<EntityDetailVO>(`/api/entities/${id}`, undefined, token);
  },
  create(payload: CreateEntityDTO, token?: string) {
    return http.post("/api/entities", payload, token);
  },
  update(id: number, payload: UpdateEntityDTO, token?: string) {
    return http.post(`/api/entities/${id}`, payload, token);
  },
};
```

Rules:
- Object method naming: `list`, `detail`, `create`, `update`, `updateStatus`, `delete`
- Every protected endpoint method takes `token?: string` as its last parameter
- Re-export DTO/QO types from `.typings.d.ts` when they're needed by hooks or pages

## Hooks

One file per domain: `hooks/use-<entity>.ts`.

Conventions:
- **Naming**: `use<Entity><Action>` — e.g., `useOrderList`, `useCreateProduct`, `useUpdateStatus`
- **Token**: always obtained from `useAuth()`, passed as `token ?? undefined`
- **Data layer**: wrap `services/<entity>.ts` calls — keep components free of direct service imports

## Page Components

Each page lives in `pages/<feature>/index.tsx`. Keep the main component under 150 lines.

List page pattern:

```typescript
// pages/<feature>/index.tsx
export function ListPage() {
  const [keyword, setKeyword] = useState("");
  const [status, setStatus] = useState("");
  const [pageNum, setPageNum] = useState(1);

  const listQuery = useEntityList({ pageNum, pageSize: 20, keyword, status });

  return (
    <div className="space-y-4">
      <h1 className="text-2xl font-semibold">Title</h1>
      <FilterBar>{/* keyword input, status select, search button */}</FilterBar>
      <DataTableShell
        summary={`Total: ${listQuery.data?.totalRow ?? 0}`}
        hasPrevious={listQuery.data?.hasPrevious}
        hasNext={listQuery.data?.hasNext}
        onPrevious={() => setPageNum((p) => p - 1)}
        onNext={() => setPageNum((p) => p + 1)}
      >
        <Table>{/* rows */}</Table>
      </DataTableShell>
      <ConfirmDialog ... />
    </div>
  );
}
```

Rules:
- Local `useState` for filters (`keyword`, `status`, `pageNum`) — no global filter state
- Extract sub-components (detail sheets, forms, row actions) into sibling files
- Use `FilterBar` + `DataTableShell` + `ConfirmDialog` for list pages
- Form pages use `useState` with an `INITIAL_FORM` constant, reset on success

## Reusable Components

Standard building blocks for list pages:

**FilterBar** — horizontal flex wrapper for filter controls:
```tsx
<div className="flex flex-wrap items-center gap-2 rounded-lg border bg-card p-3">
  {children}
</div>
```

**DataTableShell** — table card with pagination footer:
- Props: `summary`, `hasPrevious`/`hasNext`, `onPrevious`/`onNext`
- Renders prev/next buttons, disabled based on pagination flags from `PageVO`

**ConfirmDialog** — destructive action confirmation:
- Props: `open`, `title`, `description?`, `confirmText`, `cancelText`, `loading`, `onConfirm`
- Confirm button uses destructive variant

## Auth Pattern

- **AuthProvider**: React context holding `token` (persisted in `localStorage`), `user` object, `login()`, `logout()`
- **useAuth**: thin `useContext(AuthContext)` wrapper that throws if used outside provider
- **ProtectedRoute**: route guard that shows loading during bootstrap, redirects to `/login` when unauthenticated
- **401 handler**: registered callback that clears state, shows toast, redirects to `/login`

## Routing

- Use `createBrowserRouter` with `React.lazy` + `Suspense` for code splitting
- `/login` is public; all other routes wrapped in `<ProtectedRoute><AdminLayout>`
- `/` redirects to `/dashboard`
- `*` catch-all redirects to `/`

## Naming Summary

| Item | Convention | Example |
|------|-----------|---------|
| Variables, functions | camelCase | `entityList`, `formatDateTime` |
| Types, interfaces, components | PascalCase | `EntityListItemVO`, `ListPage` |
| Hooks | camelCase with `use` prefix | `useEntityList`, `useAuth` |
| Constants | UPPER_SNAKE_CASE | `NAVIGATION_ITEMS`, `PAGE_SIZE` |
| Files (services) | kebab-case | `orders.ts`, `orders.typings.d.ts` |
| Files (hooks) | kebab-case with `use-` prefix | `use-orders.ts` |
| Files (pages) | kebab-case directory | `pages/order-list/index.tsx` |

## Immutability

Always create new objects, never mutate:

```typescript
// WRONG
user.name = newName;
return user;

// CORRECT
return { ...user, name: newName };
```

Apply to state updates, array operations, and object transformations throughout.
