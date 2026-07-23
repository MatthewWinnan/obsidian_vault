# Module Card: Crafting Probability Engine

## Overview

Calculates the probability of hitting desired mods when crafting, generates optimal multi-step crafting recipes, and estimates expected costs. This is the core "craft advisor" logic — it answers "how do I make this item, and what will it cost me on average?"

---

## The Fundamental Problem

PoE2 crafting is probabilistic. When you use an Exalted Orb, you add one random modifier from a pool. The engine needs to answer:

1. **What can roll?** — Given a base type, item level, and current mods, what's the pool of available mods?
2. **What's the chance of hitting what I want?** — `P(mod) = weight(mod) / sum(all_eligible_weights)`
3. **What's the optimal crafting path?** — Which sequence of currency produces the best expected outcome for the budget?

---

## Critical Data: Mod Weights in PoE2

**Key finding from research:** Mod weights are NOT in the PoE2 game files. They've been reverse-engineered by the community:

- **Krakenbul** and the **Prohibited Library Discord** compiled weights using recombinators (before 0.5 removed them)
- **Craft of Exile** uses this data + trade-site parsing + normalization scripts
- **poe2db.tw** hosts the community-compiled weight tables

The weights are empirical approximations, not exact game data. They're "good enough" for crafting advice but not perfectly precise.

### Data Sources for Weights

| Source | Format | Completeness | Access |
|--------|--------|--------------|--------|
| Craft of Exile internal DB | Not exposed as API | Most complete, normalized | Scrape or replicate |
| poe2db.tw mod pages | HTML tables per base type | Good for affix pools | Scrape |
| Prohibited Library spreadsheets | Google Sheets | Raw community data | Public links on Discord |
| PoB-PoE2 `src/Data/` | Lua tables | Affix pools (may lack weights) | Direct file access |

**Recommended approach:** Use PoB-PoE2's `src/Data/` for affix pool definitions (which mods exist, what ilvl they require, prefix/suffix, mod groups) and layer the community weight data on top from Prohibited Library/Craft of Exile.

---

## PoE2 Crafting Currency Mechanics (Patch 0.5)

| Currency | Effect | Use Case |
|----------|--------|----------|
| **Orb of Transmutation** | Normal → Magic (1 mod) | Starting point |
| **Orb of Augmentation** | Add 1 mod to Magic item | Cheap 2-mod magic |
| **Regal Orb** | Magic → Rare (adds 1 mod) | Transition to rare |
| **Exalted Orb** | Add 1 random mod to Rare | Targeted slam |
| **Greater/Perfect Exalted** | Add 1 mod (higher tier bias) | Better odds |
| **Chaos Orb** | Remove 1 random mod, add 1 random mod | Reroll single mod |
| **Orb of Annulment** | Remove 1 random mod | Fix bad rolls |
| **Essence** | Reforge with 1 guaranteed mod | Targeted crafting |
| **Orb of Alchemy** | Normal → Rare (3-4 mods) | Bulk reforge |
| **Fracturing Orb** | Lock 1 mod permanently (4+ mod rare) | Preserve good mods |

### Key 0.5 Changes
- Recombinators removed from the game
- One crafted modifier per item (Essence/Alloy share the slot)
- Essences guarantee one specific mod; remaining mods are random

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Crafting Probability Engine                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────┐   ┌─────────────────────────────────────┐ │
│  │   Mod Database       │   │   Probability Calculator            │ │
│  │                     │   │                                     │ │
│  │  • Affix pools      │   │  P(mod) = weight / Σ(all_weights)  │ │
│  │  • Weights per mod  │   │                                     │ │
│  │  • ilvl requirements│   │  Handles:                           │ │
│  │  • Prefix/suffix    │   │  • Single-slam odds                 │ │
│  │  • Mod groups       │   │  • Multi-mod probability            │ │
│  │  • Tag restrictions │   │  • "At least T2" calculations       │ │
│  │                     │   │  • Pool shrinking (mods already on) │ │
│  └──────────┬──────────┘   └──────────────────┬──────────────────┘ │
│             │                                  │                    │
│             └──────────────┬───────────────────┘                    │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Recipe Generator                           │   │
│  │                                                             │   │
│  │  Given: target mods + budget                                │   │
│  │  Output: optimal sequence of crafting steps                 │   │
│  │                                                             │   │
│  │  Strategies:                                                │   │
│  │  • Essence spam (guarantee 1 mod, roll others)              │   │
│  │  • Alt/aug/regal (cheap 3-mod base, then exalt)             │   │
│  │  • Chaos spam (brute force for multi-mod combos)            │   │
│  │  • Fracture + reforge (lock 1 good mod, reroll rest)        │   │
│  │  • Targeted exalt (fill empty affixes)                      │   │
│  │  • Annul + slam (remove bad, add new)                       │   │
│  │                                                             │   │
│  │  Each step has: probability, cost, expected attempts        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Python API Design

