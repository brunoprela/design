# Market Data Simulator Module

## Context & Problem

The hedge fund platform depends on real-time market data from Bloomberg, Reuters, and exchange direct feeds. These data sources cost $20,000+/month per terminal, require dedicated infrastructure (Bloomberg B-PIPE appliances, co-located servers), and are bound by vendor licensing that prohibits use in non-production environments.

This creates a hard blocker for five critical workflows:

| Workflow | Why Real Data Won't Work |
|---|---|
| **Local development** | No developer has a Bloomberg terminal on their laptop |
| **Integration testing** | CI/CD pipelines can't depend on a live vendor feed |
| **Backtesting strategies** | Alpha engine needs years of history at configurable speed |
| **Load testing** | Need to simulate 10,000+ instruments at peak tick rates |
| **Demo environments** | Sales demos can't rely on market hours or live data |

Without a simulator, developers mock data inline, each in their own way. The result: tests that pass with fake data but break with real data, backtests that can't reproduce results, and no way to stress-test the system before production.

The simulator must produce events on the same `prices.normalized` Kafka topic that the real [Market Data Ingestion](market-data-ingestion.md) module uses. Downstream consumers (positions, risk, alpha) should not know or care whether data is live or simulated.

## Domain Concepts

| Concept | Definition |
|---|---|
| **Synthetic Price Generator** | Produces statistically realistic price streams using stochastic models (GBM). No historical data required -- generates from parameters alone. |
| **Historical Replay** | Replays real historical data through the system at configurable speed (1x, 10x, 100x, max). Sources: CSV, yfinance, pre-seeded TimescaleDB. |
| **Scenario Library** | Pre-built market scenarios that reproduce specific conditions: crashes, flash crashes, volatility spikes, sector rotations. Used for stress testing. |
| **Test Universe** | A curated set of ~100 instruments with realistic attributes (ISIN, sector, currency) and 2 years of seeded history. The canonical development dataset. |
| **Seed Data** | Scripts that populate PostgreSQL and TimescaleDB with instruments, historical OHLCV bars, and corporate actions for local development. |

## Architecture

```mermaid
graph TD
    subgraph "Simulator Modes"
        SYN[Synthetic Generator<br/>GBM + Correlation Matrix]
        REPLAY[Historical Replay<br/>CSV / yfinance / TimescaleDB]
        SCENARIO[Scenario Library<br/>Pre-built Stress Tests]
    end

    subgraph "Simulator Core"
        CLOCK[Simulation Clock<br/>Speed: 1x / 10x / 100x / max]
        THROTTLE[Throttle / Backpressure<br/>Respects Kafka capacity]
        TRANSFORM[PriceSnapshot Builder<br/>Canonical model output]
    end

    subgraph "Output — Same as Production"
        KAFKA[Kafka: prices.normalized]
        TSDB[(TimescaleDB<br/>market_data.prices)]
        REDIS[(Redis<br/>Latest price cache)]
    end

    subgraph "Consumers — Unchanged"
        POS[Position Keeping]
        RISK[Risk Engine]
        ALPHA[Alpha Engine]
        EXP[Exposure Calculation]
    end

    SYN --> CLOCK
    REPLAY --> CLOCK
    SCENARIO --> CLOCK

    CLOCK --> THROTTLE
    THROTTLE --> TRANSFORM
    TRANSFORM --> KAFKA
    TRANSFORM --> TSDB
    TRANSFORM --> REDIS

    KAFKA --> POS
    KAFKA --> RISK
    KAFKA --> ALPHA
    KAFKA --> EXP
```

The key design constraint: the simulator writes to exactly the same Kafka topic (`prices.normalized`), TimescaleDB table (`market_data.prices`), and Redis cache that the real ingestion module does. Downstream modules are completely decoupled from the data source.

## Design Decisions

### 1. Geometric Brownian Motion for Synthetic Prices

**Problem:** Developers need price streams that look realistic -- log-normal returns, configurable volatility, and correlated multi-asset behavior. Uniform random noise doesn't produce realistic order book dynamics or trigger real risk scenarios.

**Decision:** Use Geometric Brownian Motion (GBM) as the base model. GBM captures the log-normal distribution of asset returns observed in real markets and is the foundation of the Black-Scholes model. For multi-asset simulation, use Cholesky decomposition to introduce cross-asset correlations.

**Tradeoffs:**
- GBM assumes constant volatility and continuous prices -- it won't produce fat tails or jumps. For stress testing, use the Scenario Library instead.
- Correlated generation via Cholesky requires O(n^2) memory for n instruments. Fine for 100-1000 instruments, problematic for 10,000+.
- GBM prices can drift unrealistically over long horizons. For multi-year backtests, prefer historical replay.

```python
# simulator/synthetic.py

from dataclasses import dataclass
from datetime import datetime, timedelta
from decimal import Decimal

import numpy as np
from pydantic import BaseModel, ConfigDict


class InstrumentConfig(BaseModel):
    """Configuration for synthetic price generation."""

    model_config = ConfigDict(frozen=True)

    instrument_id: str
    initial_price: float
    annual_drift: float = 0.05       # mu: expected annual return
    annual_volatility: float = 0.20  # sigma: annualized volatility
    spread_bps: float = 5.0          # bid-ask spread in basis points


@dataclass(frozen=True)
class GeneratedTick:
    instrument_id: str
    bid: Decimal
    ask: Decimal
    mid: Decimal
    volume: Decimal
    timestamp: datetime


class SyntheticPriceGenerator:
    """Generate realistic price streams using Geometric Brownian Motion.

    GBM: dS = mu * S * dt + sigma * S * dW

    Where:
    - S = current price
    - mu = drift (annualized expected return)
    - sigma = volatility (annualized)
    - dW = Wiener process increment ~ N(0, sqrt(dt))

    For correlated assets, we decompose the correlation matrix via Cholesky
    and apply the lower triangular matrix to independent normal samples.
    """

    def __init__(self, seed: int = 42) -> None:
        self._rng = np.random.default_rng(seed)

    def generate_prices(
        self,
        instruments: list[InstrumentConfig],
        trading_days: int,
        ticks_per_day: int = 390,  # 6.5 hours * 60 minutes
        correlation_matrix: np.ndarray | None = None,
    ) -> dict[str, list[GeneratedTick]]:
        """Generate correlated price paths for multiple instruments.

        Args:
            instruments: Configuration for each instrument.
            trading_days: Number of trading days to simulate.
            ticks_per_day: Price observations per day (390 = 1 per minute for US market).
            correlation_matrix: n x n correlation matrix. Identity if None.

        Returns:
            Dict mapping instrument_id to list of GeneratedTick.
        """
        n_instruments = len(instruments)
        n_steps = trading_days * ticks_per_day
        dt = 1.0 / (252 * ticks_per_day)  # fraction of trading year

        # Build correlation structure
        if correlation_matrix is not None:
            assert correlation_matrix.shape == (n_instruments, n_instruments)
            cholesky = np.linalg.cholesky(correlation_matrix)
        else:
            cholesky = np.eye(n_instruments)

        # Generate correlated random samples
        # Shape: (n_steps, n_instruments)
        independent_normals = self._rng.standard_normal((n_steps, n_instruments))
        correlated_normals = independent_normals @ cholesky.T

        # Simulate GBM paths
        results: dict[str, list[GeneratedTick]] = {}
        base_time = datetime(2024, 1, 2, 9, 30)  # market open

        for i, config in enumerate(instruments):
            mu = config.annual_drift
            sigma = config.annual_volatility
            spread_half = config.spread_bps / 10_000 / 2

            # GBM: S(t+dt) = S(t) * exp((mu - 0.5*sigma^2)*dt + sigma*sqrt(dt)*Z)
            log_returns = (mu - 0.5 * sigma**2) * dt + sigma * np.sqrt(dt) * correlated_normals[:, i]
            price_path = config.initial_price * np.exp(np.cumsum(log_returns))

            ticks: list[GeneratedTick] = []
            for step in range(n_steps):
                day = step // ticks_per_day
                minute = step % ticks_per_day
                tick_time = base_time + timedelta(days=day, minutes=minute)

                # Skip weekends
                while tick_time.weekday() >= 5:
                    tick_time += timedelta(days=1)

                mid = price_path[step]
                spread = mid * spread_half
                volume = abs(self._rng.normal(100_000, 30_000))

                ticks.append(GeneratedTick(
                    instrument_id=config.instrument_id,
                    bid=Decimal(str(round(mid - spread, 4))),
                    ask=Decimal(str(round(mid + spread, 4))),
                    mid=Decimal(str(round(mid, 4))),
                    volume=Decimal(str(round(max(volume, 100), 0))),
                    timestamp=tick_time,
                ))

            results[config.instrument_id] = ticks

        return results

    def generate_correlation_matrix(
        self,
        n_instruments: int,
        intra_sector_corr: float = 0.6,
        inter_sector_corr: float = 0.3,
        sector_sizes: list[int] | None = None,
    ) -> np.ndarray:
        """Generate a realistic block-diagonal correlation matrix.

        Instruments within the same sector are more correlated than
        instruments across sectors, matching observed market behavior.
        """
        if sector_sizes is None:
            # Default: 5 equal sectors
            base = n_instruments // 5
            sector_sizes = [base] * 4 + [n_instruments - 4 * base]

        matrix = np.full((n_instruments, n_instruments), inter_sector_corr)
        offset = 0
        for size in sector_sizes:
            matrix[offset : offset + size, offset : offset + size] = intra_sector_corr
            offset += size

        np.fill_diagonal(matrix, 1.0)
        return matrix
```

