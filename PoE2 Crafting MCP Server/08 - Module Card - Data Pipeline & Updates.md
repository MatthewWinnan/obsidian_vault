# Module Card: Data Pipeline & Updates

## Overview

Manages all external data sources, ETL (Extract-Transform-Load) flows, local storage, deduplication, and update scheduling. Every other module depends on this for its data. The local SQLite database acts as the single source of truth — all modules query it, never the APIs directly (except for real-time pricing).

---

## Design Principles

1. **Local-first:** All data stored locally in SQLite. No redundant fetches.
2. **Deduplication:** Same data never fetched twice within its TTL.
3. **Version-tagged:** Every dataset tagged with patch version and fetch timestamp.
4. **Graceful degradation:** If an upstream source is down, serve stale data with a warning.
5. **League-aware:** Full refresh on new league; incremental during league.

---

## Data Sources Inventory

| Source | Data Type | Refresh Rate | Storage | Size Estimate |
|--------|-----------|--------------|---------|---------------|
| PoB-PoE2 `src/Data/` | Mod pools, bases, skills, uniques | Per-patch | SQLite `mods`, `bases`, `skills` | ~20MB |
| Prohibited Library / Craft of Exile | Mod weights | Per-patch (delayed 1-4 weeks) | SQLite `mod_weights` | ~2MB |
| poe.show economy API | Currency prices, fragment prices | Every 10 min | SQLite `economy_prices` | ~1MB |
| poe.ninja economy API | Uniques, bases, currency | Every 15 min | SQLite `economy_items` | ~5MB |
| poe.ninja builds API | Top player characters, gear, skills | Daily | SQLite `meta_builds` | ~50-100MB |
| GGG Leagues API | Active leagues | On startup + daily | SQLite `leagues` | ~1KB |
| poe2db.tw | Supplementary mod data, tags | Per-patch | SQLite `mod_tags` | ~5MB |
| Community weight spreadsheets | Raw empirical weight data | Per-patch (manual) | SQLite `mod_weights_raw` | ~500KB |

---

## Local Database Schema (SQLite)

