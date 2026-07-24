# Scope Extension: Conditional DPS & Farming Strategies

## 1. Conditional DPS / Effective DPS Ranges

### The Problem

Raw "TotalDPS" from PoB is the sheet number — but real combat involves:
- Shock on enemy (+25-35% damage taken, depending on build)
- Power charges (gained on crit)
- Flasks active
- Enemy debuffs (exposure, curses)
- "On full ES" or "on low life" conditional mods

### How PoB Already Handles This

PoB's `configTab.input` stores toggle states:

```lua
-- Check what config options are active in a build:
build.configTab.input.enemyShocked     -- bool
build.configTab.input.enemyChilled     -- bool
build.configTab.input.usePowerCharges  -- bool
build.configTab.input.useFrenzyCharges -- bool
build.configTab.input.conditionFullES  -- bool
build.configTab.input.customMods       -- string (arbitrary mod injection)
```

### What We Can Do

Expose an MCP tool: `effective_dps_breakdown`

```python
# Output:
{
    "base_dps": 720000,         # No buffs, no debuffs
    "with_shock": 960000,       # Enemy shocked
    "with_charges": 850000,     # Power + Frenzy charges up
    "full_buffed": 1120000,     # Everything active (realistic mapping)
    "burst_dps": 1450000,       # All flasks, all charges, shocked, etc.
    "scenarios": [
        {"label": "Mapping (sustained)", "dps": 960000, "conditions": ["shock", "power charges"]},
        {"label": "Boss (no shock)", "dps": 780000, "conditions": ["power charges"]},
        {"label": "Full burst", "dps": 1450000, "conditions": ["all"]}
    ]
}
```

We toggle configs on/off, recalculate, and present the range. This helps answer:
- "What's my real mapping DPS?" (sustained buffs)
- "What's my boss DPS?" (fewer reliable debuffs)
- "Is shock worth investing in for my build?"

### Implementation

```lua
-- Save current config
local savedConfig = {}
for k, v in pairs(build.configTab.input) do savedConfig[k] = v end

-- Test with shock disabled
build.configTab.input.enemyShocked = false
build.configTab:BuildModList()
runCallback("OnFrame")
local dps_no_shock = build.calcsTab.mainOutput.TotalDPS

-- Test with shock enabled
build.configTab.input.enemyShocked = true
build.configTab:BuildModList()
runCallback("OnFrame")
local dps_with_shock = build.calcsTab.mainOutput.TotalDPS

-- Restore
for k, v in pairs(savedConfig) do build.configTab.input[k] = v end
build.configTab:BuildModList()
runCallback("OnFrame")
```

---

## 2. Farming Strategy Advisor

### The Concept

Given your build's capabilities and the current economy, recommend the most profitable farming activities.

### Data Needed

| Data | Source | Description |
|------|--------|-------------|
| Build clear speed | PoB (DPS + movement speed + AoE) | How fast you kill |
| Build survivability | PoB (EHP, max hit, resistances) | What tier content you can run |
| Content reward tables | Community wikis / datamined | What drops where |
| Current economy prices | poe.show / poe.ninja | What's worth farming |
| Map mod compatibility | PoB config toggles | Can you run "no regen"? "reflect"? |

### Example Output

```
User: "What should I farm with this build? Goal: divine/hour"

Agent:
┌─────────────────────────────────────────────────────────┐
│ FARMING RECOMMENDATIONS (Whirling Trinity, 960K DPS)    │
│                                                         │
│ Your build: Fast clear, high single-target, CI (no     │
│ chaos damage worry), 7.8K ES                           │
│                                                         │
│ #1 T16 Maps with Ritual (1.8 divine/hour estimated)    │
│    - Your clear speed handles T16 comfortably           │
│    - Ritual rewards currently overvalued                 │
│    - Avoid: "No ES Recharge" maps                       │
│                                                         │
│ #2 Expedition (1.5 divine/hour estimated)              │
│    - Rog bases selling well this league                  │
│    - Your DPS one-shots remnants                        │
│    - Good for crafting base generation                   │
│                                                         │
│ #3 Boss farming — Tier 4 Pinnacle (1.2 divine/hour)    │
│    - Your single-target is sufficient                    │
│    - Unique drop pool currently profitable              │
│    - Risk: slower if mechanics heavy                    │
│                                                         │
│ Avoid:                                                  │
│  - Delirium (your ES is borderline for 100%)            │
│  - Maps with "Monsters reflect elemental damage"        │
└─────────────────────────────────────────────────────────┘
```

