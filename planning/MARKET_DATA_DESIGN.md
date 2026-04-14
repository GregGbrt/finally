# Market Data Design — FinAlly

Implementation reference for the market data subsystem in `backend/app/market/`. The component is **complete and tested** (73 tests, 84% coverage). This document explains the architecture and shows the actual code so the rest of the backend can integrate against it confidently.

---

## 1. Architecture

```
MarketDataSource (ABC)
├── SimulatorDataSource  ←  GBM simulator (default, no API key)
└── MassiveDataSource    ←  Polygon.io REST poller (MASSIVE_API_KEY set)
        │
        ▼
   PriceCache  (thread-safe, single source of truth)
        │
        ├──→ SSE stream  GET /api/stream/prices
        ├──→ Portfolio valuation
        └──→ Trade execution  (read current price at fill time)
```

**Key principle:** the data source never returns prices to callers directly. It writes into `PriceCache` on its own schedule. All consumers read from the cache. This decouples the SSE streamer, portfolio code, and trade executor from whichever data source is active.

---

## 2. File Map

```
backend/app/market/
  __init__.py         Public re-exports
  models.py           PriceUpdate dataclass
  cache.py            PriceCache (thread-safe)
  interface.py        MarketDataSource ABC
  seed_prices.py      SEED_PRICES, TICKER_PARAMS, correlation constants
  simulator.py        GBMSimulator + SimulatorDataSource
  massive_client.py   MassiveDataSource (Polygon.io)
  factory.py          create_market_data_source()
  stream.py           SSE FastAPI router factory
```

Import everything from the package root — do not reach into submodules:

```python
from app.market import PriceCache, PriceUpdate, MarketDataSource, create_market_data_source, create_stream_router
```

---

## 3. Data Model — `models.py`

`PriceUpdate` is the only type that crosses the market data boundary. All downstream code works with this type.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds (float)

    @property
    def change(self) -> float:          # price - previous_price, rounded 4dp
    def change_percent(self) -> float:  # % change, rounded 4dp
    def direction(self) -> str:         # "up" | "down" | "flat"
    def to_dict(self) -> dict:          # serializes all fields for JSON/SSE
```

`frozen=True` means these are immutable — safe to share across threads without copying. `slots=True` makes attribute access faster.

**`to_dict()` output** (used by SSE and API responses):

```json
{
  "ticker": "AAPL",
  "price": 191.34,
  "previous_price": 191.20,
  "timestamp": 1736012345.678,
  "change": 0.14,
  "change_percent": 0.0732,
  "direction": "up"
}
```

---

## 4. Price Cache — `cache.py`

Thread-safe in-memory store. One writer (data source background task), many readers (SSE, portfolio, trades).

```python
class PriceCache:
    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate
    def get(self, ticker: str) -> PriceUpdate | None
    def get_price(self, ticker: str) -> float | None        # convenience
    def get_all(self) -> dict[str, PriceUpdate]             # shallow copy snapshot
    def remove(self, ticker: str) -> None
    @property
    def version(self) -> int    # monotonic counter, incremented on every update
```

**Reading prices in downstream code:**

```python
# Trade execution — get current fill price
price = cache.get_price("AAPL")
if price is None:
    raise ValueError("AAPL not in price cache")

# Portfolio valuation — get all positions' current prices
all_prices = cache.get_all()
for ticker, position in positions.items():
    current_price = all_prices.get(ticker)
    if current_price:
        unrealized_pnl = (current_price.price - position.avg_cost) * position.quantity
```

**Thread safety:** all methods acquire an internal `threading.Lock`. Safe to call from any thread. The `version` property is read without the lock — on CPython this is fine (GIL makes single-int reads atomic).

**SSE change detection** — the `version` counter lets the SSE generator skip sending if nothing changed since the last send:

```python
last_version = -1
while True:
    current_version = cache.version
    if current_version != last_version:
        last_version = current_version
        data = cache.get_all()
        yield f"data: {json.dumps({t: u.to_dict() for t, u in data.items()})}\n\n"
    await asyncio.sleep(0.5)