### 2. Historical Data Replay

**Problem:** Synthetic data is useful for development but insufficient for backtesting -- backtests need to reproduce behavior against actual market conditions. The system needs to replay historical data through the same Kafka topics at configurable speed.

**Decision:** Support three data sources for replay (CSV files, Yahoo Finance via yfinance, and pre-seeded TimescaleDB) with a configurable speed multiplier. The replay service reads data, respects inter-tick timing at the specified speed, and publishes to `prices.normalized`.

**Tradeoffs:**
- yfinance is free but provides only daily OHLCV (no intraday ticks). Good enough for daily strategy backtests, not for intraday.
- CSV replay depends on data quality. No validation beyond what the normalizer does.
- "As fast as possible" mode can overwhelm Kafka consumers. The throttle must respect downstream backpressure.

```python
# simulator/replay.py

import asyncio
import csv
from datetime import datetime, timedelta
from decimal import Decimal
from enum import Enum
from pathlib import Path
from typing import AsyncIterator, Protocol

import structlog
import yfinance as yf
from pydantic import BaseModel, ConfigDict
from sqlalchemy.ext.asyncio import AsyncSession

logger = structlog.get_logger()


class ReplaySpeed(Enum):
    REAL_TIME = 1.0
    FAST_10X = 10.0
    FAST_100X = 100.0
    MAX = 0.0  # no delay between ticks


class PriceSnapshot(BaseModel):
    """Canonical price model -- matches market-data-ingestion interface."""

    model_config = ConfigDict(frozen=True)

    instrument_id: str
    bid: Decimal
    ask: Decimal
    mid: Decimal
    volume: Decimal | None = None
    timestamp: datetime
    source: str


class EventPublisher(Protocol):
    async def publish(self, topic: str, key: str, event: dict) -> None: ...


class HistoricalReplayService:
    """Replay historical data through the system at configurable speed.

    Sources:
    - CSV files exported from any data vendor
    - Yahoo Finance (yfinance) -- free, daily bars only
    - Pre-seeded TimescaleDB -- full tick data from previous recording sessions

    Speed:
    - REAL_TIME (1x): one tick per original time interval
    - FAST_10X: 10x faster than real time
    - FAST_100X: 100x faster
    - MAX: as fast as Kafka can consume (backpressure-aware)
    """

    def __init__(
        self,
        publisher: EventPublisher,
        speed: ReplaySpeed = ReplaySpeed.FAST_10X,
        max_events_per_second: int = 50_000,
    ) -> None:
        self._publisher = publisher
        self._speed = speed
        self._max_eps = max_events_per_second
        self._semaphore = asyncio.Semaphore(max_events_per_second)

    async def replay_from_csv(self, csv_path: Path) -> int:
        """Replay price data from a CSV file.

        Expected columns: timestamp, instrument_id, bid, ask, mid, volume
        """
        published = 0
        prev_timestamp: datetime | None = None

        with csv_path.open() as f:
            reader = csv.DictReader(f)
            for row in reader:
                snapshot = PriceSnapshot(
                    instrument_id=row["instrument_id"],
                    bid=Decimal(row["bid"]),
                    ask=Decimal(row["ask"]),
                    mid=Decimal(row["mid"]),
                    volume=Decimal(row["volume"]) if row.get("volume") else None,
                    timestamp=datetime.fromisoformat(row["timestamp"]),
                    source="replay-csv",
                )
                await self._throttle(snapshot.timestamp, prev_timestamp)
                await self._publish_snapshot(snapshot)
                prev_timestamp = snapshot.timestamp
                published += 1

        logger.info("csv_replay_complete", path=str(csv_path), events=published)
        return published

    async def replay_from_yfinance(
        self,
        tickers: list[str],
        start: str,
        end: str,
    ) -> int:
        """Download and replay daily bars from Yahoo Finance (free).

        Converts daily OHLCV to PriceSnapshot using close as mid,
        with a synthetic spread derived from daily volatility.
        """
        published = 0

        for ticker in tickers:
            data = yf.download(ticker, start=start, end=end, progress=False)
            if data.empty:
                logger.warning("yfinance_no_data", ticker=ticker)
                continue

            for date, row in data.iterrows():
                close = float(row["Close"])
                volume = float(row["Volume"])
                # Synthetic spread: 5 bps for liquid stocks
                spread = close * 0.0005
                snapshot = PriceSnapshot(
                    instrument_id=ticker,
                    bid=Decimal(str(round(close - spread / 2, 4))),
                    ask=Decimal(str(round(close + spread / 2, 4))),
                    mid=Decimal(str(round(close, 4))),
                    volume=Decimal(str(round(volume, 0))),
                    timestamp=date.to_pydatetime(),
                    source="replay-yfinance",
                )
                await self._publish_snapshot(snapshot)
                published += 1

            # Small delay between tickers to avoid overwhelming consumers
            await asyncio.sleep(0.01)

        logger.info("yfinance_replay_complete", tickers=len(tickers), events=published)
        return published

    async def replay_from_timescaledb(
        self,
        session: AsyncSession,
        instrument_ids: list[str],
        start: datetime,
        end: datetime,
    ) -> int:
        """Replay tick data from pre-seeded TimescaleDB.

        Reads from market_data.prices in chronological order and
        publishes with timing that respects the speed multiplier.
        """
        from sqlalchemy import text

        query = text("""
            SELECT timestamp, instrument_id, bid, ask, mid, volume
            FROM market_data.prices
            WHERE instrument_id = ANY(:ids)
              AND timestamp BETWEEN :start AND :end
            ORDER BY timestamp ASC
        """)

        result = await session.execute(
            query,
            {"ids": instrument_ids, "start": start, "end": end},
        )

        published = 0
        prev_timestamp: datetime | None = None

        for row in result:
            snapshot = PriceSnapshot(
                instrument_id=row.instrument_id,
                bid=row.bid,
                ask=row.ask,
                mid=row.mid,
                volume=row.volume,
                timestamp=row.timestamp,
                source="replay-timescaledb",
            )
            await self._throttle(snapshot.timestamp, prev_timestamp)
            await self._publish_snapshot(snapshot)
            prev_timestamp = snapshot.timestamp
            published += 1

        logger.info("timescaledb_replay_complete", events=published)
        return published

    async def _publish_snapshot(self, snapshot: PriceSnapshot) -> None:
        await self._publisher.publish(
            topic="prices.normalized",
            key=snapshot.instrument_id,
            event={
                "event_type": "price.updated",
                "event_id": f"sim-{snapshot.instrument_id}-{snapshot.timestamp.isoformat()}",
                "timestamp": snapshot.timestamp.isoformat(),
                "data": snapshot.model_dump(mode="json"),
            },
        )

    async def _throttle(
        self,
        current_ts: datetime,
        prev_ts: datetime | None,
    ) -> None:
        """Delay between ticks based on speed multiplier."""
        if self._speed == ReplaySpeed.MAX or prev_ts is None:
            return

        real_delta = (current_ts - prev_ts).total_seconds()
        if real_delta <= 0:
            return

        simulated_delay = real_delta / self._speed.value
        # Cap delay at 5 seconds to avoid long pauses on overnight gaps
        await asyncio.sleep(min(simulated_delay, 5.0))
```

