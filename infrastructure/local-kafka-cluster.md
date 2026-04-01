# Local Kafka Cluster

## Context & Problem

Developing against Kafka requires a local broker, and often Schema Registry and a UI for inspecting topics. The local setup must be as close to production behavior as possible while remaining lightweight enough to run on a laptop.

## Design Decisions

### KRaft Mode (No ZooKeeper)

Kafka 3.3+ supports KRaft mode — ZooKeeper is no longer required. This simplifies the local setup from two containers to one:

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    ports:
      - "9092:9092"
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
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
      CLUSTER_ID: "local-dev-cluster-001"
    healthcheck:
      test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092"]
      interval: 10s
      timeout: 5s
      retries: 10
```

### Topic Management

Disable auto-creation (`KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"`) to catch misconfigured topic names early. Create topics explicitly via a script:

```bash
#!/bin/bash
# scripts/create-topics.sh

KAFKA_CONTAINER="kafka"

topics=(
  "trades.executed:6:1"
  "prices.updated:12:1"
  "positions.changed:6:1"
  "compliance.violations:3:1"
  "audit.events:6:1"
)

for topic_config in "${topics[@]}"; do
  IFS=':' read -r topic partitions replication <<< "$topic_config"
  docker exec $KAFKA_CONTAINER kafka-topics \
    --bootstrap-server localhost:9092 \
    --create \
    --topic "$topic" \
    --partitions "$partitions" \
    --replication-factor "$replication" \
    --if-not-exists
  echo "Created topic: $topic (partitions=$partitions)"
done
```

### Kafka UI

```yaml
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports: ["8090:8080"]
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_NAME: connect
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS: http://kafka-connect:8083
    depends_on:
      kafka:
        condition: service_healthy
```

Access at `http://localhost:8090` — browse topics, view messages, inspect consumer groups, manage connectors.

### CLI Cheat Sheet

```bash
# List topics
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --list

# Describe a topic
docker exec kafka kafka-topics --bootstrap-server localhost:9092 \
  --describe --topic trades.executed

# Produce a test message
echo '{"trade_id":"T-1","instrument":"AAPL","quantity":100}' | \
  docker exec -i kafka kafka-console-producer \
    --bootstrap-server localhost:9092 \
    --topic trades.executed

# Consume from beginning
docker exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic trades.executed \
  --from-beginning \
  --max-messages 10

# Check consumer group lag
docker exec kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group positions-service \
  --describe
```

### Python Development Helper

```python
# scripts/kafka_dev.py — seed Kafka with test events

import json
import time
from decimal import Decimal
from datetime import datetime, timezone

from confluent_kafka import Producer

producer = Producer({"bootstrap.servers": "localhost:9092"})

instruments = ["AAPL", "MSFT", "GOOGL", "AMZN", "TSLA"]

for i in range(100):
    for instrument in instruments:
        event = {
            "event_id": f"evt-{i}-{instrument}",
            "event_type": "price.updated",
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "data": {
                "instrument_id": instrument,
                "bid": str(Decimal("150.00") + Decimal(str(i * 0.1))),
                "ask": str(Decimal("150.10") + Decimal(str(i * 0.1))),
                "mid": str(Decimal("150.05") + Decimal(str(i * 0.1))),
                "source": "test",
            },
        }
        producer.produce(
            "prices.updated",
            key=instrument.encode(),
            value=json.dumps(event).encode(),
        )
    producer.flush()
    time.sleep(0.1)

print("Seeded 500 price events")
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Broker won't start | Port 9092 already in use | `lsof -i :9092`, stop conflicting process |
| Consumer group rebalance loop | Consumer crashing and restarting | Check consumer logs, increase `session.timeout.ms` |
| Out of disk | Retention too long for local disk | Reduce `KAFKA_LOG_RETENTION_HOURS` for local dev |
| Slow startup | Cluster ID changed, metadata mismatch | Remove kafka volume: `docker compose down -v` |

## Related Documents

- [Docker Compose Patterns](docker-compose-patterns.md) — full local environment
- [Kafka Topology](../patterns/messaging/kafka-topology.md) — topic and partition design
- [Schema Registry](../patterns/messaging/schema-registry.md) — schema governance