```

---

## 5. Abstract Interface — `interface.py`

Both data sources implement this contract. Downstream code is source-agnostic.

```python
class MarketDataSource(ABC):
    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Call once at app startup."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop background task. Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add ticker to active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove ticker from active set and from the cache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return current list of tracked tickers."""
```

**Watchlist API integration pattern:**

```python
# POST /api/watchlist  { "ticker": "PYPL" }
async def add_to_watchlist(ticker: str, source: MarketDataSource, db: ...):
    await db.insert_watchlist_ticker(ticker)
    await source.add_ticker(ticker)   # starts simulating / will appear on next poll

# DELETE /api/watchlist/{ticker}
async def remove_from_watchlist(ticker: str, source: MarketDataSource, db: ...):
    await db.delete_watchlist_ticker(ticker)
    await source.remove_ticker(ticker)  # also evicts from PriceCache
```

---

## 6. Seed Prices & Parameters — `seed_prices.py`

Default tickers and their simulator parameters:

```python
SEED_PRICES = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00,
    "TSLA": 250.00, "NVDA": 800.00,  "META": 500.00, "JPM":  195.00,
    "V":    280.00, "NFLX": 600.00,
}

TICKER_PARAMS = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT": {"sigma": 0.20, "mu": 0.05},
    "AMZN": {"sigma": 0.28, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},   # High vol, low drift
    "NVDA": {"sigma": 0.40, "mu": 0.08},   # High vol, high drift
    "META": {"sigma": 0.30, "mu": 0.05},
    "JPM":  {"sigma": 0.18, "mu": 0.04},   # Bank: low vol
    "V":    {"sigma": 0.17, "mu": 0.04},   # Payments: low vol
    "NFLX": {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}   # For dynamically added tickers

CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # Tech stocks move together
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together
CROSS_GROUP_CORR   = 0.3   # Between sectors / unknown
TSLA_CORR          = 0.3   # TSLA does its own thing
```

`sigma` is annualized volatility. `mu` is annualized drift. Both are converted to per-tick values inside `GBMSimulator` using `dt = 0.5 / (252 * 6.5 * 3600)` ≈ 8.48e-8.

---

## 7. GBM Simulator — `simulator.py`

### Math

```
S(t+dt) = S(t) * exp( (mu - sigma²/2) * dt  +  sigma * sqrt(dt) * Z )
```

Where `Z` is a correlated standard normal drawn via Cholesky decomposition of the sector correlation matrix. This produces log-normal price paths — prices can't go negative.

Per-tick move magnitude: with `sigma=0.25` and `dt=8.48e-8`, a typical tick moves the price by `sigma * sqrt(dt) * Z ≈ 0.25 * 0.0003 * Z ≈ 0.007%` of price. For a $190 stock this is about 1.3 cents per tick — realistic for a 500ms update.

### Correlated Moves

```python
# Build n×n correlation matrix
corr = np.eye(n)
for i, j in pairs:
    rho = _pairwise_correlation(tickers[i], tickers[j])
    corr[i, j] = corr[j, i] = rho

# Cholesky decomposition: corr = L @ L.T
cholesky = np.linalg.cholesky(corr)

# Each step: generate independent normals, correlate them
z_independent = np.random.standard_normal(n)
z_correlated  = cholesky @ z_independent
```

Result: tech stocks move together (ρ=0.6), finance stocks together (ρ=0.5), TSLA and cross-sector pairs at ρ=0.3.

### Random Shock Events

```python
# ~0.1% chance per tick per ticker (every ~50s across 10 tickers at 2 ticks/s)
if random.random() < 0.001:
    shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
    price *= (1 + shock)   # 2-5% sudden move
```

### `GBMSimulator` Public API

```python
sim = GBMSimulator(tickers=["AAPL", "GOOGL"], event_probability=0.001)

