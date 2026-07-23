# PoB-PoE2 Headless Deep Dive

## Summary

The PoE2 fork ([PathOfBuildingCommunity/PathOfBuilding-PoE2](https://github.com/PathOfBuildingCommunity/PathOfBuilding-PoE2)) already has a fully functional headless mode. You do **not** need to build this yourself. The community maintains it and uses it for automated testing in CI.

---

## HeadlessWrapper.lua — How It Works

Location: `src/HeadlessWrapper.lua`

This file is a **stub/mock layer** that replaces all the GUI and rendering functions with no-ops, allowing the entire PoB calculation engine to run without a display. It:

1. **Stubs all rendering** — `DrawImage`, `DrawString`, `GetScreenSize`, `SetDrawColor`, etc. all do nothing or return dummy values (e.g., screen = 1920×1080).

2. **Stubs all system functions** — `MakeDir`, `Copy`, `Paste`, `OpenURL`, `GetCursorPos`, `IsKeyDown` — all mocked out.

3. **Provides module loading** — `LoadModule(fileName)` and `PLoadModule(fileName)` load Lua files from disk, which is how PoB's internal module system works.

4. **Boots the full application** — at the bottom it does:
   ```lua
   dofile("Launch.lua")
   mainObject.continuousIntegrationMode = os.getenv("CI")
   runCallback("OnInit")
   runCallback("OnFrame")  -- Need at least one frame for everything to initialise
   ```

5. **Exposes the build object** — after init:
   ```lua
   build = mainObject.main.modes["BUILD"]
   ```

6. **Provides helper functions** for programmatic use:
   ```lua
   function newBuild()
   function loadBuildFromXML(xmlText, name)
   function loadBuildFromJSON(getItemsJSON, getPassiveSkillsJSON)
   ```

### Key API Surface (from HeadlessWrapper)

| Function | Purpose |
|----------|---------|
| `newBuild()` | Reset to a fresh empty build |
| `loadBuildFromXML(xmlText, name)` | Load a complete build from PoB XML/pastebin format |
| `loadBuildFromJSON(getItemsJSON, getPassiveSkillsJSON)` | Import from GGG API JSON (items + passive tree) |
| `runCallback("OnFrame")` | Trigger a recalculation cycle |
| `build.calcsTab:BuildOutput()` | Force full recalculation of all stats |
| `build.calcsTab.mainOutput` | Access all calculated stats (DPS, crit, etc.) |
| `build.itemsTab:CreateDisplayItemFromRaw(rawText)` | Create an item from in-game copy/paste text |
| `build.itemsTab:AddDisplayItem()` | Equip the created item |
| `build.skillsTab:PasteSocketGroup(text)` | Add skills programmatically |

---

## LaunchServer.lua — What It Does

Location: `src/LaunchServer.lua`

This is **NOT** a general-purpose API server. It's specifically an **OAuth callback handler** for the official GGG API authentication flow.

It:
- Opens a TCP socket on localhost (ports 49082-49084)
- Waits up to 30 seconds for an HTTP GET with OAuth `code` and `state` parameters
- Responds with a simple HTML page ("Authentication Successful" or "Authentication Failed")
- Returns the OAuth code to PoB's main application

**Not useful for our MCP server** — it's just the OAuth redirect receiver for when users authenticate PoB with their GGG account. We'd be building our own server layer anyway.

---

## Test Suite — How It Works

### Framework: Busted (Lua BDD test framework)

Config in `.busted`:
```lua
return {
  _all = {
    coverage = false,
    verbose = true,
  },
  default = {
    directory = "src",
    lpath = "../runtime/lua/?.lua;../runtime/lua/?/init.lua",
    helper = "HeadlessWrapper.lua",    -- ← THIS IS KEY
    ROOT = { "../spec" },
    ["exclude-tags"] = "builds",
  }
}
```

**The tests use HeadlessWrapper.lua as their test helper.** This means:
- Busted loads HeadlessWrapper.lua first (which boots the entire PoB engine headlessly)
- Then test spec files can directly call `newBuild()`, manipulate items, run calculations, and assert on outputs

### Test Location: `spec/System/`

31 test files covering:

| Test File | What It Tests |
|-----------|--------------|
| `TestAttacks_spec.lua` | DPS calculations, crit chance, crit multiplier, dual wield, forced outcome |
| `TestAilments_spec.lua` | Ailment (bleed, ignite, shock) calculations |
| `TestDefence_spec.lua` | Defence calculations |
| `TestItemMods_spec.lua` | Item modifier parsing and application |
| `TestItemParse_spec.lua` | Parsing raw item text |
| `TestItemsTab_spec.lua` | Item sets, equipping, anoints, rune augments |
| `TestSkills_spec.lua` | Skill gem calculations |
| `TestTriggers_spec.lua` | Triggered skill interactions |
| `TestTradeQuery_spec.lua` | Trade query generation |
| `TestTradeQueryGenerator_spec.lua` | Trade API query building |
| `TestTradeQueryRateLimiter_spec.lua` | Rate limiting logic |
| `TestImportReimport_spec.lua` | Build import/export round-tripping |
| `TestFullDPSCache_spec.lua` | DPS caching correctness |
| ... and 18 more | |

### Test Pattern (from TestAttacks_spec.lua)

```lua
describe("TestAttacks", function()
  before_each(function()
    newBuild()
  end)

  it("creates an item and has the correct crit chance", function()
    build.itemsTab:CreateDisplayItemFromRaw([[
New Item
Heavy Bow
    ]])
    build.itemsTab:AddDisplayItem()
    runCallback("OnFrame")
    assert.are.equals(
      build.calcsTab.mainOutput.CritChance,
      5 * build.calcsTab.mainOutput.HitChance / 100
    )
  end)
end)
```

The pattern is always:
1. `newBuild()` — fresh build
2. Create/equip items via `build.itemsTab:CreateDisplayItemFromRaw(...)`
3. Add skills via `build.skillsTab:PasteSocketGroup(...)`
4. `runCallback("OnFrame")` — recalculate
5. Assert on `build.calcsTab.mainOutput.*` values

### Running Tests

```bash
cd src
busted --helper HeadlessWrapper.lua ../spec
```

Or via the `.busted` config (just run `busted` from the repo root).

---

## What This Means for the MCP Server

### Good News

1. **The headless engine is production-ready** — it's used in CI on every PR (10,063 commits, 122 contributors).
2. **Item creation from text is trivial** — paste in-game item text and it's parsed.
3. **DPS output is directly accessible** — `build.calcsTab.mainOutput.TotalDPS`, `.CritChance`, `.Speed`, `.AverageDamage`, etc.
4. **Import from GGG JSON works** — `loadBuildFromJSON()` takes the official API response directly.
5. **The trade query system is built into PoB** — they already have logic for generating trade searches.

### Integration Approach

You don't need `pob_wrapper`. That old Python library used a complex Lua embedding approach for PoE1. For PoE2, a simpler approach works:

**Option A: Subprocess (simplest)**
- Run `luajit src/HeadlessWrapper.lua` as a subprocess
- Feed it Lua commands via stdin or a script file
- Parse stdout for results

**Option B: Lua embedding via `lupa` (Python) or `mlua` (Rust)**
- Embed LuaJIT in your MCP server process
- Load HeadlessWrapper.lua once at startup
- Call functions directly (faster, no process overhead)

**Option C: Wrap as an HTTP microservice**
- Write a tiny Lua script that starts an HTTP server (using luasocket or openresty)
- Accepts JSON requests like `{"action": "loadBuild", "xml": "..."}`
- Returns calculation results as JSON
- MCP server calls this service

### Recommended: Option B for performance, Option C for isolation

For an MCP server that might handle multiple builds or concurrent requests, Option C (microservice) gives you process isolation and the ability to restart if PoB crashes. Option B is better for a single-user local tool.

---

## Key API Calls for Our Use Case

```lua
-- 1. Load user's build
loadBuildFromXML(pobPastebinXml)

-- 2. Get current DPS
local currentDPS = build.calcsTab.mainOutput.TotalDPS

-- 3. Swap an item and see the DPS change
build.itemsTab:CreateDisplayItemFromRaw(newItemText)
build.itemsTab:AddDisplayItem()
runCallback("OnFrame")
local newDPS = build.calcsTab.mainOutput.TotalDPS
local dpsGain = newDPS - currentDPS

-- 4. Access all output stats
build.calcsTab.mainOutput.MainHand.AverageHit
build.calcsTab.mainOutput.CritChance
build.calcsTab.mainOutput.CritMultiplier
build.calcsTab.mainOutput.Speed
build.calcsTab.mainOutput.MainHand.AverageDamage

-- 5. Use custom mods for "what if" testing
build.configTab.input.customMods = "+50% to critical hit chance"
build.configTab:BuildModList()
runCallback("OnFrame")
build.calcsTab:BuildOutput()
```

---

## Dependencies to Run Headless

From the `.busted` config:
```
lpath = "../runtime/lua/?.lua;../runtime/lua/?/init.lua"
```

You need:
- **LuaJIT** (standard Lua may work but LuaJIT is preferred/tested)
- The `runtime/lua/` directory from the PoB-PoE2 repo (contains utility libraries)
- The entire `src/` directory (Data, Classes, Modules, etc.)
- `luasocket` (for LaunchServer; optional for our use case)

Everything is self-contained in the repo — no external game files needed.
