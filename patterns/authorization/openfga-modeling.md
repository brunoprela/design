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

### Type-Safe Resource Registry

A common problem: the Python code and the JSON authorization model can drift. A developer adds a new relation in the JSON model but forgets to update the Python code, or vice versa. This drift is silent until a user hits a broken access check at runtime.

The solution is a type-safe resource registry with startup validation. Each resource type is declared in Python with its checkable relations, and the application validates these declarations against the JSON model on every startup.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ResourceType:
    """Declaration of an FGA object type and its checkable relations."""
    name: str
    relations: frozenset[str]

    def relation(self, name: str) -> "ResourceRelation":
        """Return a validated (type, relation) pair."""
        if name not in self.relations:
            raise ValueError(
                f"Unknown relation '{name}' for type '{self.name}'. "
                f"Valid: {sorted(self.relations)}"
            )
        return ResourceRelation(resource_type=self, relation=name)


@dataclass(frozen=True)
class ResourceRelation:
    """A bound (resource_type, relation) pair passed to require_access."""
    resource_type: ResourceType
    relation: str


_resource_registry: dict[str, ResourceType] = {}


def register_resource_type(rt: ResourceType) -> ResourceType:
    """Register a resource type. Returns it for assignment convenience."""
    _resource_registry[rt.name] = rt
    return rt
```

Resource declarations live in a single file, imported at startup:

```python
# fga_resources.py — imported during app startup to trigger registration

Portfolio = register_resource_type(ResourceType(
    name="portfolio",
    relations=frozenset({"can_view", "can_trade", "can_manage"}),
))

Fund = register_resource_type(ResourceType(
    name="fund",
    relations=frozenset({"member", "admin"}),
))
```

### Startup Validation

On startup, after loading the JSON authorization model, the application validates that every registered Python resource type and relation exists in the model. This catches drift immediately:

```python
def validate_resource_registry(model_json: dict) -> None:
    """Validate Python declarations against the FGA model JSON.

    Called during app startup. Raises ValueError with a clear
    message if any declared type or relation is missing.
    Does NOT check the reverse — JSON types not registered in
    Python are fine (e.g., the 'user' type).
    """
    json_types: dict[str, set[str]] = {}
    for td in model_json.get("type_definitions", []):
        json_types[td["type"]] = set(td.get("relations", {}).keys())

    errors: list[str] = []
    for rt_name, rt in _resource_registry.items():
        if rt_name not in json_types:
            errors.append(f"ResourceType '{rt_name}' not found in FGA model")
            continue
        missing = rt.relations - json_types[rt_name]
        if missing:
            errors.append(
                f"ResourceType '{rt_name}' declares relations "
                f"{sorted(missing)} not in FGA model"
            )

    if errors:
        raise ValueError(
            "FGA resource registry / model mismatch:\n"
            + "\n".join(f"  - {e}" for e in errors)
        )
```

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

### Generic FastAPI Dependency

Rather than writing a `require_portfolio_access()` dependency for each resource type, a generic `require_access` dependency works with any registered resource type. This eliminates boilerplate when adding new resource types:

```python
from enum import StrEnum
from fastapi import Depends, HTTPException, Request


class ParamSource(StrEnum):
    PATH = "path"
    BODY = "body"
    QUERY = "query"


def require_access(
    resource_relation: ResourceRelation,
    *,
    param: str | None = None,
    source: ParamSource = ParamSource.PATH,
):
    """Generic FastAPI dependency for FGA access checks.

    Extracts the resource ID from the request (path, body, or query),
    builds the FGA object reference, and checks the relation.
    Param defaults to "{type_name}_id" (e.g., "portfolio_id").
    """
    param_name = param or f"{resource_relation.resource_type.name}_id"

    async def _check(
        request: Request,
        ctx: RequestContext = Depends(get_actor_context),
        fga: FGAClient = Depends(_get_fga),
    ) -> None:
        resource_id = await _extract_resource_id(request, param_name, source)
        allowed = await fga.check(
            user=f"user:{ctx.actor_id}",
            relation=resource_relation.relation,
            object=f"{resource_relation.resource_type.name}:{resource_id}",
        )
        if not allowed:
            raise HTTPException(status_code=403, detail="Access denied")

    return Depends(_check)
