# Structured Logging

## Context & Problem

Unstructured log messages (`logger.info(f"Trade {trade_id} executed for {quantity} shares")`) are human-readable but machine-hostile. Searching, filtering, and aggregating across millions of log lines requires parsing free-text — which is fragile and slow.

Structured logging emits log entries as key-value pairs (typically JSON). Every field is queryable, filterable, and aggregatable without regex.

## Design Decisions

### JSON Logging Format

```python
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer(),
    ],
)

logger = structlog.get_logger()

# Output:
# {"event": "trade_executed", "trade_id": "T-123", "quantity": "1000",
#  "instrument": "AAPL", "level": "info", "timestamp": "2025-03-15T14:30:00Z"}
logger.info("trade_executed", trade_id="T-123", quantity=1000, instrument="AAPL")
```

### Context Propagation

Use `contextvars` to attach context (request ID, user ID, tenant ID) to all log entries in a request:

```python
from starlette.middleware.base import BaseHTTPMiddleware
import structlog


class LoggingContextMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            request_id=request.headers.get("x-request-id", str(uuid4())),
            user_id=getattr(request.state, "user_id", None),
            tenant_id=getattr(request.state, "tenant_id", None),
            path=request.url.path,
            method=request.method,
        )
        response = await call_next(request)
        return response
```

Every log entry within the request automatically includes `request_id`, `user_id`, etc.

### What to Log

| Level | When | Example |
|---|---|---|
| **ERROR** | Something failed that should not have | Unhandled exception, data corruption detected |
| **WARNING** | Something unexpected but handled | Retry triggered, fallback used, slow query |
| **INFO** | Significant business events | Trade executed, position updated, user logged in |
| **DEBUG** | Technical detail for troubleshooting | SQL query, cache hit/miss, serialization detail |

**Do not log:**
- Sensitive data (passwords, tokens, PII) — scrub or mask
- High-volume data at INFO level (every tick of market data) — use DEBUG or metrics
- Expected conditions (cache miss on first request) — not worth the noise

### Log Correlation

Every log entry should include a `request_id` or `correlation_id` that traces across modules:

```
# Module A logs:
{"event": "trade_received", "request_id": "req-abc", "trade_id": "T-123"}

# Module B logs:
{"event": "compliance_check_passed", "request_id": "req-abc", "trade_id": "T-123"}

# Module C logs:
{"event": "position_updated", "request_id": "req-abc", "portfolio_id": "P-1"}
```

Searching by `request_id=req-abc` shows the full journey of a request across all modules.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Log volume explosion | DEBUG left on in production | Log level config per environment, default to INFO |
| Sensitive data in logs | Token or PII logged accidentally | Structlog processor to scrub known sensitive fields |
| Log loss | Async log shipping drops under load | Buffer locally, alert on log shipping lag |
| Unstructured creep | Developer uses f-string instead of structured fields | Linting rule, code review convention |

## Related Documents

- [Distributed Tracing](distributed-tracing.md) — request-level tracing across services
- [Metrics Design](metrics-design.md) — quantitative monitoring
- [Alerting Strategy](alerting-strategy.md) — acting on log patterns