```python
from dataclasses import dataclass, field
from enum import Enum


class AffixType(Enum):
    PREFIX = "prefix"
    SUFFIX = "suffix"


class ModTier(Enum):
    T1 = 1
    T2 = 2
    T3 = 3
    T4 = 4
    T5 = 5
    T6 = 6
    ANY = 0  # Any tier acceptable


@dataclass
class Modifier:
    """A single modifier that can appear on an item."""
    mod_id: str              # Internal identifier
    name: str                # Human-readable (e.g., "Tyrannical")
    stat_text: str           # Display text (e.g., "+170 to 179% increased Physical Damage")
    affix_type: AffixType
    tier: int
    ilvl_required: int       # Minimum item level to roll this
    weight: int              # Rolling weight (higher = more common)
    mod_group: str           # Mods in same group can't coexist
    tags: list[str]          # Item tags that must match
    value_range: tuple[float, float]  # (min_roll, max_roll)


@dataclass
class AffixPool:
    """The complete pool of rollable mods for a specific item context."""
    base_type: str
    item_level: int
    available_prefixes: list[Modifier]
    available_suffixes: list[Modifier]
    total_prefix_weight: int
    total_suffix_weight: int

    def prefix_probability(self, mod: Modifier) -> float:
        """Chance of rolling this specific prefix."""
        if mod not in self.available_prefixes:
            return 0.0
        return mod.weight / self.total_prefix_weight

    def suffix_probability(self, mod: Modifier) -> float:
        """Chance of rolling this specific suffix."""
        if mod not in self.available_suffixes:
            return 0.0
        return mod.weight / self.total_suffix_weight


@dataclass
class CraftingStep:
    """One step in a crafting recipe."""
    action: str                 # e.g., "Use Exalted Orb", "Use Essence of Wrath"
    currency_name: str          # For pricing lookup
    currency_per_attempt: int   # Usually 1, but some need multiples
    target_description: str     # What we're trying to hit
    success_probability: float  # Per-attempt probability (0.0 to 1.0)
    expected_attempts: float    # 1 / success_probability
    failure_action: str         # What to do if it fails ("try again", "annul", "start over")
    notes: str                  # Explanation for the user


@dataclass
class CraftingRecipe:
    """A complete crafting recipe from base to finished item."""
    name: str                       # e.g., "Fire Damage Body Armour"
    base_type: str                  # e.g., "Vaal Regalia"
    required_ilvl: int
    target_mods: list[str]          # What the finished item should have
    steps: list[CraftingStep]
    total_expected_cost: float      # In divine orbs (computed from pricing)
    confidence: str                 # "high", "medium", "low" (based on weight data quality)
    strategy_name: str              # e.g., "Essence Spam", "Alt-Regal-Exalt"
    warnings: list[str]             # Caveats (e.g., "weights are empirical estimates")


class CraftingEngine:
    """
    Calculates crafting probabilities and generates optimal recipes.
    """

    def __init__(self, mod_database: "ModDatabase", pricing: "PricingEngine"):
        self.mods = mod_database
        self.pricing = pricing

    # ─── Probability Queries ─────────────────────────────────────

    def get_affix_pool(
        self,
        base_type: str,
        item_level: int,
        existing_mods: list[str] | None = None,
    ) -> AffixPool:
        """
        Get the available mod pool for a given item context.
        
        Excludes:
        - Mods below the item level
        - Mods in the same mod group as existing mods
        - Mods whose tags don't match the base type
        """
        ...

    def probability_of_mod(
        self,
        base_type: str,
        item_level: int,
        target_mod: str,
        existing_mods: list[str] | None = None,
    ) -> float:
        """
        Probability of hitting a specific mod on the next slam/roll.
        
        Example:
            probability_of_mod("Vaal Regalia", 86, "+1 to Level of Fire Skill Gems")
            → 0.023 (2.3%)
        """
        pool = self.get_affix_pool(base_type, item_level, existing_mods)
        mod = self.mods.find_mod(target_mod)
        
        if mod.affix_type == AffixType.PREFIX:
            return pool.prefix_probability(mod)
        else:
            return pool.suffix_probability(mod)

    def probability_of_any_tier(
        self,
        base_type: str,
        item_level: int,
        mod_group: str,
        min_tier: ModTier = ModTier.ANY,
        existing_mods: list[str] | None = None,
    ) -> float:
        """
        Probability of hitting any tier of a mod group.
        
        Example:
            probability_of_any_tier("Vaal Regalia", 86, "MaximumLife", min_tier=ModTier.T2)
            → 0.08 (8% chance of T1 or T2 life)
        """
        pool = self.get_affix_pool(base_type, item_level, existing_mods)
        
        matching = [
            m for m in (pool.available_prefixes + pool.available_suffixes)
            if m.mod_group == mod_group and m.tier <= min_tier.value
        ]
        
        total_weight = sum(m.weight for m in matching)
        
        if matching[0].affix_type == AffixType.PREFIX:
            return total_weight / pool.total_prefix_weight
        else:
            return total_weight / pool.total_suffix_weight

    def expected_attempts(
        self,
        base_type: str,
        item_level: int,
        target_mod: str,
        existing_mods: list[str] | None = None,
    ) -> float:
        """
        Expected number of attempts to hit a target mod.
        
        Example:
            expected_attempts("Vaal Regalia", 86, "+1 Fire Gems")
            → 43.5 attempts on average
        """
        prob = self.probability_of_mod(base_type, item_level, target_mod, existing_mods)
        if prob <= 0:
            return float('inf')
        return 1.0 / prob

    # ─── Recipe Generation ───────────────────────────────────────

    async def generate_recipe(
        self,
        base_type: str,
        item_level: int,
        desired_mods: list[dict],
        budget: float | None = None,
    ) -> list[CraftingRecipe]:
        """
        Generate one or more crafting recipes to achieve the desired item.
        
        Returns multiple strategies ranked by expected cost.
        
        Args:
            base_type: Item base (e.g., "Vaal Regalia")
            item_level: Required ilvl
            desired_mods: List of {"mod_group": str, "min_tier": int}
            budget: Optional budget cap in divine orbs
            
        Returns:
            List of recipes, cheapest first.
        """
        strategies = []
        
        # Strategy 1: Essence spam
        essence_recipe = await self._try_essence_strategy(
            base_type, item_level, desired_mods
        )
        if essence_recipe:
            strategies.append(essence_recipe)
        
        # Strategy 2: Alt/Aug/Regal + Exalt
        alt_regal_recipe = await self._try_alt_regal_strategy(
            base_type, item_level, desired_mods
        )
        if alt_regal_recipe:
            strategies.append(alt_regal_recipe)
        
        # Strategy 3: Chaos spam (for multi-mod combos)
        chaos_recipe = await self._try_chaos_strategy(
            base_type, item_level, desired_mods
        )
        if chaos_recipe:
            strategies.append(chaos_recipe)
        
        # Strategy 4: Fracture + reroll
        fracture_recipe = await self._try_fracture_strategy(
            base_type, item_level, desired_mods
        )
        if fracture_recipe:
            strategies.append(fracture_recipe)
        
        # Sort by expected cost
        strategies.sort(key=lambda r: r.total_expected_cost)
        
        # Filter by budget
        if budget is not None:
            strategies = [s for s in strategies if s.total_expected_cost <= budget]
        
        return strategies

    # ─── Strategy Implementations ────────────────────────────────

    async def _try_essence_strategy(
        self, base_type: str, item_level: int, desired_mods: list[dict]
    ) -> CraftingRecipe | None:
        """
        Essence strategy: Use an essence to guarantee one desired mod,
        then hope/craft for the rest.
        
        Best when: One mod is hard to roll naturally but available via essence.
        """
        # Find if any desired mod has a matching essence
        # Calculate odds of hitting remaining mods alongside the guaranteed one
        # Factor in annulment for fixing bad outcomes
        ...

    async def _try_alt_regal_strategy(
        self, base_type: str, item_level: int, desired_mods: list[dict]
    ) -> CraftingRecipe | None:
        """
        Alt/Aug/Regal strategy:
        1. Transmute (1 mod)
        2. Augment (2 mods)  
        3. Alt-spam until you hit a key mod
        4. Regal (promote to rare, adds 1 mod)
        5. Exalt remaining slots
        
        Best when: One mod is common enough to alt-spam for cheaply.
        """
        ...

    async def _try_chaos_strategy(
        self, base_type: str, item_level: int, desired_mods: list[dict]
    ) -> CraftingRecipe | None:
        """
        Chaos spam: Use Chaos Orbs (remove 1 add 1) repeatedly.
        
        Best when: Item already has some good mods and you want to
        fix one bad mod at a time. NOT for starting fresh.
        
        Note: In PoE2 0.5, Chaos Orb removes 1 + adds 1 (not full reroll).
        """
        ...

    async def _try_fracture_strategy(
        self, base_type: str, item_level: int, desired_mods: list[dict]
    ) -> CraftingRecipe | None:
        """
        Fracture strategy:
        1. Get an item with 1 amazing mod + filler
        2. Fracturing Orb to lock the good mod forever
        3. Reroll the rest freely (Alchemy/Essence spam)
        
        Best when: One mod is extremely rare/expensive and worth preserving.
        Requires: Item has 4+ mods before fracturing.
        """
        ...

    # ─── Monte Carlo Simulation ──────────────────────────────────

    def simulate_craft(
        self,
        base_type: str,
        item_level: int,
        desired_mods: list[dict],
        strategy: str,
        iterations: int = 10000,
    ) -> dict:
        """
        Run Monte Carlo simulation for complex crafting paths.
        
        Useful when the probability can't be calculated analytically
        (e.g., multi-step paths with conditional branches).
        
        Returns:
            {
                "median_cost": 5.2,
                "mean_cost": 7.8,
                "p25_cost": 3.1,     # 25th percentile (lucky)
                "p75_cost": 9.4,     # 75th percentile (unlucky)
                "p95_cost": 18.2,    # 95th percentile (very unlucky)
                "success_rate": 0.95, # % that succeeded within 100 attempts
            }
        """
        ...
```

