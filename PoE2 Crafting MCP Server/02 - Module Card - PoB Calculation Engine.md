# Module Card: PoB Calculation Engine

## Overview

A Python module that embeds the PoB-PoE2 Lua calculation engine via `lupa` (LuaJIT binding). Provides a clean Python API to load builds, swap items, and read back DPS/defence statistics without ever launching the PoB GUI.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│              MCP Server (Python)                 │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │         pob_engine.py                     │  │
│  │                                           │  │
│  │  class PoBEngine:                         │  │
│  │    - __init__(pob_path)                   │  │
│  │    - load_build_xml(xml: str)             │  │
│  │    - load_build_from_api(items, passives) │  │
│  │    - get_stats() -> BuildStats            │  │
│  │    - equip_item(raw_text, slot?)          │  │
│  │    - add_skill(gem_string)                │  │
│  │    - set_config(custom_mods)              │  │
│  │    - compare_item(raw_text) -> DPSDelta   │  │
│  │    - reset()                              │  │
│  └───────────────┬───────────────────────────┘  │
│                   │                              │
│                   │ lupa (LuaJIT embedded)       │
│                   ▼                              │
│  ┌───────────────────────────────────────────┐  │
│  │      PoB-PoE2 Lua Engine                  │  │
│  │                                           │  │
│  │  HeadlessWrapper.lua                      │  │
│  │    → Launch.lua                           │  │
│  │      → Modules/ (CalcsTab, ItemsTab...)   │  │
│  │      → Data/ (mod pools, uniques, bases)  │  │
│  │      → Classes/ (Item, Build, Skill...)   │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

---

## Dependencies

| Dependency | Version | Purpose |
|-----------|---------|---------|
| `lupa` | latest (ships with LuaJIT 2.1) | Python ↔ Lua bridge |
| PoB-PoE2 repo | `dev` branch, cloned locally | The calculation engine |
| `luasocket` | bundled in PoB runtime | Required by some PoB internals (can be stubbed if only used for OAuth) |

### Nix Integration

```nix
# In your devShell or flake
{
  buildInputs = [ pkgs.luajit pkgs.lua51Packages.luasocket ];
  pythonPackages = ps: [ ps.lupa ];
}
```

Or simply `pip install lupa` — it bundles LuaJIT statically.

---

## Python API Design

```python
from dataclasses import dataclass
from pathlib import Path


@dataclass
class BuildStats:
    """Core output stats from a PoB calculation."""
    total_dps: float
    average_hit: float
    crit_chance: float
    crit_multiplier: float
    attack_speed: float
    hit_chance: float
    total_life: float
    total_es: float
    total_mana: float
    # Resistances
    fire_res: float
    cold_res: float
    lightning_res: float
    chaos_res: float
    # Defences
    armour: float
    evasion: float
    block_chance: float


@dataclass
class DPSDelta:
    """Result of comparing a new item against the current equipped one."""
    old_dps: float
    new_dps: float
    dps_change: float
    dps_change_percent: float
    stat_changes: dict[str, float]  # key -> delta


class PoBEngine:
    """
    Embeds the PoB-PoE2 calculation engine via lupa.
    
    Usage:
        engine = PoBEngine("/path/to/PathOfBuilding-PoE2")
        engine.load_build_xml(pob_xml_string)
        stats = engine.get_stats()
        delta = engine.compare_item("New Item\\nHeavy Bow\\n+50% crit")
    """

    def __init__(self, pob_path: str | Path):
        """
        Initialize the Lua runtime and boot PoB headlessly.
        
        Args:
            pob_path: Path to the cloned PathOfBuilding-PoE2 repo root.
        """
        ...

    def load_build_xml(self, xml: str, name: str = "MCP Build") -> BuildStats:
        """
        Load a build from PoB XML/pastebin format.
        Returns the calculated stats after loading.
        """
        ...

    def load_build_from_api(
        self, items_json: str, passives_json: str
    ) -> BuildStats:
        """
        Load a build from GGG API JSON responses.
        (Uses PoB's built-in loadBuildFromJSON)
        """
        ...

    def get_stats(self) -> BuildStats:
        """Get current calculated build statistics."""
        ...

    def equip_item(self, raw_text: str, slot: str | None = None) -> BuildStats:
        """
        Equip an item from its in-game copy/paste text.
        PoB auto-detects the slot if not specified.
        Returns updated stats.
        """
        ...

    def compare_item(self, raw_text: str, slot: str | None = None) -> DPSDelta:
        """
        Compare a potential item against what's currently equipped.
        Does NOT permanently equip it — restores the original after.
        """
        ...

    def add_skill(self, gem_string: str) -> BuildStats:
        """
        Add a skill gem group.
        Format: "SkillName level/quality count\\nSupportName level/quality count"
        Example: "Fireball 20/0 1\\nIgnite I 1/0 1"
        """
        ...

    def set_config(self, custom_mods: str) -> BuildStats:
        """
        Set custom configuration mods (for "what if" testing).
        Example: "+50% to critical hit chance\\n+100 to maximum life"
        """
        ...

    def reset(self) -> None:
        """Reset to a fresh empty build."""
        ...

    def get_all_output(self) -> dict:
        """
        Get the full mainOutput table as a Python dict.
        For advanced queries not covered by BuildStats.
        """
        ...
```

