# Module Card: MCP Server Shell

## Overview

The MCP (Model Context Protocol) server that exposes all modules as tools to any LLM client. Designed for **minimal token usage** — tools return structured, compact data rather than verbose text. The LLM interprets and presents results; it doesn't need to process raw dumps.

---

## Design Principles

1. **Token-efficient tools:** Return structured JSON, not prose. Let the LLM format for the user.
2. **Coarse-grained tools:** Fewer tool calls = fewer tokens. One `get_upgrade_plan` call replaces 5 separate queries.
3. **Stateful session:** Build stays loaded between calls. No re-importing every turn.
4. **Progressive disclosure:** Summary first, details on request. Don't dump everything at once.
5. **Local-first:** Works with local LLMs (ollama, llama.cpp) — no cloud dependency for the MCP server itself.

---

## Token Budget Philosophy

The biggest token cost in MCP is:
1. **Tool definitions** sent to the LLM on every turn (~tokens per tool schema)
2. **Tool results** returned from calls (the data the LLM must read)
3. **Conversation context** accumulating over the session

### Our strategy:

| Optimisation | How |
|-------------|-----|
| Few tools (not many) | 8-12 tools max, each does more work internally |
| Compact results | Numbers/lists, not paragraphs. LLM generates the prose. |
| Summaries by default | `get_upgrade_plan` returns a ranked list, not full analysis of every slot |
| Drill-down on request | `get_slot_details(slot)` only when user asks "tell me more about body armour" |
| Session state server-side | Build is loaded once, referenced implicitly. No re-sending build data each call. |
| Avoid echoing input | Tool results don't repeat what the LLM already knows |

### Token cost estimate per interaction:

```
Tool schemas (one-time system prompt):     ~2,000 tokens
Typical tool call + result:                ~200-500 tokens
Full advisor session (5-8 turns):          ~4,000-8,000 tokens total

Compare to: ChatGPT conversation of same length: ~10,000-20,000 tokens
Savings: 50-70% from structured tools vs. freeform chat
```

---

## Tool Definitions (Compact Set)

### Tier 1: Core Tools (Always Active)

These are loaded into the LLM's context on every turn:

```python
TOOLS = [
    {
        "name": "import_build",
        "description": "Load a PoB share code. Returns build summary with stats and issues.",
        "parameters": {
            "code": {"type": "string", "description": "PoB pastebin/share code"}
        }
    },
    {
        "name": "get_upgrade_plan",
        "description": "Analyse build gaps vs meta, recommend upgrades within budget. Returns prioritised list.",
        "parameters": {
            "budget": {"type": "number", "description": "Budget in divine orbs"},
            "goal": {"type": "string", "enum": ["damage", "survivability", "balanced"], "default": "damage"}
        }
    },
    {
        "name": "craft_recipe",
        "description": "Generate optimal crafting recipe for a target item. Returns steps with costs and probabilities.",
        "parameters": {
            "base_type": {"type": "string", "description": "Item base (e.g., 'Vaal Regalia')"},
            "item_level": {"type": "integer", "description": "Required ilvl", "default": 86},
            "desired_mods": {"type": "array", "items": {"type": "string"}, "description": "Target mods"},
            "budget": {"type": "number", "description": "Max budget in divine", "optional": True}
        }
    },
    {
        "name": "compare_item",
        "description": "Simulate equipping an item. Returns DPS/defence delta vs current gear.",
        "parameters": {
            "item_text": {"type": "string", "description": "In-game item text (Ctrl+C)"},
            "slot": {"type": "string", "description": "Equipment slot", "optional": True}
        }
    },
    {
        "name": "price_check",
        "description": "Get price of a currency, unique, or base item.",
        "parameters": {
            "query": {"type": "string", "description": "Item/currency name or pasted item text"}
        }
    },
    {
        "name": "search_trade",
        "description": "Search trade for items matching criteria. Returns top 5 cheapest listings. Rate-limited.",
        "parameters": {
            "base_type": {"type": "string", "optional": True},
            "mods": {"type": "array", "items": {"type": "string"}, "optional": True},
            "max_price": {"type": "number", "description": "Max price in divine", "optional": True},
            "item_class": {"type": "string", "optional": True}
        }
    },
    {
        "name": "what_do_top_players_use",
        "description": "Show what top players use in a specific slot. Returns mod patterns with popularity %.",
        "parameters": {
            "slot": {"type": "string", "description": "Equipment slot (e.g., 'Body Armour')"}
        }
    },
    {
        "name": "data_status",
        "description": "Show data freshness, league info, and any warnings.",
        "parameters": {}
    }
]
```

