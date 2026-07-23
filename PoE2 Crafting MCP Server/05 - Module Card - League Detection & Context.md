# Module Card: League Detection & Context

## Overview

Manages the current league context for all API calls. Determines which league the user is playing, ensures all pricing/trade queries target the correct economy, and allows switching between leagues (e.g., challenge league → Standard).

---

## Why This Matters

Every API call is league-scoped:
- Trade searches return items listed in a specific league
- Currency prices differ between leagues (Standard ≠ challenge league)
- Crafting material costs can be 10x different between leagues
- Using the wrong league = completely wrong advice

---

## League Types in PoE2

| League | Description | Example |
|--------|-------------|---------|
| Challenge (current) | The seasonal league everyone plays | "Return of the Ancients" |
| Standard | Permanent league, characters migrate here after league ends | "Standard" |
| Hardcore Challenge | Seasonal + permadeath | "Hardcore Return of the Ancients" |
| Hardcore Standard | Permanent + permadeath | "Hardcore" |

Most players are on the **current challenge league**. That's our default.

---

## Detection Strategy

### Auto-Detection at Session Start

```python
async def detect_current_league() -> str:
    """
    Determine the current challenge league.
    
    Primary source: poe.show (no auth needed, always returns current league first)
    Fallback: GGG official API
    """
    # Primary: poe.show
    # GET /poe2/api/economy/leagues → [{"id": "Return of the Ancients", "name": "..."}]
    # First entry = current challenge league
    
    # Fallback: GGG API
    # GET https://api.pathofexile.com/league?realm=poe2&type=main
    # Filter for the one with category.current == true
```

### From PoB Build Import

When the user imports a build, the PoB XML contains the league the character is in:
```xml
<PathOfBuilding>
  <Build level="87" className="Witch" ascendClassName="Elementalist" 
         league="Return of the Ancients" ...>
```

We can cross-check: if the build says "Standard" but we auto-detected "Return of the Ancients", ask the user which they want.

### User Override

```
User: "I'm playing Standard"
Agent: *switches league context* → "Switched to Standard. Note: prices will 
       differ significantly from the challenge league."
```

---

## Python API Design

```python
from dataclasses import dataclass
from enum import Enum


class LeagueType(Enum):
    CHALLENGE = "challenge"
    STANDARD = "standard"
    HC_CHALLENGE = "hc_challenge"
    HC_STANDARD = "hc_standard"


@dataclass
class LeagueInfo:
    """Information about a specific league."""
    id: str              # API identifier (e.g., "Return of the Ancients")
    name: str            # Display name
    league_type: LeagueType
    realm: str           # "poe2"
    is_current: bool     # True if this is the active challenge league
    start_date: str | None
    end_date: str | None


class LeagueContext:
    """
    Manages league selection and provides the correct league ID
    to all other modules (pricing, trade, etc.).
    """

    def __init__(self, user_agent: str = "poe2-craft-mcp/1.0"):
        self._user_agent = user_agent
        self._current_league: LeagueInfo | None = None
        self._all_leagues: list[LeagueInfo] = []
        self._user_override: str | None = None

    @property
    def league_id(self) -> str:
        """
        The active league ID for API calls.
        Returns user override if set, otherwise auto-detected.
        """
        if self._user_override:
            return self._user_override
        if self._current_league:
            return self._current_league.id
        raise RuntimeError("League not initialized. Call initialize() first.")

    @property
    def league_info(self) -> LeagueInfo:
        """Full info about the active league."""
        ...

    async def initialize(self) -> LeagueInfo:
        """
        Auto-detect current league at session start.
        Called once when the MCP server starts or a new session begins.
        
        Returns:
            The detected current challenge league.
        """
        self._all_leagues = await self._fetch_leagues()
        
        # Find the current challenge league
        for league in self._all_leagues:
            if league.is_current and league.league_type == LeagueType.CHALLENGE:
                self._current_league = league
                return league
        
        # Fallback to first available
        if self._all_leagues:
            self._current_league = self._all_leagues[0]
            return self._current_league
        
        raise RuntimeError("Could not detect any active PoE2 leagues")

    def set_league(self, league_id: str) -> LeagueInfo:
        """
        Manually switch league context.
        
        Args:
            league_id: League name (e.g., "Standard", "Return of the Ancients")
                       Case-insensitive, partial matching supported.
        
        Raises:
            ValueError: If league_id doesn't match any known league.
        """
        # Fuzzy match against known leagues
        normalized = league_id.lower().strip()
        
        for league in self._all_leagues:
            if normalized in league.id.lower() or normalized in league.name.lower():
                self._user_override = league.id
                return league
        
        # Check common shortcuts
        shortcuts = {
            "standard": "Standard",
            "std": "Standard",
            "hc": "Hardcore",
            "hardcore": "Hardcore",
            "league": None,  # Reset to challenge league
            "challenge": None,
        }
        
        if normalized in shortcuts:
            if shortcuts[normalized] is None:
                self._user_override = None  # Back to auto-detected
                return self._current_league
            self._user_override = shortcuts[normalized]
            return self._find_league(shortcuts[normalized])
        
        available = [l.id for l in self._all_leagues]
        raise ValueError(
            f"Unknown league '{league_id}'. Available: {', '.join(available)}"
        )

    def reset_to_auto(self) -> LeagueInfo:
        """Reset to auto-detected challenge league."""
        self._user_override = None
        return self._current_league

    def infer_from_build(self, build_league: str | None) -> str | None:
        """
        Check if a loaded build's league conflicts with the current context.
        
        Returns:
            A warning message if there's a mismatch, None if all good.
        """
        if not build_league:
            return None
        
        if build_league.lower() == "standard" and self.league_id != "Standard":
            return (
                f"Your build is in Standard, but the current league context is "
                f"'{self.league_id}'. Prices and availability will differ. "
                f"Switch to Standard? (say 'switch to standard')"
            )
        
        if build_league != self.league_id:
            return (
                f"Your build is from '{build_league}' but we're looking at "
                f"'{self.league_id}'. This is fine if you migrated — prices "
                f"will be based on {self.league_id}."
            )
        
        return None

    async def _fetch_leagues(self) -> list[LeagueInfo]:
        """Fetch available leagues from poe.show + GGG API."""
        leagues = []
        
        # Source 1: poe.show (simple, current league first)
        poe_show_leagues = await self._fetch_poe_show_leagues()
        
        # Source 2: GGG API (more detail, includes HC variants)
        ggg_leagues = await self._fetch_ggg_leagues()
        
        # Merge and deduplicate
        ...
        
        return leagues

    async def _fetch_poe_show_leagues(self) -> list[dict]:
        """GET /poe2/api/economy/leagues"""
        import aiohttp
        async with aiohttp.ClientSession() as session:
            async with session.get(
                "https://poe.show/poe2/api/economy/leagues",
                headers={"User-Agent": self._user_agent}
            ) as resp:
                return await resp.json()

    async def _fetch_ggg_leagues(self) -> list[dict]:
        """GET https://api.pathofexile.com/league?realm=poe2&type=main"""
        import aiohttp
        async with aiohttp.ClientSession() as session:
            async with session.get(
                "https://api.pathofexile.com/league",
                params={"realm": "poe2", "type": "main"},
                headers={"User-Agent": self._user_agent}
            ) as resp:
                if resp.status == 200:
                    data = await resp.json()
                    return data.get("leagues", [])
                return []

    def _find_league(self, league_id: str) -> LeagueInfo:
        for league in self._all_leagues:
            if league.id == league_id:
                return league
        raise ValueError(f"League '{league_id}' not found")
```

