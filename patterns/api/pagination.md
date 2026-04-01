# Pagination

## Context & Problem

APIs returning unbounded result sets are a performance and reliability risk. A query that returns 10 rows today returns 10 million rows next year. Without pagination, a single list request can:

- Exhaust server memory serializing the response
- Lock database rows for the duration of a long-running query
- Time out at the load balancer, returning a partial or empty response
- Cause cascading failures when downstream consumers attempt to process the full result set

Pagination is required for every list endpoint. The question is *which pagination strategy* — and the common default (offset-limit) has serious problems at scale.

## Design Decisions

### Strategy Comparison

| Strategy | How It Works | Stability | Performance | Complexity |
|---|---|---|---|---|
| Offset-limit | `OFFSET 1000 LIMIT 50` | Unstable — inserts shift rows | Degrades with large offsets (DB scans skipped rows) | Low |
| Cursor-based (keyset) | `WHERE id > :last_id ORDER BY id LIMIT 50` | Stable — immune to inserts | Constant performance regardless of page depth | Medium |
| Page-number | `?page=5&per_page=50` (translates to offset) | Same as offset | Same as offset | Low |

**Cursor-based pagination is the recommended default.** It provides stable results and constant-time performance. Use offset-limit only for simple internal tools where datasets are small and stable.

### Why Offset-Limit Breaks

```sql
-- Page 1: returns rows 1-50
SELECT * FROM trades ORDER BY created_at LIMIT 50 OFFSET 0;

-- While the user reads page 1, a new trade is inserted...

-- Page 2: row 50 from page 1 appears again (shifted by the insert)
SELECT * FROM trades ORDER BY created_at LIMIT 50 OFFSET 50;

-- Page 100: PostgreSQL reads and discards 5000 rows before returning 50
SELECT * FROM trades ORDER BY created_at LIMIT 50 OFFSET 5000;
```

Offset pagination is O(offset + limit) in the database. At page 1000, the query reads 50,000 rows to return 50.

### Cursor-Based Implementation

The cursor encodes the sort key values of the last item returned. The next page fetches rows *after* that cursor using a WHERE clause that hits an index directly.

```python
# pagination/cursor.py
# pydantic >= 2.0

import base64
import json
from datetime import datetime, timezone
from typing import TypeVar, Generic
from pydantic import BaseModel, Field

T = TypeVar("T", bound=BaseModel)


class CursorPage(BaseModel, Generic[T]):
    """Paginated response envelope with cursor metadata."""

    items: list[T]
    next_cursor: str | None = None
    has_more: bool
    total_count: int | None = None  # optional — can be expensive to compute


class Cursor(BaseModel):
    """
    Opaque cursor encoding the sort key(s) of the last item.
    Base64-encoded JSON so clients treat it as an opaque string.
    """

    sort_value: str  # ISO timestamp or other sort key
    id: str          # tie-breaker for rows with the same sort value

    def encode(self) -> str:
        payload = json.dumps({"s": self.sort_value, "i": self.id})
        return base64.urlsafe_b64encode(payload.encode()).decode()

    @classmethod
    def decode(cls, token: str) -> "Cursor":
        payload = json.loads(base64.urlsafe_b64decode(token.encode()))
        return cls(sort_value=payload["s"], id=payload["i"])
```

### SQLAlchemy Repository with Cursor Pagination

```python
# repositories/trade.py
from datetime import datetime

from sqlalchemy import select, func, and_
from sqlalchemy.ext.asyncio import AsyncSession

from models.trade import TradeModel
from schemas.trade import Trade
from pagination.cursor import Cursor, CursorPage


class TradeRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def list_trades(
        self,
        portfolio_id: str,
        limit: int = 50,
        cursor: str | None = None,
        include_total: bool = False,
    ) -> CursorPage[Trade]:
        """
        List trades for a portfolio using cursor-based pagination.
        Sorted by created_at DESC, with trade_id as tie-breaker.
        """
        query = (
            select(TradeModel)
            .where(TradeModel.portfolio_id == portfolio_id)
            .order_by(TradeModel.created_at.desc(), TradeModel.id.desc())
            .limit(limit + 1)  # fetch one extra to detect has_more
        )

        if cursor:
            decoded = Cursor.decode(cursor)
            # Keyset condition: rows BEFORE the cursor in DESC order
            query = query.where(
                and_(
                    TradeModel.created_at <= decoded.sort_value,
                    ~and_(
                        TradeModel.created_at == decoded.sort_value,
                        TradeModel.id >= decoded.id,
                    ),
                )
            )

        result = await self._session.execute(query)
        rows = list(result.scalars().all())

        has_more = len(rows) > limit
        items = rows[:limit]

        # Build cursor from last item
        next_cursor = None
        if has_more and items:
            last = items[-1]
            next_cursor = Cursor(
                sort_value=last.created_at.isoformat(),
                id=str(last.id),
            ).encode()

        # Optional total count (use sparingly — adds a COUNT query)
        total_count = None
        if include_total:
            count_query = (
                select(func.count())
                .select_from(TradeModel)
                .where(TradeModel.portfolio_id == portfolio_id)
            )
            total_count = await self._session.scalar(count_query)

        return CursorPage(
            items=[Trade.model_validate(row) for row in items],
            next_cursor=next_cursor,
            has_more=has_more,
            total_count=total_count,
        )
```