### Tier 2: Drill-Down Tools (Loaded On Demand)

Only added to context when the conversation goes deeper:

```python
DRILL_DOWN_TOOLS = [
    {
        "name": "get_crafting_odds",
        "description": "Probability of hitting a specific mod on a base.",
        "parameters": {
            "base_type": {"type": "string"},
            "item_level": {"type": "integer"},
            "target_mod": {"type": "string"}
        }
    },
    {
        "name": "list_mod_pool",
        "description": "Show all rollable mods for a base/ilvl. Prefix or suffix.",
        "parameters": {
            "base_type": {"type": "string"},
            "item_level": {"type": "integer"},
            "affix_type": {"type": "string", "enum": ["prefix", "suffix"], "optional": True}
        }
    },
    {
        "name": "simulate_craft",
        "description": "Run Monte Carlo simulation for a crafting strategy. Returns cost distribution.",
        "parameters": {
            "base_type": {"type": "string"},
            "item_level": {"type": "integer"},
            "desired_mods": {"type": "array", "items": {"type": "string"}},
            "strategy": {"type": "string"},
            "iterations": {"type": "integer", "default": 5000}
        }
    },
    {
        "name": "progression_tiers",
        "description": "Show budget/mid/high-end progression for a slot.",
        "parameters": {
            "slot": {"type": "string"}
        }
    },
    {
        "name": "switch_league",
        "description": "Switch to a different league for all queries.",
        "parameters": {
            "league": {"type": "string"}
        }
    },
    {
        "name": "refresh_data",
        "description": "Force refresh specific data source.",
        "parameters": {
            "source": {"type": "string", "enum": ["prices", "meta", "game_data", "all"]}
        }
    }
]
```

---

## Tool Result Format (Token-Optimised)

### Bad (verbose, wastes tokens):

```json
{
  "message": "I've analyzed your build and compared it against the top 50 Fireball Elementalist players on the current league ladder. Here are my findings...",
  "detailed_analysis": "Your body armour is significantly below the meta standard. Most top players (90%) use a Vaal Regalia with +1 to Level of Fire Skill Gems..."
}
```

### Good (structured, compact — LLM generates the prose):

```json
{
  "upgrades": [
    {
      "rank": 1,
      "slot": "Body Armour",
      "current": "+100 life only",
      "target_mods": ["+1 fire gems", "spell dmg 70%+", "+90 life"],
      "meta_usage": 90,
      "dps_gain": 180000,
      "dps_gain_pct": 73,
      "buy_price": 8.0,
      "craft_cost": 5.2,
      "recommendation": "craft",
      "recipe_id": "essence_wrath_vaal_regalia"
    },
    {
      "rank": 2,
      "slot": "Weapon 1",
      "current": "+40 flat fire",
      "target_mods": ["+1 fire gems", "spell dmg 80%+", "cast speed"],
      "meta_usage": 85,
      "dps_gain": 95000,
      "dps_gain_pct": 39,
      "buy_price": 15.0,
      "craft_cost": 12.0,
      "recommendation": "skip",
      "reason": "over_budget"
    }
  ],
  "budget_remaining": 4.8,
  "data_confidence": "high",
  "league": "Return of the Ancients"
}
```

The LLM takes this compact JSON and writes a nice response for the user. The JSON costs ~200 tokens. A verbose text version would cost ~500+ tokens.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         LLM Client                                │
│  (Claude, GPT, local Llama, Mistral, etc.)                       │
│                                                                  │
│  Sees: 8 tool definitions + conversation history                 │
│  Calls: tools as needed                                          │
└────────────────────────────┬─────────────────────────────────────┘
                             │ MCP Protocol (stdio or HTTP)
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                      MCP Server (Python)                          │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Tool Router                              │  │
│  │                                                            │  │
│  │  import_build    → BuildSession.load()                     │  │
│  │  get_upgrade_plan → MetaIntelligence.get_upgrade_plan()    │  │
│  │  craft_recipe    → CraftingEngine.generate_recipe()        │  │
│  │  compare_item    → PoBEngine.compare_item()                │  │
│  │  price_check     → PricingEngine.get_*_price()             │  │
│  │  search_trade    → PricingEngine.search_trade()            │  │
│  │  what_do_top_players_use → MetaIntelligence.what_do_*()    │  │
│  │  data_status     → DataScheduler.status()                  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────────┐  │
│  │  PoB     │ │ Pricing  │ │ Crafting │ │ Meta Intelligence  │  │
│  │  Engine  │ │ Engine   │ │ Engine   │ │                    │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └─────────┬──────────┘  │
│       │             │            │                  │             │
│       └─────────────┴────────────┴──────────────────┘             │
│                              │                                    │
│                     ┌────────┴────────┐                           │
│                     │  SQLite (local) │                           │
│                     │  + Data Pipeline│                           │
│                     └─────────────────┘                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## MCP Protocol Implementation