---

## Mod Database Design

```python
class ModDatabase:
    """
    Local database of all PoE2 item modifiers, their weights,
    affix pools, and relationships.
    
    Data sources:
    - PoB-PoE2 src/Data/ (affix definitions, ilvl requirements)
    - Prohibited Library / Craft of Exile (weights)
    - poe2db.tw (supplementary data)
    """

    def __init__(self, data_path: str):
        """Load mod data from local files."""
        self._mods: dict[str, Modifier] = {}
        self._pools: dict[str, list[Modifier]] = {}  # base_type → mods
        self._load_data(data_path)

    def get_pool_for_base(
        self, base_type: str, item_level: int
    ) -> list[Modifier]:
        """All mods that can roll on this base at this ilvl."""
        ...

    def find_mod(self, query: str) -> Modifier | None:
        """Fuzzy search for a mod by name or stat text."""
        ...

    def find_essence_for_mod(self, mod_group: str) -> str | None:
        """Find which essence guarantees a mod from this group."""
        ...

    def get_mod_group_members(self, mod_group: str) -> list[Modifier]:
        """Get all tiers of a mod group."""
        ...

    def _load_data(self, data_path: str):
        """
        Load from PoB-PoE2's Lua data tables + weight overlay.
        
        PoB stores mods in: src/Data/ModItem.lua, src/Data/Bases/...
        Weights stored in: our own JSON overlay from community data
        """
        ...
```

