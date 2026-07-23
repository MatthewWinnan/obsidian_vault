# PoE2 Crafting MCP Server

## Vision

An MCP (Model Context Protocol) server that acts as a crafting advisor for Path of Exile 2. Given your current build, budget, and goal (e.g., "more damage", "cap resistances"), it determines the optimal path forward — whether that's buying specific items on trade or following a crafting recipe — all within your stated budget.

---

## Research Summary

### Available Data Sources & APIs

#### 1. Official GGG API (`api.pathofexile.com`)

OAuth 2.1 authenticated. Provides:

- **Account Characters** (`GET /character/poe2/<name>`) — Full character data: equipment, skills, jewels, passive tree.
- **Leagues** (`GET /league?realm=poe2`) — Current league names.
- **Currency Exchange** (`GET https://web.poecdn.com/api/currency-exchange/poe2/<id>`) — Hourly aggregate trade history for currency pairs.

Limitations:
- Account Stashes are PoE1 only — no official stash API for PoE2.
- No officially documented trade search API for PoE2.
- **OAuth app registration is currently closed** (GGG not processing new applications).

#### 2. PoE Trade Site (Unofficial/Internal API)

`pathofexile.com/trade2` works via:
- `POST /api/trade2/search/poe2/<league>` — JSON search query, returns item IDs.
- `GET /api/trade2/fetch/<ids>?query=<query_id>` — Fetch full item details.

Used by Exiled Exchange 2, Sidekick, etc. Functional but not officially supported. Aggressive rate limits.

#### 3. poe.show Economy API (Public, Supported)

Best source for live pricing:
- `GET /poe2/api/economy/leagues` — Active leagues
- `GET /poe2/api/economy/exchange/current/overview?league={league}&type={type}` — Pricing data

Categories: Currency, Fragments, Essences, SoulCores, Idols, Runes, Omens, Catalysts, Verisium, and more. Refreshes ~hourly.

#### 4. poe.ninja API (Undocumented, Widely Used)

Tracks PoE2 economy across 14+ currency categories and 8 unique item categories:
- `https://poe.ninja/api/data/currencyoverview?league=<league>&type=Currency`
- `https://poe.ninja/api/data/itemoverview?league=<league>&type=UniqueWeapon`

#### 5. poe2db.tw (Game Data — Mod Pools & Weightings)

The definitive source for item bases, affix pools, mod weightings, and crafting mechanics. Exposes 4 API endpoints covering item stats, skill gems, passive nodes, and league history.

#### 6. Craft of Exile (Crafting Simulator)

Full PoE2 crafting emulator at `craftofexile.com/emulator?game=poe2`. Updated to patch 0.5 (Return of the Ancients). **No public API** — web UI only. The underlying math (mod weightings per base/ilvl) can be replicated from game data exports.

---

### Build Calculation Engine: Path of Building PoE2

[PathOfBuildingCommunity/PathOfBuilding-PoE2](https://github.com/PathOfBuildingCommunity/PathOfBuilding-PoE2) — the community build planner for PoE2.

Key capability: **`HeadlessWrapper.lua`** — runs PoB calculations without the GUI.

Existing wrappers:
| Project | Language | Description |
|---------|----------|-------------|
| `coldino/pob_wrapper` | Python | Controls PoB from Python, loads builds, modifies items, reads DPS |
| `adamkilpatrick/PobWrapperDotNet` | .NET | Thin wrapper around PoB Lua internals |
| `ppoelzl/PathOfBuildingAPI` | Python | Parses PoB pastebin/XML export format |

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   MCP Server                         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Tools exposed to the LLM:                          │
│                                                     │
│  1. get_current_build(account, character)            │
│     → GGG OAuth API → character equipment/tree      │
│                                                     │
│  2. search_trade(filters, budget)                   │
│     → trade2 API (unofficial)                       │
│     → poe.show for pricing context                  │
│                                                     │
│  3. get_item_price(item_name_or_mods)               │
│     → poe.show or poe.ninja economy API             │
│                                                     │
│  4. simulate_item_swap(build_xml, slot, new_item)   │
│     → PoB headless (Lua) → returns DPS delta        │
│                                                     │
│  5. get_crafting_odds(base, ilvl, desired_mods)     │
│     → Local mod weighting DB (from poe2db/exports)  │
│     → Calculate probability & expected cost         │
│                                                     │
│  6. get_crafting_recipe(target_item)                │
│     → Lookup optimal craft path given budget        │
│     → Uses mod pool data + currency prices          │
│                                                     │
│  7. get_currency_prices()                           │
│     → poe.show /poe2/api/economy/exchange           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Key Challenges

### 1. PoB-PoE2 Headless Compatibility
The `pob_wrapper` was built for PoE1's PoB. The PoE2 fork has different data structures, skill systems, and item formats. HeadlessWrapper.lua may need patches for PoE2.

### 2. Trade API is Unofficial
Rate limits are strict (~10 requests per 5 seconds). No stability guarantee. Could break at any time.

### 3. OAuth App Registration Closed
Cannot get official API credentials from GGG right now. Character data access requires workarounds.

### 4. Mod Data Maintenance
Every patch changes mod pools. Need automated extraction from game files or dependence on poe2db.tw staying current.

### 5. Crafting Probability Engine
Must implement weighting math: `P(mod) = weight(mod) / sum(all_eligible_weights)` per prefix/suffix slot. Multi-step craft paths (e.g., "use Essence, then Exalt, then Annul if bad") need decision-tree modeling.

### 6. Build Import Format
PoE2's PoB uses XML builds. Need to convert GGG API character data → PoB XML format, or find an existing converter.

---

## Module Cards

| # | Module | Est. Effort | Purpose |
|---|--------|-------------|---------|
| 01 | PoB Headless Deep Dive | — | Research doc: how PoB-PoE2 headless works |
| 02 | PoB Calculation Engine | 5-7 days | DPS/stat simulation via lupa + LuaJIT |
| 03 | Pricing & Trade Layer | 7-9 days | Multi-source pricing, trade search, buy vs craft |
| 04 | Build Import & Onboarding | 4 days | PoB code import, build session management |
| 05 | League Detection & Context | 2.5 days | Auto-detect league, context switching |
| 06 | Meta Intelligence & Gap Analysis | 10-12 days | Top player analysis, upgrade recommendations |
| 07 | Crafting Probability Engine | 12-16 days | Mod weights, probability math, recipe generation |
| 08 | Data Pipeline & Updates | 9-11 days | ETL, SQLite, deduplication, update scheduling |
| 09 | MCP Server Shell | 9-10 days | Tool definitions, transport, token optimisation |
| | **Total** | **~59-82 days** | |

## Tech Stack

- **Language:** Python (for lupa/LuaJIT PoB integration)
- **PoB Integration:** lupa (embedded LuaJIT) calling HeadlessWrapper.lua
- **Database:** SQLite (local, single-user, ~70MB)
- **MCP Protocol:** Official Python MCP SDK (stdio + HTTP/SSE)
- **HTTP Client:** aiohttp (async API calls)
- **LLM Compatibility:** Any MCP client (Claude Desktop, local ollama, etc.)
- **Packaging:** Nix devshell for reproducible environment

## Status

- [x] Research complete
- [x] Tech stack decision
- [x] Module card drafting (9 modules)
- [ ] Implementation planning (sprint ordering)
- [ ] Implementation
