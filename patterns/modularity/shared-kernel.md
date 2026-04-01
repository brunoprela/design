# Shared Kernel

## Context & Problem

Modules in a modular monolith need some common types. Money, timestamps, entity IDs, base event classes — these appear in every module's interface. Defining them per module creates duplication and translation overhead. But a large shared module defeats the purpose of boundaries.

The shared kernel is a small, stable, jointly-owned set of types that multiple modules depend on. It is the exception to the "no shared dependencies" rule, and its scope must be carefully controlled.

## Design Decisions

### What Belongs in the Shared Kernel

**Include:**
- Value objects that appear in multiple module interfaces (`Money`, `Quantity`, `InstrumentId`)
- Base types for the event system (`DomainEvent`, `EventMetadata`)
- Shared infrastructure interfaces (`SessionFactory`, `EventPublisher`)
- Common enums used across modules (`Currency`, `Side`)

**Do not include:**
- Domain models (they belong to their bounded context)
- Business logic (it belongs to a specific module)
- Utility functions that only one module uses
- Configuration or settings

### Structure

```python
# shared/types.py — value objects

from decimal import Decimal
from typing import NewType

from pydantic import BaseModel, ConfigDict

InstrumentId = NewType("InstrumentId", str)
PortfolioId = NewType("PortfolioId", str)


class Money(BaseModel):
    model_config = ConfigDict(frozen=True)

    amount: Decimal
    currency: str

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(f"Cannot add {self.currency} and {other.currency}")
        return Money(amount=self.amount + other.amount, currency=self.currency)

    def __mul__(self, factor: Decimal) -> "Money":
        return Money(amount=self.amount * factor, currency=self.currency)
```

```python
# shared/events.py — base event types

from datetime import datetime
from uuid import UUID, uuid4

from pydantic import BaseModel, Field


class EventMetadata(BaseModel):
    event_id: UUID = Field(default_factory=uuid4)
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    correlation_id: UUID | None = None
    causation_id: UUID | None = None


class DomainEvent(BaseModel):
    """Base class for all domain events."""
    metadata: EventMetadata = Field(default_factory=EventMetadata)
    event_type: str
    event_version: int = 1
```

```python
# shared/interfaces.py — infrastructure contracts

from typing import Protocol


class EventPublisher(Protocol):
    async def publish(self, topic: str, key: str, event: DomainEvent) -> None: ...


class UnitOfWork(Protocol):
    async def commit(self) -> None: ...
    async def rollback(self) -> None: ...
```

### Size Control

The shared kernel should be small enough that any developer can read all of it in 10 minutes. Metrics to watch:

- **File count:** ≤5 files
- **Line count:** ≤500 lines total
- **Import count:** ≤10 external dependencies
- **Change frequency:** rarely (if it changes often, something is in the wrong place)

### Review Process

Changes to the shared kernel affect all modules. They require:

1. Review from at least one developer from each consuming module
2. Backward compatibility (adding is fine, removing or renaming requires migration)
3. Justification for why this belongs in shared rather than a specific module

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Kernel bloat | Convenience imports, "just put it in shared" | Size limits, require justification for additions |
| Breaking change | Shared type renamed or restructured | Treat shared kernel changes like API changes — backward compatible only |
| Hidden coupling | Shared kernel imports a specific module | Shared kernel must depend on nothing except standard library and Pydantic |
| Becoming a god module | All logic gravitates to shared for reuse | Shared kernel has types and interfaces only — no business logic |

## Related Documents

- [Module Interfaces](module-interfaces.md) — interfaces that use shared kernel types
- [Tach Enforcement](tach-enforcement.md) — shared kernel as a declared dependency
- [Bounded Contexts](../../principles/bounded-contexts.md) — why modules have separate models
