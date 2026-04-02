# Frontend RBAC — Route Guards, Component Permissions, and Menu Visibility

## Context & Problem

The backend enforces permissions at two levels: coarse-grained RBAC (`require_permission`) and fine-grained resource access (OpenFGA). The frontend must mirror RBAC to provide a coherent user experience — hiding navigation items the user can't access, disabling actions they can't perform, and blocking routes they shouldn't reach.

**Frontend RBAC is a UX concern, not a security boundary.** The backend is the authority. A determined user can bypass any client-side check. Frontend permission enforcement prevents confusion ("why did I get a 403?") and provides a polished experience ("I only see what I can do").

## Design Decisions

### Mirror the Backend Role-Permission Map

The backend defines `ROLE_PERMISSIONS` in `app/shared/auth.py`. The frontend maintains an identical map. This is a deliberate duplication — the alternative (fetching permissions from an API) adds a round-trip to every page load and creates a chicken-and-egg problem (you need permissions to know what to render, but you need to render to show a loading state).

```typescript
// src/shared/lib/permissions.ts

export const Role = {
  ADMIN: "admin",
  PORTFOLIO_MANAGER: "portfolio_manager",
  ANALYST: "analyst",
  RISK_MANAGER: "risk_manager",
  COMPLIANCE: "compliance",
  VIEWER: "viewer",
} as const;

export type Role = (typeof Role)[keyof typeof Role];

export const Permission = {
  INSTRUMENTS_READ: "instruments:read",
  INSTRUMENTS_WRITE: "instruments:write",
  PRICES_READ: "prices:read",
  POSITIONS_READ: "positions:read",
  POSITIONS_WRITE: "positions:write",
  TRADES_EXECUTE: "trades:execute",
  FUNDS_READ: "funds:read",
  FUNDS_MANAGE: "funds:manage",
} as const;

export type Permission = (typeof Permission)[keyof typeof Permission];

const ALL_PERMISSIONS = new Set(Object.values(Permission));

export const ROLE_PERMISSIONS: Record<Role, ReadonlySet<Permission>> = {
  [Role.ADMIN]: ALL_PERMISSIONS,
  [Role.PORTFOLIO_MANAGER]: new Set([
    Permission.INSTRUMENTS_READ,
    Permission.PRICES_READ,
    Permission.POSITIONS_READ,
    Permission.POSITIONS_WRITE,
    Permission.TRADES_EXECUTE,
    Permission.FUNDS_READ,
  ]),
  [Role.ANALYST]: new Set([
    Permission.INSTRUMENTS_READ,
    Permission.PRICES_READ,
    Permission.POSITIONS_READ,
    Permission.FUNDS_READ,
  ]),
  [Role.RISK_MANAGER]: new Set([
    Permission.INSTRUMENTS_READ,
    Permission.PRICES_READ,
    Permission.POSITIONS_READ,
    Permission.FUNDS_READ,
  ]),
  [Role.COMPLIANCE]: new Set([
    Permission.INSTRUMENTS_READ,
    Permission.PRICES_READ,
    Permission.POSITIONS_READ,
    Permission.FUNDS_READ,
  ]),
  [Role.VIEWER]: new Set([
    Permission.INSTRUMENTS_READ,
    Permission.PRICES_READ,
    Permission.POSITIONS_READ,
    Permission.FUNDS_READ,
  ]),
};

export function resolvePermissions(roles: readonly string[]): Set<Permission> {
  const perms = new Set<Permission>();
  for (const role of roles) {
    const rolePerms = ROLE_PERMISSIONS[role as Role];
    if (rolePerms) {
      for (const p of rolePerms) perms.add(p);
    }
  }
  return perms;
}

export function hasPermission(
  userPermissions: ReadonlySet<Permission>,
  required: Permission,
): boolean {
  return userPermissions.has(required);
}

export function hasAllPermissions(
  userPermissions: ReadonlySet<Permission>,
  required: readonly Permission[],
): boolean {
  return required.every((p) => userPermissions.has(p));
}
```