prices = sim.step()        # dict[str, float] — advance one tick
sim.add_ticker("TSLA")    # rebuilds correlation matrix
sim.remove_ticker("GOOGL") # rebuilds correlation matrix
sim.get_price("AAPL")     # float | None
sim.get_tickers()          # list[str]
```

### `SimulatorDataSource` — full implementation

```python
class SimulatorDataSource(MarketDataSource):
    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers)
        # Seed the cache immediately — SSE gets data on first connect
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)   # 0.5s default
```

Cache is seeded before the background loop starts, so the first SSE event is never empty.

---

## 8. Massive API Client — `massive_client.py`

Uses the `massive` Python package (Polygon.io wrapper). The REST client is synchronous; it's offloaded to a thread via `asyncio.to_thread` to avoid blocking the event loop.

### Poll Flow

```
start()
  │
  ├─ _poll_once()          # immediate first poll
  └─ create_task(_poll_loop())
          │
          └─ loop: sleep(interval) → _poll_once()
                                          │
                                          ├─ asyncio.to_thread(_fetch_snapshots)
                                          │       ↳ RESTClient.get_snapshot_all(STOCKS, tickers)
                                          │
                                          └─ for each snapshot:
                                                 cache.update(ticker, snap.last_trade.price,
                                                              snap.last_trade.timestamp / 1000)