### Data File Structure

```
poe2_crafting_mcp/
├── data/
│   ├── mod_weights/
│   │   ├── body_armour_int.json    # Weights per base category
│   │   ├── body_armour_str.json
│   │   ├── weapon_sceptre.json
│   │   ├── amulet.json
│   │   └── ...
│   ├── essences.json               # Essence → guaranteed mod mapping
│   ├── mod_groups.json             # Mod group definitions
│   └── currency_effects.json      # What each currency does
```

### Weight Data Format

```json
{
  "base_category": "body_armour_int",
  "source": "prohibited_library_v3",
  "patch": "0.5",
  "last_updated": "2026-06-15",
  "prefixes": [
    {
      "mod_id": "LocalIncreasedEnergyShield7",
      "name": "Incandescent",
      "stat": "+150 to 179 to maximum Energy Shield",
      "tier": 1,
      "ilvl": 82,
      "weight": 250,
      "mod_group": "LocalEnergyShield"
    },
    {
      "mod_id": "FireSkillGemLevel1",
      "name": "Cremating",
      "stat": "+1 to Level of all Fire Skill Gems",
      "tier": 1,
      "ilvl": 55,
      "weight": 50,
      "mod_group": "FireGemLevel"
    }
  ],
  "suffixes": [...]
}
```

---

## Example: Full Crafting Calculation

**User wants:** Body Armour with +1 Fire Gems, Spell Damage, +90 Life

**Engine process:**

