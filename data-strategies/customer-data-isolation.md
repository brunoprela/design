# Customer Data Isolation

## Context & Problem

In forward-deployed engineering (FDE) and multi-tenant platforms, customer data must be isolated — not just logically (authorization) but physically or structurally. A compliance team at Customer A should never risk seeing Customer B's data, even through a bug. A noisy query from one customer should not degrade performance for another.

Data isolation is a spectrum. The right strategy depends on the number of tenants, regulatory requirements, data volume per tenant, and the operational capacity of the team.

## Design Decisions

### Isolation Spectrum

```mermaid
graph LR
    A[Shared Table + RLS] -->|more isolation| B[Schema Per Tenant]
    B -->|more isolation| C[Database Per Tenant]
    C -->|more isolation| D[Instance Per Tenant]

    style A fill:#1a3a5c,stroke:#333
    style D fill:#5c1a1a,stroke:#333
```

| Strategy | Tenant Count | Isolation | Ops Cost | Migration Cost | Best For |
|---|---|---|---|---|---|
| **Shared table + RLS** | 10–10,000+ | Logical | Lowest | Lowest | SaaS, many small tenants |
| **Schema per tenant** | 10–500 | Logical + structural | Medium | Medium | Mid-market, moderate data |
| **Database per tenant** | 1–50 | Full database | High | High | Enterprise, regulated |
| **Instance per tenant** | 1–10 | Full infrastructure | Highest | Highest | Extreme compliance, large enterprise |

### Shared Table with Row-Level Security

Every table has a `tenant_id` column. PostgreSQL RLS policies enforce isolation at the database level — even if the application has a bug that omits the `WHERE tenant_id = ...` filter, RLS blocks cross-tenant access.

```sql
-- Enable RLS on all tenant tables
ALTER TABLE trades ENABLE ROW LEVEL SECURITY;
ALTER TABLE positions ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Policy: rows are only visible when tenant matches session variable
CREATE POLICY tenant_isolation ON trades
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

CREATE POLICY tenant_isolation ON positions
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Force RLS even for table owners (defense in depth)
ALTER TABLE trades FORCE ROW LEVEL SECURITY;
```

Application sets tenant context per request:

```python
from contextvars import ContextVar
from sqlalchemy import event, text

_current_tenant: ContextVar[str] = ContextVar("current_tenant")


class TenantMiddleware:
    async def __call__(self, request: Request, call_next):
        tenant_id = extract_tenant_from_token(request)
        _current_tenant.set(tenant_id)
        response = await call_next(request)
        return response


@event.listens_for(engine.sync_engine, "before_cursor_execute")
def set_tenant_context(conn, cursor, statement, parameters, context, executemany):
    tenant_id = _current_tenant.get(None)
    if tenant_id:
        cursor.execute(f"SET LOCAL app.current_tenant = '{tenant_id}'")
```

### Schema Per Tenant

Each tenant gets a PostgreSQL schema with identical tables:

```python
class TenantRouter:
    """Routes database operations to the correct tenant schema."""

    async def get_session(self, tenant_id: str) -> AsyncSession:
        async with self._session_factory() as session:
            await session.execute(
                text(f"SET search_path TO tenant_{tenant_id}, shared")
            )
            yield session
```

**Tenant lifecycle:**

```python
class TenantProvisioner:
    async def create_tenant(self, tenant_id: str) -> None:
        async with self._engine.begin() as conn:
            # Create schema
            await conn.execute(text(f"CREATE SCHEMA tenant_{tenant_id}"))
            # Run migrations in the new schema
            await self._run_migrations(conn, f"tenant_{tenant_id}")
            # Seed default data
            await self._seed_defaults(conn, f"tenant_{tenant_id}")

    async def delete_tenant(self, tenant_id: str) -> None:
        async with self._engine.begin() as conn:
            await conn.execute(text(f"DROP SCHEMA tenant_{tenant_id} CASCADE"))
```

### Database Per Tenant

Each tenant gets a separate PostgreSQL database. Maximum isolation, but connection management becomes the challenge:

```python
class TenantConnectionManager:
    """Maintains a connection pool per tenant database."""

    def __init__(self) -> None:
        self._pools: dict[str, async_sessionmaker] = {}

    async def get_session(self, tenant_id: str) -> AsyncSession:
        if tenant_id not in self._pools:
            engine = create_async_engine(
                f"postgresql+asyncpg://app:pass@localhost:5432/tenant_{tenant_id}",
                pool_size=5,
            )
            self._pools[tenant_id] = async_sessionmaker(engine)

        async with self._pools[tenant_id]() as session:
            yield session
```

## Cross-Tenant Operations

Some operations need to span tenants: billing aggregation, platform analytics, global search. These are **admin-only** operations that bypass tenant isolation deliberately:

```python
class PlatformAdminService:
    """Operates across all tenants. Never exposed to tenant-scoped APIs."""

    async def get_usage_report(self) -> list[TenantUsage]:
        tenants = await self._tenant_registry.list_all()
        results = []
        for tenant in tenants:
            async with self._get_session(tenant.id) as session:
                usage = await self._calculate_usage(session, tenant.id)
                results.append(usage)
        return results
```

Cross-tenant access must be:
- Explicit (separate service, separate routes, separate auth)
- Audited (every cross-tenant query logged)
- Minimal (only what's needed for platform operations)

## Data Residency

Some customers require data to stay in specific geographic regions (EU data stays in EU). This extends isolation to infrastructure:

```mermaid
graph TD
    subgraph "US Region"
        US_APP[App Instance] --> US_DB[(PostgreSQL)]
    end
    subgraph "EU Region"
        EU_APP[App Instance] --> EU_DB[(PostgreSQL)]
    end
    
    ROUTER[Tenant Router] -->|US tenants| US_APP
    ROUTER -->|EU tenants| EU_APP
```

The tenant registry stores the region, and the application routes requests accordingly.

## Backup and Recovery

| Strategy | Backup Approach | Recovery Granularity |
|---|---|---|
| Shared table | Full database backup, filter by tenant_id on restore | Requires filtering — slow, error-prone |
| Schema per tenant | `pg_dump --schema=tenant_X` | Per-tenant restore, clean |
| Database per tenant | `pg_dump tenant_X` | Full database restore, cleanest |

Schema-per-tenant and database-per-tenant allow per-tenant backup and restore without affecting other tenants. Shared-table requires careful data extraction.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Cross-tenant data leak | Missing RLS policy on new table | CI check that all tables have RLS enabled, integration tests |
| Noisy neighbor | One tenant's heavy queries degrade others | Per-tenant connection pools, resource quotas, query timeouts |
| Schema migration drift | One tenant schema behind others | Automated migration runner across all schemas, migration status dashboard |
| Tenant deletion incomplete | Foreign keys or external references remain | Cascade delete in schema, cleanup job for external stores (S3, Elasticsearch) |
| Connection pool exhaustion | Too many per-tenant pools with database-per-tenant | Lazy pool creation, pool eviction for inactive tenants, PgBouncer |

## Related Documents

- [Multi-Tenancy](../patterns/data-access/multi-tenancy.md) — implementation details at the data access layer
- [Authentication MFA](../patterns/api/authentication-mfa.md) — extracting tenant identity
- [Authorization RBAC](../patterns/api/authorization-rbac.md) — tenant-scoped roles
- [Alembic Migrations](../patterns/data-access/alembic-migrations.md) — per-tenant migration strategy