### Capabilities Assessment (from PoB data)

```python
@dataclass
class BuildCapabilities:
    """What content this build can comfortably do."""
    # Offence
    clear_speed_tier: str        # "fast", "medium", "slow" (based on DPS + AoE + speed)
    single_target_dps: float     # Boss DPS
    mapping_dps: float           # Sustained DPS with reasonable buffs

    # Defence
    max_affordable_tier: int     # Highest map tier survivable (T1-T16)
    can_do_bosses: bool          # Enough EHP + DPS for pinnacle bosses
    dangerous_map_mods: list[str]  # ["no ES recharge", "reflect elemental"]
    safe_map_mods: list[str]     # ["extra physical", "monster speed"]

    # Farming style
    movement_speed: float        # For clear speed estimation
    has_aoe: bool                # Can clear packs efficiently
    is_ci: bool                  # Immune to chaos = no "60% less chaos res" worry
```

### Content Profitability (Economy-Driven)

This changes every league and even week-to-week. We'd pull from poe.show/poe.ninja:

```python
@dataclass
class FarmingActivity:
    name: str                       # "T16 Maps with Ritual"
    estimated_divine_per_hour: float
    requirements: dict              # {"min_dps": 500000, "min_ehp": 5000}
    rewards: list[str]              # What drops are valuable
    avoid_mods: list[str]           # Map mods incompatible with build
    notes: str
```

### Where Does the Content Reward Data Come From?

This is the hardest part. Options:
1. **Community-maintained tables** — sites like onlyfarms.gg, timesaver.gg publish farming guides with divine/hour estimates
2. **poe.ninja economy data** — we know what items are worth, can estimate drop value
3. **Hardcoded heuristics** — "Expedition = bases + currency", "Ritual = deferred value", etc. Updated per league
4. **User-reported data** — "I made 2 divine/hour running X" — crowdsourced

For MVP, option 3 (heuristics + economy prices) is sufficient. The agent knows:
- What content exists (Ritual, Expedition, Delirium, Breach, Maps, Bosses)
- What each drops (currency, bases, uniques, fragments)
- What those drops are worth right now (from pricing module)
- Whether your build can handle it (from PoB stats)

---

## MCP Tools to Add

| Tool | Description |
|------|-------------|
| `effective_dps_breakdown(scenarios?)` | Show DPS under different buff/debuff conditions |
| `toggle_config(option, value)` | Toggle a PoB config option and show effect |
| `farming_recommendations(goal?)` | Suggest profitable content for your build |
| `assess_build_capabilities()` | What content can this build handle? |
| `dangerous_map_mods()` | Which map mods should you avoid? |

---

## Priority

| Feature | Priority | Depends On |
|---------|----------|------------|
| Effective DPS breakdown | High (Sprint 3) | PoB engine only — cheap to add |
| Config toggle | High (Sprint 3) | PoB engine only |
| Build capability assessment | Medium (Sprint 6) | PoB engine + heuristics |
| Farming recommendations | Low (Sprint 8+) | All of the above + economy data + content tables |
| Dangerous map mods | Medium (Sprint 6) | PoB config toggling |

The DPS breakdown is almost free to implement — it's just toggling PoB configs and reading the output. Farming strategies require more data infrastructure but build naturally on everything else.