Using the official [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk):

```python
from mcp.server import Server
from mcp.types import Tool, TextContent
import json


app = Server("poe2-craft-advisor")


# ─── Session State (persists across tool calls) ─────────────

class SessionState:
    def __init__(self):
        self.build_session: BuildSession | None = None
        self.league_context = LeagueContext()
        self.pob_engine = PoBEngine(POB_PATH)
        self.db = Database(DB_PATH)
        self.pricing = PricingEngine(self.league_context, self.db)
        self.crafting = CraftingEngine(ModDatabase(self.db), self.pricing)
        self.meta = MetaIntelligence(self.league_context, self.pob_engine, self.pricing)
        self.scheduler = DataScheduler(self.db, self.league_context)

state = SessionState()


# ─── Tool Definitions ────────────────────────────────────────

@app.tool()
async def import_build(code: str) -> str:
    """Load a PoB share code. Returns build summary."""
    state.build_session = BuildSession(state.pob_engine)
    summary = state.build_session.load(code)

    # Compact result
    return json.dumps({
        "name": summary.name,
        "class": f"{summary.ascendancy} ({summary.class_name})",
        "level": summary.level,
        "skill": summary.main_skill,
        "dps": round(summary.total_dps),
        "life": round(summary.life),
        "es": round(summary.energy_shield),
        "res": {
            "fire": round(summary.fire_res),
            "cold": round(summary.cold_res),
            "light": round(summary.lightning_res),
            "chaos": round(summary.chaos_res)
        },
        "issues": summary.issues[:3]  # Max 3 issues to save tokens
    })


@app.tool()
async def get_upgrade_plan(budget: float, goal: str = "damage") -> str:
    """Analyse build gaps, recommend upgrades within budget."""
    if not state.build_session or not state.build_session.is_loaded:
        return json.dumps({"error": "No build loaded. Use import_build first."})

    plan = await state.meta.get_upgrade_plan(
        state.build_session, budget=budget, goal=goal
    )

    # Compact: only essential fields
    return json.dumps({
        "upgrades": [
            {
                "rank": r.rank,
                "slot": r.slot,
                "target": r.target_mods[:3],  # Top 3 mods only
                "dps_gain_pct": round(r.dps_gain_percent, 1),
                "cost": round(r.cost, 1),
                "action": r.action,  # "buy" | "craft" | "skip"
            }
            for r in plan[:5]  # Max 5 recommendations
        ],
        "budget_used": round(sum(r.cost for r in plan if r.action != "skip"), 1),
    })


@app.tool()
async def craft_recipe(
    base_type: str,
    item_level: int = 86,
    desired_mods: list[str] = [],
    budget: float | None = None,
) -> str:
    """Generate crafting recipe. Returns steps with costs."""
    recipes = await state.crafting.generate_recipe(
        base_type, item_level, desired_mods, budget
    )

    if not recipes:
        return json.dumps({"error": "No viable recipe found within budget."})

    best = recipes[0]
    return json.dumps({
        "strategy": best.strategy_name,
        "total_cost": round(best.total_expected_cost, 1),
        "confidence": best.confidence,
        "steps": [
            {
                "step": i + 1,
                "action": s.action,
                "target": s.target_description,
                "probability": round(s.success_probability * 100, 1),
                "avg_attempts": round(s.expected_attempts, 1),
                "cost_per_try": round(s.cost_per_attempt, 2),
            }
            for i, s in enumerate(best.steps)
        ],
        "warnings": best.warnings[:2],
    })


@app.tool()
async def compare_item(item_text: str, slot: str | None = None) -> str:
    """Simulate equipping an item. Returns stat deltas."""
    if not state.build_session or not state.build_session.is_loaded:
        return json.dumps({"error": "No build loaded."})

    delta = state.pob_engine.compare_item(item_text, slot)
    return json.dumps({
        "dps_change": round(delta.dps_change),
        "dps_change_pct": round(delta.dps_change_percent, 1),
        "changes": {k: round(v, 1) for k, v in delta.stat_changes.items() if v != 0},
        "verdict": "upgrade" if delta.dps_change > 0 else "downgrade",
    })


@app.tool()
async def price_check(query: str) -> str:
    """Price check a currency, unique, or base."""
    # Try currency first
    price = await state.pricing.get_currency_price(query)
    if price:
        return json.dumps({
            "name": price.name,
            "price_divine": round(price.price_in_divine, 3),
            "type": "currency"
        })

    # Try unique/base
    item_price = await state.pricing.get_item_price(query)
    if item_price:
        return json.dumps({
            "name": item_price.name,
            "price_divine": round(item_price.price_in_divine, 2),
            "listings": item_price.listings_count,
            "type": item_price.item_type
        })

    return json.dumps({"error": f"No price data for '{query}'"})


@app.tool()
async def search_trade(
    base_type: str | None = None,
    mods: list[str] | None = None,
    max_price: float | None = None,
    item_class: str | None = None,
) -> str:
    """Search trade. Returns top 5 cheapest. Rate-limited (max 3/session)."""
    results = await state.pricing.search_trade(
        base_type=base_type,
        min_mods={m: 0 for m in (mods or [])},
        max_price=max_price,
        item_class=item_class,
    )

    if not results:
        # Fall back to trade URL
        url = await state.pricing.get_trade_url(base_type=base_type, min_mods=mods)
        return json.dumps({"results": [], "trade_url": url, "note": "No results or rate limited. Use the link to search manually."})

    return json.dumps({
        "results": [
            {
                "price": round(r.price_in_divine, 2),
                "mods": r.mods[:4],  # Max 4 mods shown
                "seller": r.seller[:15],
            }
            for r in results[:5]
        ],
        "total_found": len(results),
    })


@app.tool()
async def what_do_top_players_use(slot: str) -> str:
    """What do top players use in this slot?"""
    if not state.build_session or not state.build_session.is_loaded:
        return json.dumps({"error": "No build loaded."})

    meta = await state.meta.what_do_top_players_use(
        slot,
        state.build_session.summary.main_skill,
        state.build_session.summary.ascendancy,
    )

    return json.dumps({
        "slot": slot,
        "sample_size": meta.sample_size,
        "common_bases": meta.common_base_types[:3],
        "key_mods": [
            {"mod": m.mod_text, "usage_pct": round(m.occurrence_percent)}
            for m in meta.key_mods[:6]  # Top 6 mods
        ],
        "top_uniques": meta.common_uniques[:3] if meta.common_uniques else [],
    })


@app.tool()
async def data_status() -> str:
    """Show data freshness and league info."""
    status = state.scheduler.get_status()
    return json.dumps({
        "league": state.league_context.league_id,
        "sources": {
            name: {
                "age_min": round(s.age_seconds / 60, 1),
                "confidence": s.confidence,
                "warning": s.warning,
            }
            for name, s in status.items()
        }
    })
```

