# Health Checks

## Context & Problem

Container orchestrators (Kubernetes) and load balancers need to know whether a service instance is functioning. Without health checks, traffic routes to broken instances — ones with severed database connections, exhausted connection pools, or deadlocked event loops. Users experience errors that the infrastructure could have avoided by simply routing to a healthy instance.

Three distinct questions need answering:

1. **Is the process alive?** (Liveness) — Has it deadlocked or entered an unrecoverable state? If not alive, kill and restart it.
2. **Is it ready to serve traffic?** (Readiness) — Has it finished startup, connected to dependencies, loaded configuration? If not ready, stop sending requests but do not restart.
3. **Has it finished starting up?** (Startup) — For slow-starting services, give extra time before liveness checks begin.

Conflating these three — or implementing only one — causes cascading failures. A service that fails liveness because PostgreSQL is slow gets restarted, which makes the connection storm worse. A service that reports healthy before its connection pool is initialized serves 500s on the first requests.

## Design Decisions

### Probe Architecture

```mermaid
graph TD
    K8S[Kubernetes / Load Balancer] -->|liveness probe| LIVE[/health/live]
    K8S -->|readiness probe| READY[/health/ready]
    K8S -->|startup probe| STARTUP[/health/startup]

    READY -->|checks| PG[(PostgreSQL)]
    READY -->|checks| RD[(Redis)]
    READY -->|checks| KF[(Kafka)]

    LIVE -->|no dependency checks| SELF[Process is alive]
```

**Key rule:** Liveness probes must NOT check external dependencies. If PostgreSQL is down and liveness fails, Kubernetes restarts the pod. The new pod also cannot reach PostgreSQL, so it fails liveness again. Every pod in the deployment restarts in a loop — a cascading failure caused by the health check itself.

### Health Check Models

```python
# health/models.py
# pydantic >= 2.0

from enum import Enum
from pydantic import BaseModel


class HealthStatus(str, Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"


class DependencyHealth(BaseModel):
    name: str
    status: HealthStatus
    latency_ms: float | None = None
    detail: str | None = None


class HealthResponse(BaseModel):
    status: HealthStatus
    version: str
    dependencies: list[DependencyHealth] = []
```

### Dependency Check Protocol

```python
# health/checks.py
import time
import asyncio
import logging
from typing import Protocol

import redis.asyncio as redis
from sqlalchemy.ext.asyncio import AsyncEngine
from sqlalchemy import text

from health.models import DependencyHealth, HealthStatus

logger = logging.getLogger(__name__)


class HealthCheck(Protocol):
    """Interface for a dependency health check."""

    name: str

    async def check(self) -> DependencyHealth: ...


class PostgresHealthCheck:
    """Check PostgreSQL connectivity via a lightweight query."""

    name = "postgresql"

    def __init__(self, engine: AsyncEngine, timeout: float = 3.0) -> None:
        self._engine = engine
        self._timeout = timeout

    async def check(self) -> DependencyHealth:
        start = time.monotonic()
        try:
            async with asyncio.timeout(self._timeout):
                async with self._engine.connect() as conn:
                    await conn.execute(text("SELECT 1"))
            latency = (time.monotonic() - start) * 1000
            return DependencyHealth(
                name=self.name,
                status=HealthStatus.HEALTHY,
                latency_ms=round(latency, 1),
            )
        except TimeoutError:
            return DependencyHealth(
                name=self.name,
                status=HealthStatus.UNHEALTHY,
                detail=f"Timeout after {self._timeout}s",
            )
        except Exception as e:
            return DependencyHealth(
                name=self.name,
                status=HealthStatus.UNHEALTHY,
                detail=str(e),
            )


class RedisHealthCheck:
    """Check Redis connectivity via PING."""

    name = "redis"

    def __init__(self, client: redis.Redis, timeout: float = 2.0) -> None:
        self._client = client
        self._timeout = timeout

    async def check(self) -> DependencyHealth:
        start = time.monotonic()
        try:
            async with asyncio.timeout(self._timeout):
                await self._client.ping()
            latency = (time.monotonic() - start) * 1000
            return DependencyHealth(
                name=self.name,
                status=HealthStatus.HEALTHY,
                latency_ms=round(latency, 1),
            )
        except Exception as e:
            return DependencyHealth(
                name=self.name,
                status=HealthStatus.UNHEALTHY,
                detail=str(e),
            )


class KafkaHealthCheck:
    """Check Kafka broker connectivity via metadata request."""

    name = "kafka"

    def __init__(self, bootstrap_servers: str, timeout: float = 5.0) -> None:
        self._bootstrap_servers = bootstrap_servers
        self._timeout = timeout

    async def check(self) -> DependencyHealth:
        # aiokafka >= 0.9.0
        from aiokafka import AIOKafkaProducer

        start = time.monotonic()
        try:
            producer = AIOKafkaProducer(
                bootstrap_servers=self._bootstrap_servers,
                request_timeout_ms=int(self._timeout * 1000),
            )
            async with asyncio.timeout(self._timeout):
                await producer.start()
                await producer.stop()
            latency = (time.monotonic() - start) * 1000
            return DependencyHealth(
                name=self.name,
                status=HealthStatus.HEALTHY,
                latency_ms=round(latency, 1),
            )
        except Exception as e:
            return DependencyHealth(
                name=self.name,
                status=HealthStatus.UNHEALTHY,
                detail=str(e),
            )
```

