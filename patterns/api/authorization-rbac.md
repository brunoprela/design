# Authorization — RBAC

## Context & Problem

Authentication answers "who are you?" Authorization answers "what are you allowed to do?" These are separate concerns.

Role-Based Access Control (RBAC) is the most common authorization model: users are assigned roles, roles have permissions, and the application checks permissions before allowing actions. It works well for coarse-grained access control — PMs can view all portfolios, analysts can view but not trade, compliance can view everything but modify nothing.

For fine-grained, relationship-based access control (e.g., "user X can see document Y because they belong to team Z"), see [OpenFGA Modeling](../authorization/openfga-modeling.md).

## Design Decisions

### Permission Model

```
User → has → Role(s) → grants → Permission(s)
```

```python
from enum import StrEnum


class Role(StrEnum):
    ADMIN = "admin"
    PORTFOLIO_MANAGER = "portfolio_manager"
    ANALYST = "analyst"
    RISK_MANAGER = "risk_manager"
    COMPLIANCE = "compliance"
    VIEWER = "viewer"


class Permission(StrEnum):
    # Positions
    POSITIONS_READ = "positions:read"
    POSITIONS_WRITE = "positions:write"

    # Trading
    TRADES_READ = "trades:read"
    TRADES_EXECUTE = "trades:execute"
    TRADES_APPROVE = "trades:approve"

    # Risk
    RISK_READ = "risk:read"
    RISK_CONFIGURE = "risk:configure"

    # Admin
    USERS_MANAGE = "users:manage"
    SYSTEM_CONFIGURE = "system:configure"


ROLE_PERMISSIONS: dict[Role, set[Permission]] = {
    Role.ADMIN: set(Permission),  # all permissions
    Role.PORTFOLIO_MANAGER: {
        Permission.POSITIONS_READ,
        Permission.POSITIONS_WRITE,
        Permission.TRADES_READ,
        Permission.TRADES_EXECUTE,
        Permission.RISK_READ,
    },
    Role.ANALYST: {
        Permission.POSITIONS_READ,
        Permission.TRADES_READ,
        Permission.RISK_READ,
    },
    Role.RISK_MANAGER: {
        Permission.POSITIONS_READ,
        Permission.RISK_READ,
        Permission.RISK_CONFIGURE,
    },
    Role.COMPLIANCE: {
        Permission.POSITIONS_READ,
        Permission.TRADES_READ,
        Permission.TRADES_APPROVE,
        Permission.RISK_READ,
    },
    Role.VIEWER: {
        Permission.POSITIONS_READ,
        Permission.TRADES_READ,
    },
}
```

### Permission Checking as a Dependency

```python
# shared/auth.py

from functools import wraps
from fastapi import Depends, HTTPException, status


def require_permission(*permissions: Permission):
    """FastAPI dependency that checks permissions."""
    async def check(user: CurrentUser) -> CurrentUser:
        user_permissions: set[Permission] = set()
        for role in user.roles:
            user_permissions |= ROLE_PERMISSIONS.get(Role(role), set())

        missing = set(permissions) - user_permissions
        if missing:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Missing permissions: {', '.join(missing)}",
            )
        return user
    return Depends(check)


# Usage in routes
@router.post("/trades")
async def execute_trade(
    request: TradeRequest,
    user: CurrentUser = require_permission(Permission.TRADES_EXECUTE),
):
    ...


@router.get("/positions/{portfolio_id}")
async def get_positions(
    portfolio_id: str,
    user: CurrentUser = require_permission(Permission.POSITIONS_READ),
):
    ...
```

### Principle of Least Privilege

Default to no access. Every endpoint requires an explicit permission. There is no "authenticated = authorized" assumption.

```python
# This is wrong — any authenticated user can access
@router.get("/admin/users")
async def list_users(user: CurrentUser):
    ...

# This is correct — requires explicit permission
@router.get("/admin/users")
async def list_users(
    user: CurrentUser = require_permission(Permission.USERS_MANAGE),
):
    ...
```

## When RBAC Is Not Enough

RBAC handles role-based decisions well. It does not handle:

- **Resource-level access** — "can user X access portfolio Y?" (need ABAC or ReBAC)
- **Dynamic policies** — "PMs can only trade during market hours" (need policy engine)
- **Relationship-based access** — "user can see this document because their team owns it" (need OpenFGA)

For these cases, use RBAC for coarse-grained checks and OpenFGA or a policy engine for fine-grained decisions.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Permission bypass | Endpoint missing permission check | Require explicit auth on all routes, test for missing auth |
| Role explosion | Too many roles for every combination | Use permission composition, not one role per scenario |
| Stale roles | User changes role but token has old roles | Short-lived tokens (15 min), or check roles against DB for sensitive ops |
| Privilege escalation | User modifies their own roles | Roles come from IdP, not user input. Token is signed |

## Related Documents

- [Authentication MFA](authentication-mfa.md) — identity verification before authorization
- [OpenFGA Modeling](../authorization/openfga-modeling.md) — fine-grained relationship-based access
- [Attribute-Based Access](../authorization/attribute-based-access.md) — dynamic policy evaluation