### 3. Stress Test Scenario Library

**Problem:** Neither synthetic nor historical replay lets you reliably test specific market conditions on demand. If you want to test "what happens when the market drops 40% over 6 months," you don't want to wait for it in replay -- you want to trigger it now.

**Decision:** A library of pre-built scenarios that overlay specific price dynamics onto the test universe. Each scenario defines price multipliers and volatility adjustments over a time window, applied on top of the synthetic generator's baseline.

```python
# simulator/scenarios.py

from dataclasses import dataclass, field
from datetime import timedelta
from enum import Enum

import numpy as np


class ScenarioType(Enum):
    CRASH = "crash"
    FLASH_CRASH = "flash_crash"
    RALLY = "rally"
    VOLATILITY_SPIKE = "volatility_spike"
    SECTOR_ROTATION = "sector_rotation"
    RATES_SHOCK = "rates_shock"


@dataclass(frozen=True)
class ScenarioPhase:
    """A phase within a scenario (e.g., the sell-off, then the recovery)."""

    duration: timedelta
    drift_override: float          # annualized drift during this phase
    volatility_multiplier: float   # multiply base volatility by this
    affected_sectors: list[str] = field(default_factory=list)  # empty = all sectors


@dataclass(frozen=True)
class Scenario:
    name: str
    description: str
    scenario_type: ScenarioType
    phases: list[ScenarioPhase]
    total_duration: timedelta


class ScenarioLibrary:
    """Pre-built market scenarios for stress testing and demo environments.

    Each scenario is a sequence of phases that override drift and volatility
    parameters in the synthetic generator. This lets you compose scenarios:
    run normal market for 30 days, inject a crash, observe recovery.

    Usage:
        library = ScenarioLibrary()
        scenario = library.get("flash_crash_2010")
        generator = SyntheticPriceGenerator(seed=42)
        # Apply scenario phases to modify generator parameters
    """

    def __init__(self) -> None:
        self._scenarios: dict[str, Scenario] = {
            s.name: s for s in self._build_library()
        }

    def get(self, name: str) -> Scenario:
        if name not in self._scenarios:
            available = ", ".join(sorted(self._scenarios.keys()))
            raise KeyError(f"Unknown scenario '{name}'. Available: {available}")
        return self._scenarios[name]

    def list_scenarios(self) -> list[Scenario]:
        return list(self._scenarios.values())

    @staticmethod
    def _build_library() -> list[Scenario]:
        return [
            Scenario(
                name="market_crash_2008",
                description=(
                    "S&P drops 40% over 6 months. Volatility spikes to 80% annualized. "
                    "Financial sector hit hardest (-60%). Slow recovery over 12 months."
                ),
                scenario_type=ScenarioType.CRASH,
                phases=[
                    ScenarioPhase(
                        duration=timedelta(days=126),
                        drift_override=-1.2,   # aggressive downward drift
                        volatility_multiplier=4.0,
                        affected_sectors=["financials", "real_estate"],
                    ),
                    ScenarioPhase(
                        duration=timedelta(days=63),
                        drift_override=-0.3,
                        volatility_multiplier=2.5,
                    ),
                    ScenarioPhase(
                        duration=timedelta(days=252),
                        drift_override=0.4,
                        volatility_multiplier=1.5,
                    ),
                ],
                total_duration=timedelta(days=441),
            ),
            Scenario(
                name="flash_crash_2010",
                description=(
                    "Market drops 10% in 20 minutes, recovers most losses in 16 minutes. "
                    "Tests system behavior under extreme tick rates and rapid reversal."
                ),
                scenario_type=ScenarioType.FLASH_CRASH,
                phases=[
                    ScenarioPhase(
                        duration=timedelta(minutes=20),
                        drift_override=-150.0,  # extreme short-term drift
                        volatility_multiplier=20.0,
                    ),
                    ScenarioPhase(
                        duration=timedelta(minutes=16),
                        drift_override=120.0,   # sharp recovery
                        volatility_multiplier=15.0,
                    ),
                    ScenarioPhase(
                        duration=timedelta(hours=5),
                        drift_override=-0.1,
                        volatility_multiplier=3.0,
                    ),
                ],
                total_duration=timedelta(hours=6),
            ),
            Scenario(
                name="covid_march_2020",
                description=(
                    "Rapid sell-off across all sectors. SPX drops 34% in 23 trading days. "
                    "VIX hits 82. Circuit breakers triggered. Travel and hospitality crushed."
                ),
                scenario_type=ScenarioType.CRASH,
                phases=[
                    ScenarioPhase(
                        duration=timedelta(days=23),
                        drift_override=-6.0,
                        volatility_multiplier=6.0,
                    ),
                    ScenarioPhase(
                        duration=timedelta(days=15),
                        drift_override=-1.0,
                        volatility_multiplier=3.0,
                    ),
                    ScenarioPhase(
                        duration=timedelta(days=90),
                        drift_override=2.0,
                        volatility_multiplier=2.0,
                    ),
                ],
                total_duration=timedelta(days=128),
            ),
            Scenario(
                name="rates_shock",
                description=(
                    "Interest rates increase 300bps over 3 months. Bond proxies sell off. "
                    "Utilities and REITs drop 15-20%. Financials rally on wider net interest margins."
                ),
                scenario_type=ScenarioType.RATES_SHOCK,
                phases=[
                    ScenarioPhase(
                        duration=timedelta(days=63),
                        drift_override=-0.8,
                        volatility_multiplier=2.0,
                        affected_sectors=["utilities", "real_estate"],
                    ),
                    ScenarioPhase(
                        duration=timedelta(days=63),
                        drift_override=0.5,
                        volatility_multiplier=1.5,
                        affected_sectors=["financials"],
                    ),
                ],
                total_duration=timedelta(days=63),
            ),
            Scenario(
                name="sector_rotation",
                description=(
                    "Growth-to-value rotation. Tech sells off 15% while financials "
                    "and energy rally 20% over 2 months. Tests sector exposure rebalancing."
                ),
                scenario_type=ScenarioType.SECTOR_ROTATION,
                phases=[
                    ScenarioPhase(
                        duration=timedelta(days=42),
                        drift_override=-1.0,
                        volatility_multiplier=2.0,
                        affected_sectors=["technology", "communication_services"],
                    ),
                    ScenarioPhase(
                        duration=timedelta(days=42),
                        drift_override=1.2,
                        volatility_multiplier=1.5,
                        affected_sectors=["financials", "energy", "industrials"],
                    ),
                ],
                total_duration=timedelta(days=42),
            ),
            Scenario(
                name="low_volatility",
                description=(
                    "Calm market with low volatility for 60 trading days. "
                    "Tests normal operations, ensures system works in boring conditions."
                ),
                scenario_type=ScenarioType.RALLY,
                phases=[
                    ScenarioPhase(
                        duration=timedelta(days=60),
                        drift_override=0.08,
                        volatility_multiplier=0.5,
                    ),
                ],
                total_duration=timedelta(days=60),
            ),
        ]
```

