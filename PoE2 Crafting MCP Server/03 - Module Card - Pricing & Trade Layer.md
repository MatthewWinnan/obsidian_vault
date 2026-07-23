# Module Card: Pricing & Trade Layer

## Overview

A multi-source pricing module that fetches live economy data, searches for specific items on trade, and provides the cost basis for both "buy" and "craft" decision paths. The crafting advisor needs to know: what does the finished item cost to buy, what do the materials cost, and is crafting cheaper?

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Pricing & Trade Module                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │  Economy Cache   │  │  Trade Search    │  │  Price Oracle  │  │
│  │                 │  │                  │  │               │  │
│  │  poe.show      │  │  trade2 API      │  │  Aggregates   │  │
│  │  poe.ninja     │  │  (rate-limited)  │  │  all sources  │  │
│  │                 │  │                  │  │               │  │
│  │  Refreshes     │  │  User-triggered  │  │  "What's the  │  │
│  │  every 10 min  │  │  or budget-aware │  │   cheapest    │  │
│  │                 │  │  batch searches  │  │   path?"      │  │
│  └────────┬────────┘  └────────┬─────────┘  └───────┬───────┘  │
│           │                    │                     │           │
│           └────────────────────┴─────────────────────┘           │
│                              │                                   │
│                              ▼                                   │
│                    Unified Price API                              │
│                                                                 │
│  get_currency_price("Exalted Orb") → 0.85 divine                │
│  get_base_price("Vaal Regalia", ilvl=86) → 0.2 divine           │
│  get_item_listings(filters) → [TradeResult, ...]                │
│  get_craft_material_cost(recipe) → 3.5 divine                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Sources & Their Roles

### 1. poe.show — Currency & Commodity Pricing (Primary)

**What it gives us:** Live exchange rates for all tradeable currencies, essences, catalysts, fragments, runes, soul cores, etc.

**Endpoints:**
```
GET /poe2/api/economy/leagues
GET /poe2/api/economy/exchange/current/overview?league={league}&type={type}
GET /poe2/api/economy/stash/current/overview?league={league}&type={type}
```

**PoE2 types:** Currency, Fragments, Essences, SoulCores, Runes, Ritual (Omens), Breach (Catalysts), Expedition, Delirium, Verisium, Abyss, UncutGems, LineageSupportGems, Idols

**Use for:**
- "How much does an Exalted Orb cost?"
- "How much does a Deafening Essence of Wrath cost?"
- Total crafting material cost calculation
- Currency conversion (everything normalized to Divine/Exalt)

**Refresh rate:** Hourly (we poll every 10 minutes, use cached data)

### 2. poe.ninja — Unique Items, Bases, Broader Economy

**What it gives us:** Pricing for uniques, base items, skill gems, and market trends.

**Endpoints (undocumented, stable):**
```
GET /api/data/currencyoverview?league={league}&type=Currency
GET /api/data/itemoverview?league={league}&type=UniqueWeapon
GET /api/data/itemoverview?league={league}&type=UniqueArmour
GET /api/data/itemoverview?league={league}&type=BaseType
```

**Use for:**
- "How much does a finished Vaal Regalia base cost at ilvl 86?"
- "What's this unique worth?" (for comparison: is crafting better than buying a unique?)
- Historical price trends

### 3. Official Trade2 API — Specific Item Search

**What it gives us:** Actual live listings matching exact mod criteria.

**Endpoints:**
```
POST /api/trade2/search/poe2/{league}  → returns { id, result: [item_ids], total }
GET  /api/trade2/fetch/{ids}?query={query_id}  → returns full item details with prices
```

**Use for:**
- "Find me a Vaal Regalia with +100 life and T1 fire res listed under 2 divine"
- "What does the finished item I want actually cost on trade right now?"
- Comparing "buy finished" vs "craft it yourself"

**Rate limits:** ~10 requests per 5 seconds. We budget max 3-5 trade searches per advisor session.

### 4. poeprices.info — ML Rare Item Valuation (Optional)

**What it gives us:** A predicted price for a rare item based on its mods.

**Use for:**
- Quick ballpark: "Is this item I could craft worth 5 divine or 50 divine?"
- Sanity-checking whether a crafting project's output has market value

---

## Python API Design

