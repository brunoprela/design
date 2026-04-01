# Local PostgreSQL & TimescaleDB

## Context & Problem

PostgreSQL is the transactional backbone. TimescaleDB extends it for time-series data. Locally, both run in the same container — TimescaleDB's image is a superset of PostgreSQL, so modules that do not need time-series features still use plain PostgreSQL features.

## Design Decisions

### Single Container, Multiple Schemas

One PostgreSQL instance with schema-per-module isolation:

```yaml
services:
  postgres:
    image: timescale/timescaledb:latest-pg16
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/01-init.sql
      - ./scripts/extensions.sql:/docker-entrypoint-initdb.d/02-extensions.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 5
    command:
      - postgres
      - -c
      - shared_preload_libraries=timescaledb
      - -c
      - max_connections=200
      - -c
      - log_statement=all       # log all SQL in dev
      - -c
      - log_min_duration_statement=100  # log slow queries > 100ms
```

### Initialization Scripts

```sql
-- scripts/init-db.sql
-- Runs once on first container start

-- Bounded context schemas
CREATE SCHEMA IF NOT EXISTS market_data;
CREATE SCHEMA IF NOT EXISTS positions;
CREATE SCHEMA IF NOT EXISTS risk;
CREATE SCHEMA IF NOT EXISTS compliance;
CREATE SCHEMA IF NOT EXISTS audit;
CREATE SCHEMA IF NOT EXISTS read_models;

-- OpenFGA needs its own database
SELECT 'CREATE DATABASE openfga'
WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'openfga')\gexec
```

```sql
-- scripts/extensions.sql

CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;
CREATE EXTENSION IF NOT EXISTS vector;          -- pgvector
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;         -- trigram similarity for fuzzy search
CREATE EXTENSION IF NOT EXISTS btree_gist;      -- exclusion constraints
```

### Connection Configuration

```python
# Application settings for local development

DATABASE_URL = "postgresql+asyncpg://app:dev@localhost:5432/app"

# Pool tuned for local dev (smaller than production)
POOL_SIZE = 10
MAX_OVERFLOW = 5
```

### Running Migrations

```bash
# Run all module migrations
alembic -c migrations/alembic.ini -n positions upgrade head
alembic -c migrations/alembic.ini -n market_data upgrade head
alembic -c migrations/alembic.ini -n risk upgrade head

# Or use a Makefile target
make db-migrate
```

### Seed Data

```python
# scripts/seed_data.py

"""Seed local database with realistic test data."""

import asyncio
from datetime import datetime, timedelta
from decimal import Decimal
from uuid import uuid4

from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker


async def seed():
    engine = create_async_engine("postgresql+asyncpg://app:dev@localhost:5432/app")
    session_factory = async_sessionmaker(engine)

    async with session_factory() as session:
        # Seed instruments
        instruments = [
            {"id": "AAPL", "name": "Apple Inc.", "currency": "USD", "sector": "Technology"},
            {"id": "MSFT", "name": "Microsoft Corp.", "currency": "USD", "sector": "Technology"},
            {"id": "JPM", "name": "JPMorgan Chase", "currency": "USD", "sector": "Financials"},
            {"id": "NVDA", "name": "NVIDIA Corp.", "currency": "USD", "sector": "Technology"},
        ]

        # Seed price history (30 days of OHLCV)
        base_prices = {"AAPL": 175, "MSFT": 420, "JPM": 195, "NVDA": 880}
        for day_offset in range(30):
            trade_date = datetime.utcnow() - timedelta(days=30 - day_offset)
            for instrument_id, base in base_prices.items():
                noise = (hash(f"{instrument_id}{day_offset}") % 1000) / 100 - 5
                await session.execute(text("""
                    INSERT INTO market_data.ohlcv (timestamp, instrument_id, interval,
                        open, high, low, close, volume)
                    VALUES (:ts, :inst, '1d', :o, :h, :l, :c, :v)
                    ON CONFLICT DO NOTHING
                """), {
                    "ts": trade_date, "inst": instrument_id, "o": base + noise,
                    "h": base + noise + 3, "l": base + noise - 2,
                    "c": base + noise + 1, "v": 1_000_000 + day_offset * 10_000,
                })

        await session.commit()

    await engine.dispose()
    print("Seed data loaded")


asyncio.run(seed())
```

### CLI Cheat Sheet

```bash
# Connect to database
psql postgresql://app:dev@localhost:5432/app

# List schemas
\dn

# List tables in a schema
\dt positions.*

# Check TimescaleDB hypertables
SELECT * FROM timescaledb_information.hypertables;

# Check active connections
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

# Find slow queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - pg_stat_activity.query_start > interval '1 second';

# Check table sizes
SELECT schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename))
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC;
```

### pgAdmin (Optional UI)

```yaml
services:
  pgadmin:
    image: dpage/pgadmin4:latest
    profiles: [dev-tools]
    ports: ["5050:80"]
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@dev.local
      PGADMIN_DEFAULT_PASSWORD: dev
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Container won't start | Port 5432 in use (local PostgreSQL running) | Stop local PostgreSQL or change port mapping |
| Extension not found | Extension not included in image | TimescaleDB image includes most extensions; check image docs |
| Migration fails | Schema drift between branches | `docker compose down -v` to reset, re-run migrations |
| Slow queries in dev | Missing indexes, large seed data | Check `EXPLAIN ANALYZE`, add indexes in migrations |
| Data lost on restart | Forgot to mount volume | Always use named volumes for PostgreSQL data |

## Related Documents

- [Docker Compose Patterns](docker-compose-patterns.md) — full local environment
- [SQLAlchemy Repository](../patterns/data-access/sqlalchemy-repository.md) — connecting to this database
- [Alembic Migrations](../patterns/data-access/alembic-migrations.md) — managing schema changes
- [TimescaleDB Hypertables](../patterns/data-access/timescaledb-hypertables.md) — time-series tables
