# Next.js App Router — Project Structure for Large Codebases

## Context & Problem

Frontend applications that start in a flat `pages/` directory with shared utilities quickly become unnavigable. Components, hooks, API calls, and types end up scattered across generic folders (`components/`, `hooks/`, `utils/`) with no domain cohesion. A developer adding a new feature has to touch six directories and guess where things belong.

The App Router (Next.js 14+) introduces Server Components, nested layouts, and route groups — powerful primitives that impose structure but don't prescribe organization. Without a deliberate architecture, teams end up with a `components/` folder containing 200 files.

This document defines a feature-based project structure that scales to a large codebase while maintaining alignment with backend bounded contexts.

## Design Decisions

### Feature-Based, Not Layer-Based

**Layer-based** (`components/`, `hooks/`, `services/`, `types/`) forces developers to scatter a single feature across many directories. Adding "portfolio positions" touches `components/PositionTable.tsx`, `hooks/usePositions.ts`, `services/positionApi.ts`, `types/position.ts` — four directories for one concept.

**Feature-based** co-locates everything about a domain concept. Adding "portfolio positions" means creating or editing files in `features/portfolio/`. The directory is the feature boundary:

```
features/portfolio/
├── components/
│   ├── position-table.tsx
│   ├── trade-form.tsx
│   └── portfolio-summary.tsx
├── hooks/
│   ├── use-positions.ts
│   └── use-trade-mutation.ts
├── api.ts                  # TanStack Query options factories
├── types.ts                # Feature-specific types
└── index.ts                # Public exports
```

**Rule:** if a component is used by only one feature, it lives in that feature's directory. If it's used by two or more, it moves to `shared/components/`.

### Server vs Client Component Boundaries

The App Router defaults to Server Components. Client Components require `"use client"`. The boundary matters for performance, security, and data access:

| Component Type | Use When | Example |
|---|---|---|
| **Server Component** | Fetching data, accessing session, rendering static/dynamic content | Dashboard layout, position table (data-driven) |
| **Client Component** | Interactivity, browser APIs, state, effects | Trade form, fund selector dropdown, price ticker |
| **Shared UI** | Pure presentational, no data fetching | Button, DataTable, Badge, Card |

**Pattern:** Server Components fetch data and pass it as props to Client Component "islands" that handle interactivity:

```tsx
// app/(dashboard)/[fundSlug]/portfolio/[portfolioId]/page.tsx — Server Component
import { auth } from "@/shared/lib/auth";
import { getPositions } from "@/features/portfolio/api";
import { PositionTable } from "@/features/portfolio/components/position-table";

export default async function PortfolioPage({
  params,
}: {
  params: Promise<{ fundSlug: string; portfolioId: string }>;
}) {
  const { fundSlug, portfolioId } = await params;
  const session = await auth();
  const positions = await getPositions(session, fundSlug, portfolioId);

  return <PositionTable positions={positions} fundSlug={fundSlug} />;
}
```

```tsx
// features/portfolio/components/position-table.tsx — Client Component
"use client";

import { usePositions } from "../hooks/use-positions";

export function PositionTable({
  positions: initialData,
  fundSlug,
}: {
  positions: Position[];
  fundSlug: string;
}) {
  // Hydrate with server data, then keep fresh via polling
  const { data } = usePositions(fundSlug, { initialData });
  // ... interactive table with sorting, filtering
}
```

This gives fast initial render (Server Component), then hands off to TanStack Query for freshness.

### Route Groups and Layouts

Route groups `(name)` organize routes without affecting the URL. Use them to separate authentication states and apply distinct layouts:

```
app/
├── (auth)/                        # Unauthenticated layout (centered card)
│   ├── layout.tsx
│   ├── login/page.tsx
│   └── unauthorized/page.tsx
├── (dashboard)/                   # Authenticated layout (sidebar + header)
│   ├── layout.tsx                 # Auth check, sidebar, fund context
│   ├── [fundSlug]/                # Fund-scoped routes
│   │   ├── layout.tsx             # Fund context provider, nav
│   │   ├── page.tsx               # Fund overview / dashboard
│   │   ├── portfolio/
│   │   │   ├── page.tsx           # Portfolio list
│   │   │   └── [portfolioId]/
│   │   │       └── page.tsx       # Portfolio positions
│   │   ├── instruments/
│   │   │   └── page.tsx           # Instrument search/browse
│   │   └── market-data/
│   │       └── page.tsx           # Price dashboard
│   └── settings/                  # Non-fund-scoped
│       └── page.tsx               # User settings, API keys
├── api/                           # Route Handlers (BFF proxy)
│   ├── auth/[...nextauth]/route.ts
│   └── proxy/[...path]/route.ts   # Proxy to FastAPI
└── layout.tsx                     # Root layout (providers, fonts)
```

