# Authorization — RBAC & Actor Context

## Context & Problem

Authentication answers "who are you?" Authorization answers "what are you allowed to do?" These are separate concerns.

Role-Based Access Control (RBAC) is the most common authorization model: users are assigned roles, roles have permissions, and the application checks permissions before allowing actions. It works well for coarse-grained access control — PMs can view all portfolios, analysts can view but not trade, compliance can view everything but modify nothing.

For fine-grained, relationship-based access control (e.g., "user X can see document Y because they belong to team Z"), see [OpenFGA Modeling](../authorization/openfga-modeling.md).

## Design Decisions

### Actor Context — Who Is Making This Request?

Before checking permissions, you need to know *who* is acting. In a multi-tenant system with multiple actor types (human users, API keys, LLM agents), every request carries an **actor context** that answers:

- **Who** — actor ID (user UUID, API key ID, agent ID)
- **What kind** — actor type (user, apikey, agent, system)
- **Where** — which fund/tenant they're operating in
- **What they can do** — roles and resolved permissions
- **On whose behalf** — delegation chain (for agent tokens)

```python
from contextvars import ContextVar
from enum import StrEnum
from uuid import uuid4

from pydantic import BaseModel, ConfigDict, Field


class ActorType(StrEnum):
    USER = "user"
    API_KEY = "apikey"
    AGENT = "agent"
    SYSTEM = "system"


class RequestContext(BaseModel):
    """Immutable identity + authorization context for a single request."""

    model_config = ConfigDict(frozen=True)

    actor_id: str
    actor_type: ActorType
    fund_slug: str
    roles: frozenset[str] = frozenset()
    permissions: frozenset[str] = frozenset()
    request_id: str = Field(default_factory=lambda: str(uuid4()))
    delegated_by: str | None = None


_current_context: ContextVar[RequestContext] = ContextVar("request_context")
```

**Why `contextvars`, not dependency injection?** `contextvars` is async-safe and scoped to the current task — concurrent requests never leak context. It's accessible from any layer (service, handler, repository) without threading the context through every function signature. FastAPI dependencies set the context; downstream code reads it.

### Context Propagation — Strict vs. System Fallback

A critical safety decision: what happens when code reads the request context outside of a request (e.g., an event handler triggered by the simulator)?

```python
def get_request_context() -> RequestContext:
    """Get the current request context. Raises if no context is set.

    Use get_request_context_or_system() for code paths that
    legitimately run outside a request (e.g. event handlers).
    """
    try:
        return _current_context.get()
    except LookupError:
        raise RuntimeError(
            "No request context set. This code path requires an authenticated request."
        ) from None


def get_request_context_or_system() -> RequestContext:
    """Get the current request context, falling back to SYSTEM_CONTEXT.

    Only use this for code paths that legitimately run outside a
    request -- e.g. event handlers triggered by the simulator.
    """
    try:
        return _current_context.get()
    except LookupError:
        return SYSTEM_CONTEXT
```

**Why two functions?** A single function that silently falls back to system context is dangerous — a bug in middleware could grant every request admin-level access. The strict version forces developers to think about whether system-context fallback is appropriate for their code path.

### Agent Token Delegation

LLM agents act on behalf of users. The token carries a delegation chain so downstream systems can audit who authorized the agent:

```python
class AgentTokenRequest(BaseModel):
    agent_name: str
    roles: list[str] = ["viewer"]


@app.post("/auth/agent-token")
async def create_agent_token(
    request: AgentTokenRequest,
    ctx: RequestContext = require_permission(Permission.FUNDS_MANAGE),
) -> TokenResponse:
    """Issue a JWT for an LLM agent.

    Requires funds:manage permission. The agent token is:
    - Scoped to the caller's fund (not user-specified)
    - Carries delegated_by = caller's actor ID for audit
    """
    token = auth.issue_agent_token(
        agent_id=str(uuid4()),
        fund_slug=ctx.fund_slug,
        roles=request.roles,
        delegated_by=ctx.actor_id,
    )
    ...
```

Key security decisions:
- **Agent tokens require authentication** — only authenticated users with `funds:manage` can mint agent tokens
- **Fund scoping is automatic** — the agent token inherits the caller's fund, preventing cross-fund token creation
- **Delegation is mandatory** — every agent token carries the creating user's ID

### Permission Model