```

### `MassiveDataSource` — full implementation

```python
class MassiveDataSource(MarketDataSource):
    def __init__(self, api_key: str, price_cache: PriceCache, poll_interval: float = 15.0):
        self._client = RESTClient(api_key=api_key)  # Polygon.io REST client
        self._cache  = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._tickers = list(tickers)
        await self._poll_once()   # Populate cache before loop starts
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            for snap in snapshots:
                try:
                    self._cache.update(
                        ticker=snap.ticker,
                        price=snap.last_trade.price,
                        timestamp=snap.last_trade.timestamp / 1000.0,  # ms → s
                    )
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping snapshot for %s: %s",
                                   getattr(snap, "ticker", "???"), e)
        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Continue — next interval will retry

    def _fetch_snapshots(self) -> list:
        """Runs in thread pool (blocking I/O)."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### Rate Limits

| Tier | Requests/min | Recommended poll interval |
|------|-------------|--------------------------|
| Free | 5           | 15s (default)            |
| Starter | 60      | 2s                       |
| Developer | 120  | 1s                       |

Set `poll_interval` via constructor or environment variable if needed.

---

## 9. Factory — `factory.py`

Selects the data source at startup based on environment:

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        logger.info("Market data source: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

No `MASSIVE_API_KEY` → simulator. Key present → Massive. Downstream code never needs to know which is active.

---

## 10. SSE Streaming Endpoint — `stream.py`

Endpoint: `GET /api/stream/prices`  
Content-Type: `text/event-stream`

### Event Format

Each event is a single `data:` line containing a JSON object keyed by ticker:

```
retry: 1000

data: {"AAPL": {"ticker":"AAPL","price":191.34,"previous_price":191.20,"timestamp":1736012345.678,"change":0.14,"change_percent":0.0732,"direction":"up"}, "GOOGL": {...}, ...}

data: {"AAPL": {"ticker":"AAPL","price":191.41,...}, ...}
```

The `retry: 1000` directive tells the browser's `EventSource` to reconnect after 1 second if the connection drops.

### Implementation

```python
router = APIRouter(prefix="/api/stream", tags=["streaming"])

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",   # Disable nginx buffering
            },
        )
    return router

async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    yield "retry: 1000\n\n"
    last_version = -1
    while True:
        if await request.is_disconnected():
            break
        current_version = price_cache.version
        if current_version != last_version:
            last_version = current_version
            prices = price_cache.get_all()
            if prices:
                data = {ticker: u.to_dict() for ticker, u in prices.items()}
                yield f"data: {json.dumps(data)}\n\n"
        await asyncio.sleep(interval)
```

**Version-based change detection** avoids sending duplicate events when the cache hasn't changed (e.g., between Massive polls).

### Frontend Connection

```typescript
const source = new EventSource("/api/stream/prices");

source.onmessage = (event) => {
  const prices: Record<string, PriceUpdate> = JSON.parse(event.data);
  // prices["AAPL"].price, prices["AAPL"].direction, etc.
};

source.onerror = () => {
  // EventSource auto-reconnects after `retry` ms — nothing to do here
  // Optionally update a connection status indicator
};
```

---

## 11. FastAPI Lifecycle Integration

Wire everything together in the FastAPI app startup/shutdown:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.market import PriceCache, create_market_data_source, create_stream_router

price_cache = PriceCache()
market_source = create_market_data_source(price_cache)

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: load initial tickers from DB, start data source
    initial_tickers = await db.get_watchlist_tickers(user_id="default")
    await market_source.start(initial_tickers)

    yield   # App is running

    # Shutdown: stop data source cleanly
    await market_source.stop()

app = FastAPI(lifespan=lifespan)

# Register the SSE router
stream_router = create_stream_router(price_cache)
app.include_router(stream_router)
```

Pass `price_cache` and `market_source` to route handlers via FastAPI dependency injection:

```python
from fastapi import Depends

def get_price_cache() -> PriceCache:
    return price_cache

def get_market_source() -> MarketDataSource:
    return market_source

@app.post("/api/watchlist")
async def add_ticker(
    body: AddTickerRequest,
    source: MarketDataSource = Depends(get_market_source),
    cache: PriceCache = Depends(get_price_cache),
):
    await db.insert_watchlist_ticker(body.ticker)
    await source.add_ticker(body.ticker)
    return {"ticker": body.ticker, "price": cache.get_price(body.ticker)}
```

---

## 12. Trade Execution — Using the Cache

At the moment a market order is filled, read the current price from the cache:

```python
@app.post("/api/portfolio/trade")
async def execute_trade(
    body: TradeRequest,
    cache: PriceCache = Depends(get_price_cache),
):
    current_price = cache.get_price(body.ticker)
    if current_price is None:
        raise HTTPException(400, f"{body.ticker} price unavailable")

    # Execute at current_price — instant fill, no slippage
    fill_price = current_price
    ...
```

---

## 13. Environment Variables

| Variable | Required | Default | Effect |
|----------|----------|---------|--------|
| `MASSIVE_API_KEY` | No | (empty) | If set, uses Massive/Polygon.io API |
| `LLM_MOCK` | No | `false` | Unrelated to market data |

The backend reads `.env` from the project root via `python-dotenv` (or Docker `--env-file`).

---

## 14. Test Suite

73 tests across 6 modules in `backend/tests/market/`. Run with:

```bash
cd backend
uv run --extra dev pytest tests/market/ -v
uv run --extra dev pytest tests/market/ --cov=app/market
```

| Test module | What it covers |
|-------------|----------------|
| `test_models.py` (11 tests) | `PriceUpdate` properties, `to_dict()`, edge cases |
| `test_cache.py` (13 tests) | Thread safety, version counter, all CRUD operations |
| `test_simulator.py` (17 tests) | GBM math, Cholesky, shock events, add/remove tickers |
| `test_simulator_source.py` (10 tests) | `SimulatorDataSource` lifecycle, cache seeding |
| `test_factory.py` (7 tests) | Env var routing, correct class returned |
| `test_massive.py` (13 tests) | `MassiveDataSource` — requires `massive` package installed |

---

## 15. Demo

A Rich terminal dashboard for visual verification:

```bash
cd backend
uv run market_data_demo.py
```

Shows all 10 tickers with live prices, sparklines, color-coded direction arrows, and a shock event log. Runs for 60 seconds or until Ctrl+C.