### 4. Test Universe and Seed Data

**Problem:** Every developer needs a consistent, realistic dataset to work with. Without a canonical test universe, each developer invents their own instruments, leading to non-reproducible tests and demo data that doesn't look believable.

**Decision:** A seed script that creates 100 instruments across 10 sectors, generates 2 years of daily OHLCV history, and populates corporate actions (splits and dividends). The script is idempotent -- safe to run multiple times.

```python
# scripts/seed_market_data.py

"""Seed local development environment with realistic market data.

Usage:
    python scripts/seed_market_data.py --instruments 100 --days 504
    python scripts/seed_market_data.py --instruments 20 --days 60 --fast

Creates:
    - Instruments in security_master.instruments (PostgreSQL)
    - 2 years of daily OHLCV bars in market_data.prices (TimescaleDB)
    - Corporate actions in security_master.corporate_actions
    - Instrument attributes (ISIN, sector, currency, exchange)
"""

import argparse
import hashlib
from dataclasses import dataclass
from datetime import datetime, timedelta
from decimal import Decimal

import numpy as np
from pydantic import BaseModel, ConfigDict


class SeedInstrument(BaseModel):
    """Instrument definition for seed data."""

    model_config = ConfigDict(frozen=True)

    instrument_id: str
    name: str
    isin: str
    exchange: str
    currency: str
    sector: str
    initial_price: float
    volatility: float


# ---- Test Universe Definition ---- #

SECTORS: dict[str, list[SeedInstrument]] = {
    "technology": [
        SeedInstrument(instrument_id="AAPL", name="Apple Inc.", isin="US0378331005", exchange="NASDAQ", currency="USD", sector="technology", initial_price=175.0, volatility=0.25),
        SeedInstrument(instrument_id="MSFT", name="Microsoft Corp.", isin="US5949181045", exchange="NASDAQ", currency="USD", sector="technology", initial_price=380.0, volatility=0.22),
        SeedInstrument(instrument_id="GOOGL", name="Alphabet Inc.", isin="US02079K3059", exchange="NASDAQ", currency="USD", sector="technology", initial_price=140.0, volatility=0.28),
        SeedInstrument(instrument_id="NVDA", name="NVIDIA Corp.", isin="US67066G1040", exchange="NASDAQ", currency="USD", sector="technology", initial_price=480.0, volatility=0.45),
        SeedInstrument(instrument_id="META", name="Meta Platforms Inc.", isin="US30303M1027", exchange="NASDAQ", currency="USD", sector="technology", initial_price=350.0, volatility=0.35),
        SeedInstrument(instrument_id="ADBE", name="Adobe Inc.", isin="US00724F1012", exchange="NASDAQ", currency="USD", sector="technology", initial_price=550.0, volatility=0.30),
        SeedInstrument(instrument_id="CRM", name="Salesforce Inc.", isin="US79466L3024", exchange="NYSE", currency="USD", sector="technology", initial_price=260.0, volatility=0.28),
        SeedInstrument(instrument_id="ORCL", name="Oracle Corp.", isin="US68389X1054", exchange="NYSE", currency="USD", sector="technology", initial_price=120.0, volatility=0.25),
        SeedInstrument(instrument_id="INTC", name="Intel Corp.", isin="US4581401001", exchange="NASDAQ", currency="USD", sector="technology", initial_price=45.0, volatility=0.35),
        SeedInstrument(instrument_id="AMD", name="Advanced Micro Devices", isin="US0079031078", exchange="NASDAQ", currency="USD", sector="technology", initial_price=150.0, volatility=0.42),
    ],
    "financials": [
        SeedInstrument(instrument_id="JPM", name="JPMorgan Chase", isin="US46625H1005", exchange="NYSE", currency="USD", sector="financials", initial_price=170.0, volatility=0.22),
        SeedInstrument(instrument_id="BAC", name="Bank of America", isin="US0605051046", exchange="NYSE", currency="USD", sector="financials", initial_price=33.0, volatility=0.25),
        SeedInstrument(instrument_id="GS", name="Goldman Sachs", isin="US38141G1040", exchange="NYSE", currency="USD", sector="financials", initial_price=380.0, volatility=0.25),
        SeedInstrument(instrument_id="MS", name="Morgan Stanley", isin="US6174464486", exchange="NYSE", currency="USD", sector="financials", initial_price=90.0, volatility=0.25),
        SeedInstrument(instrument_id="WFC", name="Wells Fargo", isin="US9497461015", exchange="NYSE", currency="USD", sector="financials", initial_price=48.0, volatility=0.23),
        SeedInstrument(instrument_id="BLK", name="BlackRock Inc.", isin="US09247X1019", exchange="NYSE", currency="USD", sector="financials", initial_price=750.0, volatility=0.22),
        SeedInstrument(instrument_id="SCHW", name="Charles Schwab", isin="US8085131055", exchange="NYSE", currency="USD", sector="financials", initial_price=65.0, volatility=0.28),
        SeedInstrument(instrument_id="AXP", name="American Express", isin="US0258161092", exchange="NYSE", currency="USD", sector="financials", initial_price=185.0, volatility=0.22),
        SeedInstrument(instrument_id="C", name="Citigroup Inc.", isin="US1729674242", exchange="NYSE", currency="USD", sector="financials", initial_price=52.0, volatility=0.26),
        SeedInstrument(instrument_id="USB", name="U.S. Bancorp", isin="US9029733048", exchange="NYSE", currency="USD", sector="financials", initial_price=42.0, volatility=0.24),
    ],
    "healthcare": [
        SeedInstrument(instrument_id="JNJ", name="Johnson & Johnson", isin="US4781601046", exchange="NYSE", currency="USD", sector="healthcare", initial_price=160.0, volatility=0.15),
        SeedInstrument(instrument_id="UNH", name="UnitedHealth Group", isin="US91324P1021", exchange="NYSE", currency="USD", sector="healthcare", initial_price=530.0, volatility=0.20),
        SeedInstrument(instrument_id="PFE", name="Pfizer Inc.", isin="US7170811035", exchange="NYSE", currency="USD", sector="healthcare", initial_price=28.0, volatility=0.25),
        SeedInstrument(instrument_id="MRK", name="Merck & Co.", isin="US58933Y1055", exchange="NYSE", currency="USD", sector="healthcare", initial_price=110.0, volatility=0.18),
        SeedInstrument(instrument_id="ABBV", name="AbbVie Inc.", isin="US00287Y1091", exchange="NYSE", currency="USD", sector="healthcare", initial_price=155.0, volatility=0.20),
        SeedInstrument(instrument_id="TMO", name="Thermo Fisher Scientific", isin="US8835561023", exchange="NYSE", currency="USD", sector="healthcare", initial_price=540.0, volatility=0.22),
        SeedInstrument(instrument_id="LLY", name="Eli Lilly", isin="US5324571083", exchange="NYSE", currency="USD", sector="healthcare", initial_price=600.0, volatility=0.28),
        SeedInstrument(instrument_id="BMY", name="Bristol-Myers Squibb", isin="US1101221083", exchange="NYSE", currency="USD", sector="healthcare", initial_price=52.0, volatility=0.22),
        SeedInstrument(instrument_id="AMGN", name="Amgen Inc.", isin="US0311621009", exchange="NASDAQ", currency="USD", sector="healthcare", initial_price=280.0, volatility=0.20),
        SeedInstrument(instrument_id="GILD", name="Gilead Sciences", isin="US3755581036", exchange="NASDAQ", currency="USD", sector="healthcare", initial_price=80.0, volatility=0.22),
    ],
    "energy": [
        SeedInstrument(instrument_id="XOM", name="Exxon Mobil", isin="US30231G1022", exchange="NYSE", currency="USD", sector="energy", initial_price=105.0, volatility=0.25),
        SeedInstrument(instrument_id="CVX", name="Chevron Corp.", isin="US1667641005", exchange="NYSE", currency="USD", sector="energy", initial_price=155.0, volatility=0.23),
        SeedInstrument(instrument_id="COP", name="ConocoPhillips", isin="US20825C1045", exchange="NYSE", currency="USD", sector="energy", initial_price=120.0, volatility=0.28),
        SeedInstrument(instrument_id="SLB", name="Schlumberger Ltd.", isin="AN8068571086", exchange="NYSE", currency="USD", sector="energy", initial_price=55.0, volatility=0.30),
        SeedInstrument(instrument_id="EOG", name="EOG Resources", isin="US26875P1012", exchange="NYSE", currency="USD", sector="energy", initial_price=125.0, volatility=0.28),
        SeedInstrument(instrument_id="MPC", name="Marathon Petroleum", isin="US56585A1025", exchange="NYSE", currency="USD", sector="energy", initial_price=150.0, volatility=0.28),
        SeedInstrument(instrument_id="PSX", name="Phillips 66", isin="US7185461040", exchange="NYSE", currency="USD", sector="energy", initial_price=120.0, volatility=0.26),
        SeedInstrument(instrument_id="VLO", name="Valero Energy", isin="US91913Y1001", exchange="NYSE", currency="USD", sector="energy", initial_price=135.0, volatility=0.28),
        SeedInstrument(instrument_id="OXY", name="Occidental Petroleum", isin="US6745991058", exchange="NYSE", currency="USD", sector="energy", initial_price=62.0, volatility=0.35),
        SeedInstrument(instrument_id="HAL", name="Halliburton Co.", isin="US4062161017", exchange="NYSE", currency="USD", sector="energy", initial_price=38.0, volatility=0.32),
    ],
    "consumer_discretionary": [
        SeedInstrument(instrument_id="AMZN", name="Amazon.com Inc.", isin="US0231351067", exchange="NASDAQ", currency="USD", sector="consumer_discretionary", initial_price=155.0, volatility=0.30),
        SeedInstrument(instrument_id="TSLA", name="Tesla Inc.", isin="US88160R1014", exchange="NASDAQ", currency="USD", sector="consumer_discretionary", initial_price=250.0, volatility=0.50),
        SeedInstrument(instrument_id="HD", name="Home Depot", isin="US4370761029", exchange="NYSE", currency="USD", sector="consumer_discretionary", initial_price=340.0, volatility=0.22),
        SeedInstrument(instrument_id="MCD", name="McDonald's Corp.", isin="US5801351017", exchange="NYSE", currency="USD", sector="consumer_discretionary", initial_price=290.0, volatility=0.18),
        SeedInstrument(instrument_id="NKE", name="Nike Inc.", isin="US6541061031", exchange="NYSE", currency="USD", sector="consumer_discretionary", initial_price=108.0, volatility=0.28),
        SeedInstrument(instrument_id="SBUX", name="Starbucks Corp.", isin="US8552441094", exchange="NASDAQ", currency="USD", sector="consumer_discretionary", initial_price=100.0, volatility=0.25),
        SeedInstrument(instrument_id="TJX", name="TJX Companies", isin="US8725401090", exchange="NYSE", currency="USD", sector="consumer_discretionary", initial_price=95.0, volatility=0.22),
        SeedInstrument(instrument_id="LOW", name="Lowe's Companies", isin="US5486611073", exchange="NYSE", currency="USD", sector="consumer_discretionary", initial_price=230.0, volatility=0.24),
        SeedInstrument(instrument_id="BKNG", name="Booking Holdings", isin="US09857L1089", exchange="NASDAQ", currency="USD", sector="consumer_discretionary", initial_price=3400.0, volatility=0.28),
        SeedInstrument(instrument_id="TGT", name="Target Corp.", isin="US87612E1064", exchange="NYSE", currency="USD", sector="consumer_discretionary", initial_price=140.0, volatility=0.28),
    ],
    "consumer_staples": [
        SeedInstrument(instrument_id="PG", name="Procter & Gamble", isin="US7427181091", exchange="NYSE", currency="USD", sector="consumer_staples", initial_price=155.0, volatility=0.15),
        SeedInstrument(instrument_id="KO", name="Coca-Cola Co.", isin="US1912161007", exchange="NYSE", currency="USD", sector="consumer_staples", initial_price=60.0, volatility=0.14),
        SeedInstrument(instrument_id="PEP", name="PepsiCo Inc.", isin="US7134481081", exchange="NASDAQ", currency="USD", sector="consumer_staples", initial_price=175.0, volatility=0.15),
        SeedInstrument(instrument_id="COST", name="Costco Wholesale", isin="US22160K1051", exchange="NASDAQ", currency="USD", sector="consumer_staples", initial_price=600.0, volatility=0.20),
        SeedInstrument(instrument_id="WMT", name="Walmart Inc.", isin="US9311421039", exchange="NYSE", currency="USD", sector="consumer_staples", initial_price=165.0, volatility=0.16),
    ],
    "industrials": [
        SeedInstrument(instrument_id="CAT", name="Caterpillar Inc.", isin="US1491231015", exchange="NYSE", currency="USD", sector="industrials", initial_price=280.0, volatility=0.25),
        SeedInstrument(instrument_id="BA", name="Boeing Co.", isin="US0970231058", exchange="NYSE", currency="USD", sector="industrials", initial_price=210.0, volatility=0.35),
        SeedInstrument(instrument_id="HON", name="Honeywell Intl.", isin="US4385161066", exchange="NASDAQ", currency="USD", sector="industrials", initial_price=200.0, volatility=0.20),
        SeedInstrument(instrument_id="UPS", name="United Parcel Service", isin="US9113121068", exchange="NYSE", currency="USD", sector="industrials", initial_price=155.0, volatility=0.22),
        SeedInstrument(instrument_id="DE", name="Deere & Co.", isin="US2441991054", exchange="NYSE", currency="USD", sector="industrials", initial_price=400.0, volatility=0.25),
    ],
    "utilities": [
        SeedInstrument(instrument_id="NEE", name="NextEra Energy", isin="US65339F1012", exchange="NYSE", currency="USD", sector="utilities", initial_price=72.0, volatility=0.18),
        SeedInstrument(instrument_id="DUK", name="Duke Energy", isin="US26441C2044", exchange="NYSE", currency="USD", sector="utilities", initial_price=100.0, volatility=0.15),
        SeedInstrument(instrument_id="SO", name="Southern Company", isin="US8425871071", exchange="NYSE", currency="USD", sector="utilities", initial_price=72.0, volatility=0.14),
        SeedInstrument(instrument_id="D", name="Dominion Energy", isin="US25746U1097", exchange="NYSE", currency="USD", sector="utilities", initial_price=48.0, volatility=0.16),
        SeedInstrument(instrument_id="AEP", name="American Electric Power", isin="US0255371017", exchange="NASDAQ", currency="USD", sector="utilities", initial_price=85.0, volatility=0.15),
    ],
    "real_estate": [
        SeedInstrument(instrument_id="AMT", name="American Tower", isin="US03027X1000", exchange="NYSE", currency="USD", sector="real_estate", initial_price=210.0, volatility=0.22),
        SeedInstrument(instrument_id="PLD", name="Prologis Inc.", isin="US74340W1036", exchange="NYSE", currency="USD", sector="real_estate", initial_price=125.0, volatility=0.22),
        SeedInstrument(instrument_id="CCI", name="Crown Castle Intl.", isin="US22822V1017", exchange="NYSE", currency="USD", sector="real_estate", initial_price=110.0, volatility=0.22),
        SeedInstrument(instrument_id="EQIX", name="Equinix Inc.", isin="US29444U7000", exchange="NASDAQ", currency="USD", sector="real_estate", initial_price=800.0, volatility=0.22),
        SeedInstrument(instrument_id="SPG", name="Simon Property Group", isin="US8288061091", exchange="NYSE", currency="USD", sector="real_estate", initial_price=145.0, volatility=0.25),
    ],
    "communication_services": [
        SeedInstrument(instrument_id="NFLX", name="Netflix Inc.", isin="US64110L1061", exchange="NASDAQ", currency="USD", sector="communication_services", initial_price=480.0, volatility=0.35),
        SeedInstrument(instrument_id="DIS", name="Walt Disney Co.", isin="US2546871060", exchange="NYSE", currency="USD", sector="communication_services", initial_price=92.0, volatility=0.28),
        SeedInstrument(instrument_id="CMCSA", name="Comcast Corp.", isin="US20030N1019", exchange="NASDAQ", currency="USD", sector="communication_services", initial_price=42.0, volatility=0.22),
        SeedInstrument(instrument_id="T", name="AT&T Inc.", isin="US00206R1023", exchange="NYSE", currency="USD", sector="communication_services", initial_price=17.0, volatility=0.20),
        SeedInstrument(instrument_id="VZ", name="Verizon Communications", isin="US92343V1044", exchange="NYSE", currency="USD", sector="communication_services", initial_price=38.0, volatility=0.18),
    ],
}


def get_all_instruments() -> list[SeedInstrument]:
    """Flatten all sectors into a single instrument list."""
    instruments = []
    for sector_instruments in SECTORS.values():
        instruments.extend(sector_instruments)
    return instruments


@dataclass
class CorporateAction:
    instrument_id: str
    action_type: str       # "split" or "dividend"
    ex_date: datetime
    ratio: float | None    # split ratio (e.g., 4.0 for 4:1)
    amount: float | None   # dividend per share


def generate_corporate_actions(
    instruments: list[SeedInstrument],
    trading_days: int,
    n_splits: int = 10,
    n_dividends: int = 50,
    seed: int = 42,
) -> list[CorporateAction]:
    """Generate realistic corporate actions for the test universe.

    Splits: assigned to high-priced stocks, random dates.
    Dividends: quarterly for dividend-paying sectors (staples, utilities, financials).
    """
    rng = np.random.default_rng(seed)
    actions: list[CorporateAction] = []
    base_date = datetime(2022, 1, 3)

    # Splits: pick n_splits random instruments, assign split ratios
    split_ratios = [2.0, 3.0, 4.0, 5.0, 10.0]
    split_instruments = rng.choice(len(instruments), size=min(n_splits, len(instruments)), replace=False)
    for idx in split_instruments:
        inst = instruments[idx]
        day_offset = int(rng.integers(60, trading_days - 60))
        actions.append(CorporateAction(
            instrument_id=inst.instrument_id,
            action_type="split",
            ex_date=base_date + timedelta(days=day_offset),
            ratio=float(rng.choice(split_ratios)),
            amount=None,
        ))

    # Dividends: quarterly for select sectors
    dividend_sectors = {"financials", "consumer_staples", "utilities", "energy", "healthcare"}
    dividend_instruments = [i for i in instruments if i.sector in dividend_sectors]
    quarters_in_range = trading_days // 63  # ~63 trading days per quarter

    for inst in dividend_instruments[:n_dividends]:
        annual_yield = rng.uniform(0.015, 0.04)  # 1.5% to 4% annual yield
        quarterly_dividend = inst.initial_price * annual_yield / 4

        for q in range(quarters_in_range):
            ex_day = 63 * q + int(rng.integers(10, 50))
            if ex_day >= trading_days:
                break
            actions.append(CorporateAction(
                instrument_id=inst.instrument_id,
                action_type="dividend",
                ex_date=base_date + timedelta(days=ex_day),
                ratio=None,
                amount=round(quarterly_dividend, 2),
            ))

    return actions


async def seed_database(
    n_instruments: int = 100,
    trading_days: int = 504,
) -> None:
    """Main seed function. Generates instruments, prices, and corporate actions.

    This function is idempotent: it truncates and re-inserts.

    Steps:
        1. Insert instruments into security_master.instruments
        2. Generate OHLCV history using SyntheticPriceGenerator
        3. Insert price history into market_data.prices (TimescaleDB)
        4. Generate and insert corporate actions
    """
    from sqlalchemy import text
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine("postgresql+asyncpg://localhost:5432/hedgefund")
    instruments = get_all_instruments()[:n_instruments]

    async with engine.begin() as conn:
        # 1. Seed instruments
        await conn.execute(text("TRUNCATE security_master.instruments CASCADE"))
        for inst in instruments:
            await conn.execute(
                text("""
                    INSERT INTO security_master.instruments
                        (instrument_id, name, isin, exchange, currency, sector)
                    VALUES (:id, :name, :isin, :exchange, :currency, :sector)
                """),
                {
                    "id": inst.instrument_id,
                    "name": inst.name,
                    "isin": inst.isin,
                    "exchange": inst.exchange,
                    "currency": inst.currency,
                    "sector": inst.sector,
                },
            )

        # 2. Generate and insert price history
        from simulator.synthetic import InstrumentConfig, SyntheticPriceGenerator

        generator = SyntheticPriceGenerator(seed=42)
        configs = [
            InstrumentConfig(
                instrument_id=inst.instrument_id,
                initial_price=inst.initial_price,
                annual_drift=0.05,
                annual_volatility=inst.volatility,
            )
            for inst in instruments
        ]

        corr_matrix = generator.generate_correlation_matrix(
            n_instruments=len(instruments),
            intra_sector_corr=0.6,
            inter_sector_corr=0.3,
            sector_sizes=[len(v) for v in SECTORS.values()][:len(instruments)],
        )

        prices = generator.generate_prices(
            instruments=configs,
            trading_days=trading_days,
            ticks_per_day=1,  # daily bars for seed data
            correlation_matrix=corr_matrix,
        )

        await conn.execute(text(
            "DELETE FROM market_data.prices WHERE source = 'seed'"
        ))

        for instrument_id, ticks in prices.items():
            for tick in ticks:
                await conn.execute(
                    text("""
                        INSERT INTO market_data.prices
                            (timestamp, instrument_id, bid, ask, mid, volume, source)
                        VALUES (:ts, :id, :bid, :ask, :mid, :vol, 'seed')
                    """),
                    {
                        "ts": tick.timestamp,
                        "id": tick.instrument_id,
                        "bid": tick.bid,
                        "ask": tick.ask,
                        "mid": tick.mid,
                        "vol": tick.volume,
                    },
                )

        # 3. Seed corporate actions
        corp_actions = generate_corporate_actions(instruments, trading_days)
        await conn.execute(text("TRUNCATE security_master.corporate_actions CASCADE"))
        for ca in corp_actions:
            await conn.execute(
                text("""
                    INSERT INTO security_master.corporate_actions
                        (instrument_id, action_type, ex_date, ratio, amount)
                    VALUES (:id, :type, :date, :ratio, :amount)
                """),
                {
                    "id": ca.instrument_id,
                    "type": ca.action_type,
                    "date": ca.ex_date,
                    "ratio": ca.ratio,
                    "amount": ca.amount,
                },
            )

    print(f"Seeded {len(instruments)} instruments, {trading_days} days of history, "
          f"{len(corp_actions)} corporate actions")


if __name__ == "__main__":
    import asyncio

    parser = argparse.ArgumentParser(description="Seed market data for local development")
    parser.add_argument("--instruments", type=int, default=100, help="Number of instruments")
    parser.add_argument("--days", type=int, default=504, help="Trading days of history")
    args = parser.parse_args()

    asyncio.run(seed_database(n_instruments=args.instruments, trading_days=args.days))
```

