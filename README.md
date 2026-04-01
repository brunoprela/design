# Design

A design reference library for modular software systems. Each document captures the reasoning, tradeoffs, and implementation patterns behind production-grade architecture — from universal principles to concrete system designs.

## Philosophy

**Patterns are universal, systems are specific.** Reusable patterns (CQRS, event sourcing, RBAC, data pipelines) live independently. Concrete system designs (hedge fund desk, FDE data platform, LLM authorization) compose those patterns. Adding a new system means assembling existing building blocks with domain-specific glue.

Everything here is designed to be **developed and run locally** — no cloud accounts required.

## Repository Structure

```
design/
├── principles/          Universal design philosophy
├── patterns/            Reusable, technology-specific implementation patterns
│   ├── data-access/     SQLAlchemy, Alembic, TimescaleDB, pooling, multi-tenancy
│   ├── api/             FastAPI, OpenAPI, authentication, authorization, versioning
│   ├── authorization/   OpenFGA, policy-as-code, ABAC
│   ├── messaging/       Kafka topology, schema registry, exactly-once semantics
│   ├── data-processing/ Ingestion, CDC, ETL/ELT, data quality
│   ├── ai-ml/           LLM gateway, RAG, embeddings, prompt management
│   ├── resilience/      Circuit breakers, retries, idempotency, bulkheads
│   ├── observability/   Logging, tracing, metrics, alerting
│   ├── testing/         Integration, contract, fixtures, testcontainers
│   └── modularity/      Tach, module interfaces, shared kernel
├── data-strategies/     Cross-cutting data architecture
├── infrastructure/      Local dev and operational patterns
└── systems/             Concrete system designs
    ├── hedge-fund-desk/ PM desk platform (market data, positions, risk, compliance)
    ├── fde-data-platform/ Forward-deployed engineering data platform
    └── llm-platform/    LLM-powered application with fine-grained authorization
```

## How to Read This Repository

Each document follows a consistent structure:

1. **Context & Problem** — why this matters
2. **Design Decisions** — options considered, tradeoffs, rationale
3. **Architecture** — Mermaid diagrams, component relationships
4. **Domain Model** — entities, value objects, aggregates
5. **Interface Contract** — Python Protocols, Pydantic schemas, event schemas
6. **Code Skeleton** — real, runnable Python
7. **Failure Modes** — what breaks, blast radius, recovery
8. **Performance Profile** — latency, throughput, scaling
9. **Observability** — metrics, logs, traces
10. **Testing Approach** — verification strategy
11. **Local Development** — Docker, seed data, how to run

Not every section applies to every document. Principles documents are shorter and more conceptual. Pattern documents include code. System documents compose patterns and add domain context.

## How Patterns Compose Into Systems

System designs reference the patterns they use:

```
systems/hedge-fund-desk/position-keeping.md
  → principles/cqrs-event-sourcing.md
  → patterns/data-access/sqlalchemy-repository.md
  → patterns/messaging/kafka-topology.md
  → patterns/resilience/idempotency.md

systems/llm-platform/openfga-document-access.md
  → patterns/authorization/openfga-modeling.md
  → patterns/ai-ml/rag-architecture.md
  → patterns/data-processing/embedding-pipelines.md
```

## Tech Stack

The patterns and systems in this repo are built around:

- **Python** — primary language
- **FastAPI** — API layer
- **SQLAlchemy + Alembic** — data access and migrations
- **PostgreSQL** — transactional data
- **TimescaleDB** — time-series data
- **Apache Kafka** — event streaming
- **OpenFGA** — fine-grained authorization
- **Docker Compose** — local infrastructure
- **Tach** — module boundary enforcement