**Keeping the map in sync:** This is tested in CI. A Vitest test imports the FastAPI OpenAPI spec (or a generated snapshot), extracts the role-permission matrix, and asserts it matches the TypeScript constant. Drift is caught before merge.

### Three Layers of Enforcement

```
┌─────────────────────────────────────────────────┐
│  middleware.ts        Route-level gates          │  Runs on every navigation
│                       (redirect if unauthorized) │  Before any component renders
├─────────────────────────────────────────────────┤
│  <Can> / usePermission  Component-level checks   │  Show/hide/disable elements
│                         (conditional rendering)  │  Within rendered pages
├─────────────────────────────────────────────────┤
│  Navigation config     Menu visibility           │  Filter sidebar/nav items
│                        (static config)           │  In layout components
└─────────────────────────────────────────────────┘
```

### Where Do Roles Come From?

The user's role is **fund-scoped** — a user might be `admin` in Fund Alpha but `analyst` in Fund Beta. The role comes from the `FundMembershipRecord` in the database, resolved by the backend during authentication.

In the frontend, the role is available from the `/api/v1/me/funds` endpoint:

```json
[
  { "fund_slug": "fund-alpha", "fund_name": "Alpha Capital", "role": "admin" },
  { "fund_slug": "fund-beta", "fund_name": "Beta Partners", "role": "analyst" }
]
```

The active role is determined by the current `[fundSlug]` in the URL. This data is fetched once on login (or fund switch) and cached via TanStack Query.

## Architecture

### Layer 1: Route Protection via Middleware