### 5. Mock Broker Simulator

**Problem:** The [Order Management](order-management.md) module routes orders to brokers via FIX protocol. In local development, there is no broker to receive orders. Without simulated fills, developers can't test the full trade lifecycle from intent to position update.

**Decision:** A mock broker that listens for orders on the `trades.pending` topic and produces fill events with configurable latency, partial fill probability, and slippage.

```python
# simulator/mock_broker.py

import asyncio
import random
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal
from typing import Protocol

import structlog

logger = structlog.get_logger()


@dataclass(frozen=True)
class BrokerConfig:
    """Configuration for mock broker behavior."""

    fill_latency_min_ms: int = 10
    fill_latency_max_ms: int = 500
    partial_fill_probability: float = 0.15     # 15% of orders get partial fills
    rejection_probability: float = 0.02        # 2% of orders get rejected
    slippage_bps: float = 2.0                  # average slippage in basis points
    market_impact_coefficient: float = 0.1     # price impact per 1% of daily volume


class EventPublisher(Protocol):
    async def publish(self, topic: str, key: str, event: dict) -> None: ...


class MockBrokerSimulator:
    """Simulates broker behavior for local development and testing.

    Receives order events and produces execution reports with realistic:
    - Fill latency (configurable 10ms-500ms, uniformly distributed)
    - Partial fills (configurable probability, random fill ratio 20-80%)
    - Rejections (invalid instrument, market closed, insufficient liquidity)
    - Slippage (market impact model based on order size vs. daily volume)

    This replaces the FIX adapter in local/dev environments.
    """

    def __init__(
        self,
        publisher: EventPublisher,
        config: BrokerConfig = BrokerConfig(),
        seed: int = 42,
    ) -> None:
        self._publisher = publisher
        self._config = config
        self._rng = random.Random(seed)
        self._daily_volumes: dict[str, Decimal] = {}

    def set_daily_volumes(self, volumes: dict[str, Decimal]) -> None:
        """Set reference volumes for market impact calculation."""
        self._daily_volumes = volumes

    async def handle_order(self, order: dict) -> None:
        """Process an incoming order and produce fill/reject events."""
        instrument_id = order["instrument_id"]
        order_id = order["order_id"]
        side = order["side"]           # "buy" or "sell"
        quantity = Decimal(str(order["quantity"]))
        limit_price = Decimal(str(order.get("limit_price", 0)))

        # Simulate network + matching engine latency
        latency_ms = self._rng.uniform(
            self._config.fill_latency_min_ms,
            self._config.fill_latency_max_ms,
        )
        await asyncio.sleep(latency_ms / 1000)

        # Random rejection
        if self._rng.random() < self._config.rejection_probability:
            reasons = [
                "Instrument not found",
                "Market closed",
                "Insufficient liquidity",
                "Order size exceeds limit",
            ]
            await self._publisher.publish(
                topic="trades.executed",
                key=instrument_id,
                event={
                    "event_type": "order.rejected",
                    "order_id": order_id,
                    "instrument_id": instrument_id,
                    "reason": self._rng.choice(reasons),
                    "timestamp": datetime.utcnow().isoformat(),
                },
            )
            logger.info("order_rejected", order_id=order_id)
            return

        # Calculate slippage
        slippage = self._calculate_slippage(instrument_id, quantity, side)

        # Partial fill logic
        if self._rng.random() < self._config.partial_fill_probability:
            fill_ratio = Decimal(str(round(self._rng.uniform(0.2, 0.8), 2)))
            first_fill = (quantity * fill_ratio).quantize(Decimal("1"))
            remaining = quantity - first_fill

            await self._publish_fill(order_id, instrument_id, side, first_fill, limit_price, slippage)
            await asyncio.sleep(self._rng.uniform(0.05, 0.5))
            await self._publish_fill(order_id, instrument_id, side, remaining, limit_price, slippage)
        else:
            await self._publish_fill(order_id, instrument_id, side, quantity, limit_price, slippage)

    def _calculate_slippage(
        self,
        instrument_id: str,
        quantity: Decimal,
        side: str,
    ) -> Decimal:
        """Market impact model: slippage increases with order size relative to volume."""
        base_slippage = Decimal(str(self._config.slippage_bps)) / Decimal("10000")

        daily_volume = self._daily_volumes.get(instrument_id, Decimal("1000000"))
        if daily_volume > 0:
            participation_rate = quantity / daily_volume
            impact = Decimal(str(self._config.market_impact_coefficient)) * participation_rate
        else:
            impact = Decimal("0")

        total_slippage = base_slippage + impact
        # Slippage is adverse: buys pay more, sells receive less
        return total_slippage if side == "buy" else -total_slippage

    async def _publish_fill(
        self,
        order_id: str,
        instrument_id: str,
        side: str,
        quantity: Decimal,
        limit_price: Decimal,
        slippage: Decimal,
    ) -> None:
        fill_price = limit_price * (1 + slippage)
        await self._publisher.publish(
            topic="trades.executed",
            key=instrument_id,
            event={
                "event_type": "order.filled",
                "order_id": order_id,
                "instrument_id": instrument_id,
                "side": side,
                "quantity": str(quantity),
                "fill_price": str(fill_price.quantize(Decimal("0.0001"))),
                "slippage_bps": str((slippage * 10000).quantize(Decimal("0.01"))),
                "timestamp": datetime.utcnow().isoformat(),
            },
        )
        logger.info(
            "order_filled",
            order_id=order_id,
            instrument_id=instrument_id,
            quantity=str(quantity),
            fill_price=str(fill_price),
        )
```

