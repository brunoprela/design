# Multi-Tenancy

## Context & Problem

Multi-tenant applications serve multiple customers (tenants) from the same deployment. The core tension is between isolation (tenants must not see each other's data) and efficiency (shared infrastructure reduces cost and operational burden).

This is especially relevant for forward-deployed engineering (FDE) roles, where you deploy a platform that serves multiple customers, each with their own data, configuration, and sometimes their own integrations.

## Design Decisions

### Isolation Strategies

| Strategy | Isolation | Complexity | Cost | When to Use |
|---|---|---|---|---|
| **Shared table, tenant column** | Lowest (row-level) | Lowest | Lowest | SaaS with many small tenants |
| **Schema per tenant** | Medium (schema-level) | Medium | Medium | Moderate tenant count, need logical separation |
| **Database per tenant** | Highest (full DB) | Highest | Highest | Regulated industries, large enterprise tenants |

### Shared Table with Row-Level Security (Recommended Start)

For most applications, a shared table with a `tenant_id` column and PostgreSQL Row-Level Security (RLS) provides strong isolation with minimal overhead:

```python
# models.py

class TradeRecord(Base):
    __tablename__ = "trades"

    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    tenant_id: Mapped[UUID] = mapped_column(index=True)  # every table has this
    instrument_id: Mapped[str] = mapped_column(String(32))
    # ... other fields
```

```sql
-- Row-Level Security policy
ALTER TABLE trades ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON trades
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

The application sets `app.current_tenant` at the start of each request:

```python
# middleware.py

class TenantMiddleware:
    async def __call__(self, request: Request, call_next):
        tenant_id = extract_tenant_from_token(request)
        # Set tenant context for RLS policies.
        # IMPORTANT: In production, SET LOCAL must run on the same database
        # session/connection used by the request handler — otherwise the
        # setting is lost. Prefer a SQLAlchemy session event listener
        # (e.g., after_begin) or ensure the middleware and request handler
        # share the same session via dependency injection.
        async with db.session() as session:
            await session.execute(
                text("SET LOCAL app.current_tenant = :tid"),
                {"tid": str(tenant_id)},
            )
        response = await call_next(request)
        return response
```

### Schema Per Tenant

Each tenant gets their own PostgreSQL schema. Tables are identical across schemas, but data is physically separated:

```
database: app
├── tenant_abc/
│   ├── trades
│   ├── positions
│   └── ...
├── tenant_xyz/
│   ├── trades
│   ├── positions
│   └── ...
└── shared/
    ├── tenants (registry)
    └── plans
```

SQLAlchemy can dynamically set the schema per request:

```python
from sqlalchemy import event


@event.listens_for(engine.sync_engine, "before_cursor_execute")
def set_search_path(conn, cursor, statement, parameters, context, executemany):
    tenant_id = get_current_tenant()
    cursor.execute(f"SET search_path TO tenant_{tenant_id}, shared")
```

**Tradeoff:** stronger isolation, but migrations must run per-schema. For 10 tenants this is manageable. For 10,000 it is not.

### Tenant Context Propagation

The tenant identity must flow through every layer without being manually threaded through function arguments:

```python
from contextvars import ContextVar

_current_tenant: ContextVar[str] = ContextVar("current_tenant")


def get_current_tenant() -> str:
    return _current_tenant.get()


def set_current_tenant(tenant_id: str) -> None:
    _current_tenant.set(tenant_id)
```

`contextvars` is async-safe and scoped to the current task — no risk of tenant leakage across concurrent requests.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Tenant data leakage | Missing `tenant_id` filter in a query | RLS as safety net, query auditing, integration tests per tenant |
| Noisy neighbor | One tenant's heavy queries slow others | Per-tenant rate limiting, resource quotas, separate connection pools |
| Migration drift | Schema-per-tenant with one schema behind | Automated migration runner that iterates all tenant schemas |
| Tenant context not set | Middleware bug, background job without context | Default-deny: RLS blocks all rows if `current_tenant` is not set |
| Tenant deletion | Need to fully remove a tenant's data | Schema-per-tenant makes this easy (`DROP SCHEMA`); shared-table needs careful cascading deletes |

## Related Documents

- [SQLAlchemy Repository](sqlalchemy-repository.md) — repository pattern with tenant awareness
- [Customer Data Isolation](../../data-strategies/customer-data-isolation.md) — broader FDE data isolation strategies
- [Authentication MFA](../api/authentication-mfa.md) — extracting tenant identity from tokens