Next.js middleware runs on the edge before Server Components. Use it for coarse authentication checks — not fine-grained permission logic (middleware doesn't have easy access to the full session with fund-scoped roles):

```typescript
// src/middleware.ts

import { auth } from "@/shared/lib/auth";
import { NextResponse } from "next/server";

const PUBLIC_PATHS = new Set(["/login", "/unauthorized"]);

export default auth((req) => {
  const { pathname } = req.nextUrl;

  // Allow public paths
  if (PUBLIC_PATHS.has(pathname)) {
    return NextResponse.next();
  }

  // Allow Auth.js API routes
  if (pathname.startsWith("/api/auth")) {
    return NextResponse.next();
  }

  // Require authentication for everything else
  if (!req.auth) {
    return NextResponse.redirect(new URL("/login", req.url));
  }

  // Check for expired session
  if (req.auth.error === "RefreshTokenError") {
    return NextResponse.redirect(new URL("/login", req.url));
  }

  return NextResponse.next();
});

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

### Layer 2: Permission Hook and Component Gate

```typescript
// src/shared/hooks/use-permission.ts
"use client";

import { useMemo } from "react";
import {
  type Permission,
  hasAllPermissions,
  hasPermission,
  resolvePermissions,
} from "@/shared/lib/permissions";
import { useFundContext } from "./use-fund-context";

export function usePermission() {
  const { role } = useFundContext();

  const permissions = useMemo(
    () => resolvePermissions(role ? [role] : []),
    [role],
  );

  return {
    permissions,
    can: (permission: Permission) => hasPermission(permissions, permission),
    canAll: (perms: Permission[]) => hasAllPermissions(permissions, perms),
    role,
  };
}
```

```typescript
// src/shared/hooks/use-fund-context.ts
"use client";

import { useParams } from "next/navigation";
import { useQuery } from "@tanstack/react-query";

interface FundInfo {
  fund_slug: string;
  fund_name: string;
  role: string;
}

export function useFundContext() {
  const params = useParams<{ fundSlug: string }>();

  const { data: funds = [] } = useQuery<FundInfo[]>({
    queryKey: ["me", "funds"],
    queryFn: () => fetch("/api/proxy/me/funds").then((r) => r.json()),
    staleTime: 5 * 60 * 1000, // 5 minutes — fund memberships rarely change
  });

  const activeFund = funds.find((f) => f.fund_slug === params.fundSlug);

  return {
    fundSlug: params.fundSlug,
    fundName: activeFund?.fund_name ?? params.fundSlug,
    role: activeFund?.role ?? null,
    funds,
  };
}
```

**`<Can>` component — conditional rendering based on permissions:**

```tsx
// src/shared/components/can.tsx
"use client";

import type { Permission } from "@/shared/lib/permissions";
import { usePermission } from "@/shared/hooks/use-permission";
import type { ReactNode } from "react";

interface CanProps {
  permission: Permission | Permission[];
  /** Require all permissions (default) or any one */
  mode?: "all" | "any";
  /** Render when permission is denied */
  fallback?: ReactNode;
  children: ReactNode;
}

export function Can({
  permission,
  mode = "all",
  fallback = null,
  children,
}: CanProps) {
  const { can, canAll } = usePermission();

  const perms = Array.isArray(permission) ? permission : [permission];
  const allowed =
    mode === "all" ? canAll(perms) : perms.some((p) => can(p));

  return allowed ? <>{children}</> : <>{fallback}</>;
}
```

**Usage in feature components:**

```tsx
import { Can } from "@/shared/components/can";
import { Permission } from "@/shared/lib/permissions";

function PortfolioActions({ portfolioId }: { portfolioId: string }) {
  return (
    <div>
      {/* Always visible — read-only */}
      <ViewPositionsButton portfolioId={portfolioId} />

      {/* Only visible to users who can trade */}
      <Can permission={Permission.TRADES_EXECUTE}>
        <NewTradeButton portfolioId={portfolioId} />
      </Can>

      {/* Disabled with tooltip for insufficient permissions */}
      <Can
        permission={Permission.FUNDS_MANAGE}
        fallback={
          <Button disabled title="Requires fund management permission">
            Manage Fund
          </Button>
        }
      >
        <ManageFundButton />
      </Can>
    </div>
  );
}
```

### Layer 3: Navigation Visibility

Navigation items are filtered through the permission map in the layout:

```typescript
// src/shared/lib/navigation.ts

import { Permission } from "./permissions";

export interface NavItem {
  label: string;
  href: string;
  icon: string;
  /** Permission required to see this item. Omit for always-visible. */
  permission?: Permission;
}

export const NAV_ITEMS: NavItem[] = [
  { label: "Dashboard", href: "", icon: "LayoutDashboard" },
  {
    label: "Portfolios",
    href: "/portfolio",
    icon: "Briefcase",
    permission: Permission.POSITIONS_READ,
  },
  {
    label: "Instruments",
    href: "/instruments",
    icon: "Search",
    permission: Permission.INSTRUMENTS_READ,
  },
  {
    label: "Market Data",
    href: "/market-data",
    icon: "TrendingUp",
    permission: Permission.PRICES_READ,
  },
  {
    label: "Settings",
    href: "/settings",
    icon: "Settings",
    permission: Permission.FUNDS_MANAGE,
  },
];
```

```tsx
// In the dashboard layout sidebar:
function Sidebar({ fundSlug }: { fundSlug: string }) {
  const { can } = usePermission();

  const visibleItems = NAV_ITEMS.filter(
    (item) => !item.permission || can(item.permission),
  );

  return (
    <nav>
      {visibleItems.map((item) => (
        <NavLink
          key={item.href}
          href={`/${fundSlug}${item.href}`}
          icon={item.icon}
        >
          {item.label}
        </NavLink>
      ))}
    </nav>
  );
}
```

### Server Component Permission Checks

For pages that should be entirely inaccessible (not just missing a button), check permissions in the Server Component and redirect:

```tsx
// app/(dashboard)/[fundSlug]/settings/page.tsx

import { auth } from "@/shared/lib/auth";
import { redirect } from "next/navigation";
import { resolvePermissions, Permission } from "@/shared/lib/permissions";

export default async function SettingsPage({
  params,
}: {
  params: Promise<{ fundSlug: string }>;
}) {
  const { fundSlug } = await params;
  const session = await auth();
  if (!session) redirect("/login");

  // Fetch user's role for this fund (server-side)
  const fundsResponse = await fetch(`${process.env.API_URL}/api/v1/me/funds`, {
    headers: { Authorization: `Bearer ${session.accessToken}` },
  });
  const funds = await fundsResponse.json();
  const fund = funds.find((f: { fund_slug: string }) => f.fund_slug === fundSlug);

  if (!fund) redirect(`/unauthorized`);

  const perms = resolvePermissions([fund.role]);
  if (!perms.has(Permission.FUNDS_MANAGE)) redirect(`/${fundSlug}`);

  return <SettingsContent />;
}
```

## Failure Modes

| Scenario | Impact | Mitigation |
|---|---|---|
| Frontend permission map drifts from backend | User sees UI elements they can't actually use (403 on click) | CI test that compares TypeScript and Python maps |
| User's role changes while session is active | Stale permissions until TanStack Query refetches `/me/funds` | Cache TTL of 5 minutes is acceptable. Mutations that change roles (rare) should invalidate the query. |
| Fund membership revoked mid-session | API calls fail with 403 | BFF proxy catches 403, frontend shows "access revoked" with a prompt to select a different fund |
| New permission added to backend but not frontend | Feature is accessible via API but not surfaced in UI | The CI sync test catches missing permissions |

## Testing Approach

```typescript
// Test the permission map matches the backend
import { describe, it, expect } from "vitest";
import { ROLE_PERMISSIONS, Role, Permission } from "@/shared/lib/permissions";

describe("RBAC permission map", () => {
  it("admin has all permissions", () => {
    const adminPerms = ROLE_PERMISSIONS[Role.ADMIN];
    for (const perm of Object.values(Permission)) {
      expect(adminPerms.has(perm)).toBe(true);
    }
  });

  it("viewer cannot execute trades", () => {
    const viewerPerms = ROLE_PERMISSIONS[Role.VIEWER];
    expect(viewerPerms.has(Permission.TRADES_EXECUTE)).toBe(false);
  });

  it("portfolio_manager can execute trades but cannot manage funds", () => {
    const pmPerms = ROLE_PERMISSIONS[Role.PORTFOLIO_MANAGER];
    expect(pmPerms.has(Permission.TRADES_EXECUTE)).toBe(true);
    expect(pmPerms.has(Permission.FUNDS_MANAGE)).toBe(false);
  });
});
```

```tsx
// Test the <Can> component
import { render, screen } from "@testing-library/react";
import { Can } from "@/shared/components/can";
import { Permission } from "@/shared/lib/permissions";

// With MSW or a test provider that sets the role to "viewer"
it("hides trade button for viewer", () => {
  render(
    <Can permission={Permission.TRADES_EXECUTE}>
      <button>Trade</button>
    </Can>,
  );
  expect(screen.queryByText("Trade")).not.toBeInTheDocument();
});

it("shows fallback for denied permission", () => {
  render(
    <Can
      permission={Permission.TRADES_EXECUTE}
      fallback={<span>No access</span>}
    >
      <button>Trade</button>
    </Can>,
  );
  expect(screen.getByText("No access")).toBeInTheDocument();
});
```

## Related Documents

- [Authentication & RBAC](../api/authorization-rbac.md) — backend RBAC model and actor context
- [OIDC Auth Flow](./oidc-auth-flow.md) — how the user gets authenticated
- [Next.js App Router](./nextjs-app-router.md) — where permission-gated pages live in the route tree
- [OpenFGA Modeling](../authorization/openfga-modeling.md) — resource-level access (not mirrored in frontend)
