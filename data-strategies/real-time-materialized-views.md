# Real-Time Materialized Views

## Context & Problem

Dashboards and APIs need fast reads. Computing aggregations on the fly — summing positions across a portfolio, calculating real-time P&L, counting active users per tenant — is expensive when the underlying data is large or scattered.

Traditional materialized views are refreshed periodically (every N minutes). But for use cases like a PM dashboard showing real-time P&L, even a 1-minute delay is too much.

Real-time materialized views are maintained incrementally: as events arrive, the view is updated — not rebuilt from scratch. The view is always current, and reads are a simple lookup.

## Design Decisions

### Approaches

| Approach | Update Latency | Complexity | Best For |
|---|---|---|---|
| **PostgreSQL `REFRESH MATERIALIZED VIEW`** | Minutes (full rebuild) | Lowest | Small datasets, periodic reports |
| **TimescaleDB continuous aggregates** | Seconds–minutes | Low | Time-series aggregations |
| **Kafka Streams / Faust** | Milliseconds–seconds | Medium | Event-driven aggregations |
| **Application-maintained projections** | Milliseconds | Medium | CQRS read models |
| **Redis computed views** | Milliseconds | Medium | High-read, low-write aggregations |

### TimescaleDB Continuous Aggregates

For time-series data, TimescaleDB maintains aggregations automatically as new data arrives:

```sql
-- Continuous aggregate: 1-minute OHLCV bars, auto-maintained
CREATE MATERIALIZED VIEW market_data.ohlcv_1m
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', timestamp) AS bucket,
    instrument_id,
    first(mid, timestamp)  AS open,
    max(mid)               AS high,
    min(mid)               AS low,
    last(mid, timestamp)   AS close,
    sum(volume)            AS volume,
    count(*)               AS tick_count
FROM market_data.prices
GROUP BY bucket, instrument_id
WITH NO DATA;

-- Refresh: materialize everything older than 1 minute, check every 30 seconds
SELECT add_continuous_aggregate_policy('market_data.ohlcv_1m',
    start_offset    => INTERVAL '1 hour',
    end_offset      => INTERVAL '1 minute',
    schedule_interval => INTERVAL '30 seconds'
);

-- Hierarchical: 1-hour bars from 1-minute bars (not from raw data)
CREATE MATERIALIZED VIEW market_data.ohlcv_1h
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', bucket) AS bucket,
    instrument_id,
    first(open, bucket)  AS open,
    max(high)            AS high,
    min(low)             AS low,
    last(close, bucket)  AS close,
    sum(volume)          AS volume,
    sum(tick_count)      AS tick_count
FROM market_data.ohlcv_1m
GROUP BY time_bucket('1 hour', bucket), instrument_id
WITH NO DATA;
```

### Event-Driven Projections (CQRS Read Models)

For non-time-series aggregations (portfolio P&L, position summaries), maintain read models from events:

```python
class PortfolioDashboardProjection:
    """Maintains a denormalized read model for the PM cockpit.
    Updated in real-time as events arrive."""

    def __init__(self, session_factory: async_sessionmaker) -> None:
        self._session_factory = session_factory

    async def handle(self, event: dict) -> None:
        match event["event_type"]:
            case "trade.executed":
                await self._apply_trade(event)
            case "price.updated":
                await self._mark_to_market(event)
            case "corporate_action.applied":
                await self._apply_corporate_action(event)

    async def _apply_trade(self, event: dict) -> None:
        async with self._session_factory() as session:
            # Upsert into denormalized positions view
            await session.execute(text("""
                INSERT INTO read_models.portfolio_positions
                    (portfolio_id, instrument_id, quantity, avg_cost, last_updated)
                VALUES (:portfolio_id, :instrument_id, :quantity, :cost, NOW())
                ON CONFLICT (portfolio_id, instrument_id) DO UPDATE SET
                    quantity = portfolio_positions.quantity + :signed_quantity,
                    last_updated = NOW()
            """), {
                "portfolio_id": event["data"]["portfolio_id"],
                "instrument_id": event["data"]["instrument_id"],
                "quantity": abs(Decimal(event["data"]["quantity"])),
                "signed_quantity": self._signed_quantity(event),
                "cost": Decimal(event["data"]["price"]),
            })
            await session.commit()

    async def _mark_to_market(self, event: dict) -> None:
        async with self._session_factory() as session:
            # Update market value for all positions in this instrument
            await session.execute(text("""
                UPDATE read_models.portfolio_positions
                SET market_price = :price,
                    market_value = quantity * :price,
                    unrealized_pnl = (quantity * :price) - (quantity * avg_cost),
                    last_updated = NOW()
                WHERE instrument_id = :instrument_id
            """), {
                "instrument_id": event["data"]["instrument_id"],
                "price": Decimal(event["data"]["mid"]),
            })
            await session.commit()
```

### Redis for High-Read Views

When reads are extremely frequent (thousands per second) and the view is small:

```python
class RedisPortfolioView:
    """Real-time portfolio summary maintained in Redis."""

    async def update_on_trade(self, event: dict) -> None:
        portfolio_id = event["data"]["portfolio_id"]
        instrument_id = event["data"]["instrument_id"]
        quantity = Decimal(event["data"]["quantity"])
        side = event["data"]["side"]
        signed = quantity if side == "buy" else -quantity

        pipe = self._redis.pipeline()
        # Update position quantity
        pipe.hincrbyfloat(
            f"portfolio:{portfolio_id}:positions",
            instrument_id,
            float(signed),
        )
        # Update metadata
        pipe.hset(
            f"portfolio:{portfolio_id}:meta",
            "last_trade_at",
            datetime.utcnow().isoformat(),
        )
        await pipe.execute()

    async def get_portfolio_summary(self, portfolio_id: str) -> dict:
        positions = await self._redis.hgetall(f"portfolio:{portfolio_id}:positions")
        meta = await self._redis.hgetall(f"portfolio:{portfolio_id}:meta")
        return {"positions": positions, **meta}
```

### Rebuilding Views

Every materialized view must be rebuildable from the source of truth (events or source tables). If a bug corrupts the view, replay events to reconstruct it:

```python
class ProjectionRebuilder:
    async def rebuild(self, projection, from_timestamp: datetime | None = None) -> None:
        """Rebuild a projection by replaying events from the event store."""
        events = await self._event_store.get_all(
            after=from_timestamp,
            order_by="timestamp",
        )
        # Clear the current view
        await projection.clear()
        # Replay all events
        for event in events:
            await projection.handle(event)
        logger.info(f"Rebuilt projection from {len(events)} events")
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| View inconsistency | Event processing bug | Rebuild from source events |
| Stale view | Event consumer lagging | Monitor consumer lag, alert on staleness |
| View too large | Unbounded aggregation | Partition views by time, expire old data |
| Rebuild too slow | Millions of events to replay | Snapshots — rebuild from last snapshot + recent events |
| Dual-write inconsistency | Write to source and view separately | Single event source, view derived from events only |

## Related Documents

- [CQRS & Event Sourcing](../principles/cqrs-event-sourcing.md) — event-driven read models
- [TimescaleDB Hypertables](../patterns/data-access/timescaledb-hypertables.md) — continuous aggregates
- [Kafka Topology](../patterns/messaging/kafka-topology.md) — event streaming for view updates
- [Event-Driven Architecture](../principles/event-driven-architecture.md) — events as the source of view updates