**Fund slug in the URL** — `[fundSlug]` is a dynamic segment, not client state. This makes fund context shareable (URLs work), tab-independent, and accessible in Server Components without client-side stores.

## Full Project Structure

```
mini-hedge-ui/
├── src/
│   ├── app/                              # Routes only — thin, delegate to features
│   │   ├── (auth)/
│   │   ├── (dashboard)/
│   │   ├── api/
│   │   └── layout.tsx
│   │
│   ├── features/                         # Domain logic, co-located by bounded context
│   │   ├── portfolio/                    # Positions, trades, P&L
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── api.ts
│   │   │   ├── types.ts
│   │   │   └── index.ts
│   │   ├── market-data/                  # Prices, tickers
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── api.ts
│   │   │   └── types.ts
│   │   ├── instruments/                  # Security master search/browse
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── api.ts
│   │   │   └── types.ts
│   │   └── platform/                     # Fund selector, user profile
│   │       ├── components/
│   │       ├── hooks/
│   │       └── api.ts
│   │
│   ├── shared/                           # Cross-feature shared code
│   │   ├── components/
│   │   │   └── ui/                       # Design system (shadcn/ui primitives)
│   │   │       ├── button.tsx
│   │   │       ├── data-table.tsx        # TanStack Table wrapper
│   │   │       ├── card.tsx
│   │   │       └── ...
│   │   ├── lib/
│   │   │   ├── auth.ts                   # Auth.js config
│   │   │   ├── api-client.ts             # Typed fetch wrapper
│   │   │   ├── formatters.ts             # Number, date, currency formatting
│   │   │   └── permissions.ts            # Role → Permission map, helpers
│   │   ├── hooks/
│   │   │   ├── use-permission.ts         # Permission check hook
│   │   │   └── use-fund-context.ts       # Read fundSlug from URL params
│   │   └── types/
│   │       ├── api.d.ts                  # Generated from OpenAPI spec
│   │       └── auth.d.ts                 # Session, token types
│   │
│   ├── middleware.ts                     # Route protection, redirects
│   └── env.ts                            # Typed env vars (t3-env or manual)
│
├── public/
├── next.config.ts
├── tailwind.config.ts
├── biome.json                            # Linting + formatting (replaces ESLint + Prettier)
├── tsconfig.json
├── vitest.config.ts
├── playwright.config.ts
├── Dockerfile
└── package.json
```

### Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Files | kebab-case | `position-table.tsx`, `use-positions.ts` |
| Components | PascalCase export | `export function PositionTable()` |
| Hooks | camelCase with `use` prefix | `usePositions`, `usePermission` |
| Types | PascalCase | `Position`, `TradeRequest` |
| API query keys | tuple with domain prefix | `['positions', fundSlug, portfolioId]` |
| Route segments | kebab-case | `market-data/`, `portfolio/` |

### Import Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@/features/*": ["./src/features/*"],
      "@/shared/*": ["./src/shared/*"]
    }
  }
}
```

`@/features/portfolio/components/position-table` is unambiguous. Never use relative paths across feature boundaries — `../../../shared/` is a code smell that the import should use an alias.

## Testing Approach

| Layer | Tool | Scope |
|---|---|---|
| Unit | Vitest | Formatters, permission logic, pure utilities |
| Component | Vitest + Testing Library | Feature components with MSW for API mocking |
| Integration | Playwright | Full user flows (login, trade, fund switch) |
| Visual | Storybook (optional) | Design system components in isolation |

MSW (Mock Service Worker) mocks at the network level — components use real fetch/TanStack Query logic. The same MSW handlers work in Vitest, Storybook, and development preview mode.

## Failure Modes

| Scenario | Impact | Mitigation |
|---|---|---|
| Feature grows too large | Single `features/portfolio/` has 50+ files | Split into sub-features: `features/portfolio/positions/`, `features/portfolio/trades/` |
| Shared component drift | `shared/components/` becomes a dumping ground | Gate moves to shared: a component must be used by 2+ features to qualify |
| Circular feature imports | `portfolio` imports from `instruments` and vice versa | Extract shared type to `shared/types/`, or introduce a new feature for the overlap |
| Server/Client boundary confusion | `"use client"` at the wrong level fetches data client-side unnecessarily | Rule: pages are Server Components; only interactive widgets are Client Components |

## Related Documents

- [OIDC Auth Flow](./oidc-auth-flow.md) — Auth.js + Keycloak integration
- [Frontend RBAC](./rbac-frontend.md) — permission enforcement in the UI
- [API Client Codegen](./api-client-codegen.md) — typed API client with OpenAPI
- [Frontend Dashboard](../../systems/hedge-fund-desk/frontend-dashboard.md) — system design composing these patterns