```sql
-- ═══════════════════════════════════════════════════════
-- METADATA & VERSIONING
-- ═══════════════════════════════════════════════════════

CREATE TABLE data_versions (
    source TEXT PRIMARY KEY,          -- 'pob_data', 'mod_weights', 'poe_show', etc.
    patch_version TEXT,               -- '0.5.0', '0.6.0'
    last_updated_at REAL,             -- Unix timestamp
    last_fetch_at REAL,               -- When we last pulled
    data_hash TEXT,                   -- SHA256 of the dataset (for change detection)
    confidence TEXT DEFAULT 'high',   -- 'high', 'medium', 'low'
    notes TEXT
);

CREATE TABLE leagues (
    id TEXT PRIMARY KEY,
    name TEXT,
    league_type TEXT,                 -- 'challenge', 'standard', 'hc_challenge', 'hc_standard'
    realm TEXT DEFAULT 'poe2',
    is_current INTEGER DEFAULT 0,
    start_date TEXT,
    end_date TEXT,
    fetched_at REAL
);

-- ═══════════════════════════════════════════════════════
-- MOD POOLS & WEIGHTS (from PoB + community)
-- ═══════════════════════════════════════════════════════

CREATE TABLE item_bases (
    base_id TEXT PRIMARY KEY,
    base_type TEXT,                   -- 'Vaal Regalia', 'Infernal Blade'
    item_class TEXT,                  -- 'Body Armour', 'One Hand Sword'
    base_category TEXT,               -- 'body_armour_int', 'weapon_sword_1h'
    tags TEXT,                        -- JSON array of tags
    implicit_mods TEXT,               -- JSON array
    requirements TEXT,                -- JSON: {level, str, dex, int}
    patch_version TEXT
);

CREATE TABLE modifiers (
    mod_id TEXT PRIMARY KEY,
    name TEXT,                        -- 'Tyrannical', 'Incandescent'
    stat_text TEXT,                   -- '+170 to 179% increased Physical Damage'
    affix_type TEXT,                  -- 'prefix', 'suffix'
    tier INTEGER,
    mod_group TEXT,                   -- Mods in same group are mutually exclusive
    ilvl_required INTEGER,
    tags TEXT,                        -- JSON array: which item tags enable this mod
    value_min REAL,
    value_max REAL,
    patch_version TEXT
);

CREATE TABLE mod_weights (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    mod_id TEXT REFERENCES modifiers(mod_id),
    base_category TEXT,              -- Which base category this weight applies to
    weight INTEGER,                  -- The rolling weight
    source TEXT,                     -- 'prohibited_library', 'trade_parsed', 'assumed'
    confidence TEXT,                 -- 'high', 'medium', 'low'
    patch_version TEXT,
    updated_at REAL,
    UNIQUE(mod_id, base_category, patch_version)
);

CREATE TABLE essences (
    essence_id TEXT PRIMARY KEY,
    name TEXT,                        -- 'Shrieking Essence of Wrath'
    tier TEXT,                        -- 'shrieking', 'deafening', 'greater'
    guaranteed_mod_group TEXT,        -- Which mod group it guarantees
    guaranteed_stat_text TEXT,
    item_classes TEXT,                -- JSON: which item classes it works on
    patch_version TEXT
);

-- ═══════════════════════════════════════════════════════
-- ECONOMY DATA (from poe.show / poe.ninja)
-- ═══════════════════════════════════════════════════════

CREATE TABLE economy_currency (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    league TEXT,
    name TEXT,
    price_divine REAL,
    price_exalt REAL,
    price_chaos REAL,
    listings_count INTEGER,
    source TEXT,                      -- 'poe_show', 'poe_ninja'
    fetched_at REAL,
    UNIQUE(league, name, source)
);

CREATE TABLE economy_items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    league TEXT,
    name TEXT,
    item_type TEXT,                   -- 'unique_weapon', 'unique_armour', 'base_type'
    base_type TEXT,
    item_level INTEGER,
    price_divine REAL,
    listings_count INTEGER,
    source TEXT,
    fetched_at REAL,
    UNIQUE(league, name, item_type, source)
);

-- ═══════════════════════════════════════════════════════
-- META BUILDS (from poe.ninja builds)
-- ═══════════════════════════════════════════════════════

CREATE TABLE meta_characters (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    league TEXT,
    character_name TEXT,
    account_name TEXT,
    class TEXT,
    ascendancy TEXT,
    level INTEGER,
    main_skill TEXT,
    dps REAL,
    life REAL,
    energy_shield REAL,
    equipment TEXT,                   -- JSON: full gear snapshot
    passives TEXT,                    -- JSON: passive node hashes
    fetched_at REAL
);

CREATE TABLE meta_item_patterns (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    league TEXT,
    ascendancy TEXT,
    main_skill TEXT,
    slot TEXT,                        -- 'Body Armour', 'Weapon 1', etc.
    mod_text TEXT,
    occurrence_count INTEGER,
    sample_size INTEGER,
    occurrence_percent REAL,
    avg_tier REAL,
    computed_at REAL,
    UNIQUE(league, ascendancy, main_skill, slot, mod_text)
);

-- ═══════════════════════════════════════════════════════
-- INDICES
-- ═══════════════════════════════════════════════════════

CREATE INDEX idx_modifiers_group ON modifiers(mod_group);
CREATE INDEX idx_modifiers_category ON mod_weights(base_category);
CREATE INDEX idx_economy_currency_league ON economy_currency(league, fetched_at);
CREATE INDEX idx_economy_items_league ON economy_items(league, item_type);
CREATE INDEX idx_meta_chars_skill ON meta_characters(league, main_skill, ascendancy);
CREATE INDEX idx_meta_patterns ON meta_item_patterns(league, ascendancy, main_skill, slot);
```

---

## ETL Pipelines

### Pipeline 1: PoB Data Sync (Mod Pools)

**Trigger:** New patch detected, or manual `refresh_data` command.

```
┌────────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│  PoB-PoE2 Git Repo │────▶│  Lua → JSON Parser   │────▶│  SQLite tables  │
│  src/Data/         │     │                      │     │  item_bases     │
│                    │     │  Reads:              │     │  modifiers      │
│  - ModItem.lua     │     │  - Base definitions  │     │  essences       │
│  - Bases/*.lua     │     │  - Mod pools         │     │                 │
│  - Uniques/*.lua   │     │  - Essence mappings  │     │                 │
│  - Skills/*.lua    │     │  - Skill data        │     │                 │
└────────────────────┘     └──────────────────────┘     └─────────────────┘
```

