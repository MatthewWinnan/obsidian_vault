# MVP Implementation Plan

## MVP Scope

The minimum viable product that delivers real value: **Load a build, get crafting/buying advice for one slot, within a budget.**

Not everything needs to be perfect for the MVP. We need:
- ✅ Load a build from PoB code
- ✅ Show current stats
- ✅ Price check currencies and items (cached)
- ✅ Generate a basic crafting recipe for a target item
- ✅ Compare: buy vs craft
- ✅ MCP server running on stdio (works with Claude Desktop or local LLM)

We defer to post-MVP:
- ❌ Meta intelligence (poe.ninja builds) — nice but not essential day 1
- ❌ Monte Carlo simulation — basic probability is enough
- ❌ Trade API search (just generate trade URLs instead)
- ❌ HTTP/SSE transport — stdio is fine for personal use
- ❌ Full weight data coverage — start with common bases, expand later

---

## Sprint Order (MVP)

```
Sprint 1 (Week 1-2):  Foundation
  ├── Dev environment (Nix flake)
  ├── SQLite database + schema
  ├── PoB-PoE2 repo clone + headless boot test
  └── Basic lupa integration (load build, read DPS)

Sprint 2 (Week 3-4):  Build Engine + Import
  ├── PoBEngine class (load XML, get stats, equip item)
  ├── Build import (decode share code → load)
  ├── compare_item (swap + read delta)
  └── Tests with real build XMLs

Sprint 3 (Week 5-6):  Pricing + Data
  ├── poe.show client (currency prices)
  ├── poe.ninja client (base/unique prices)
  ├── Economy data cache (SQLite + TTL)
  ├── League detection
  └── Price check tool working

Sprint 4 (Week 7-8):  Crafting Engine (Basic)
  ├── Mod database loader (from PoB Lua data)
  ├── Weight data import (initial set of common bases)
  ├── Probability calculator (single mod, mod group)
  ├── Recipe generator (Essence + Alt-Regal strategies only)
  └── Cost estimation (integrates pricing)

Sprint 5 (Week 9-10): MCP Server + Integration
  ├── MCP SDK setup (stdio transport)
  ├── Wire all tools (import_build, craft_recipe, compare_item, price_check)
  ├── Token-optimised result formatters
  ├── End-to-end test: full advisor session
  └── Claude Desktop / ollama config

Total MVP: ~10 weeks solo
```

---

## Post-MVP Roadmap

```
Sprint 6-7:   Meta Intelligence (poe.ninja builds, gap analysis)
Sprint 8:     Trade API integration (live search, rate limiting)
Sprint 9:     Monte Carlo simulation + advanced craft strategies (Fracture)
Sprint 10:    Weight data expansion (all base types)
Sprint 11:    HTTP transport + web UI option
Sprint 12:    Polish, error handling, documentation
```

---

## Development Environment (Nix)

### Flake Structure

```
poe2-crafting-mcp/
├── flake.nix
├── flake.lock
├── shell.nix              ← For non-flake users
├── .envrc                 ← direnv integration
├── pyproject.toml         ← Python project config (uv/pip)
├── src/
│   └── poe2_crafting_mcp/
│       ├── __init__.py
│       ├── server.py
│       ├── ...
├── tests/
├── vendor/
│   └── PathOfBuilding-PoE2/  ← Git submodule
└── data/
    └── weights/
```

### flake.nix

