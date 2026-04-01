# Contract Testing

## Context & Problem

Module A calls Module B's API. Both have their own test suites. Both pass. But A expects a field called `total` and B returns `totalAmount`. Integration test would catch this — but integration tests are slow, late, and test the whole stack.

Contract testing verifies that two sides agree on their shared interface without running both together. The consumer defines what it expects (the contract). The provider verifies it can satisfy the contract. If the contract breaks, the test fails before deployment.

## Design Decisions

### Consumer-Driven Contracts

The consumer (caller) writes a contract describing the requests it makes and the responses it expects. The provider (callee) runs the contract against its implementation. This ensures the provider does not break anything the consumer depends on.

### Schemathesis for API Contract Testing

For OpenAPI-based APIs, Schemathesis generates test cases from the OpenAPI spec:

```python
# test_api_contract.py
import schemathesis

schema = schemathesis.from_url("http://localhost:8000/openapi.json")


@schema.parametrize()
def test_api_contract(case):
    """Auto-generated tests from OpenAPI spec.
    Tests: valid inputs return expected status codes, response matches schema."""
    response = case.call()
    case.validate_response(response)
```

This catches:
- Responses that do not match the declared schema
- Endpoints that return undocumented status codes
- Validation errors on valid-looking inputs

### Module Interface Contract Testing

Within the modular monolith, contract tests verify that module implementations satisfy their Protocols:

```python
from typing import runtime_checkable, Protocol


@runtime_checkable
class MarketDataReader(Protocol):
    async def get_latest_price(self, instrument_id: str) -> PriceSnapshot: ...


def test_postgres_repo_satisfies_protocol():
    """Verify the implementation satisfies the Protocol at type level."""
    assert isinstance(PostgresPriceRepository(...), MarketDataReader)


async def test_market_data_contract():
    """Verify the implementation returns data in the expected shape."""
    repo = PostgresPriceRepository(test_session_factory)
    # Seed test data
    ...
    price = await repo.get_latest_price("AAPL")

    assert isinstance(price, PriceSnapshot)
    assert price.instrument_id == "AAPL"
    assert price.bid > 0
    assert price.ask >= price.bid
    assert price.mid == (price.bid + price.ask) / 2
```

### Event Contract Testing

For Kafka events, verify that produced events match the registered schema:

```python
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer


def test_trade_event_matches_schema():
    """Verify that the event we produce matches the registered Avro schema."""
    registry = SchemaRegistryClient({"url": "http://localhost:8081"})
    serializer = AvroSerializer(registry, TRADE_EXECUTED_SCHEMA)

    event = {
        "event_id": "evt-1",
        "event_version": 1,
        "timestamp": 1710500000000,
        "trade_id": "T-123",
        "portfolio_id": "P-1",
        "instrument_id": "AAPL",
        "side": "BUY",
        "quantity": "1000",
        "price": "150.00",
        "currency": "USD",
    }

    # This will raise if the event does not match the schema
    serialized = serializer(event, SerializationContext("trades.executed", MessageField.VALUE))
    assert serialized is not None
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Contract too strict | Tests break on non-breaking changes (new optional field) | Test required fields only, use "contains" not "equals" |
| Contract too loose | Does not catch real incompatibilities | Include specific field names, types, and business invariants |
| Stale contracts | Consumer evolves, contract not updated | Run contract tests in CI for both consumer and provider |

## Related Documents

- [OpenAPI Contracts](../api/openapi-contracts.md) — API contract spec
- [Schema Registry](../messaging/schema-registry.md) — event contract enforcement
- [Contract-First Design](../../principles/contract-first-design.md) — the underlying principle