---

## Integration With Other Modules

The `LeagueContext` is a **shared dependency** injected into every other module:

```python
class MCPServer:
    def __init__(self):
        self.league = LeagueContext()
        self.pricing = PricingEngine(league_context=self.league)
        self.trade = TradeClient(league_context=self.league)
        self.pob = PoBEngine(POB_PATH)
        self.session = BuildSession(self.pob)
    
    async def start(self):
        # First thing: detect league
        league_info = await self.league.initialize()
        print(f"Active league: {league_info.id}")
```

Every pricing/trade call reads `self.league.league_id` at call time:
```python
class PricingEngine:
    async def get_currency_price(self, name: str) -> CurrencyPrice:
        league = self.league_context.league_id  # Always current
        return await self._poe_show.fetch_price(league, name)
```

This means switching leagues mid-session immediately affects all subsequent queries — no restart needed.

---

## Session Start Sequence

```
1. MCP server starts
2. LeagueContext.initialize()
   → Fetches poe.show leagues → "Return of the Ancients"
   → Fetches GGG leagues → confirms, gets HC variants too
3. User imports build (PoB code)
4. LeagueContext.infer_from_build(build.league)
   → If mismatch: warn user, ask if they want to switch
5. All pricing/trade queries use league_context.league_id
```

---

## MCP Tool Exposure

| MCP Tool | Description |
|----------|-------------|
| `get_current_league()` | Show which league is active and what's available |
| `switch_league(league_name)` | Switch to a different league (e.g., "Standard") |

---

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Between leagues (no challenge active) | Default to Standard, inform user |
| League just launched (no economy data yet) | poe.show returns empty → warn user that prices may be inaccurate for the first few hours |
| User types "HC" | Switch to Hardcore variant of current challenge league |
| Build from an old/ended league | Warn user, suggest Standard (where the character migrated) |
| poe.show down | Fall back to GGG API for league detection, cached prices for economy |

---

## Sprint Estimate

| Task | Effort |
|------|--------|
| League fetch from poe.show + GGG API | 0.5 day |
| LeagueContext class with auto-detection | 0.5 day |
| User override + fuzzy matching | 0.5 day |
| Build-league mismatch detection | 0.5 day |
| Integration as shared dependency | 0.5 day |
| **Total** | **2.5 days** |