```nix
{
  description = "PoE2 Crafting MCP Server";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};

        python = pkgs.python312;
        pythonPackages = python.pkgs;

        # LuaJIT for PoB headless (also bundled in lupa, but useful for testing standalone)
        luajit = pkgs.luajit;

      in {
        devShells.default = pkgs.mkShell {
          name = "poe2-craft-mcp";

          buildInputs = [
            # Python
            python
            pythonPackages.pip
            pythonPackages.virtualenv

            # LuaJIT (for standalone PoB testing)
            luajit
            pkgs.lua51Packages.luasocket  # PoB runtime dependency

            # Build tools
            pkgs.git
            pkgs.sqlite

            # For lupa compilation (if building from source)
            pkgs.gcc
            pkgs.pkg-config

            # Utilities
            pkgs.jq           # JSON inspection
            pkgs.httpie       # API testing
            pkgs.direnv
          ];

          shellHook = ''
            # Create venv if it doesn't exist
            if [ ! -d .venv ]; then
              echo "Creating Python virtual environment..."
              python -m venv .venv
            fi
            source .venv/bin/activate

            # Install Python deps if needed
            if [ ! -f .venv/.installed ]; then
              echo "Installing Python dependencies..."
              pip install -e ".[dev]" --quiet
              touch .venv/.installed
            fi

            # Set PoB path
            export POB_PATH="$PWD/vendor/PathOfBuilding-PoE2"
            export POE2_CRAFT_DB="$PWD/data/poe2_craft.db"

            echo "╔═══════════════════════════════════════════╗"
            echo "║  PoE2 Crafting MCP - Dev Environment      ║"
            echo "║                                           ║"
            echo "║  Python:  $(python --version)             ║"
            echo "║  LuaJIT:  $(luajit -v 2>&1 | head -1)    ║"
            echo "║  PoB:     $POB_PATH                      ║"
            echo "║  DB:      $POE2_CRAFT_DB                  ║"
            echo "╚═══════════════════════════════════════════╝"
          '';

          # Ensure lupa can find LuaJIT headers
          LUAJIT_INCLUDE_DIR = "${luajit}/include/luajit-2.1";
          LUAJIT_LIB = "${luajit}/lib";
        };
      }
    );
}
```

### .envrc (direnv)

```bash
use flake
```

### pyproject.toml

```toml
[project]
name = "poe2-crafting-mcp"
version = "0.1.0"
description = "PoE2 Crafting Advisor MCP Server"
requires-python = ">=3.11"

dependencies = [
    "lupa>=2.0",              # LuaJIT embedding
    "mcp>=1.0",              # MCP protocol SDK
    "aiohttp>=3.9",          # Async HTTP client
    "aiosqlite>=0.19",       # Async SQLite
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "ruff>=0.5",              # Linter
    "mypy>=1.10",             # Type checking
]

local-llm = [
    "httpx>=0.27",            # For ollama adapter
]

[project.scripts]
poe2-craft = "poe2_crafting_mcp.server:main"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 100
target-version = "py311"
```

---

## Initial Setup Commands

```bash
# 1. Clone the project
mkdir -p ~/projects/poe2-crafting-mcp
cd ~/projects/poe2-crafting-mcp
git init

# 2. Create the flake + pyproject (files above)

# 3. Enter dev shell
direnv allow
# or: nix develop

# 4. Clone PoB-PoE2 as submodule
git submodule add https://github.com/PathOfBuildingCommunity/PathOfBuilding-PoE2.git vendor/PathOfBuilding-PoE2
git submodule update --init

# 5. Verify PoB headless works standalone
cd vendor/PathOfBuilding-PoE2/src
luajit HeadlessWrapper.lua
# Should print nothing and exit (means it booted successfully)
cd ../../..

# 6. Verify lupa works
python -c "
import lupa.luajit21 as lupa
lua = lupa.LuaRuntime()
print(lua.eval('1 + 1'))  # Should print 2
print('lupa OK')
"

# 7. Verify MCP SDK
python -c "from mcp.server import Server; print('MCP SDK OK')"

# 8. Create initial database
python -c "
import sqlite3, pathlib
db_path = pathlib.Path('data/poe2_craft.db')
db_path.parent.mkdir(exist_ok=True)
conn = sqlite3.connect(db_path)
# Run schema.sql here
print(f'Database created: {db_path}')
"
```

---

## Sprint 1 Detailed Breakdown

### Day 1-2: Nix flake + project skeleton

```
- [ ] Create flake.nix (from above)
- [ ] Create pyproject.toml
- [ ] Create src/poe2_crafting_mcp/__init__.py
- [ ] Create tests/ directory
- [ ] direnv integration
- [ ] Verify: `nix develop` works, Python + lupa + MCP importable
```

### Day 3-4: PoB headless boot via lupa

```
- [ ] Clone PoB-PoE2 as git submodule
- [ ] Write src/poe2_crafting_mcp/engine/pob_engine.py (skeleton)
- [ ] Get lupa to dofile("HeadlessWrapper.lua") without errors
- [ ] Troubleshoot path issues (LoadModule, package.path)
- [ ] Verify: `newBuild()` callable from Python
- [ ] Verify: can read `build.calcsTab.mainOutput.TotalDPS`
```

### Day 5-6: SQLite schema + database module

