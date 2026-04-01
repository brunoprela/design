# FastAPI Modular Layout

## Context & Problem

FastAPI applications that start in a single `main.py` quickly become unmanageable. Routes, models, services, and dependencies are mixed together. Adding a new feature means scrolling through thousands of lines or guessing which file to put things in.

A modular layout aligns the FastAPI application structure with bounded contexts. Each module owns its routes, schemas, services, and dependencies. The main application composes modules by including their routers.

## Design Decisions

### Module-Per-Router

Each bounded context registers its own `APIRouter`. The main app includes routers — it does not define routes directly:

```
app/
├── main.py                      # Composition root: creates app, includes routers
├── config.py                    # Settings via pydantic-settings
├── container.py                 # Dependency wiring
├── shared/
│   ├── database.py              # Engine, session factory
│   ├── types.py                 # Shared value objects
│   ├── middleware.py            # Cross-cutting middleware
│   └── exceptions.py           # Global exception handlers
└── modules/
    ├── market_data/
    │   ├── __init__.py
    │   ├── routes.py            # APIRouter with route handlers
    │   ├── schemas.py           # Pydantic request/response models
    │   ├── service.py           # Business logic
    │   ├── repository.py        # Data access
    │   ├── models.py            # SQLAlchemy models
    │   ├── interface.py         # Protocol for other modules
    │   └── dependencies.py      # FastAPI Depends() for this module
    ├── positions/
    │   ├── ...
    └── risk/
        ├── ...
```

### Route Registration

```python
# main.py

from fastapi import FastAPI

from app.modules.market_data.routes import router as market_data_router
from app.modules.positions.routes import router as positions_router
from app.modules.risk.routes import router as risk_router
from app.shared.exceptions import register_exception_handlers
from app.shared.middleware import register_middleware


def create_app() -> FastAPI:
    app = FastAPI(
        title="Platform API",
        version="1.0.0",
        docs_url="/docs",
    )

    register_middleware(app)
    register_exception_handlers(app)

    app.include_router(market_data_router, prefix="/api/v1/market-data", tags=["Market Data"])
    app.include_router(positions_router, prefix="/api/v1/positions", tags=["Positions"])
    app.include_router(risk_router, prefix="/api/v1/risk", tags=["Risk"])

    return app

app = create_app()
```

### Module Routes

```python
# modules/market_data/routes.py

from datetime import datetime
from fastapi import APIRouter, Depends, Query

from app.modules.market_data.schemas import PriceResponse, PriceHistoryResponse
from app.modules.market_data.dependencies import get_market_data_service
from app.modules.market_data.service import MarketDataService

router = APIRouter()


@router.get("/prices/{instrument_id}", response_model=PriceResponse)
async def get_latest_price(
    instrument_id: str,
    service: MarketDataService = Depends(get_market_data_service),
) -> PriceResponse:
    price = await service.get_latest_price(instrument_id)
    return PriceResponse.model_validate(price)


@router.get("/prices/{instrument_id}/history", response_model=PriceHistoryResponse)
async def get_price_history(
    instrument_id: str,
    start: datetime = Query(...),
    end: datetime = Query(...),
    service: MarketDataService = Depends(get_market_data_service),
) -> PriceHistoryResponse:
    prices = await service.get_price_history(instrument_id, start, end)
    return PriceHistoryResponse(prices=prices)
```

### Module Schemas (Pydantic v2)

```python
# modules/market_data/schemas.py

from datetime import datetime
from decimal import Decimal

from pydantic import BaseModel, ConfigDict


class PriceResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    instrument_id: str
    bid: Decimal
    ask: Decimal
    mid: Decimal
    timestamp: datetime
    source: str


class PriceHistoryResponse(BaseModel):
    prices: list[PriceResponse]
```

### Dependency Injection via FastAPI Depends

```python
# modules/market_data/dependencies.py

from functools import lru_cache

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.shared.database import get_session_factory
from app.modules.market_data.repository import PostgresPriceRepository
from app.modules.market_data.service import MarketDataService


async def get_market_data_service(
    session_factory=Depends(get_session_factory),
) -> MarketDataService:
    repo = PostgresPriceRepository(session_factory)
    return MarketDataService(price_repo=repo)
```

## URL Structure Convention

```
/api/v1/{module}/{resource}
/api/v1/{module}/{resource}/{id}
/api/v1/{module}/{resource}/{id}/{sub-resource}
```

Examples:
```
GET    /api/v1/positions/portfolios/abc/holdings
POST   /api/v1/positions/trades
GET    /api/v1/market-data/prices/AAPL
GET    /api/v1/risk/portfolios/abc/var
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Circular imports | Module A's routes import module B's service which imports A | Depend on interfaces (Protocols), not implementations |
| Fat route handlers | Business logic in route functions | Routes only do: parse request → call service → format response |
| Missing validation | Endpoint accepts `dict` instead of typed schema | Every endpoint has explicit request/response Pydantic models |
| Dependency graph explosion | Deep chains of `Depends()` | Keep dependency chains shallow (max 2-3 levels) |

## Related Documents

- [OpenAPI Contracts](openapi-contracts.md) — generated API documentation
- [Authentication MFA](authentication-mfa.md) — securing routes
- [Authorization RBAC](authorization-rbac.md) — role-based access on routes
- [Modular Monolith](../../principles/modular-monolith.md) — the architecture behind this layout