### Health Aggregator

The aggregator runs all dependency checks concurrently and determines overall status:

```python
# health/aggregator.py
import asyncio

from health.models import HealthResponse, HealthStatus, DependencyHealth
from health.checks import HealthCheck


class HealthAggregator:
    """
    Runs all registered health checks and computes aggregate status.

    Rules:
    - All healthy → HEALTHY
    - At least one unhealthy critical dependency → UNHEALTHY
    - Non-critical dependency down → DEGRADED
    """

    def __init__(self, version: str) -> None:
        self._version = version
        self._checks: list[tuple[HealthCheck, bool]] = []  # (check, is_critical)

    def register(self, check: HealthCheck, critical: bool = True) -> None:
        """Register a health check. Critical checks cause UNHEALTHY status."""
        self._checks.append((check, critical))

    async def evaluate(self) -> HealthResponse:
        """Run all checks concurrently and aggregate results."""
        tasks = [check.check() for check, _ in self._checks]
        results: list[DependencyHealth] = await asyncio.gather(*tasks)

        overall = HealthStatus.HEALTHY
        for result, (_, critical) in zip(results, self._checks):
            if result.status == HealthStatus.UNHEALTHY:
                if critical:
                    overall = HealthStatus.UNHEALTHY
                elif overall != HealthStatus.UNHEALTHY:
                    overall = HealthStatus.DEGRADED

        return HealthResponse(
            status=overall,
            version=self._version,
            dependencies=results,
        )
```

### FastAPI Endpoints

```python
# health/routes.py
import os
from fastapi import APIRouter, Response

from health.aggregator import HealthAggregator
from health.models import HealthResponse, HealthStatus

router = APIRouter(tags=["health"])

# Set to True once startup is complete (lifespan event sets this)
_startup_complete = False


def mark_startup_complete() -> None:
    global _startup_complete
    _startup_complete = True


@router.get("/health/live", response_model=dict)
async def liveness() -> dict:
    """
    Liveness probe. Checks only that the process is responsive.
    No dependency checks — if the event loop responds, the process is alive.
    """
    return {"status": "alive"}


@router.get("/health/ready", response_model=HealthResponse)
async def readiness(
    response: Response,
    aggregator: HealthAggregator = ...,  # injected via Depends
) -> HealthResponse:
    """
    Readiness probe. Checks all dependencies.
    Returns 503 if any critical dependency is unhealthy.
    """
    health = await aggregator.evaluate()
    if health.status == HealthStatus.UNHEALTHY:
        response.status_code = 503
    return health


@router.get("/health/startup", response_model=dict)
async def startup_check(response: Response) -> dict:
    """
    Startup probe. Returns 200 only after initialization is complete.
    Kubernetes uses this to know when to start liveness/readiness probes.
    """
    if not _startup_complete:
        response.status_code = 503
        return {"status": "starting"}
    return {"status": "started"}
```