### Offset-Limit Implementation (When Appropriate)

For admin dashboards and internal tools where simplicity matters and datasets are small:

```python
# pagination/offset.py
from pydantic import BaseModel, Field
from typing import TypeVar, Generic

T = TypeVar("T", bound=BaseModel)


class OffsetPage(BaseModel, Generic[T]):
    """Offset-based paginated response."""

    items: list[T]
    total: int
    offset: int
    limit: int
    has_more: bool


class OffsetParams(BaseModel):
    """Query parameters for offset pagination."""

    offset: int = Field(default=0, ge=0, le=10_000)  # cap offset to prevent abuse
    limit: int = Field(default=50, ge=1, le=200)
```

```python
# SQLAlchemy query with offset
async def list_with_offset(
    self,
    params: OffsetParams,
) -> OffsetPage[Trade]:
    count_query = select(func.count()).select_from(TradeModel)
    total = await self._session.scalar(count_query)

    query = (
        select(TradeModel)
        .order_by(TradeModel.created_at.desc())
        .offset(params.offset)
        .limit(params.limit)
    )
    result = await self._session.execute(query)
    rows = result.scalars().all()

    return OffsetPage(
        items=[Trade.model_validate(row) for row in rows],
        total=total,
        offset=params.offset,
        limit=params.limit,
        has_more=(params.offset + params.limit) < total,
    )
```

### FastAPI Endpoint

```python
# api/routes/trades.py
from fastapi import APIRouter, Query, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from schemas.trade import Trade
from pagination.cursor import CursorPage
from repositories.trade import TradeRepository

router = APIRouter(prefix="/portfolios/{portfolio_id}/trades", tags=["trades"])


@router.get("", response_model=CursorPage[Trade])
async def list_trades(
    portfolio_id: str,
    limit: int = Query(default=50, ge=1, le=200),
    cursor: str | None = Query(default=None, description="Opaque pagination cursor"),
    include_total: bool = Query(default=False, description="Include total count (slower)"),
    session: AsyncSession = Depends(get_session),
) -> CursorPage[Trade]:
    """
    List trades for a portfolio with cursor-based pagination.

    Pass the `next_cursor` value from the response as the `cursor`
    parameter to fetch the next page.
    """
    repo = TradeRepository(session)
    return await repo.list_trades(
        portfolio_id=portfolio_id,
        limit=limit,
        cursor=cursor,
        include_total=include_total,
    )
```

### Response Shape

```json
{
  "items": [
    {"id": "t_001", "instrument": "AAPL", "side": "buy", "quantity": 100, "created_at": "2025-03-15T14:30:00Z"},
    {"id": "t_002", "instrument": "MSFT", "side": "sell", "quantity": 50, "created_at": "2025-03-15T14:29:00Z"}
  ],
  "next_cursor": "eyJzIjoiMjAyNS0wMy0xNVQxNDoyOTowMFoiLCJpIjoidF8wMDIifQ==",
  "has_more": true,
  "total_count": null
}
```

### Link Headers (RFC 8288)

For clients that prefer standard HTTP link headers over response body metadata:

```python
# pagination/links.py
from fastapi import Request, Response


def add_pagination_links(
    request: Request,
    response: Response,
    next_cursor: str | None,
    has_more: bool,
) -> None:
    """Add RFC 8288 Link headers for pagination."""
    if not has_more or not next_cursor:
        return

    base_url = str(request.url).split("?")[0]
    # Preserve existing query params, replace cursor
    params = dict(request.query_params)
    params["cursor"] = next_cursor
    query_string = "&".join(f"{k}={v}" for k, v in params.items())

    response.headers["Link"] = f'<{base_url}?{query_string}>; rel="next"'
```

### Choosing `total_count`

Computing total count requires a separate `COUNT(*)` query. On large tables this can be expensive (full index scan). Guidelines:

- **Omit by default.** Most paginated UIs show "Load more" or infinite scroll — they only need `has_more`.
- **Include on first page only** if the UI needs to show "X results found." Cache the count for subsequent pages.
- **Use an estimate** for very large tables: `SELECT reltuples FROM pg_class WHERE relname = 'trades'`. This is O(1) and accurate within a few percent.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Large offset performance degradation | User or crawler requests `?offset=1000000` | Cap offset (e.g., max 10,000), use cursor pagination instead |
| Cursor invalidation | Row pointed to by cursor was deleted | Handle gracefully — if cursor target is missing, start from nearest valid position |
| Duplicate or skipped items (offset) | Inserts or deletes between page requests | Use cursor pagination for correctness-sensitive endpoints |
| Expensive `COUNT(*)` on every request | `total_count` computed on every page | Make `include_total` opt-in, cache result, or use `pg_class` estimate |
| Cursor tampering | Client modifies base64 cursor to inject SQL | Decode and validate cursor fields; use parameterized queries (SQLAlchemy handles this) |
| Sort column lacks index | Cursor query does a sequential scan | Ensure `(sort_column, id)` composite index exists |

## Related Documents

- [FastAPI Modular Layout](fastapi-modular-layout.md) — where pagination routes fit in the project structure
- [SQLAlchemy Repository](../data-access/sqlalchemy-repository.md) — repository pattern that encapsulates pagination queries
- [API Versioning](api-versioning.md) — changing pagination strategy across API versions
