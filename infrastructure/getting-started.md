# Getting Started

## Context & Problem

A new developer cloning this repository needs to get the full local development environment running. Without a clear sequence, they must read 3+ infrastructure documents and guess the correct order.

## Prerequisites

- Docker Desktop (4.0+) with at least **12GB RAM** allocated (Settings → Resources)
  - Minimum: 8GB RAM (core profile only)
  - Recommended: 16GB RAM (full profile with observability)
- Python 3.12+
- Docker Compose v2

## Hardware Requirements

| Profile | Services | RAM | Disk | CPU |
|---|---|---|---|---|
| `core` | PostgreSQL, Redis, Kafka | ~2GB | ~1GB | 2 cores |
| `full` | + Schema Registry, Connect, OpenFGA | ~4GB | ~2GB | 4 cores |
| `observability` | + Prometheus, Grafana, Jaeger | ~6GB | ~3GB | 4 cores |
| `all` | Everything including Elasticsearch, MinIO | ~10GB | ~5GB | 6 cores |

## Quick Start (5 minutes)

### Step 1: Start Infrastructure
```bash
# Start core services (PostgreSQL + TimescaleDB, Redis, Kafka)
docker compose --profile core up -d

# Verify services are healthy
docker compose ps
```

### Step 2: Initialize Databases
```bash
# The init-db.sql script runs automatically on first start.
# To run migrations manually:
alembic upgrade head
```

### Step 3: Create Kafka Topics
```bash
# Run the topic creation script
./scripts/create-topics.sh
```

### Step 4: Seed Test Data
```bash
# Install Python dependencies
pip install -r requirements-dev.txt

# Seed instruments, historical prices, and reference data
python scripts/seed_market_data.py --instruments 100 --days 504
```

### Step 5: Verify
```bash
# Check PostgreSQL
psql postgresql://app:app@localhost:5432/app -c "SELECT count(*) FROM security_master.instruments;"

# Check Kafka topics
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --list

# Check Redis
docker exec redis redis-cli ping
```

## Profiles

| Profile | Use Case |
|---|---|
| `core` | Day-to-day development |
| `full` | Testing authorization (OpenFGA), schema evolution |
| `observability` | Debugging latency, tracing request flows |
| `all` | Full system integration testing |

## Stopping

```bash
docker compose --profile core down        # Stop, keep data
docker compose --profile core down -v     # Stop and delete all data
```

## Troubleshooting

| Problem | Solution |
|---|---|
| Kafka not starting | Ensure port 9092 is free: `lsof -i :9092` |
| PostgreSQL OOM | Increase Docker RAM allocation to 8GB+ |
| Slow startup | First pull takes ~5min for images. Subsequent starts are <30s |
| Schema Registry errors | Start with `--profile full` not `--profile core` |

## Related Documents

- [Docker Compose Patterns](docker-compose-patterns.md) — profiles, health checks, and the full compose file
- [Local Kafka Cluster](local-kafka-cluster.md) — Kafka-specific setup, topic management, and CLI cheat sheet
- [Local PostgreSQL & TimescaleDB](local-postgres-timescale.md) — database configuration, migrations, and seed data
