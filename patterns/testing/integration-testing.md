# Integration Testing

## Context & Problem

Unit tests verify logic in isolation. But the most common production failures happen at integration boundaries: the SQL query that works in tests but fails against real PostgreSQL constraints, the Kafka consumer that handles the mock event but not the real serialized format, the API that passes schema validation but violates a database uniqueness constraint.

Integration tests verify that components work together correctly, using real infrastructure (databases, message brokers, caches).

## Design Decisions

### What to Integration Test

| Test | Real Infrastructure | What It Verifies |
|---|---|---|
| Repository against PostgreSQL | Real PostgreSQL | SQL correctness, constraints, migrations |
| API endpoint end-to-end | Real DB + real app | Request → validation → service → DB → response |
| Kafka consumer | Real Kafka + real DB | Deserialization → processing → state change |
| External API adapter | Mocked external, real internal | Adapter translates correctly, retries work |

### Test Pyramid

```
         /  E2E  \          — few, slow, high confidence
        / Integration \     — moderate, medium speed
       /   Unit Tests   \   — many, fast, focused
```

Unit tests cover business logic. Integration tests cover boundaries. E2E tests cover critical user flows. Most teams under-invest in integration tests.

### Testcontainers for Real Infrastructure

```python
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

from app.modules.positions.models import Base


@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("timescale/timescaledb:latest-pg16") as pg:
        yield pg


@pytest.fixture
async def session_factory(postgres):
    url = postgres.get_connection_url().replace("psycopg2", "asyncpg")
    engine = create_async_engine(url)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    factory = async_sessionmaker(engine, expire_on_commit=False)
    yield factory
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


async def test_trade_repository_persists_trade(session_factory):
    repo = PostgresTradeRepository(session_factory)

    trade = TradeRecord(
        portfolio_id=uuid4(),
        instrument_id="AAPL",
        side="buy",
        quantity=Decimal("1000"),
        price=Decimal("150.00"),
        currency="USD",
        executed_at=datetime.utcnow(),
    )

    saved = await repo.add(trade)
    fetched = await repo.get_by_id(saved.id)

    assert fetched is not None
    assert fetched.instrument_id == "AAPL"
    assert fetched.quantity == Decimal("1000")
```

### Database Isolation Between Tests

Each test should start with a clean state. Options:

1. **Transaction rollback** — wrap each test in a transaction, rollback after. Fast but does not test commit behavior.
2. **Truncate tables** — truncate all tables between tests. Medium speed, tests real commits.
3. **Fresh database** — create a new database per test. Slowest, strongest isolation.

**Recommendation:** Transaction rollback for most tests. Truncate for tests that need to verify commit behavior. Fresh database only for migration tests.

```python
@pytest.fixture
async def session(session_factory):
    """Provides a session that rolls back after each test."""
    async with session_factory() as session:
        async with session.begin():
            yield session
            await session.rollback()
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Flaky tests | Test depends on timing or external state | Deterministic fixtures, no sleep, testcontainers |
| Slow CI | Too many integration tests | Parallelize, use transaction rollback, cache Docker images |
| Test pollution | One test leaves state that affects another | Proper cleanup fixtures, isolation per test |
| Port conflicts | Multiple test suites run simultaneously | Testcontainers assigns random ports |

## Related Documents

- [Test Containers](test-containers.md) — infrastructure for integration tests
- [Contract Testing](contract-testing.md) — testing API contracts between modules
- [Fixture Factories](fixture-factories.md) — generating test data
