# Docker Compose Patterns

## Context & Problem

A modular system depends on multiple infrastructure services — PostgreSQL, TimescaleDB, Kafka, Redis, OpenFGA, Schema Registry, Elasticsearch. Without a reproducible local setup, developers waste hours configuring services, debugging version mismatches, and explaining "you need to install X first."

Docker Compose provides a single `docker compose up` that starts the entire local environment identically on every machine.

## Design Decisions

### Profiles for Selective Startup

Not every developer needs every service. A frontend developer does not need Kafka. A data engineer does not need OpenFGA. Profiles let developers start only what they need:

```yaml
# docker-compose.yml

services:
  # === Core (always started) ===
  postgres:
    image: timescale/timescaledb:latest-pg16
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s

  # === Messaging (profile: messaging) ===
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    profiles: [messaging, full]
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      CLUSTER_ID: "local-dev-cluster-001"
    healthcheck:
      test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092"]
      interval: 10s
      timeout: 5s
      retries: 10

  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    profiles: [messaging, full]
    ports: ["8081:8081"]
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      kafka:
        condition: service_healthy

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.6.0
    profiles: [messaging, full]
    ports: ["8083:8083"]
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: connect-local
      CONNECT_CONFIG_STORAGE_TOPIC: connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: connect-status
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_PLUGIN_PATH: /usr/share/java,/usr/share/confluent-hub-components
    depends_on:
      kafka:
        condition: service_healthy
      schema-registry:
        condition: service_started

  # === Authorization (profile: auth) ===
  openfga:
    image: openfga/openfga:latest
    profiles: [auth, full]
    ports:
      - "8080:8080"
      - "3000:3000"
    command: run
    environment:
      OPENFGA_DATASTORE_ENGINE: postgres
      OPENFGA_DATASTORE_URI: postgres://app:dev@postgres:5432/openfga
      OPENFGA_PLAYGROUND_ENABLED: "true"
    depends_on:
      postgres:
        condition: service_healthy

  # === Observability (profile: observability) ===
  prometheus:
    image: prom/prometheus:latest
    profiles: [observability, full]
    ports: ["9090:9090"]
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    profiles: [observability, full]
    ports: ["3001:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: dev
      GF_AUTH_ANONYMOUS_ENABLED: "true"
    volumes:
      - ./config/grafana/dashboards:/var/lib/grafana/dashboards
      - ./config/grafana/provisioning:/etc/grafana/provisioning

  jaeger:
    image: jaegertracing/all-in-one:latest
    profiles: [observability, full]
    ports:
      - "16686:16686"
      - "4317:4317"
    environment:
      COLLECTOR_OTLP_ENABLED: "true"

  # === Search (profile: search) ===
  elasticsearch:
    image: elasticsearch:8.13.0
    profiles: [search, full]
    ports: ["9200:9200"]
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"

  # === Object Storage (profile: storage) ===
  minio:
    image: minio/minio:latest
    profiles: [storage, full]
    ports:
      - "9000:9000"
      - "9001:9001"
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin

  # === Policy Engine (profile: policy) ===
  opa:
    image: openpolicyagent/opa:latest
    profiles: [policy, full]
    ports: ["8181:8181"]
    command: run --server --watch /policies
    volumes:
      - ./policy:/policies

volumes:
  postgres_data:
```

### Usage

```bash
# Core only (Postgres + Redis)
docker compose up

# Core + messaging
docker compose --profile messaging up

# Everything
docker compose --profile full up

# Specific combination
docker compose --profile messaging --profile auth up

# Background
docker compose --profile full up -d

# Tear down and remove volumes
docker compose --profile full down -v
```

### Database Initialization Script

```sql
-- scripts/init-db.sql

-- Create schemas for each bounded context
CREATE SCHEMA IF NOT EXISTS market_data;
CREATE SCHEMA IF NOT EXISTS positions;
CREATE SCHEMA IF NOT EXISTS risk;
CREATE SCHEMA IF NOT EXISTS compliance;
CREATE SCHEMA IF NOT EXISTS audit;

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS vector;  -- pgvector for embeddings
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create OpenFGA database
CREATE DATABASE openfga;

-- Create a read-only user for analytics
CREATE ROLE analyst WITH LOGIN PASSWORD 'dev';
GRANT USAGE ON SCHEMA market_data, positions, risk TO analyst;
ALTER DEFAULT PRIVILEGES IN SCHEMA market_data, positions, risk
    GRANT SELECT ON TABLES TO analyst;
```

### Health Check Dependencies

Services start in dependency order using health checks:

```mermaid
graph TD
    PG[postgres] --> SR[schema-registry]
    PG --> FGA[openfga]
    K[kafka] --> SR
    K --> KC[kafka-connect]
    SR --> KC
```

`depends_on` with `condition: service_healthy` ensures downstream services do not start until their dependencies are ready — not just running.

### Environment Variables

Use a `.env` file for configuration:

```bash
# .env
POSTGRES_DB=app
POSTGRES_USER=app
POSTGRES_PASSWORD=dev
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
REDIS_URL=redis://localhost:6379
SCHEMA_REGISTRY_URL=http://localhost:8081
OPENFGA_API_URL=http://localhost:8080
```

```yaml
# docker-compose.yml (excerpt)
services:
  postgres:
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Port conflict | Another service using same port | Use unique ports, check `lsof -i :5432` |
| Container OOM | Service exceeds memory limit | Set `mem_limit`, tune JVM heap for Kafka/ES |
| Volume permission error | Host filesystem permissions | Use named volumes, avoid bind mounts for data |
| Stale containers | Old image cached | `docker compose pull` before `up` |
| Startup order race | Service starts before dependency is ready | Health checks + `depends_on: condition: service_healthy` |

## Related Documents

- [Local Kafka Cluster](local-kafka-cluster.md) — Kafka-specific local setup
- [Local Postgres TimescaleDB](local-postgres-timescale.md) — database-specific local setup
- [Secret Management](secret-management.md) — managing credentials beyond `.env`
