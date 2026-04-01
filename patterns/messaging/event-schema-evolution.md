# Event Schema Evolution

## Context & Problem

Event schemas change as the system evolves. New fields are added, old fields become irrelevant, types need adjustment. Unlike database schemas (where you control both writer and reader), event schemas have multiple independent consumers that deploy on different schedules.

A schema change that breaks even one consumer causes a production incident. Schema evolution strategies ensure that producers and consumers can evolve independently without coordinated deployments.

## Design Decisions

### Evolution Rules

| Change | Backward Compatible | Forward Compatible | Strategy |
|---|---|---|---|
| Add optional field (with default) | Yes | Yes | Safe — do it |
| Add required field (no default) | No | Yes | Add with default first, make required later |
| Remove field | Yes | No | Deprecate with default, remove after all consumers migrate |
| Rename field | No | No | Add new field, deprecate old, remove old later |
| Change field type | No | No | New event version |

### Versioned Events

When a breaking change is unavoidable, create a new event version:

```python
# v1 — original
class TradeExecutedV1(BaseModel):
    event_type: str = "trade.executed"
    event_version: int = 1
    trade_id: str
    quantity: Decimal
    price: Decimal

# v2 — breaking change: price split into components
class TradeExecutedV2(BaseModel):
    event_type: str = "trade.executed"
    event_version: int = 2
    trade_id: str
    quantity: Decimal
    gross_price: Decimal     # renamed from price
    commission: Decimal      # new field
    net_price: Decimal       # new field
```

### Upcasters

Upcasters transform old events into the latest version at read time. This lets consumers always work with the latest schema:

```python
class TradeEventUpcaster:
    def upcast(self, event: dict) -> dict:
        version = event.get("event_version", 1)

        if version == 1:
            event = self._v1_to_v2(event)
        # Add more version steps as needed
        return event

    def _v1_to_v2(self, event: dict) -> dict:
        """Transform v1 → v2: split price into components."""
        price = Decimal(str(event["price"]))
        return {
            **event,
            "event_version": 2,
            "gross_price": str(price),
            "commission": "0",
            "net_price": str(price),
        }
```

### Dual-Publish Transition

For major schema changes, publish both old and new versions during a transition period:

```
Phase 1: Producer publishes v1
Phase 2: Producer publishes v1 AND v2 (dual-publish)
Phase 3: Consumers migrate to v2
Phase 4: Producer stops publishing v1
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Consumer crashes on new field | Consumer uses strict deserialization | Use lenient deserialization (ignore unknown fields) |
| Data loss on old field removal | Consumer depended on removed field | Deprecation period, monitor field usage |
| Upcaster bug | Incorrect transformation logic | Unit test upcasters with real historical events |
| Version confusion | Multiple versions in flight | Monitor active versions per topic, sunset old versions |

## Related Documents

- [Schema Registry](schema-registry.md) — enforcing compatibility rules
- [Contract-First Design](../../principles/contract-first-design.md) — schemas as contracts
- [Kafka Topology](kafka-topology.md) — topic configuration for versioned events
