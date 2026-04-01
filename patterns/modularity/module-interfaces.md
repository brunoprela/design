# Module Interfaces

## Context & Problem

In a modular monolith, modules must interact without coupling to each other's internals. If the positions module directly imports the market data module's `PostgresPriceRepository`, it depends on the implementation, not the contract. Changing the repository requires changing every consumer.

Module interfaces define the contract: what a module offers and what shape the data takes. Other modules depend on this contract, never on the implementation behind it.

## Design Decisions

### Python Protocols as Contracts

Protocols (PEP 544) define structural interfaces. Any class with the right methods satisfies the Protocol — no inheritance required:

```python
# modules/market_data/interface.py

from typing import Protocol
from datetime import datetime
from decimal import Decimal

from pydantic import BaseModel


class PriceSnapshot(BaseModel):
    """Value object — the data shape exposed to other modules."""
    instrument_id: str
    bid: Decimal
    ask: Decimal
    mid: Decimal
    timestamp: datetime
    source: str


class MarketDataReader(Protocol):
    """What other modules can ask the market data module."""

    async def get_latest_price(self, instrument_id: str) -> PriceSnapshot: ...
    async def get_price_at(self, instrument_id: str, timestamp: datetime) -> PriceSnapshot: ...
    async def get_price_history(
        self,
        instrument_id: str,
        start: datetime,
        end: datetime,
    ) -> list[PriceSnapshot]: ...
```

### Interface Design Rules

1. **Expose behavior, not structure** — `get_latest_price()` not `get_session()`. Consumers should not know about the database.

2. **Use value objects at the boundary** — Pydantic `BaseModel` instances, not SQLAlchemy model instances. Returning ORM objects leaks the persistence layer.

3. **Keep interfaces small** — each Protocol should represent one coherent capability. Split `MarketDataReader` and `MarketDataWriter` rather than one large `MarketDataService`.

4. **Immutable at the boundary** — data objects crossing module boundaries should be immutable (Pydantic models with `frozen=True` or plain dataclasses).

### Module Exports

Each module's `__init__.py` explicitly exports only its public interface:

```python
# modules/market_data/__init__.py

from app.modules.market_data.interface import (
    MarketDataReader,
    PriceSnapshot,
)

__all__ = [
    "MarketDataReader",
    "PriceSnapshot",
]
```

Everything else in the module is private. Tach enforces that other modules only import from `__init__.py`.

### Consumer Usage

```python
# modules/positions/service.py

from app.modules.market_data import MarketDataReader, PriceSnapshot


class PositionService:
    def __init__(self, price_reader: MarketDataReader, ...) -> None:
        self._price_reader = price_reader

    async def get_market_value(self, portfolio_id: str, instrument_id: str) -> Decimal:
        position = await self._get_position(portfolio_id, instrument_id)
        price: PriceSnapshot = await self._price_reader.get_latest_price(instrument_id)
        return position.quantity * price.mid
```

The positions module knows `MarketDataReader` and `PriceSnapshot` — it does not know PostgreSQL, TimescaleDB, or any implementation detail.

### Wiring in the Composition Root

```python
# container.py

from app.modules.market_data.service import MarketDataServiceImpl
from app.modules.positions.service import PositionService

market_data = MarketDataServiceImpl(price_repo=PostgresPriceRepository(session_factory))
positions = PositionService(price_reader=market_data, ...)
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Leaky interface | Protocol method returns SQLAlchemy model | Return Pydantic BaseModel or dataclass |
| Fat interface | Protocol has 20+ methods | Split into focused Protocols (Reader, Writer, Subscriber) |
| Implementation drift | Implementation method signature diverges from Protocol | Run mypy/pyright in CI with strict Protocol checking |
| Circular interfaces | Module A's interface references Module B's types | Use shared kernel types, or events for decoupling |

## Related Documents

- [Dependency Inversion](../../principles/dependency-inversion.md) — the principle behind Protocol-based interfaces
- [Tach Enforcement](tach-enforcement.md) — enforcing that modules use interfaces
- [Shared Kernel](shared-kernel.md) — types shared across module interfaces