```python
from dataclasses import dataclass
from enum import Enum


class CurrencyType(Enum):
    DIVINE = "divine"
    EXALTED = "exalted"
    CHAOS = "chaos"


@dataclass
class CurrencyPrice:
    """Price of a currency/material in terms of a reference currency."""
    name: str
    price_in_divine: float
    price_in_exalt: float
    stack_size: int
    icon_url: str | None = None


@dataclass
class BaseItemPrice:
    """Price of a base item type at a given ilvl."""
    base_type: str
    item_level: int
    price_in_divine: float
    listings_count: int
    confidence: str  # "high", "medium", "low"


@dataclass
class TradeResult:
    """A single item listing from the trade site."""
    item_id: str
    price_amount: float
    price_currency: str
    price_in_divine: float
    seller: str
    item_text: str  # Raw item text (for feeding into PoB)
    mods: list[str]
    listed_time: str


@dataclass
class CraftingCostEstimate:
    """Total estimated cost of a crafting recipe."""
    materials: list[tuple[str, int, float]]  # (name, quantity, cost_per_unit)
    total_cost_divine: float
    base_item_cost: float
    currency_cost: float


class PricingEngine:
    """
    Multi-source pricing aggregator for PoE2.
    
    Handles:
    - Currency/material pricing (poe.show)
    - Base item pricing (poe.ninja)  
    - Specific item searches (trade2 API)
    - Crafting cost calculation
    """

    def __init__(self, league: str | None = None, user_agent: str = "poe2-craft-mcp/1.0"):
        """
        Args:
            league: Override league name. If None, auto-detects current league.
            user_agent: User-Agent header for API requests.
        """
        ...

    # ─── Currency & Material Pricing ─────────────────────────────────

    async def get_currency_price(self, currency_name: str) -> CurrencyPrice:
        """Get current price of a currency/crafting material."""
        ...

    async def get_all_currency_prices(self) -> dict[str, CurrencyPrice]:
        """Get all currency prices (cached, refreshes every 10 min)."""
        ...

    async def get_essence_price(self, essence_name: str) -> CurrencyPrice:
        """Get price of a specific essence."""
        ...

    # ─── Base Item Pricing ───────────────────────────────────────────

    async def get_base_price(
        self, base_type: str, item_level: int = 86, influence: str | None = None
    ) -> BaseItemPrice:
        """Get price of a base item type."""
        ...

    # ─── Trade Search (Rate-Limited) ─────────────────────────────────

    async def search_trade(
        self,
        base_type: str | None = None,
        min_mods: dict[str, float] | None = None,
        max_price: float | None = None,
        item_class: str | None = None,
        min_ilvl: int | None = None,
    ) -> list[TradeResult]:
        """
        Search the trade site for specific items.
        
        Rate-limited: max 3-5 calls per session.
        Results cached for 5 minutes.
        """
        ...

    async def get_trade_url(
        self,
        base_type: str | None = None,
        min_mods: dict[str, float] | None = None,
        max_price: float | None = None,
    ) -> str:
        """
        Generate a trade site URL for the user to open manually.
        No rate limit cost — just builds the URL.
        """
        ...

    # ─── Crafting Cost Calculation ───────────────────────────────────

    async def estimate_craft_cost(
        self, recipe: "CraftingRecipe"
    ) -> CraftingCostEstimate:
        """
        Calculate total cost of executing a crafting recipe.
        
        Considers:
        - Base item cost (from trade/poe.ninja)
        - Currency cost per attempt (from poe.show)
        - Expected number of attempts (from crafting probability engine)
        """
        ...

    async def compare_buy_vs_craft(
        self,
        target_mods: dict[str, float],
        base_type: str,
        crafting_recipe: "CraftingRecipe",
    ) -> dict:
        """
        Compare: buy the finished item on trade vs. craft it yourself.
        
        Returns:
            {
                "buy_price": 15.0,      # cheapest listing matching target
                "craft_cost": 8.5,      # expected cost to craft
                "recommendation": "craft",
                "savings": 6.5,
                "confidence": "medium",
                "trade_url": "https://...",   # if buy is recommended
                "craft_steps": [...]          # if craft is recommended
            }
        """
        ...

    # ─── Cache Management ────────────────────────────────────────────

    async def refresh_cache(self) -> None:
        """Force refresh all cached pricing data."""
        ...

    def get_cache_age(self) -> float:
        """Return age of economy cache in seconds."""
        ...
```

---

## Crafting Cost Calculation — The Key Logic

The crafting advisor needs to answer: **"How much will it cost me to craft this item?"**

This requires combining data from multiple sources:

```python
async def estimate_craft_cost(self, recipe: CraftingRecipe) -> CraftingCostEstimate:
    materials = []
    
    # 1. Base item cost
    base_price = await self.get_base_price(
        recipe.base_type, 
        recipe.required_ilvl
    )
    materials.append((recipe.base_type, 1, base_price.price_in_divine))
    
    # 2. For each crafting step, calculate expected currency cost
    for step in recipe.steps:
        currency_price = await self.get_currency_price(step.currency_name)
        
        # Expected attempts = 1 / probability_of_success
        # (probability comes from the Crafting Probability Engine module)
        expected_attempts = 1.0 / step.success_probability
        cost_per_attempt = currency_price.price_in_divine * step.currency_per_attempt
        
        step_cost = cost_per_attempt * expected_attempts
        materials.append((
            step.currency_name,
            int(expected_attempts * step.currency_per_attempt),
            currency_price.price_in_divine
        ))
    
    total = sum(cost for _, _, cost in materials)
    
    return CraftingCostEstimate(
        materials=materials,
        total_cost_divine=total,
        base_item_cost=base_price.price_in_divine,
        currency_cost=total - base_price.price_in_divine,
    )
```

### Example: "I want a Vaal Regalia with +100 Life and T1 Fire Res"