---

## Transport Options

The MCP server supports multiple transports depending on how the LLM connects:

### Option A: stdio (for local LLMs / Claude Desktop)

```python
if __name__ == "__main__":
    import asyncio
    from mcp.server.stdio import stdio_server

    async def main():
        await state.scheduler.on_startup()
        async with stdio_server() as (read, write):
            await app.run(read, write, app.create_initialization_options())

    asyncio.run(main())
```

**Usage with Claude Desktop `config.json`:**
```json
{
  "mcpServers": {
    "poe2-craft": {
      "command": "python",
      "args": ["-m", "poe2_crafting_mcp.server"],
      "env": {"POB_PATH": "/path/to/PathOfBuilding-PoE2"}
    }
  }
}
```

### Option B: HTTP/SSE (for remote LLMs or web UIs)

```python
from mcp.server.sse import SseServerTransport
from starlette.applications import Starlette
from starlette.routing import Route

transport = SseServerTransport("/messages")

async def handle_sse(request):
    async with transport.connect_sse(request.scope, request.receive, request._send) as streams:
        await app.run(streams[0], streams[1], app.create_initialization_options())

starlette_app = Starlette(routes=[
    Route("/sse", endpoint=handle_sse),
    Route("/messages", endpoint=transport.handle_post_message, methods=["POST"]),
])
```

