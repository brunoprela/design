# Test Containers

## Context & Problem

Integration tests need real infrastructure — PostgreSQL, Kafka, Redis, OpenFGA. Running these as shared services leads to test pollution and "works on my machine" failures. Mocking them defeats the purpose of integration testing.

Testcontainers spins up disposable Docker containers per test session. Each test run gets fresh infrastructure, identical to production, torn down automatically after the tests complete.

## Design Decisions

### Per-Session Containers

Start containers once per test session (not per test) for speed. Isolate tests within the session using transactions or table truncation:

```python
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.kafka import KafkaContainer
from testcontainers.redis import RedisContainer


@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer(
        image="timescale/timescaledb:latest-pg16",
        username="test",
        password="test",
        dbname="test",
    ) as pg:
        yield pg


@pytest.fixture(scope="session")
def kafka_container():
    with KafkaContainer(image="confluentinc/cp-kafka:7.6.0") as kafka:
        yield kafka


@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer(image="redis:7-alpine") as redis:
        yield redis
```

### Database Fixtures

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker


@pytest.fixture(scope="session")
async def db_engine(postgres_container):
    url = postgres_container.get_connection_url().replace("psycopg2", "asyncpg")
    engine = create_async_engine(url)

    # Run migrations
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield engine
    await engine.dispose()


@pytest.fixture
async def db_session(db_engine):
    """Per-test session with automatic rollback."""
    factory = async_sessionmaker(db_engine, expire_on_commit=False)
    async with factory() as session:
        async with session.begin():
            yield session
            await session.rollback()
```

### Kafka Test Fixtures

```python
@pytest.fixture(scope="session")
def kafka_bootstrap(kafka_container):
    return kafka_container.get_bootstrap_server()


@pytest.fixture
def kafka_producer(kafka_bootstrap):
    from confluent_kafka import Producer
    producer = Producer({"bootstrap.servers": kafka_bootstrap})
    yield producer
    producer.flush()


@pytest.fixture
def kafka_consumer(kafka_bootstrap):
    from confluent_kafka import Consumer
    consumer = Consumer({
        "bootstrap.servers": kafka_bootstrap,
        "group.id": f"test-{uuid4()}",  # unique group per test
        "auto.offset.reset": "earliest",
    })
    yield consumer
    consumer.close()
```

### CI Configuration

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      # Testcontainers need Docker
      docker:
        image: docker:dind
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e ".[test]"
      - run: pytest --timeout=60
        env:
          TESTCONTAINERS_RYUK_DISABLED: "false"  # cleanup container
```

## Performance Tips

| Technique | Impact |
|---|---|
| Session-scoped containers | Start once, reuse across all tests |
| Transaction rollback per test | Faster than truncating tables |
| Pre-pull images in CI | Avoid download time on every run |
| Parallel test execution | `pytest-xdist` with separate containers per worker |

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Docker not available | CI runner missing Docker | Ensure Docker-in-Docker or Docker socket is available |
| Port conflict | Another process using the same port | Testcontainers assigns random ports by default |
| Slow startup | Container image pull on every run | Cache images in CI, use `--pull=missing` |
| Container leak | Test crashes before cleanup | Testcontainers Ryuk sidecar auto-removes orphaned containers |

## Related Documents

- [Integration Testing](integration-testing.md) — testing strategy using testcontainers
- [Fixture Factories](fixture-factories.md) — generating test data
- [Docker Compose Patterns](../../infrastructure/docker-compose-patterns.md) — local development infrastructure
