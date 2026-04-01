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

### SQLAlchemy Session-Level Schema Binding

For schema-per-tenant, the `search_path` must be set on the **same connection** used by the request handler. The safest approach is a SQLAlchemy session event that fires when a transaction begins:

```python
from sqlalchemy import event, text
from sqlalchemy.ext.asyncio import AsyncSession


def bind_tenant_schema(session_factory):
    """Attach a listener that sets search_path when a session begins a transaction."""

    @event.listens_for(session_factory.sync_session_class, "after_begin")
    def set_search_path(session, transaction, connection):
        tenant = get_current_tenant()
        # Fund schemas first, then shared schemas for reference data
        connection.execute(
            text(f"SET search_path TO {tenant}, shared, public")
        )
```

This guarantees every query within the session resolves tables against the correct fund schema. The `shared` schema is always appended so that cross-fund reference data (instruments, market data) is accessible without qualification.

### Fund Onboarding & Offboarding

Schema-per-tenant requires explicit lifecycle operations:

```python
class FundLifecycleService:
    """Manages fund schema creation and teardown."""

    # Module schemas created per fund (matches bounded context boundaries)
    FUND_SCHEMAS = [
        "positions", "orders", "risk", "compliance",
        "cash", "audit", "exposure", "performance", "alpha",
    ]

    async def onboard_fund(self, fund_slug: str) -> None:
        """Create all schemas and tables for a new fund."""
        async with self._engine.begin() as conn:
            for module in self.FUND_SCHEMAS:
                schema = f"{fund_slug}_{module}"
                await conn.execute(text(f"CREATE SCHEMA IF NOT EXISTS {schema}"))

            # Run module migrations against the new fund schemas
            await self._run_migrations(conn, fund_slug)

            # Register fund in platform.funds table
            await conn.execute(
                text("INSERT INTO platform.funds (slug, name, status) VALUES (:s, :n, 'active')"),
                {"s": fund_slug, "n": fund_slug},
            )

    async def offboard_fund(self, fund_slug: str) -> None:
        """Export data for retention, then drop fund schemas."""
        # 1. Export all fund data to cold storage (regulatory retention)
        await self._export_to_cold_storage(fund_slug)

        # 2. Verify export completeness
        await self._verify_export(fund_slug)

        # 3. Drop fund schemas
        async with self._engine.begin() as conn:
            for module in self.FUND_SCHEMAS:
                schema = f"{fund_slug}_{module}"
                await conn.execute(text(f"DROP SCHEMA IF EXISTS {schema} CASCADE"))

            # Mark fund as offboarded (don't delete — audit trail)
            await conn.execute(
                text("UPDATE platform.funds SET status = 'offboarded', offboarded_at = NOW() WHERE slug = :s"),
                {"s": fund_slug},
            )
```

### Fund-Scoped Kafka Topics

When the tenant boundary extends to the event bus, topics must be scoped per fund. A topic naming convention keeps this consistent:

```python
def fund_topic(fund_slug: str, topic: str) -> str:
    """Build a fund-scoped Kafka topic name.
    
    fund_topic("fund-alpha", "positions.changed")
    → "fund-alpha.positions.changed"
    """
    return f"{fund_slug}.{topic}"
```

Shared topics (market data, instruments, corporate actions) have no fund prefix — all funds consume the same prices and reference data. See [Hedge Fund Platform Overview](../../systems/hedge-fund-desk/overview.md#kafka-topic-scoping) for the full topic map.

### Migrations Per Schema

Schema-per-tenant means migrations must run against every fund's schemas. Alembic supports this via a multi-schema migration runner:

```python
async def run_migrations_all_funds(engine, migration_path: str) -> None:
    """Run a migration against every active fund's schemas."""
    async with engine.begin() as conn:
        funds = await conn.execute(
            text("SELECT slug FROM platform.funds WHERE status = 'active'")
        )
        for fund in funds:
            await conn.execute(
                text(f"SET search_path TO {fund.slug}, shared, public")
            )
            sql = Path(migration_path).read_text()
            await conn.execute(text(sql))
```

For 10–100 funds this is manageable (minutes). For 10,000+ tenants, schema-per-tenant becomes operationally expensive — consider shared tables with RLS instead.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Tenant data leakage | Missing `tenant_id` filter in a query | RLS as safety net, query auditing, integration tests per tenant |
| Noisy neighbor | One tenant's heavy queries slow others | Per-tenant rate limiting, resource quotas, separate connection pools |
| Migration drift | Schema-per-tenant with one schema behind | Automated migration runner that iterates all tenant schemas |
| Tenant context not set | Middleware bug, background job without context | Default-deny: RLS blocks all rows if `current_tenant` is not set |
| Tenant deletion | Need to fully remove a tenant's data | Schema-per-tenant makes this easy (`DROP SCHEMA`); shared-table needs careful cascading deletes |
| Schema drift | Migration applied to some fund schemas but not all | Automated migration runner iterates all active funds; version table per schema |
| Cross-fund data leak via Kafka | Consumer subscribed to wrong fund topic | Topic ACLs per fund, consumer group naming convention includes fund slug |
| search_path not set | Session created without `after_begin` listener | Integration test: assert `current_setting('search_path')` starts with fund schema |
| Fund offboarding data loss | Schema dropped before cold storage export verified | Two-phase offboarding: export → verify checksums → drop. Never drop without verified export |

## Related Documents

- [SQLAlchemy Repository](sqlalchemy-repository.md) — repository pattern with tenant awareness
- [Customer Data Isolation](../../data-strategies/customer-data-isolation.md) — broader FDE data isolation strategies
- [Authentication MFA](../api/authentication-mfa.md) — extracting tenant identity from tokens
- [Hedge Fund Platform Overview](../../systems/hedge-fund-desk/overview.md#tenancy-model) — schema-per-fund applied to financial system
- [Kafka Topology](../messaging/kafka-topology.md) — fund-scoped topic naming and ACLs
- [Disaster Recovery](../../infrastructure/disaster-recovery.md) — fund-aware backup and offboarding retention