```
User --> has --> Role(s) --> grants --> Permission(s)
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
    INSTRUMENTS_READ = "instruments:read"
    INSTRUMENTS_WRITE = "instruments:write"
    PRICES_READ = "prices:read"
    POSITIONS_READ = "positions:read"
    POSITIONS_WRITE = "positions:write"
    TRADES_EXECUTE = "trades:execute"
    FUNDS_READ = "funds:read"
    FUNDS_MANAGE = "funds:manage"


ROLE_PERMISSIONS: dict[Role, frozenset[Permission]] = {
    Role.ADMIN: frozenset(Permission),  # all permissions
    Role.PORTFOLIO_MANAGER: frozenset({
        Permission.INSTRUMENTS_READ,
        Permission.PRICES_READ,
        Permission.POSITIONS_READ,
        Permission.POSITIONS_WRITE,
        Permission.TRADES_EXECUTE,
        Permission.FUNDS_READ,
    }),
    Role.ANALYST: frozenset({
        Permission.INSTRUMENTS_READ,
        Permission.PRICES_READ,
        Permission.POSITIONS_READ,
        Permission.FUNDS_READ,
    }),
    Role.RISK_MANAGER: frozenset({
        Permission.INSTRUMENTS_READ,
        Permission.PRICES_READ,
        Permission.POSITIONS_READ,
        Permission.FUNDS_READ,
    }),
    Role.COMPLIANCE: frozenset({
        Permission.INSTRUMENTS_READ,
        Permission.PRICES_READ,
        Permission.POSITIONS_READ,
        Permission.FUNDS_READ,
    }),
    Role.VIEWER: frozenset({
        Permission.INSTRUMENTS_READ,
        Permission.PRICES_READ,
        Permission.POSITIONS_READ,
        Permission.FUNDS_READ,
    }),
}
```

### Permission Checking as a Dependency

```python
from fastapi import Depends, HTTPException


def get_actor_context(request: Request) -> RequestContext:
    """FastAPI dependency -- returns the authenticated RequestContext.
    Raises 401 if no user is authenticated.
    """
    ctx = get_request_context()
    if ctx.actor_type == ActorType.SYSTEM:
        raise HTTPException(status_code=401, detail="Authentication required")
    return ctx


def require_permission(*perms: Permission):
    """FastAPI dependency that checks the caller has all listed permissions."""
    async def _check(
        ctx: RequestContext = Depends(get_actor_context),
    ) -> RequestContext:
        missing = {p.value for p in perms} - set(ctx.permissions)
        if missing:
            raise HTTPException(
                status_code=403,
                detail=f"Missing permissions: {', '.join(sorted(missing))}",
            )
        return ctx
    return Depends(_check)
```

Usage in routes — RBAC and OpenFGA compose naturally as stacked dependencies:

```python
@router.get("/{portfolio_id}/positions")
async def get_positions(
    portfolio_id: UUID,
    ctx: RequestContext = require_permission(Permission.POSITIONS_READ),
    _access: None = require_access(Portfolio.relation("can_view")),
) -> list[Position]:
    ...
```

The RBAC dependency runs first (does the actor have the `positions:read` permission?), then the OpenFGA dependency runs (does the actor have `can_view` on this specific portfolio?). Both must pass.

### Principle of Least Privilege

Default to no access. Every endpoint requires an explicit permission. There is no "authenticated = authorized" assumption.

```python
# Wrong -- any authenticated user can access
@router.get("/admin/users")
async def list_users(user: CurrentUser):
    ...

# Correct -- requires explicit permission
@router.get("/admin/users")
async def list_users(
    ctx: RequestContext = require_permission(Permission.USERS_MANAGE),
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
| Privilege escalation | User modifies their own roles | Roles come from IdP or DB lookup, not user input. Token is signed |
| Silent admin fallback | Context not set, falls back to system context | Strict `get_request_context()` raises error; explicit opt-in for system fallback |
| Cross-fund token minting | Unauthenticated agent token creation | Agent tokens require authentication + funds:manage permission |

## Related Documents

- [Authentication MFA](authentication-mfa.md) — identity verification before authorization
- [OpenFGA Modeling](../authorization/openfga-modeling.md) — fine-grained relationship-based access and type-safe resource registry
- [Attribute-Based Access](../authorization/attribute-based-access.md) — dynamic policy evaluation