```python
class PoBDataSync:
    """Syncs mod/base data from PoB-PoE2 Lua files into SQLite."""

    def __init__(self, pob_path: Path, db: Database):
        self.pob_path = pob_path
        self.db = db

    async def sync(self, force: bool = False) -> SyncResult:
        """
        Pull latest PoB data and update local database.
        
        Steps:
        1. git pull the PoB-PoE2 repo (or check if already up to date)
        2. Parse Lua data tables → intermediate JSON
        3. Diff against current DB (skip if unchanged)
        4. Upsert new/changed records
        5. Update data_versions table
        """
        # Check if update needed
        current_hash = self._hash_data_directory()
        stored_hash = self.db.get_data_version("pob_data").data_hash

        if current_hash == stored_hash and not force:
            return SyncResult(status="up_to_date", records_changed=0)

        # Parse Lua → JSON
        bases = self._parse_bases()
        mods = self._parse_modifiers()
        essences = self._parse_essences()

        # Upsert into SQLite
        self.db.upsert_bases(bases)
        self.db.upsert_modifiers(mods)
        self.db.upsert_essences(essences)

        # Update version
        self.db.set_data_version("pob_data", 
            patch_version=self._detect_patch_version(),
            data_hash=current_hash)

        return SyncResult(status="updated", records_changed=len(mods))

    def _parse_modifiers(self) -> list[dict]:
        """Parse PoB's Lua mod tables into Python dicts."""
        # PoB stores mods in src/Data/ModItem.lua and related files
        # Use lupa to execute the Lua and extract the tables
        ...
```

### Pipeline 2: Weight Data Import

**Trigger:** Manual (after community publishes new weights), or on new patch.

```
┌──────────────────────┐     ┌────────────────────┐     ┌─────────────────┐
│  Weight Sources      │────▶│  Normalizer        │────▶│  SQLite         │
│                      │     │                    │     │  mod_weights    │
│  - Prohibited Library│     │  - Validate ranges │     │                 │
│    spreadsheet (CSV) │     │  - Tag confidence  │     │                 │
│  - poe2db.tw scrape  │     │  - Merge sources   │     │                 │
│  - Craft of Exile    │     │  - Deduplicate     │     │                 │
│    (manual export)   │     │                    │     │                 │
└──────────────────────┘     └────────────────────┘     └─────────────────┘
```

```python
class WeightDataImporter:
    """Imports and merges mod weight data from community sources."""

    def import_prohibited_library(self, csv_path: Path) -> int:
        """
        Import from Prohibited Library spreadsheet export.
        Format: mod_id, base_category, weight, sample_size
        """
        ...

    def import_from_poe2db(self) -> int:
        """
        Scrape poe2db.tw modifier pages for weight data.
        Falls back to this when no direct spreadsheet available.
        """
        ...

    def import_from_craft_of_exile(self, json_path: Path) -> int:
        """
        Import from manually exported Craft of Exile data.
        (No API — must be manually captured from their site.)
        """
        ...

    def fill_missing_weights(self):
        """
        For mods with no community-gathered weight:
        - Assign default weight based on tier (T1=low, T6=high)
        - Mark confidence as 'low'
        - Log which mods are missing data
        """
        ...
```

### Pipeline 3: Economy Data (Continuous)

**Trigger:** Background timer (every 10 minutes), or on-demand.

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  poe.show API   │────▶│  Price Fetcher   │────▶│  SQLite         │
│  poe.ninja API  │     │                  │     │  economy_*      │
│                 │     │  - Dedup by key  │     │                 │
│  (HTTP GET)     │     │  - Normalize to  │     │                 │
│                 │     │    divine/exalt   │     │                 │
│                 │     │  - TTL check     │     │                 │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

```python
class EconomyDataFetcher:
    """Fetches and caches economy data with deduplication."""

    CURRENCY_TTL = 600       # 10 minutes
    ITEMS_TTL = 900          # 15 minutes
    BASE_TYPES_TTL = 1800    # 30 minutes

    async def refresh_if_stale(self):
        """Only fetch if data is older than TTL."""
        currency_age = self.db.get_age("economy_currency", self.league)
        if currency_age > self.CURRENCY_TTL:
            await self._fetch_currency()

        items_age = self.db.get_age("economy_items", self.league)
        if items_age > self.ITEMS_TTL:
            await self._fetch_items()

    async def _fetch_currency(self):
        """Fetch all currency types from poe.show, upsert into DB."""
        types = ["Currency", "Fragments", "Essences", "SoulCores",
                 "Runes", "Ritual", "Breach", "Expedition"]

        for t in types:
            data = await self.poe_show.get_exchange_overview(self.league, t)
            for item in data:
                self.db.upsert_economy_currency(
                    league=self.league,
                    name=item["name"],
                    price_divine=item["divine_value"],
                    source="poe_show",
                )
```

### Pipeline 4: Meta Builds (Daily)

**Trigger:** Once per day, or when user first asks for meta analysis.