### Wiring with FastAPI Lifespan

```python
# main.py
from contextlib import asynccontextmanager

from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine
import redis.asyncio as redis

from health.routes import router as health_router, mark_startup_complete
from health.aggregator import HealthAggregator
from health.checks import PostgresHealthCheck, RedisHealthCheck


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    engine = create_async_engine("postgresql+asyncpg://user:pass@db:5432/app")
    redis_client = redis.Redis(host="localhost", port=6379)

    aggregator = HealthAggregator(version="1.4.2")
    aggregator.register(PostgresHealthCheck(engine), critical=True)
    aggregator.register(RedisHealthCheck(redis_client), critical=False)

    app.state.health_aggregator = aggregator
    mark_startup_complete()

    yield

    # Shutdown
    await engine.dispose()
    await redis_client.aclose()


app = FastAPI(lifespan=lifespan)
app.include_router(health_router)
```

### Kubernetes Probe Configuration

```yaml
# k8s/deployment.yaml (container spec)
livenessProbe:
  httpGet:
    path: /health/live
    port: 8000
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3      # 3 consecutive failures → restart

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8000
  initialDelaySeconds: 0
  periodSeconds: 5
  timeoutSeconds: 5
  failureThreshold: 2      # 2 consecutive failures → remove from service

startupProbe:
  httpGet:
    path: /health/startup
    port: 8000
  initialDelaySeconds: 2
  periodSeconds: 3
  timeoutSeconds: 2
  failureThreshold: 20     # 20 * 3s = 60s max startup time
```

Configuration guidance:

| Probe | Period | Timeout | Failure Threshold | Notes |
|---|---|---|---|---|
| Liveness | 10s | 2s | 3 | Slow — do not restart aggressively |
| Readiness | 5s | 5s | 2 | Faster — remove from LB quickly |
| Startup | 3s | 2s | 20 | Allow generous startup time |

### Docker Compose Healthcheck

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/ready"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Cascading restarts | Liveness probe checks a shared dependency (DB); dependency goes down → all pods restart simultaneously | Never check external dependencies in liveness; only check process health |
| Flapping readiness | Dependency latency oscillates around the timeout threshold | Add hysteresis: require N consecutive failures before going unready, N consecutive successes before going ready |
| Slow dependency check blocks probe | Health check query takes longer than probe timeout | Set strict `asyncio.timeout` inside each check; probe timeout > sum of check timeouts |
| Health endpoint overwhelmed | Too many probes from too many sources | Cache health check results for 2-3 seconds; do not re-run checks on every probe |
| Startup probe expires | Service takes longer to start than `failureThreshold * periodSeconds` | Increase startup probe `failureThreshold`; optimize startup (lazy connections) |
| False healthy during graceful shutdown | Pod reports ready while draining connections | Set readiness to unhealthy before beginning shutdown; use `preStop` hook in Kubernetes |
| Degraded treated as unhealthy | Non-critical dependency down causes traffic removal | Distinguish DEGRADED from UNHEALTHY; return 200 for degraded, 503 only for unhealthy |

## Related Documents

- [Circuit Breakers](../resilience/circuit-breakers.md) — circuit state can inform health status (all circuits open → degraded)
- [Graceful Degradation](../resilience/graceful-degradation.md) — what the service does when dependencies are down but the service is still "ready"
- [Metrics Design](metrics-design.md) — expose health check latencies as Prometheus metrics
