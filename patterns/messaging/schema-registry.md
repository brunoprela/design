# Schema Registry

## Context & Problem

Kafka stores bytes. Without schema governance, producers and consumers must agree on serialization format through convention — which breaks silently when someone changes a field. A producer adds a field, renames another, and consumers crash or silently produce wrong results.

Schema Registry is a centralized service that stores and enforces schemas for Kafka topics. Producers register schemas before publishing. Consumers fetch schemas for deserialization. Compatibility rules prevent breaking changes.

## Design Decisions

### Serialization Format

| Format | Schema Evolution | Performance | Human Readable | Ecosystem |
|---|---|---|---|---|
| **Avro** | Excellent | Fast (binary) | No | Best Kafka integration |
| **Protobuf** | Excellent | Fast (binary) | No | Strong cross-language |
| **JSON Schema** | Good | Slower (text) | Yes | Easiest to adopt |

**Avro** is the default choice for Kafka — best compatibility support in Confluent Schema Registry, compact binary encoding, and strong schema evolution rules.

### Compatibility Modes

| Mode | Allows | Use Case |
|---|---|---|
| **BACKWARD** (default) | New schema can read old data | Consumers deploy before producers |
| **FORWARD** | Old schema can read new data | Producers deploy before consumers |
| **FULL** | Both backward and forward | Safest — any deployment order works |
| **NONE** | Any change | Development/testing only |

**FULL** compatibility is recommended for production. It guarantees that any consumer version can read data from any producer version.

### Subject Naming

```
<topic>-key     — schema for message keys
<topic>-value   — schema for message values
```

Example: topic `trades.executed` has subjects `trades.executed-key` and `trades.executed-value`.

## Code Skeleton

### Avro Schema Definition

```json
{
  "type": "record",
  "name": "TradeExecuted",
  "namespace": "com.platform.trades",
  "fields": [
    {"name": "event_id", "type": "string"},
    {"name": "event_version", "type": "int", "default": 1},
    {"name": "timestamp", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "trade_id", "type": "string"},
    {"name": "portfolio_id", "type": "string"},
    {"name": "instrument_id", "type": "string"},
    {"name": "side", "type": {"type": "enum", "name": "Side", "symbols": ["BUY", "SELL"]}},
    {"name": "quantity", "type": "string"},
    {"name": "price", "type": "string"},
    {"name": "currency", "type": "string", "default": "USD"}
  ]
}
```

### Producer with Schema Registry

```python
from confluent_kafka import SerializingProducer
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer


def create_avro_producer(
    bootstrap_servers: str,
    schema_registry_url: str,
    schema_str: str,
) -> SerializingProducer:
    schema_registry = SchemaRegistryClient({"url": schema_registry_url})
    avro_serializer = AvroSerializer(schema_registry, schema_str)

    return SerializingProducer({
        "bootstrap.servers": bootstrap_servers,
        "acks": "all",
        "enable.idempotence": True,
        "value.serializer": avro_serializer,
    })
```

### Consumer with Schema Registry

```python
from confluent_kafka import DeserializingConsumer
from confluent_kafka.schema_registry.avro import AvroDeserializer


def create_avro_consumer(
    bootstrap_servers: str,
    schema_registry_url: str,
    group_id: str,
) -> DeserializingConsumer:
    schema_registry = SchemaRegistryClient({"url": schema_registry_url})
    avro_deserializer = AvroDeserializer(schema_registry)

    return DeserializingConsumer({
        "bootstrap.servers": bootstrap_servers,
        "group.id": group_id,
        "auto.offset.reset": "earliest",
        "value.deserializer": avro_deserializer,
    })
```

## Schema Evolution Examples

**Safe: adding an optional field**
```json
{"name": "venue", "type": ["null", "string"], "default": null}
```

**Safe: adding a default value**
```json
{"name": "currency", "type": "string", "default": "USD"}
```

**Unsafe: removing a field without default** — breaks backward compatibility.

**Unsafe: changing field type** — `string` → `int` breaks everything.

## Local Development

```yaml
services:
  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    ports: ["8081:8081"]
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
    depends_on: [kafka]
```

Test compatibility before deploying:
```bash
# Check if a new schema is compatible
curl -X POST http://localhost:8081/compatibility/subjects/trades.executed-value/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "{...}"}'
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Incompatible schema registered | Compatibility check disabled | Always use FULL or BACKWARD mode in production |
| Schema Registry down | Service failure | Producers/consumers cache schemas locally, temporary outage is tolerable |
| Schema ID not found | Consumer encounters unknown schema version | Ensure Schema Registry has all historical schemas |
| Deserialization failure | Schema mismatch between producer/consumer | Schema Registry enforces compatibility before registration |

## Related Documents

- [Kafka Topology](kafka-topology.md) — topic and partition design
- [Event Schema Evolution](event-schema-evolution.md) — strategies for evolving schemas over time
- [Contract-First Design](../../principles/contract-first-design.md) — schemas as contracts
