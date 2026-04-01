# Metrics Design

## Context & Problem

Logs tell you what happened. Traces tell you how a request flowed. Metrics tell you *how the system is performing* — quantitatively, over time, at a glance.

Without metrics, you discover problems from user reports. With metrics, you see degradation before users notice. The key is choosing the right metrics and structuring them for actionable dashboards and alerts.

## Design Decisions

### The Four Golden Signals

From Google's SRE handbook — monitor these for every service:

| Signal | What It Measures | Example Metric |
|---|---|---|
| **Latency** | Time to serve a request | `http_request_duration_seconds` |
| **Traffic** | Demand on the system | `http_requests_total` |
| **Errors** | Rate of failed requests | `http_errors_total` |
| **Saturation** | How full the system is | `db_connection_pool_usage_ratio` |

### RED Method (Request-focused)

For request-driven services (APIs):
- **Rate** — requests per second
- **Errors** — errors per second
- **Duration** — latency distribution (p50, p95, p99)

### USE Method (Resource-focused)

For infrastructure components (databases, queues):
- **Utilization** — % of resource capacity used
- **Saturation** — work waiting (queue depth)
- **Errors** — error count

### Prometheus Metrics in FastAPI

```python
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response


# Define metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "path", "status"],
)

REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration",
    ["method", "path"],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
)

DB_POOL_SIZE = Gauge(
    "db_connection_pool_size",
    "Database connection pool size",
    ["pool"],
)

KAFKA_CONSUMER_LAG = Gauge(
    "kafka_consumer_lag",
    "Kafka consumer lag by topic and partition",
    ["topic", "partition", "consumer_group"],
)


class MetricsMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        with REQUEST_DURATION.labels(
            method=request.method,
            path=request.url.path,
        ).time():
            response = await call_next(request)

        REQUEST_COUNT.labels(
            method=request.method,
            path=request.url.path,
            status=response.status_code,
        ).inc()

        return response


# Metrics endpoint
@app.get("/metrics")
async def metrics():
    return Response(content=generate_latest(), media_type="text/plain")
```

### Business Metrics

Beyond infrastructure, track domain-specific metrics:

```python
TRADES_EXECUTED = Counter(
    "trades_executed_total",
    "Total trades executed",
    ["portfolio", "side", "instrument_type"],
)

POSITION_VALUE = Gauge(
    "position_market_value_dollars",
    "Current market value of positions",
    ["portfolio", "instrument"],
)

COMPLIANCE_VIOLATIONS = Counter(
    "compliance_violations_total",
    "Compliance rule violations",
    ["rule_id", "severity"],
)
```

### Label Cardinality

Labels add dimensions to metrics. High cardinality labels (e.g., `user_id`, `trade_id`) create millions of time series and crash Prometheus. Rules:

- **Safe labels:** method, status_code, module, instrument_type, side
- **Dangerous labels:** user_id, trade_id, request_id, IP address
- **Rule:** if a label can have >100 unique values, do not use it as a metric label. Log it instead.

## Local Development

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: dev
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: app
    scrape_interval: 15s
    static_configs:
      - targets: ["host.docker.internal:8000"]
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Metric cardinality explosion | High-cardinality labels | Limit labels, review before adding new ones |
| Stale metrics | Gauge not updated after component restart | Use push-based for short-lived processes |
| Alert fatigue | Too many noisy alerts | Alert on symptoms, not causes. Tune thresholds |
| Missing metrics | New module deployed without instrumentation | Standardize metrics middleware, check in code review |

## Related Documents

- [Structured Logging](structured-logging.md) — qualitative complement to metrics
- [Distributed Tracing](distributed-tracing.md) — request-level detail
- [Alerting Strategy](alerting-strategy.md) — acting on metrics