---

## Implementation Details

### Initialization Sequence

```python
import os
from pathlib import Path

try:
    import lupa.luajit21 as lupa
except ImportError:
    import lupa.lua54 as lupa


class PoBEngine:
    def __init__(self, pob_path: str | Path):
        self.pob_path = Path(pob_path)
        self.src_path = self.pob_path / "src"

        # Create Lua runtime
        self.lua = lupa.LuaRuntime(unpack_returned_tuples=True)

        # Set up Lua package path to find PoB modules
        runtime_lua = self.pob_path / "runtime" / "lua"
        self.lua.execute(f'''
            package.path = "{self.src_path}/?.lua;"
                        .. "{runtime_lua}/?.lua;"
                        .. "{runtime_lua}/?/init.lua;"
                        .. package.path
        ''')

        # Change working directory to src/ (PoB expects this)
        original_dir = os.getcwd()
        os.chdir(self.src_path)

        try:
            # Boot the headless engine
            self.lua.execute('dofile("HeadlessWrapper.lua")')
        finally:
            os.chdir(original_dir)

        # Grab references to key objects
        g = self.lua.globals()
        self._build = g.build
        self._new_build = g.newBuild
        self._load_xml = g.loadBuildFromXML
        self._load_json = g.loadBuildFromJSON
        self._run_frame = g.runCallback

    def _recalc(self):
        """Trigger a full recalculation frame."""
        self._run_frame("OnFrame")

    def _force_calc(self):
        """Force rebuild of all calculation outputs."""
        self._build.calcsTab.BuildOutput(self._build.calcsTab)
        self._recalc()
```

### Reading Stats

```python
    def get_stats(self) -> BuildStats:
        output = self._build.calcsTab.mainOutput
        return BuildStats(
            total_dps=output.TotalDPS or 0,
            average_hit=output.AverageHit or 0,
            crit_chance=output.CritChance or 0,
            crit_multiplier=output.CritMultiplier or 0,
            attack_speed=output.Speed or 0,
            hit_chance=output.HitChance or 0,
            total_life=output.Life or 0,
            total_es=output.EnergyShield or 0,
            total_mana=output.Mana or 0,
            fire_res=output.FireResist or 0,
            cold_res=output.ColdResist or 0,
            lightning_res=output.LightningResist or 0,
            chaos_res=output.ChaosResist or 0,
            armour=output.Armour or 0,
            evasion=output.Evasion or 0,
            block_chance=output.BlockChance or 0,
        )
```

### Equipping Items

```python
    def equip_item(self, raw_text: str, slot: str | None = None) -> BuildStats:
        items_tab = self._build.itemsTab
        items_tab.CreateDisplayItemFromRaw(items_tab, raw_text)
        items_tab.AddDisplayItem(items_tab, True)  # True = auto-equip
        self._recalc()
        return self.get_stats()
```

### Comparing Items (Non-Destructive)

```python
    def compare_item(self, raw_text: str, slot: str | None = None) -> DPSDelta:
        # Snapshot current state
        old_stats = self.get_stats()
        old_dps = old_stats.total_dps

        # Create and equip the new item
        items_tab = self._build.itemsTab
        items_tab.CreateDisplayItemFromRaw(items_tab, raw_text)
        items_tab.AddDisplayItem(items_tab, True)
        self._recalc()

        new_stats = self.get_stats()
        new_dps = new_stats.total_dps

        # TODO: Undo the equip (use item set save/restore or undo system)
        # For now, caller must reload build if they want to restore

        dps_change = new_dps - old_dps
        pct_change = (dps_change / old_dps * 100) if old_dps > 0 else 0

        return DPSDelta(
            old_dps=old_dps,
            new_dps=new_dps,
            dps_change=dps_change,
            dps_change_percent=pct_change,
            stat_changes={
                "life": new_stats.total_life - old_stats.total_life,
                "es": new_stats.total_es - old_stats.total_es,
                "crit_chance": new_stats.crit_chance - old_stats.crit_chance,
                "attack_speed": new_stats.attack_speed - old_stats.attack_speed,
            },
        )
```

