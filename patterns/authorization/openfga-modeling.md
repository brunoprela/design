# OpenFGA — Relationship-Based Access Control

## Context & Problem

RBAC answers "does this user have the `trades:execute` permission?" That is sufficient for coarse-grained access. But many real-world authorization questions are relationship-based:

- "Can user X view document Y?" — depends on whether X is a member of the team that owns Y
- "Can user X approve this trade?" — depends on whether X is the compliance officer for this portfolio
- "Can this LLM query access document Z?" — depends on whether the user who initiated the query has access to Z

These are ReBAC (Relationship-Based Access Control) questions. OpenFGA is an open-source authorization engine (based on Google Zanzibar) that models and evaluates these relationships.

## Design Decisions

### When to Use OpenFGA vs. RBAC

| Question | Use |
|---|---|
| "Does user have role X?" | RBAC (simple, in-app) |
| "Can user do action X on resource Y?" | OpenFGA |
| "Can user see resources owned by their team?" | OpenFGA |
| "Which resources can user X access?" (list filtering) | OpenFGA |

Use RBAC for global permissions. Use OpenFGA when the answer depends on the relationship between the user and the specific resource.

### Authorization Model

OpenFGA uses a type system to define object types, relations, and how relations compose:

```dsl
# OpenFGA authorization model

model
  schema 1.1

type user

type team
  relations
    define member: [user]
    define lead: [user]

type portfolio
  relations
    define owner: [team]
    define viewer: [user, team#member]
    define editor: [user, team#member]
    define can_view: viewer or editor or owner
    define can_edit: editor or owner
    define can_trade: [user] and editor

type document
  relations
    define owner: [user, team]
    define viewer: [user, team#member]
    define can_view: viewer or owner
```

This reads as: a `portfolio` has `owner` (a team), `viewer` and `editor` relations. `can_view` is granted if the user is a viewer, editor, or member of the owning team.

### Writing Relationship Tuples

When a user joins a team or a portfolio is created, write the relationship:

```python
from openfga_sdk import OpenFgaClient, ClientWriteRequest, ClientTupleKey


class AuthorizationService:
    def __init__(self, fga_client: OpenFgaClient) -> None:
        self._fga = fga_client

    async def grant_team_membership(self, user_id: str, team_id: str) -> None:
        await self._fga.write(ClientWriteRequest(
            writes=[
                ClientTupleKey(
                    user=f"user:{user_id}",
                    relation="member",
                    object=f"team:{team_id}",
                ),
            ],
        ))

    async def assign_portfolio_owner(self, team_id: str, portfolio_id: str) -> None:
        await self._fga.write(ClientWriteRequest(
            writes=[
                ClientTupleKey(
                    user=f"team:{team_id}",
                    relation="owner",
                    object=f"portfolio:{portfolio_id}",
                ),
            ],
        ))
```

### Checking Authorization

```python
from openfga_sdk import ClientCheckRequest


class AuthorizationService:
    async def can_view_portfolio(self, user_id: str, portfolio_id: str) -> bool:
        response = await self._fga.check(ClientCheckRequest(
            user=f"user:{user_id}",
            relation="can_view",
            object=f"portfolio:{portfolio_id}",
        ))
        return response.allowed

    async def can_trade_portfolio(self, user_id: str, portfolio_id: str) -> bool:
        response = await self._fga.check(ClientCheckRequest(
            user=f"user:{user_id}",
            relation="can_trade",
            object=f"portfolio:{portfolio_id}",
        ))
        return response.allowed
```

### FastAPI Integration

```python
from fastapi import Depends, HTTPException


def require_relation(relation: str, object_type: str, id_param: str = "portfolio_id"):
    """FastAPI dependency for OpenFGA authorization checks."""
    async def check(
        request: Request,
        user: CurrentUser = Depends(get_current_user),
        auth: AuthorizationService = Depends(get_auth_service),
    ):
        object_id = request.path_params.get(id_param)
        allowed = await auth.check(
            user_id=user.sub,
            relation=relation,
            object=f"{object_type}:{object_id}",
        )
        if not allowed:
            raise HTTPException(status_code=403, detail="Access denied")
        return user
    return Depends(check)


@router.get("/portfolios/{portfolio_id}")
async def get_portfolio(
    portfolio_id: str,
    user=require_relation("can_view", "portfolio"),
):
    ...


@router.post("/portfolios/{portfolio_id}/trades")
async def create_trade(
    portfolio_id: str,
    request: TradeRequest,
    user=require_relation("can_trade", "portfolio"),
):
    ...
```

### List Filtering

"Show me all portfolios I can view" requires listing objects, not checking one:

```python
from openfga_sdk import ClientListObjectsRequest


async def list_accessible_portfolios(self, user_id: str) -> list[str]:
    response = await self._fga.list_objects(ClientListObjectsRequest(
        user=f"user:{user_id}",
        relation="can_view",
        type="portfolio",
    ))
    return [obj.replace("portfolio:", "") for obj in response.objects]
```

## Local Development

OpenFGA runs locally via Docker:

```yaml
services:
  openfga:
    image: openfga/openfga:latest
    ports:
      - "8080:8080"   # HTTP API
      - "8081:8081"   # gRPC
      - "3000:3000"   # Playground UI
    command: run
    environment:
      OPENFGA_DATASTORE_ENGINE: postgres
      OPENFGA_DATASTORE_URI: postgres://openfga:dev@postgres:5432/openfga
```

The playground at `localhost:3000` lets you visually test authorization models and relationship tuples.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| OpenFGA unavailable | Service down | Circuit breaker, fail-closed (deny access when unsure) |
| Stale relationships | User removed from team but tuple not deleted | Sync tuples on role changes, periodic reconciliation |
| Performance on list operations | Large number of objects | Use OpenFGA's contextual tuples, cache list results with short TTL |
| Model too permissive | Incorrect relation composition | Test model with assertions before deploying |

## Related Documents

- [OpenFGA for LLM Permissions](openfga-llm-permissions.md) — applying ReBAC to RAG and LLM access
- [Authorization RBAC](../api/authorization-rbac.md) — coarse-grained role checks
- [Policy as Code](policy-as-code.md) — OPA/Cedar for policy-based decisions
