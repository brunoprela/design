# Distributed Tracing

## Context & Problem

A single user request may traverse multiple modules, databases, caches, and external APIs. When something is slow or fails, logs tell you *what* happened in each component but not *how the request flowed* or *where time was spent*.

Distributed tracing creates a trace — a tree of spans — that follows a request from entry to completion. Each span represents a unit of work (a function call, a DB query, an HTTP request) with timing and metadata.

## Design Decisions

### OpenTelemetry (OTEL)

OpenTelemetry is the standard. It provides:
- Vendor-neutral API for creating traces and spans
- Auto-instrumentation for common libraries (FastAPI, SQLAlchemy, httpx, confluent-kafka)
- Exporters to any backend (Jaeger, Zipkin, Grafana Tempo)

### What to Trace

| Operation | Trace? | Rationale |
|---|---|---|
| Incoming HTTP requests | Yes (auto) | FastAPI auto-instrumentation |
| Outgoing HTTP requests | Yes (auto) | httpx auto-instrumentation |
| Database queries | Yes (auto) | SQLAlchemy auto-instrumentation |
| Kafka produce/consume | Yes (auto) | confluent-kafka instrumentation |
| Business logic operations | Selective | Manual spans for critical paths |
| Cache lookups | Optional | Useful for debugging latency |

### Setup

```python
# shared/telemetry.py

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor


def setup_tracing(service_name: str, otlp_endpoint: str) -> None:
    provider = TracerProvider(
        resource=Resource.create({"service.name": service_name})
    )
    provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter(endpoint=otlp_endpoint))
    )
    trace.set_tracer_provider(provider)

    # Auto-instrument libraries
    FastAPIInstrumentor.instrument()
    SQLAlchemyInstrumentor().instrument()
    HTTPXClientInstrumentor().instrument()


# Manual spans for business logic
tracer = trace.get_tracer(__name__)


async def calculate_portfolio_risk(portfolio_id: str) -> RiskResult:
    with tracer.start_as_current_span(
        "calculate_portfolio_risk",
        attributes={"portfolio.id": portfolio_id},
    ) as span:
        positions = await get_positions(portfolio_id)
        span.set_attribute("position.count", len(positions))

        with tracer.start_as_current_span("calculate_var"):
            var = compute_var(positions)

        with tracer.start_as_current_span("calculate_stress"):
            stress = compute_stress_test(positions)

        return RiskResult(var=var, stress=stress)
```

### Trace Context Propagation

Trace context propagates automatically via HTTP headers (`traceparent`) and Kafka headers. For the modular monolith, in-process propagation is automatic — child spans inherit the parent from the current context.

## Local Development

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # UI
      - "4317:4317"     # OTLP gRPC
    environment:
      COLLECTOR_OTLP_ENABLED: true
```

Access traces at `http://localhost:16686`.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Missing spans | Library not instrumented | Check OTEL instrumentation registry |
| Broken trace | Context not propagated across async boundary | Use OTEL context-aware async utilities |
| High overhead | Tracing every operation | Sample traces (e.g., 10% in production) |
| Trace storage cost | High-volume traces | Tail-based sampling (only keep interesting traces) |

## Related Documents

- [Structured Logging](structured-logging.md) — logs correlated with trace IDs
- [Metrics Design](metrics-design.md) — quantitative view complementing traces