### Option C: Local LLM Direct (ollama / llama.cpp)

For local models without native MCP support, wrap as a simple function-calling loop:

```python
class LocalLLMAdapter:
    """Bridges local LLMs that don't speak MCP natively."""

    def __init__(self, model_url: str = "http://localhost:11434"):
        self.model_url = model_url
        self.tools = TOOLS  # Our tool definitions

    async def chat(self, user_message: str) -> str:
        """
        1. Send user message + tool defs to local LLM
        2. If LLM returns tool_call → execute it → feed result back
        3. Repeat until LLM returns final text response
        """
        ...
```

---

## Token Usage Breakdown

### Per-Session Overhead (One-Time)

| Component | Tokens |
|-----------|--------|
| System prompt (role description) | ~300 |
| 8 tool schemas | ~1,200 |
| Initial `import_build` result | ~150 |
| **Total one-time** | **~1,650** |

### Per-Turn Average

| Operation | Tool Call Tokens | Result Tokens | LLM Response |
|-----------|-----------------|---------------|--------------|
| `get_upgrade_plan` | ~50 | ~300 | ~200 |
| `craft_recipe` | ~60 | ~250 | ~300 |
| `compare_item` | ~100 (item text) | ~80 | ~150 |
| `price_check` | ~20 | ~50 | ~80 |
| `what_do_top_players_use` | ~20 | ~200 | ~200 |

### Full Session Estimate (Typical 5-Turn Interaction)

```
Turn 1: User pastes build           → ~400 tokens
Turn 2: "I want more damage, 10 div" → ~600 tokens  
Turn 3: "Tell me about the body armour craft" → ~500 tokens
Turn 4: "Show me the recipe"         → ~400 tokens
Turn 5: "What about buying instead?" → ~400 tokens

Total: ~3,950 tokens (input+output combined)
```

**At Anthropic Haiku prices (~$0.25/M tokens): ~$0.001 per session**
**At local Llama 3.1 8B: electricity only**

---

## System Prompt (Token-Optimised)

```
You are a Path of Exile 2 crafting advisor. You help players upgrade their gear efficiently.

You have tools to: load builds, analyse gaps vs top players, generate crafting recipes, check prices, and search trade.

Rules:
- Always load a build first (import_build) before giving gear advice
- Prefer crafting over buying when it saves >20% cost
- State data confidence when giving probability-based advice
- Keep responses concise — players want actionable steps, not essays
- When recommending crafts, always include expected cost and probability
```

~100 tokens. Minimal.

---

## File Layout

```
poe2_crafting_mcp/
├── server.py              ← MCP server entrypoint
├── tools.py               ← Tool definitions + handlers
├── session.py             ← SessionState management
├── adapters/
│   ├── stdio.py           ← stdio transport (Claude Desktop)
│   ├── http.py            ← HTTP/SSE transport
│   └── local_llm.py       ← Adapter for ollama/llama.cpp
├── engine/                ← PoB engine module
├── pricing/               ← Pricing & trade module
├── crafting/              ← Crafting probability module
├── meta/                  ← Meta intelligence module
├── data/                  ← Data pipeline module
└── config.py              ← Paths, secrets, settings
```

---

## Configuration

```toml
# config.toml

[paths]
pob_repo = "~/tools/PathOfBuilding-PoE2"
database = "~/.local/share/poe2-craft-mcp/poe2_craft.db"
weight_data = "~/.local/share/poe2-craft-mcp/weights/"

[server]
transport = "stdio"       # "stdio" | "http"
http_port = 8765          # Only if transport = "http"

[limits]
trade_searches_per_session = 3
meta_refresh_hours = 24
economy_refresh_minutes = 10

[llm]
# For local LLM adapter mode
model_url = "http://localhost:11434"
model_name = "llama3.1:8b"
```

---

## Sprint Estimate

| Task | Effort |
|------|--------|
| MCP SDK setup + tool registration | 1 day |
| Tool handlers (wiring to modules) | 2 days |
| Result formatters (token-optimised JSON) | 1 day |
| Session state management | 0.5 day |
| stdio transport + Claude Desktop config | 0.5 day |
| HTTP/SSE transport | 0.5 day |
| Local LLM adapter (ollama) | 1 day |
| System prompt engineering | 0.5 day |
| Integration testing (full flow) | 2 days |
| **Total** | **9-10 days** |