---

## Key Considerations

### Working Directory

PoB's `LoadModule()` uses relative paths. The Lua runtime must either:
- Execute from within `src/` (simplest), or
- Override `LoadModule` to resolve paths relative to `src/`

### Method Calls on Lua Objects

In lupa, Lua method calls (`:` syntax) need the object passed explicitly:
```python
# Lua:  build.itemsTab:CreateDisplayItemFromRaw(text)
# Python equivalent:
items_tab = self._build.itemsTab
items_tab.CreateDisplayItemFromRaw(items_tab, raw_text)
```

### The `runCallback("OnFrame")` Requirement

After any modification (equip item, change skill, modify config), you must call `runCallback("OnFrame")` to trigger PoB's internal recalculation. Without this, `mainOutput` will have stale values.

### Thread Safety

Each `LuaRuntime` instance has a GIL-like lock. If you need concurrent build comparisons (e.g., comparing multiple items in parallel), create multiple `PoBEngine` instances. Each boots its own PoB instance (~100-200MB RAM each).

### Error Handling

PoB may set `mainObject.promptMsg` if something goes wrong during initialization. Check this after boot:
```python
g = self.lua.globals()
if g.mainObject.promptMsg:
    raise RuntimeError(f"PoB init failed: {g.mainObject.promptMsg}")
```

---

## MCP Tool Exposure

This module backs the following MCP tools:

| MCP Tool | Method Used |
|----------|-------------|
| `simulate_item_swap(build, slot, item_text)` | `compare_item()` |
| `get_build_stats(build_xml)` | `load_build_xml()` + `get_stats()` |
| `test_custom_mod(build, mod_text)` | `set_config()` + `get_stats()` |

---

## Testing Strategy

### Unit Tests (Python)

```python
def test_engine_boots():
    engine = PoBEngine("/path/to/PathOfBuilding-PoE2")
    assert engine._build is not None

def test_load_build_returns_stats():
    engine = PoBEngine(POB_PATH)
    stats = engine.load_build_xml(SAMPLE_BUILD_XML)
    assert stats.total_dps > 0

def test_equip_item_changes_dps():
    engine = PoBEngine(POB_PATH)
    engine.load_build_xml(SAMPLE_BUILD_XML)
    old_stats = engine.get_stats()
    engine.equip_item("New Item\nHeavy Bow\n+100% increased Physical Damage")
    new_stats = engine.get_stats()
    assert new_stats.total_dps != old_stats.total_dps

def test_compare_item_reports_delta():
    engine = PoBEngine(POB_PATH)
    engine.load_build_xml(SAMPLE_BUILD_XML)
    delta = engine.compare_item("New Item\nHeavy Bow\n+50% to Critical Damage Bonus")
    assert delta.dps_change != 0
    assert abs(delta.dps_change_percent) > 0
```

### Integration Tests

Run the existing PoB test suite via busted to ensure the Lua engine is working:
```bash
cd PathOfBuilding-PoE2/src && busted --helper HeadlessWrapper.lua ../spec
```

---

## File Layout

```
poe2_crafting_mcp/
├── engine/
│   ├── __init__.py
│   ├── pob_engine.py      ← This module
│   ├── models.py          ← BuildStats, DPSDelta dataclasses
│   └── exceptions.py      ← PoBInitError, PoBCalcError
├── vendor/
│   └── PathOfBuilding-PoE2/  ← Git submodule or clone
└── tests/
    ├── test_pob_engine.py
    └── fixtures/
        └── sample_build.xml
```

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| lupa can't load luasocket | Boot fails | Stub `require("socket")` in Lua before loading HeadlessWrapper (it already stubs `require("lcurl.safe")`) |
| PoB update breaks API | Calculations stop working | Pin to a specific PoB-PoE2 release tag; update deliberately |
| Memory usage per engine instance | ~100-200MB | Only run 1-2 instances; don't parallelize unless needed |
| `LoadModule` path resolution fails | Import errors in Lua | Use `os.chdir(src_path)` or inject absolute paths into package.path |
| Some stat keys differ between builds | KeyError on `mainOutput.X` | Use `or 0` / `getattr` with defaults; discover keys dynamically |

---

## Sprint Estimate

| Task | Effort |
|------|--------|
| Bootstrap lupa + PoB headless in Python | 1-2 days |
| Implement `load_build_xml` + `get_stats` | 1 day |
| Implement `equip_item` + `compare_item` | 1-2 days |
| Handle edge cases (path resolution, luasocket stub) | 1 day |
| Write tests with a real build XML | 1 day |
| **Total** | **5-7 days** |