## Docker Compose Integration

The simulator integrates as a service with Docker Compose profiles, so it only runs in development and backtest environments -- never in production.

```yaml
# docker-compose.yml (excerpt)

services:
  market-data-simulator:
    build:
      context: ./services/market-data-simulator
      dockerfile: Dockerfile
    profiles: ["dev", "backtest"]
    depends_on:
      kafka:
        condition: service_healthy
      timescaledb:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      MODE: synthetic               # "synthetic", "replay", or "scenario"
      INSTRUMENTS: 100              # number of instruments to simulate
      SPEED: "10x"                  # replay speed (ignored in synthetic mode)
      SCENARIO: ""                  # scenario name (only in scenario mode)
      KAFKA_BOOTSTRAP: kafka:9092
      TIMESCALEDB_URL: postgresql://dev:dev@timescaledb:5432/hedgefund
      REDIS_URL: redis://redis:6379/0
      SEED: 42                      # deterministic output for reproducibility
      TICKS_PER_DAY: 390            # 1 per minute during market hours
    restart: unless-stopped

  mock-broker:
    build:
      context: ./services/mock-broker
      dockerfile: Dockerfile
    profiles: ["dev"]
    depends_on:
      kafka:
        condition: service_healthy
    environment:
      KAFKA_BOOTSTRAP: kafka:9092
      FILL_LATENCY_MIN_MS: 10
      FILL_LATENCY_MAX_MS: 500
      PARTIAL_FILL_PROBABILITY: 0.15
      REJECTION_PROBABILITY: 0.02
    restart: unless-stopped

  # Seed job — runs once, then exits
  seed-market-data:
    build:
      context: .
      dockerfile: Dockerfile
    profiles: ["dev"]
    depends_on:
      timescaledb:
        condition: service_healthy
    command: python scripts/seed_market_data.py --instruments 100 --days 504
    environment:
      DATABASE_URL: postgresql+asyncpg://dev:dev@timescaledb:5432/hedgefund
    restart: "no"
```