```
┌─────────────────────┐     ┌────────────────────────┐     ┌─────────────────┐
│  poe.ninja builds   │────▶│  Build Analyser        │────▶│  SQLite         │
│  API                │     │                        │     │  meta_*         │
│                     │     │  - Parse characters    │     │                 │
│  (single large GET) │     │  - Extract gear per    │     │                 │
│                     │     │    slot                │     │                 │
│  ~50-100MB JSON     │     │  - Tally mod patterns  │     │                 │
│                     │     │  - Compute popularity  │     │                 │
└─────────────────────┘     └────────────────────────┘     └─────────────────┘
```

---

## Update Scheduling & Triggers

### Automatic Triggers

```python
class DataScheduler:
    """Manages all data refresh schedules."""

    SCHEDULES = {
        "economy_currency":  {"interval": 600,    "source": "poe_show"},     # 10 min
        "economy_items":     {"interval": 900,    "source": "poe_ninja"},    # 15 min
        "meta_builds":       {"interval": 86400,  "source": "poe_ninja"},    # 24 hours
        "leagues":           {"interval": 3600,   "source": "ggg_api"},      # 1 hour
        "pob_data":          {"interval": None,   "source": "git"},          # Manual/patch
        "mod_weights":       {"interval": None,   "source": "community"},    # Manual/patch
    }

    async def on_startup(self):
        """Run on MCP server start."""
        await self.detect_league_change()
        await self.economy.refresh_if_stale()
        # Don't auto-fetch meta builds on every startup (100MB)
        # Only fetch if cache is empty or >24h old

    async def detect_league_change(self):
        """
        Check if a new league has started since last run.
        If yes: trigger full refresh of ALL data.
        """
        current_league = await self.league_context.initialize()
        stored_league = self.db.get_active_league()

        if current_league.id != stored_league:
            logger.info(f"New league detected: {current_league.id}")
            await self.full_league_refresh()

    async def full_league_refresh(self):
        """
        Complete data refresh for a new league.
        Called when: new league starts, or user switches leagues.
        """
        # 1. Update league info
        await self.leagues.sync()

        # 2. Economy data (fresh for new league)
        await self.economy.force_refresh()

        # 3. PoB data (new patch usually accompanies new league)
        await self.pob_sync.sync(force=True)

        # 4. Clear old meta builds (they're from last league)
        self.db.clear_meta_builds(old_league)

        # 5. Fetch new meta builds (may be sparse day 1)
        await self.meta_builds.sync()

        # 6. Weights: keep old weights, flag as "previous patch"
        self.db.flag_weights_as_unverified()

        logger.info("Full league refresh complete")
```

### Manual Triggers (User Commands)

| Command | What It Does |
|---------|-------------|
| `refresh_prices` | Force-fetch economy data right now |
| `refresh_meta` | Re-pull poe.ninja builds data |
| `update_pob` | Git pull PoB-PoE2, re-parse mod pools |
| `import_weights(file)` | Load new weight data from file |
| `data_status` | Show all data sources, freshness, confidence |

---

## Deduplication & Freshness

### How We Avoid Duplicate Fetches

```python
class Database:
    def should_fetch(self, source: str, league: str) -> bool:
        """Check if data needs refreshing based on TTL."""
        row = self.conn.execute("""
            SELECT fetched_at FROM data_versions
            WHERE source = ? AND league = ?
        """, (source, league)).fetchone()

        if row is None:
            return True  # Never fetched

        age = time.time() - row["fetched_at"]
        ttl = DataScheduler.SCHEDULES[source]["interval"]

        if ttl is None:
            return False  # Manual-only source

        return age > ttl

    def upsert_economy_currency(self, league, name, price_divine, source):
        """Insert or update — never duplicate."""
        self.conn.execute("""
            INSERT INTO economy_currency (league, name, price_divine, source, fetched_at)
            VALUES (?, ?, ?, ?, ?)
            ON CONFLICT(league, name, source)
            DO UPDATE SET price_divine = excluded.price_divine,
                         fetched_at = excluded.fetched_at
        """, (league, name, price_divine, source, time.time()))
```

### Data Staleness Indicator

Every query response includes freshness metadata:

```python
@dataclass
class DataFreshness:
    source: str
    age_seconds: float
    is_stale: bool        # age > TTL
    confidence: str       # 'high', 'medium', 'low'
    patch_version: str
    warning: str | None   # e.g., "Weights from previous patch"
```

---

## New League Day-1 Strategy

The first day of a new league is special — data is sparse:

```
Hour 0-2:   League launches
            → League detection triggers full refresh
            → Economy data: EMPTY (no trades yet)
            → Meta builds: EMPTY (nobody at endgame yet)
            → Mod pools: Updated (PoB usually ready day 0-1)
            → Weights: Previous patch (flagged as unverified)

Hour 2-6:   First currency trades appear
            → Economy data starts populating
            → Prices volatile (use wide confidence bands)

Day 1-3:    Ladder fills, first characters reach maps
            → Meta builds start appearing (sparse)
            → Economy stabilizing

Day 3-7:    Full economy established
            → Meta builds meaningful (50+ per archetype)
            → Old weights likely still valid (GGG rarely changes weights between patches)

Week 2-4:   Community verifies weights
            → Prohibited Library publishes updated weights
            → We import them → confidence goes back to 'high'
```

**Agent behaviour during sparse data:**

```
Agent: "Note: It's day 1 of the new league. Economy prices are based on
       only 12 listings — they'll stabilize over the next 24 hours.
       Crafting probability estimates use previous patch weights
       (typically accurate unless GGG specifically noted weight changes
       in patch notes). I'll flag if anything seems off."
```

---

## File Layout

```
poe2_crafting_mcp/
├── data/
│   ├── pipeline/
│   │   ├── __init__.py
│   │   ├── scheduler.py       ← DataScheduler
│   │   ├── pob_sync.py        ← PoB Lua → SQLite ETL
│   │   ├── weight_import.py   ← Community weight data import
│   │   ├── economy_fetch.py   ← poe.show / poe.ninja fetcher
│   │   ├── meta_fetch.py      ← poe.ninja builds fetcher
│   │   └── lua_parser.py      ← Lua table → Python dict converter
│   ├── db/
│   │   ├── __init__.py
│   │   ├── database.py        ← SQLite connection, query helpers
│   │   ├── schema.sql         ← Table definitions
│   │   └── migrations/        ← Schema version migrations
│   ├── weights/
│   │   ├── README.md          ← How to update weights
│   │   ├── patch_0.5/
│   │   │   ├── body_armour_int.json
│   │   │   ├── weapon_sceptre.json
│   │   │   └── ...
│   │   └── patch_0.6/         ← Added when new patch drops
│   └── poe2_craft.db          ← The SQLite database file
```

---

## Database Size & Performance

| Table | Expected Rows | Size | Query Pattern |
|-------|---------------|------|---------------|
| `modifiers` | ~3,000-5,000 | ~2MB | Filter by base_category + ilvl |
| `mod_weights` | ~15,000 | ~3MB | Join with modifiers on mod_id |
| `economy_currency` | ~200 per league | ~50KB | Lookup by name |
| `economy_items` | ~5,000 per league | ~2MB | Filter by item_type |
| `meta_characters` | ~10,000 per league | ~50MB | Filter by skill + class |
| `meta_item_patterns` | ~50,000 per league | ~10MB | Filter by archetype + slot |
| **Total** | | **~70MB** | |

SQLite handles this easily. No need for Postgres or Redis — single-user local tool.

---

## MCP Tool Exposure

| MCP Tool | Description |
|----------|-------------|
| `data_status()` | Show all data sources, freshness, last update, confidence |
| `refresh_prices()` | Force-refresh economy data |
| `refresh_meta()` | Force-refresh meta builds |
| `update_game_data()` | Pull latest PoB data + re-parse |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| poe.ninja builds endpoint returns 100MB+ | Slow/fails on bad connections | Stream + parse incrementally; store compressed; only fetch daily |
| PoB-PoE2 Lua format changes | Parser breaks | Version-check PoB repo; pin to known-good release tags |
| Weight data becomes stale after stealth nerf | Bad probability estimates | Flag confidence; encourage user to report if results seem off |
| SQLite corruption | Data loss | WAL mode for crash safety; periodic backup; can always re-sync |
| Multiple modules fetching same data | Duplicate requests | All fetches go through DataScheduler with TTL check |
| New league has no data | Agent gives no advice | Serve previous-patch data with warnings; focus on PoB sim (works without economy data) |

---

## Sprint Estimate

| Task | Effort |
|------|--------|
| SQLite schema + database class | 1 day |
| PoB Lua → SQLite parser | 2-3 days |
| Economy fetcher (poe.show + poe.ninja) | 1-2 days |
| Meta builds fetcher + pattern analyser | 2 days |
| Weight data importer (CSV/JSON) | 1 day |
| DataScheduler (TTL, triggers, league detection) | 1 day |
| Deduplication + upsert logic | 0.5 day |
| `data_status` reporting | 0.5 day |
| **Total** | **9-11 days** |