```
1. Get affix pool for "Vaal Regalia" ilvl 86:
   - 45 possible prefixes (total weight: 12,500)
   - 52 possible suffixes (total weight: 14,200)

2. Identify desired mods:
   - "+1 Fire Gems" = prefix, weight 50 → P = 50/12500 = 0.4%
   - "Spell Damage 70%+" (T1-T2) = prefix, weight 200+350 = 550 → P = 4.4%
   - "+90 Life" (T1-T2) = prefix, weight 800+600 = 1400 → P = 11.2%

3. Problem: All 3 are prefixes! Max 3 prefixes on a rare item.
   Need ALL THREE in prefix slots.

4. Strategy evaluation:
   
   A) Essence of Wrath (guarantees Spell Damage):
      - Use Shrieking Essence of Wrath → guarantees 70%+ spell damage
      - Item becomes rare with spell damage + 2-3 random mods
      - Need: +1 Fire AND +Life to also land in the other prefix slots
      - P(fire gems in remaining 2 prefix slots) ≈ 0.4% per prefix
      - P(life in remaining prefix slot after fire) ≈ 11.2%
      - Combined per essence: ~0.045%
      - Expected essences: ~2,222
      - Cost per essence: 0.3 divine
      - Expected cost: 667 divine (TOO EXPENSIVE)
   
   B) Fracture approach:
      - Buy/roll an item with +1 Fire Gems (hardest mod)
      - Fracture it (lock the +1 fire)
      - Essence spam for spell damage (guaranteed) + pray for life
      - P(life alongside essence spam) ≈ 11.2% per attempt
      - Expected essences: ~9
      - Cost: fracture orb (8 divine) + 9 essences (2.7 divine) + base (0.5 divine)
      - Expected cost: ~11.2 divine ← VIABLE
   
   C) Just buy it on trade:
      - Search: 15 divine for this combo
   
   RECOMMENDATION: Strategy B (Fracture) saves ~4 divine vs buying.
```

---

## MCP Tool Exposure

| MCP Tool | Description |
|----------|-------------|
| `get_crafting_odds(base, ilvl, target_mod)` | Probability of hitting a specific mod |
| `generate_crafting_recipe(base, ilvl, desired_mods, budget)` | Full recipe with steps and costs |
| `simulate_craft(base, ilvl, mods, strategy, iterations)` | Monte Carlo for complex paths |
| `list_available_mods(base, ilvl, affix_type?)` | Show what can roll on this item |
| `find_essence_for_mod(mod_description)` | Which essence guarantees this mod? |

---

## Integration Points

| Module | How Crafting Engine Uses It |
|--------|----------------------------|
| **Pricing Engine** | Prices each currency in the recipe → total cost |
| **Meta Intelligence** | Identifies which mods to target (what top players use) |
| **PoB Engine** | Validates the crafted item is actually an upgrade (DPS sim) |
| **Build Session** | Knows which slot we're crafting for |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Weights are empirical approximations | Probability estimates may be 10-20% off | Always show confidence level; warn user |
| Weights change with patches | Stale data gives wrong advice | Version-tag weight files per patch; monitor Prohibited Library for updates |
| No weights for new item types | Can't calculate for new bases | Fall back to "equal weight" assumption; flag to user |
| Complex multi-step paths are hard to model analytically | Wrong expected cost | Use Monte Carlo simulation for anything beyond 2 steps |
| Some strategies depend on item state (what's already on it) | Conditional logic is complex | Model as decision tree; each node = "what to do if X happens" |

---

## Weight Data Maintenance Strategy

```
1. On new league/patch:
   - Check Prohibited Library Discord for updated spreadsheets
   - Check Craft of Exile changelog for data updates
   - Pull PoB-PoE2 latest src/Data/ for new bases/mods

2. Automated:
   - Script to parse poe2db.tw mod pages for new affixes
   - Script to convert PoB Lua data tables → our JSON format
   
3. Confidence scoring:
   - Mods with verified weights (recombinator data): confidence = "high"
   - Mods with trade-parsed weights: confidence = "medium"  
   - New/unverified mods (assumed weights): confidence = "low"
```

---

## Sprint Estimate

| Task | Effort |
|------|--------|
| Mod database schema + loader (from PoB data) | 2-3 days |
| Weight data compilation (Prohibited Library + poe2db) | 2-3 days |
| Probability calculator (single mod, mod group, conditional) | 2 days |
| Recipe generator (Essence, Alt-Regal, Fracture strategies) | 3-4 days |
| Monte Carlo simulator | 1-2 days |
| Cost estimation (integrate with Pricing Engine) | 1 day |
| Testing against known Craft of Exile results | 1-2 days |
| **Total** | **12-16 days** |

This is the most labour-intensive module because of the data compilation work. The actual code is straightforward probability math — the hard part is getting the weight data right and keeping it current.