```

Usage in routes — note how both RBAC permission and FGA relation are composed:

```python
@router.get("/{portfolio_id}/positions")
async def list_positions(
    portfolio_id: UUID,
    ctx: RequestContext = require_permission(Permission.POSITIONS_READ),
    _access: None = require_access(Portfolio.relation("can_view")),
) -> list[Position]:
    ...


@router.post("/trades", status_code=201)
async def execute_trade(
    request: TradeRequest,
    ctx: RequestContext = require_permission(Permission.TRADES_EXECUTE),
    _access: None = require_access(
        Portfolio.relation("can_trade"),
        source=ParamSource.BODY,
    ),
) -> Position:
    ...
```

### Adding a New Resource Type

With the generic system, adding FGA protection for a new resource type is a 3-step process:

1. **JSON model** — add the type definition with relations
2. **Resource declaration** — `Strategy = register_resource_type(ResourceType(name="strategy", relations=frozenset({...})))`
3. **Route** — `_access: None = require_access(Strategy.relation("can_view"))`

Mismatches are caught at startup. No new dependency functions, no new enums.

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
  openfga-migrate:
    image: openfga/openfga:latest
    command: migrate
    environment:
      OPENFGA_DATASTORE_ENGINE: postgres
      OPENFGA_DATASTORE_URI: "postgres://minihedge:minihedge@postgres:5432/minihedge"
    depends_on:
      postgres:
        condition: service_healthy

  openfga:
    image: openfga/openfga:latest
    command: run
    environment:
      OPENFGA_DATASTORE_ENGINE: postgres
      OPENFGA_DATASTORE_URI: "postgres://minihedge:minihedge@postgres:5432/minihedge"
      OPENFGA_PLAYGROUND_ENABLED: "true"
    ports:
      - "8080:8080"   # HTTP API
      - "3000:3000"   # Playground UI
    depends_on:
      openfga-migrate:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD", "grpc_health_probe", "-addr=localhost:8081"]
      interval: 5s
      timeout: 3s
      retries: 5
```

The playground at `localhost:3000` lets you visually test authorization models and relationship tuples.

**Note:** The OpenFGA Docker image does not include `curl` or `wget`. Use `grpc_health_probe` (included in the image) for healthchecks.

### SDK Gotchas

When writing tuples idempotently (e.g., during seeding), use the `ConflictOptions` wrapper — not a bare `on_duplicate_writes` key:

```python
from openfga_sdk.client.models import (
    ClientWriteRequest,
    ClientWriteRequestOnDuplicateWrites,
    ConflictOptions,
)

await client.write(
    body=ClientWriteRequest(writes=tuples),
    options={
        "conflict": ConflictOptions(
            on_duplicate_writes=ClientWriteRequestOnDuplicateWrites.IGNORE,
        ),
    },
)
```

The enum value is `IGNORE`, not `SKIP`.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| OpenFGA unavailable | Service down | Circuit breaker, fail-closed (deny access when unsure) |
| Stale relationships | User removed from team but tuple not deleted | Sync tuples on role changes, periodic reconciliation |
| Performance on list operations | Large number of objects | Use OpenFGA's contextual tuples, cache list results with short TTL |
| Model too permissive | Incorrect relation composition | Test model with assertions before deploying |
| Python/model drift | Developer adds relation in one but not the other | Startup validation via `validate_resource_registry()` |

## Related Documents

- [OpenFGA for LLM Permissions](openfga-llm-permissions.md) — applying ReBAC to RAG and LLM access
- [Authorization RBAC](../api/authorization-rbac.md) — coarse-grained role checks and actor context
- [Policy as Code](policy-as-code.md) — OPA/Cedar for policy-based decisions