**Usage:**

```bash
# Start everything including simulator
docker compose --profile dev up -d

# Start only backtest infrastructure (no mock broker)
docker compose --profile backtest up -d

# Run seed script standalone
docker compose --profile dev run --rm seed-market-data

# Override simulator mode for a specific test
docker compose --profile dev run -e MODE=scenario -e SCENARIO=flash_crash_2010 market-data-simulator
```

## Failure Modes

| Failure | Cause | Impact | Mitigation |
|---|---|---|---|
| **Synthetic data unrealistic** | GBM doesn't produce fat tails, jumps, or mean reversion | Strategies that work in simulation fail in production | Use historical replay for backtesting, synthetic only for development. Document that GBM is a simplification. |
| **Replay clock drift** | System clock and replay clock diverge under load | Events published out of order, confusing downstream consumers | Use monotonic clock for inter-tick delays. Publish with original timestamps but track wall-clock offset for monitoring. |
| **Kafka backpressure during fast replay** | "Max speed" replay produces events faster than consumers can process | Consumer lag grows unboundedly, OOM in consumer processes | Rate-limit publisher to `max_events_per_second`. Monitor consumer lag and pause replay if lag exceeds threshold. |
| **Seed data schema mismatch** | Seed script assumes table schema that has since changed via migration | Seed script fails, new developers blocked | Run seed script in CI against latest migrations. Pin seed script to migration version. |
| **Correlation matrix not positive-definite** | User provides invalid correlation matrix | Cholesky decomposition fails with `LinAlgError` | Validate matrix is symmetric and positive-definite before decomposition. Fall back to nearest valid matrix via spectral projection. |
| **yfinance rate limiting** | Too many concurrent downloads | Incomplete historical data | Sequential download with backoff. Cache downloaded data in TimescaleDB for subsequent replays. |
| **Mock broker diverges from real broker** | Real broker has behaviors not modeled (specific rejection codes, fill algorithms) | Tests pass locally but orders fail in production | Maintain a "known differences" document. Record real broker behavior and update mock periodically. |

