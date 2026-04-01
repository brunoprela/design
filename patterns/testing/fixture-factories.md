# Fixture Factories

## Context & Problem

Tests need data. Constructing test objects by hand is verbose, fragile, and obscures the intent of the test:

```python
# Bad: every test manually constructs full objects
trade = TradeRecord(
    id=uuid4(), portfolio_id=uuid4(), instrument_id="AAPL", side="buy",
    quantity=Decimal("1000"), price=Decimal("150"), currency="USD",
    executed_at=datetime(2025, 3, 15, 14, 30), created_at=datetime.utcnow(),
)
```

Most of those fields are irrelevant to what the test is checking. A fixture factory provides sensible defaults and lets the test override only the fields that matter:

```python
# Good: test specifies only what matters
trade = make_trade(side="sell", quantity=Decimal("500"))
```

## Design Decisions

### Factory Functions Over Factory Libraries

Simple factory functions are preferred over factory libraries (factory_boy, etc.) for most codebases. They are explicit, have no learning curve, and are easy to customize:

```python
# tests/factories.py

from datetime import datetime
from decimal import Decimal
from uuid import uuid4


def make_trade(
    *,
    id=None,
    portfolio_id=None,
    instrument_id="AAPL",
    side="buy",
    quantity=Decimal("1000"),
    price=Decimal("150.00"),
    currency="USD",
    executed_at=None,
) -> TradeRecord:
    return TradeRecord(
        id=id or uuid4(),
        portfolio_id=portfolio_id or uuid4(),
        instrument_id=instrument_id,
        side=side,
        quantity=quantity,
        price=price,
        currency=currency,
        executed_at=executed_at or datetime.utcnow(),
    )


def make_position(
    *,
    portfolio_id=None,
    instrument_id="AAPL",
    quantity=Decimal("1000"),
    cost_basis=Decimal("150000"),
) -> Position:
    return Position(
        portfolio_id=portfolio_id or uuid4(),
        instrument_id=instrument_id,
        quantity=quantity,
        cost_basis=cost_basis,
    )


def make_price_quote(
    *,
    instrument_id="AAPL",
    bid=Decimal("149.50"),
    ask=Decimal("150.50"),
    mid=None,
    timestamp=None,
) -> PriceQuote:
    return PriceQuote(
        instrument_id=instrument_id,
        bid=bid,
        ask=ask,
        mid=mid or (bid + ask) / 2,
        timestamp=timestamp or datetime.utcnow(),
        source="test",
    )
```

### Usage in Tests

```python
async def test_sell_reduces_position():
    position = make_position(quantity=Decimal("1000"))
    trade = make_trade(side="sell", quantity=Decimal("300"))

    position.apply_trade(trade)

    assert position.quantity == Decimal("700")


async def test_position_valuation_uses_mid_price():
    position = make_position(instrument_id="AAPL", quantity=Decimal("100"))
    price = make_price_quote(instrument_id="AAPL", mid=Decimal("150.00"))

    value = position.market_value(price)

    assert value == Decimal("15000.00")
```

The test reads like a specification — only the relevant fields are visible.

### Factories for API Requests

```python
def make_trade_request(
    *,
    instrument_id="AAPL",
    side="buy",
    quantity="1000",
    price="150.00",
) -> dict:
    return {
        "instrument_id": instrument_id,
        "side": side,
        "quantity": quantity,
        "price": price,
    }


async def test_create_trade_endpoint(client):
    response = await client.post(
        "/api/v1/trades",
        json=make_trade_request(side="sell"),
    )
    assert response.status_code == 201
```

### Sequences for Unique Values

When tests need unique-but-predictable values:

```python
_counter = 0

def next_id() -> str:
    global _counter
    _counter += 1
    return f"test-{_counter}"


def make_instrument(*, id=None, symbol=None):
    _id = id or next_id()
    return Instrument(id=_id, symbol=symbol or f"SYM-{_id}")
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Factory drift | Factory defaults diverge from real constraints | Factories should produce valid objects by default |
| Over-specified tests | Tests override every field (defeating the purpose) | Only override what the test cares about |
| Hidden coupling | Factory creates related objects with hardcoded IDs | Use factory composition: `make_position(portfolio=make_portfolio())` |

## Related Documents

- [Integration Testing](integration-testing.md) — where factories are used
- [Test Containers](test-containers.md) — infrastructure for seeding test data