```
- [ ] Write data/schema.sql (from module card 08)
- [ ] Write src/poe2_crafting_mcp/data/database.py
- [ ] Schema creation on first run
- [ ] Basic CRUD: upsert, query, TTL check
- [ ] Test: create DB, insert data, query back
```

### Day 7-8: First integration test

```
- [ ] Find a real PoB share code for testing
- [ ] Decode share code → XML (base64 + zlib)
- [ ] Load into PoB engine via lupa
- [ ] Read back: DPS, life, resistances
- [ ] Write test_pob_engine.py with assertions
- [ ] Verify: full round-trip works
```

### Day 9-10: Buffer / troubleshooting

```
- [ ] Fix whatever broke (path resolution, luasocket stub, etc.)
- [ ] Document any PoB-specific quirks found
- [ ] Ensure tests pass in clean nix shell
- [ ] Git commit: "Sprint 1 complete: foundation"
```

---

## Testing Strategy

### Unit Tests (Fast, No Network)

```python
# tests/test_pob_engine.py
def test_engine_boots():
    """PoB headless initializes without error."""
    engine = PoBEngine(POB_PATH)
    assert engine._build is not None

def test_load_build_from_xml():
    """Loading a build XML produces non-zero DPS."""
    engine = PoBEngine(POB_PATH)
    stats = engine.load_build_xml(FIXTURE_BUILD_XML)
    assert stats.total_dps > 0
    assert stats.total_life > 0

def test_equip_item_changes_stats():
    """Equipping an item changes calculated output."""
    engine = PoBEngine(POB_PATH)
    engine.load_build_xml(FIXTURE_BUILD_XML)
    old_dps = engine.get_stats().total_dps
    engine.equip_item("New Item\nHeavy Bow\n+100% increased Physical Damage")
    new_dps = engine.get_stats().total_dps
    assert new_dps != old_dps
```

### Integration Tests (Network, Slow)

```python
# tests/test_integration.py
@pytest.mark.integration
async def test_full_advisor_session():
    """Simulate a complete advisory session end-to-end."""
    server = create_test_server()

    # Import build
    result = await server.call_tool("import_build", {"code": TEST_POB_CODE})
    assert "dps" in result

    # Get upgrade plan
    result = await server.call_tool("get_upgrade_plan", {"budget": 10, "goal": "damage"})
    assert "upgrades" in result
    assert len(result["upgrades"]) > 0
```

### Fixture Data

```
tests/
├── fixtures/
│   ├── sample_build_fireball.xml     ← Exported from PoB
│   ├── sample_build_lightning.xml
│   ├── sample_pob_code.txt           ← Share code
│   ├── sample_economy_response.json  ← Mocked poe.show response
│   └── sample_item_text.txt          ← In-game item copy
```

---

## Claude Desktop Config (For Testing MCP)

Once the MVP is working:

```json
// ~/.config/claude-desktop/config.json (Linux)
// ~/Library/Application Support/Claude/claude_desktop_config.json (Mac)
{
  "mcpServers": {
    "poe2-craft": {
      "command": "/home/matthew/projects/poe2-crafting-mcp/.venv/bin/python",
      "args": ["-m", "poe2_crafting_mcp.server"],
      "env": {
        "POB_PATH": "/home/matthew/projects/poe2-crafting-mcp/vendor/PathOfBuilding-PoE2",
        "POE2_CRAFT_DB": "/home/matthew/projects/poe2-crafting-mcp/data/poe2_craft.db"
      }
    }
  }
}
```

### For Local LLM (ollama)

```bash
# Start ollama with a model
ollama run llama3.1:8b

# In another terminal, start our MCP server in local-llm adapter mode
python -m poe2_crafting_mcp.server --adapter=ollama --model=llama3.1:8b
```

---

## Definition of Done (MVP)

The MVP is complete when:

```
- [ ] `nix develop` → clean dev environment, all deps available
- [ ] `pytest` → all tests pass (unit + integration mocked)
- [ ] Load a real PoB build → see stats printed
- [ ] Equip an item → see DPS change
- [ ] `price_check("Exalted Orb")` → returns current price from poe.show
- [ ] `craft_recipe("Vaal Regalia", 86, ["+1 fire gems", "+90 life"])` → returns steps + cost
- [ ] MCP server starts via stdio → Claude Desktop can call tools
- [ ] Full session: paste build → ask for upgrade → get crafting recipe → under 5k tokens
```