## Performance Profile

| Metric | Target | Notes |
|---|---|---|
| Synthetic generation rate | 1M ticks/sec | Numpy vectorized, not per-tick Python loops |
| Kafka publish throughput | 50,000 events/sec | Batched producer, `linger.ms=5`, `batch.size=64KB` |
| Historical replay (max speed) | 100,000 events/sec | Bounded by Kafka producer and TimescaleDB read speed |
| Seed script (100 instruments, 504 days) | < 60 seconds | Bulk insert with `COPY`, not row-by-row |
| Mock broker fill latency | 10ms-500ms (configurable) | Matches observed real-broker latency range |
| Memory (synthetic, 100 instruments) | < 500MB | Price paths stored as numpy arrays, converted to ticks on publish |

## Dependencies

```
market-data-simulator
  ├── depends on: shared kernel (PriceSnapshot, OHLCVBar, event types)
  ├── depends on: nothing else (development-only tool, not a platform module)
  ├── publishes to: prices.normalized (same topic as real market data)
  ├── publishes to: trades.executed (mock broker fills)
  ├── writes to: market_data.prices (TimescaleDB, seed data)
  ├── writes to: security_master.instruments (PostgreSQL, seed data)
  └── consumed by: all downstream modules (they don't know it's simulated)
```

**Python dependencies:**

```
numpy>=1.26           # GBM simulation, correlation matrices
yfinance>=0.2.31      # Free historical data download
pandas>=2.1           # DataFrame operations for yfinance data
pydantic>=2.5         # Schema validation
sqlalchemy[asyncpg]>=2.0
confluent-kafka>=2.3  # Kafka producer
structlog>=23.2       # Structured logging
```

## Related Documents

- [Market Data Ingestion](market-data-ingestion.md) -- the production module this simulator replaces in dev/test
- [Risk Engine](risk-engine.md) -- primary consumer of simulated data for stress testing
- [Order Management](order-management.md) -- consumes mock broker fills
- [Docker Compose Patterns](../../infrastructure/docker-compose-patterns.md) -- profile-based service activation
- [Local Kafka Cluster](../../infrastructure/local-kafka-cluster.md) -- Kafka setup for local development
