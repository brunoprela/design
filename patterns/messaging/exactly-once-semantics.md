# Exactly-Once Semantics

## Context & Problem

In a distributed system, message delivery has three guarantees: at-most-once, at-least-once, and exactly-once. Most systems use at-least-once (messages may be duplicated but never lost). For financial operations — trade settlement, position updates, cash movements — duplicates can cause real financial harm.

Kafka supports exactly-once semantics (EOS) through idempotent producers and transactional consumers. This is not free — it adds latency and complexity — so it should be used selectively on critical paths.

## Design Decisions

### Where to Use Exactly-Once

| Path | Guarantee | Rationale |
|---|---|---|
| Trade execution → position update | Exactly-once | Duplicate would double-count position |
| Market data price updates | At-least-once + idempotent consumer | Duplicate price is harmless (latest wins) |
| Audit log writes | At-least-once | Duplicate audit entry is acceptable |
| Metrics and analytics | At-most-once or at-least-once | Approximate is fine |

### Idempotent Producers

An idempotent producer ensures that retries do not create duplicates. Kafka assigns a sequence number to each message and deduplicates on the broker:

```python
producer = Producer({
    "bootstrap.servers": bootstrap_servers,
    "enable.idempotence": True,   # Enables exactly-once production
    "acks": "all",                # Required for idempotence
    "retries": 5,
    "max.in.flight.requests.per.connection": 5,
})
```

This is low-cost and should be enabled for all producers.

### Transactional Consume-Transform-Produce

For the consume → process → produce pattern (e.g., consume trade event, update position, produce position-changed event), Kafka transactions ensure atomicity:

```python
from confluent_kafka import Consumer, Producer


class TransactionalProcessor:
    def __init__(self, bootstrap_servers: str, group_id: str) -> None:
        self._consumer = Consumer({
            "bootstrap.servers": bootstrap_servers,
            "group.id": group_id,
            "enable.auto.commit": False,
            "isolation.level": "read_committed",  # Only read committed messages
        })

        self._producer = Producer({
            "bootstrap.servers": bootstrap_servers,
            "transactional.id": f"{group_id}-producer",  # Unique per producer
            "enable.idempotence": True,
        })
        self._producer.init_transactions()

    def process(self, handler) -> None:
        while True:
            msg = self._consumer.poll(1.0)
            if msg is None or msg.error():
                continue

            try:
                # Begin transaction
                self._producer.begin_transaction()

                # Process and produce output
                result = handler(msg)
                if result:
                    self._producer.produce(
                        topic=result["topic"],
                        key=result["key"],
                        value=result["value"],
                    )

                # Commit consumer offsets and producer messages atomically
                self._producer.send_offsets_to_transaction(
                    self._consumer.position(self._consumer.assignment()),
                    self._consumer.consumer_group_metadata(),
                )
                self._producer.commit_transaction()

            except Exception:
                self._producer.abort_transaction()
                raise
```

The key: consumer offset commit and producer message publish happen in one atomic transaction. Either both succeed or neither does.

### Idempotent Consumers (Alternative)

If Kafka transactions are too complex or too slow, use at-least-once delivery with idempotent consumers:

```python
class IdempotentHandler:
    """Uses event_id to deduplicate at the consumer level."""

    def __init__(self, session_factory) -> None:
        self._session_factory = session_factory

    async def handle(self, event: dict) -> None:
        event_id = event["event_id"]

        async with self._session_factory() as session:
            # Check if already processed
            existing = await session.execute(
                select(ProcessedEvent).where(ProcessedEvent.event_id == event_id)
            )
            if existing.scalar():
                return  # Already processed, skip

            # Process the event
            await self._apply_event(session, event)

            # Record that we processed it (in the same transaction)
            session.add(ProcessedEvent(event_id=event_id, processed_at=datetime.utcnow()))
            await session.commit()
```

This is simpler than Kafka transactions and works across any message broker, but requires a deduplication table per consumer.

## Performance Impact

| Configuration | Throughput Impact | Latency Impact |
|---|---|---|
| Idempotent producer | <5% reduction | Negligible |
| Transactional produce | 10-20% reduction | +5-10ms per transaction |
| `isolation.level=read_committed` | Slight reduction | May add ms if transactions are in-flight |

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Transaction timeout | Processing too slow | Increase `transaction.timeout.ms`, or simplify processing |
| Zombie producer | Old producer instance with same transactional.id | Kafka fences zombies automatically via epoch |
| Deduplication table growth | Processed event records accumulate | Periodic cleanup of old records (older than retention period) |

## Related Documents

- [Kafka Topology](kafka-topology.md) — topic configuration for transactions
- [Idempotency](../resilience/idempotency.md) — consumer-side deduplication
- [Event-Driven Architecture](../../principles/event-driven-architecture.md) — delivery guarantees