```
Step 1: Buy ilvl 86 Vaal Regalia base → 0.1 divine (from poe.ninja)
Step 2: Spam Chaos Orbs until +100 life prefix → probability 1/45
         Chaos Orb price: 0.003 divine each
         Expected cost: 45 × 0.003 = 0.135 divine
Step 3: Exalt for fire res suffix → probability 1/12  
         Exalted Orb price: 0.85 divine each
         Expected cost: 12 × 0.85 = 10.2 divine
         
Total craft cost: ~10.4 divine
Buy finished on trade: ~15 divine (from trade search)
Recommendation: CRAFT (save ~4.6 divine)
```

---

## Rate Limiting Strategy

```python
class RateLimiter:
    """Respects GGG's X-Rate-Limit headers with exponential backoff."""
    
    def __init__(self):
        self.requests_made = 0
        self.window_start = time.time()
        self.retry_after = 0
        self.max_per_session = 5  # Hard cap for MCP advisor sessions
    
    async def acquire(self) -> bool:
        """Returns True if we can make a request, False if budget exhausted."""
        if self.requests_made >= self.max_per_session:
            return False
        if time.time() < self.retry_after:
            await asyncio.sleep(self.retry_after - time.time())
        return True
    
    def update_from_headers(self, headers: dict):
        """Parse X-Rate-Limit-Client and X-Rate-Limit-Client-State headers."""
        if "Retry-After" in headers:
            self.retry_after = time.time() + int(headers["Retry-After"])
        # Parse: "10:5:10" = 10 max hits, 5 second window, 10 second penalty
        if "X-Rate-Limit-Client" in headers:
            parts = headers["X-Rate-Limit-Client"].split(":")
            self.max_hits = int(parts[0])
            self.window_seconds = int(parts[1])
```

---

## Caching Strategy

| Data Source | Cache Duration | Refresh Trigger |
|-------------|---------------|-----------------|
| poe.show economy | 10 minutes | Background timer |
| poe.ninja items | 15 minutes | Background timer |
| Trade search results | 5 minutes | Per-query key |
| Base item prices | 30 minutes | On-demand |

```python
class PriceCache:
    """Simple TTL cache for pricing data."""
    
    def __init__(self):
        self._store: dict[str, tuple[float, Any]] = {}  # key -> (expires_at, value)
    
    def get(self, key: str) -> Any | None:
        if key in self._store:
            expires_at, value = self._store[key]
            if time.time() < expires_at:
                return value
            del self._store[key]
        return None
    
    def set(self, key: str, value: Any, ttl_seconds: float):
        self._store[key] = (time.time() + ttl_seconds, value)
```

---

## MCP Tool Exposure

| MCP Tool | Method(s) Used | Rate-Limited? |
|----------|---------------|---------------|
| `get_currency_prices()` | `get_all_currency_prices()` | No (cached) |
| `get_item_price(item_text)` | poeprices.info or trade search | Maybe |
| `search_trade(filters, budget)` | `search_trade()` | Yes |
| `get_trade_url(filters)` | `get_trade_url()` | No (URL only) |
| `estimate_craft_cost(recipe)` | `estimate_craft_cost()` | No (uses cached prices) |
| `compare_buy_vs_craft(target, base, recipe)` | `compare_buy_vs_craft()` | Yes (1 trade search) |

---

## Integration with Crafting Advisor Flow

```
User: "I want more fire damage on my weapon, budget 5 divine"

Agent flow:
1. get_build_stats() → current DPS
2. Identify slot + mods that would increase fire damage
3. get_currency_prices() → current orb costs [cached, free]
4. get_base_price("Infernal Blade", ilvl=83) → 0.05 divine [cached, free]  
5. get_crafting_odds("Infernal Blade", 83, desired_mods) → probability data [local calc]
6. estimate_craft_cost(recipe) → 3.2 divine expected [uses cached prices]
7. search_trade(base="Infernal Blade", mods=desired, max_price=5.0) → listings [1 API call]
8. compare results:
   - Cheapest on trade: 4.8 divine
   - Expected craft cost: 3.2 divine
9. Recommend: "Craft it. Here's the step-by-step recipe..."
```

---

## File Layout

```
poe2_crafting_mcp/
├── pricing/
│   ├── __init__.py
│   ├── engine.py          ← PricingEngine class
│   ├── models.py          ← CurrencyPrice, TradeResult, etc.
│   ├── cache.py           ← PriceCache with TTL
│   ├── rate_limiter.py    ← RateLimiter with header parsing
│   ├── sources/
│   │   ├── __init__.py
│   │   ├── poe_show.py    ← poe.show client
│   │   ├── poe_ninja.py   ← poe.ninja client
│   │   ├── trade_api.py   ← Official trade2 client
│   │   └── poeprices.py   ← poeprices.info client (optional)
│   └── trade_query.py     ← Query builder (mods → trade JSON)
```

---

## Sprint Estimate

| Task | Effort |
|------|--------|
| poe.show client + caching | 1 day |
| poe.ninja client + caching | 1 day |
| Trade2 API client + rate limiter | 2 days |
| Trade query builder (mod filters → JSON) | 1-2 days |
| `estimate_craft_cost()` integration | 1 day |
| `compare_buy_vs_craft()` logic | 1 day |
| **Total** | **7-9 days** |
