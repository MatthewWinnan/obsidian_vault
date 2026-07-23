# Module Card: Meta Intelligence & Build Gap Analysis

## Overview

Analyses what top players and the broader community are using for your specific build archetype, then compares their gear/tree against yours to identify the highest-impact upgrade paths. This ensures the agent's recommendations follow proven community theory rather than guessing.

---

## Core Concept

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   YOUR BUILD                          TOP PLAYERS (same archetype)   │
│                                                                      │
│   Fireball Elementalist               Top 50 Fireball Elementalists  │
│   Level 87, 245k DPS                  Level 95+, 1.2M+ DPS          │
│                                                                      │
│   Body: Rare Vaal Regalia             Body: +1 fire, spell dmg, life │
│         (+100 life only)                    (90% use this pattern)   │
│                                                                      │
│   Weapon: Random sceptre             Weapon: +1 fire, cast speed,   │
│           (flat damage only)                  spell dmg %            │
│                                              (85% use this pattern)  │
│                                                                      │
│   ─── GAP ANALYSIS ───────────────────────────────────────────────   │
│                                                                      │
│   #1 Priority: Body armour (biggest DPS gap vs. meta)                │
│   #2 Priority: Weapon (missing +1 gems and cast speed)               │
│   #3 Priority: Amulet (top players have crit multi here)             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Data Sources

### 1. poe.ninja Builds API (Primary)

**Endpoint (undocumented, reverse-engineered):**
```
GET https://poe.ninja/api/data/0/getbuildoverview
    ?overview={league_slug}
    &type=exp
    &language=en
```

**Returns:**
```json
{
  "names": ["CharName1", "CharName2", ...],
  "classes": [3, 5, 3, ...],
  "classNames": ["Marauder", "Witch", "Elementalist", ...],
  "levels": [97, 96, 95, ...],
  "activeSkills": [{"name": "Fireball", "id": 42}, ...],
  "activeSkillUse": {"42": [0, 5, 12, ...], ...},
  "skillDetails": [{"name": "Fireball", "dps": {"0": [1200000], "5": [980000]}}],
  "uniqueItems": [...],
  "keystones": [...],
  "items": [...]  // equipment per character
}
```

**Individual character detail (poe.ninja web):**
```
GET https://poe.ninja/poe2/builds/char/{account}/{character}
```

### 2. GGG Official Ladder API (Fallback / Supplement)

```
GET https://api.pathofexile.com/league/{league}/ladder?realm=poe2&limit=500
```

Returns ranked characters. For each public character:
```
GET https://api.pathofexile.com/character/poe2/{name}
```

Returns full equipment, skills, passive tree as JSON.

### 3. poe.ninja Data Dumps (Bulk Analysis)

poe.ninja provides periodic data dumps of build statistics at `/poe1/data` (and presumably similar for PoE2). Useful for offline analysis if the API changes.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  Meta Intelligence Module                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────┐    ┌────────────────────────────┐   │
│  │   Build Corpus        │    │   Gap Analyser             │   │
│  │                       │    │                            │   │
│  │   Fetches & caches    │    │   Compares user build      │   │
│  │   top player data     │    │   vs corpus patterns       │   │
│  │   from poe.ninja      │    │   per slot                 │   │
│  │                       │    │                            │   │
│  │   Filters by:         │    │   Outputs:                 │   │
│  │   - Class/ascendancy  │    │   - Slot priority ranking  │   │
│  │   - Main skill        │    │   - Target mod patterns    │   │
│  │   - Level range       │    │   - Popularity %           │   │
│  │   - DPS threshold     │    │   - Estimated DPS gain     │   │
│  └───────────┬───────────┘    └─────────────┬──────────────┘   │
│              │                               │                  │
│              └───────────────┬───────────────┘                  │
│                              │                                  │
│                              ▼                                  │
│              ┌───────────────────────────────┐                  │
│              │   Upgrade Recommender         │                  │
│              │                               │                  │
│              │   For each slot gap:          │                  │
│              │   1. What mods to target      │                  │
│              │   2. Buy price (trade)        │                  │
│              │   3. Craft cost (probability) │                  │
│              │   4. DPS gain (PoB sim)       │                  │
│              │   5. Priority score           │                  │
│              └───────────────────────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Python API Design

```python
from dataclasses import dataclass, field


@dataclass
class ModPattern:
    """A frequently-occurring mod on a slot among top players."""
    mod_text: str
    occurrence_percent: float  # e.g., 87.5 = 87.5% of top players have this
    typical_tier: str          # e.g., "T1", "T1-T2"
    typical_value_range: tuple[float, float]  # e.g., (90, 100) for +life


@dataclass
class SlotMeta:
    """What top players use in a specific equipment slot."""
    slot: str                         # e.g., "Body Armour", "Weapon 1"
    common_base_types: list[tuple[str, float]]  # (base_name, %) 
    key_mods: list[ModPattern]        # Most important mods for this slot
    common_uniques: list[tuple[str, float]]  # (unique_name, usage %)
    sample_size: int                  # How many characters in this analysis


@dataclass
class SlotGap:
    """A specific upgrade opportunity identified by gap analysis."""
    slot: str
    priority_rank: int               # 1 = most impactful upgrade
    priority_score: float            # Composite score (DPS weight + popularity)
    your_item_summary: str           # What you currently have
    target_mods: list[ModPattern]    # What you should aim for
    estimated_dps_gain: float | None # From PoB sim (if available)
    estimated_dps_gain_percent: float | None
    meta_popularity: float           # % of top players using this pattern
    suggested_base: str | None       # Recommended base type
    notes: str                       # Human-readable explanation


@dataclass 
class MetaSnapshot:
    """A cached snapshot of the meta for a specific archetype."""
    archetype: str           # e.g., "Fireball Elementalist"
    league: str
    sample_size: int         # Total characters analysed
    avg_dps: float           # Average DPS of the cohort
    median_level: int
    slots: dict[str, SlotMeta]   # slot_name → meta data
    top_keystones: list[tuple[str, float]]   # (keystone, usage %)
    top_support_gems: list[tuple[str, float]]
    fetched_at: float        # timestamp


class MetaIntelligence:
    """
    Analyses top player builds to identify upgrade paths
    grounded in proven community theory.
    """

    def __init__(
        self,
        league_context: "LeagueContext",
        pob_engine: "PoBEngine",
        pricing: "PricingEngine",
        user_agent: str = "poe2-craft-mcp/1.0",
    ):
        self.league = league_context
        self.pob = pob_engine
        self.pricing = pricing
        self._user_agent = user_agent
        self._cache: dict[str, MetaSnapshot] = {}

    # ─── Data Fetching ───────────────────────────────────────────

    async def fetch_meta_snapshot(
        self,
        main_skill: str,
        ascendancy: str,
        min_level: int = 90,
        top_n: int = 50,
    ) -> MetaSnapshot:
        """
        Fetch and analyse top player data for a specific archetype.
        
        Caches results for 24 hours (meta doesn't change that fast).
        
        Args:
            main_skill: The primary damage skill (e.g., "Fireball")
            ascendancy: The ascendancy class (e.g., "Elementalist")
            min_level: Minimum character level to include
            top_n: Number of top characters to analyse
        """
        cache_key = f"{main_skill}:{ascendancy}:{self.league.league_id}"
        
        if cache_key in self._cache:
            snapshot = self._cache[cache_key]
            if time.time() - snapshot.fetched_at < 86400:  # 24h
                return snapshot
        
        # Fetch from poe.ninja builds API
        raw_data = await self._fetch_builds_data()
        
        # Filter to matching archetype
        characters = self._filter_characters(
            raw_data, main_skill, ascendancy, min_level, top_n
        )
        
        # Analyse gear patterns per slot
        snapshot = self._analyse_patterns(characters, main_skill, ascendancy)
        self._cache[cache_key] = snapshot
        return snapshot

    # ─── Gap Analysis ────────────────────────────────────────────

    async def analyse_gaps(
        self,
        build_session: "BuildSession",
        top_n: int = 50,
    ) -> list[SlotGap]:
        """
        Compare your build against top players of the same archetype.
        Returns a prioritised list of upgrade opportunities.
        
        Args:
            build_session: Your currently loaded build
            top_n: How many top players to compare against
            
        Returns:
            List of SlotGap objects, sorted by priority (highest impact first)
        """
        summary = build_session.summary
        
        # Get meta for your archetype
        meta = await self.fetch_meta_snapshot(
            main_skill=summary.main_skill,
            ascendancy=summary.ascendancy,
            top_n=top_n,
        )
        
        gaps = []
        for slot_name, slot_meta in meta.slots.items():
            # Get your current item in this slot
            your_item = self._get_user_item_for_slot(build_session, slot_name)
            
            # Compare your item's mods against the meta patterns
            gap = self._compute_slot_gap(
                slot_name, your_item, slot_meta, meta
            )
            
            if gap and gap.priority_score > 0:
                gaps.append(gap)
        
        # Sort by priority score (highest first)
        gaps.sort(key=lambda g: g.priority_score, reverse=True)
        
        # Assign ranks
        for i, gap in enumerate(gaps):
            gap.priority_rank = i + 1
        
        return gaps

    # ─── Upgrade Recommendations ─────────────────────────────────

    async def get_upgrade_plan(
        self,
        build_session: "BuildSession",
        budget: float,
        goal: str = "damage",
        max_slots: int = 3,
    ) -> list["UpgradeRecommendation"]:
        """
        Generate a concrete upgrade plan within budget.
        
        Combines:
        - Gap analysis (what to upgrade)
        - Pricing (what it costs)
        - Crafting probability (craft vs buy)
        - PoB simulation (actual DPS gain)
        
        Args:
            build_session: Your loaded build
            budget: Total budget in divine orbs
            goal: "damage", "survivability", or "balanced"
            max_slots: Max number of slots to recommend upgrading
        """
        gaps = await self.analyse_gaps(build_session)
        
        # Filter by goal
        if goal == "damage":
            gaps = [g for g in gaps if self._is_damage_slot(g)]
        elif goal == "survivability":
            gaps = [g for g in gaps if self._is_defence_slot(g)]
        
        recommendations = []
        remaining_budget = budget
        
        for gap in gaps[:max_slots]:
            if remaining_budget <= 0:
                break
            
            rec = await self._build_recommendation(
                gap, build_session, remaining_budget
            )
            
            if rec and rec.cost <= remaining_budget:
                recommendations.append(rec)
                remaining_budget -= rec.cost
        
        return recommendations

    # ─── Specific Queries ────────────────────────────────────────

    async def what_do_top_players_use(
        self, slot: str, main_skill: str, ascendancy: str
    ) -> SlotMeta:
        """
        Direct query: "What do top Fireball Elementalists use in Body Armour?"
        """
        meta = await self.fetch_meta_snapshot(main_skill, ascendancy)
        return meta.slots.get(slot)

    async def find_similar_characters(
        self,
        main_skill: str,
        ascendancy: str,
        min_dps: float | None = None,
        has_item: str | None = None,
    ) -> list[dict]:
        """
        Find specific characters matching criteria.
        Useful for "show me someone doing what I want to do but better".
        
        Returns character names/accounts for the user to look up on poe.ninja.
        """
        ...

    async def get_progression_tiers(
        self, main_skill: str, ascendancy: str, slot: str
    ) -> list[dict]:
        """
        Show progression tiers for a slot:
        - Budget tier (1-5 divine): what to aim for
        - Mid tier (5-20 divine): next step up
        - High-end (20+ divine): best in slot
        
        Based on what players at different levels/DPS thresholds use.
        """
        ...

    # ─── Internal Methods ────────────────────────────────────────

    async def _fetch_builds_data(self) -> dict:
        """Fetch raw build overview from poe.ninja."""
        import aiohttp
        
        league_slug = self.league.league_id.lower().replace(" ", "-")
        url = "https://poe.ninja/api/data/0/getbuildoverview"
        params = {
            "overview": league_slug,
            "type": "exp",
            "language": "en",
        }
        
        async with aiohttp.ClientSession() as session:
            async with session.get(
                url,
                params=params,
                headers={"User-Agent": self._user_agent}
            ) as resp:
                return await resp.json()

    def _filter_characters(
        self, raw_data: dict, main_skill: str, ascendancy: str,
        min_level: int, top_n: int
    ) -> list[dict]:
        """
        Filter the raw poe.ninja data to characters matching our archetype.
        """
        # Find skill index
        skill_idx = None
        for i, skill in enumerate(raw_data.get("activeSkills", [])):
            if skill["name"].lower() == main_skill.lower():
                skill_idx = i
                break
        
        if skill_idx is None:
            return []
        
        # Find class index
        class_idx = None
        for i, name in enumerate(raw_data.get("classNames", [])):
            if name.lower() == ascendancy.lower():
                class_idx = i
                break
        
        # Get characters using this skill + class, filter by level
        matching = []
        skill_users = raw_data.get("activeSkillUse", {}).get(str(skill_idx), [])
        
        # poe.ninja encodes as delta-encoded indices
        current_idx = 0
        for i, delta in enumerate(skill_users):
            if i == 0:
                current_idx = delta
            else:
                current_idx += delta
            
            if current_idx < len(raw_data.get("classes", [])):
                char_class = raw_data["classes"][current_idx]
                char_level = raw_data.get("levels", [0])[current_idx]
                
                if char_class == class_idx and char_level >= min_level:
                    matching.append(current_idx)
        
        return matching[:top_n]

    def _analyse_patterns(
        self, character_indices: list[int], main_skill: str, ascendancy: str
    ) -> MetaSnapshot:
        """
        Analyse gear patterns across the filtered character set.
        Identify most common mods per slot, common bases, common uniques.
        """
        # For each equipment slot, tally:
        # - Base type frequency
        # - Mod frequency (which mods appear most)
        # - Unique usage
        ...

    def _compute_slot_gap(
        self, slot_name: str, your_item: dict | None,
        slot_meta: SlotMeta, meta: MetaSnapshot
    ) -> SlotGap | None:
        """
        Compare your item in a slot against what top players use.
        Score the gap based on:
        - How many key mods you're missing
        - Popularity of those mods (higher = more important)
        - Whether it's a unique slot (less upgrade potential)
        """
        if not slot_meta.key_mods:
            return None
        
        your_mods = set()  # Extract mod types from your item
        meta_mods = {m.mod_text for m in slot_meta.key_mods if m.occurrence_percent > 50}
        
        missing_mods = meta_mods - your_mods
        
        if not missing_mods:
            return None  # You already match the meta
        
        # Score: sum of occurrence_percent for missing mods
        score = sum(
            m.occurrence_percent
            for m in slot_meta.key_mods
            if m.mod_text in missing_mods
        ) / 100
        
        return SlotGap(
            slot=slot_name,
            priority_rank=0,  # Assigned later
            priority_score=score,
            your_item_summary=str(your_item),
            target_mods=[m for m in slot_meta.key_mods if m.mod_text in missing_mods],
            estimated_dps_gain=None,  # Filled by PoB sim later
            estimated_dps_gain_percent=None,
            meta_popularity=max(m.occurrence_percent for m in slot_meta.key_mods if m.mod_text in missing_mods),
            suggested_base=slot_meta.common_base_types[0][0] if slot_meta.common_base_types else None,
            notes=f"Top players ({meta.sample_size} analysed) commonly use these mods. You're missing {len(missing_mods)} key mods.",
        )
```

---

## Example Interaction

```
User: "I want more damage. Budget: 10 divine."

Agent:
┌─────────────────────────────────────────────────────────────────────┐
│ Build Gap Analysis: Fireball Elementalist (vs top 50 players)       │
│                                                                     │
│ Your DPS: 245,312                                                   │
│ Top 50 average DPS: 1,240,000                                       │
│                                                                     │
│ Priority upgrades:                                                   │
│                                                                     │
│ #1 BODY ARMOUR (biggest gap)                                         │
│    You have: Vaal Regalia (+100 life only)                           │
│    Meta uses (90%): +1 to Level of Fire Skill Gems,                  │
│                     70%+ Spell Damage, +90 Life                      │
│    Estimated DPS gain: +180,000 (+73%)                               │
│    Buy: ~8 divine | Craft: ~5 divine (Essence of Wrath spam)         │
│    → RECOMMEND: Craft using Shrieking Essence of Wrath               │
│                                                                     │
│ #2 WEAPON                                                            │
│    You have: Sceptre (+40 flat fire damage)                          │
│    Meta uses (85%): +1 Fire Gems, 80%+ Spell Damage, Cast Speed      │
│    Estimated DPS gain: +95,000 (+39%)                                │
│    Buy: ~15 divine (over budget) | Craft: ~12 divine (over budget)   │
│    → SKIP: Over budget. Revisit after body armour.                   │
│                                                                     │
│ Recommendation: Craft a body armour first.                           │
│ Recipe: [See crafting steps below]                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Progression Tiers

The agent can also show upgrade tiers so the user knows what to aim for at each budget level:

```python
@dataclass
class ProgressionTier:
    tier_name: str          # "Budget", "Mid", "High-end", "Mirror-tier"
    budget_range: tuple[float, float]  # (min_divine, max_divine)
    target_mods: list[str]
    example_item_text: str  # A representative item at this tier
    player_dps_range: tuple[float, float]  # DPS range of players at this tier
    percentage_of_meta: float  # What % of top players are at this tier
```

```
Progression for Body Armour (Fireball Elementalist):

Budget (1-5 divine):
  - +1 Fire Gems (Essence craft), +70 Life
  - Gets you to ~400k DPS
  - 15% of players at this tier

Mid (5-20 divine):
  - +1 Fire Gems, Spell Damage 70%+, +90 Life
  - Gets you to ~800k DPS  
  - 50% of players at this tier

High-end (20-50 divine):
  - +1 Fire Gems, Spell Damage 100%+, +100 Life, Fire Pen
  - Gets you to ~1.2M DPS
  - 30% of players at this tier

Mirror-tier (50+ divine):
  - Perfect rolls, additional influence mods
  - 5% of players at this tier
```

---

## Caching Strategy

| Data | Cache Duration | Reason |
|------|---------------|--------|
| poe.ninja build overview | 24 hours | Meta doesn't shift faster than daily |
| Filtered archetype analysis | 24 hours | Same data, just filtered |
| Individual character details | 7 days | Character gear changes infrequently |
| Progression tiers | 24 hours | Derived from build overview |

At ~50-100MB for the full build overview JSON, we fetch once per day and keep it in memory for instant filtering.

---

## Integration Points

| Module | How Meta Intelligence Uses It |
|--------|-------------------------------|
| **Build Session** | Reads your class, skill, equipped items |
| **PoB Engine** | Simulates DPS gain of proposed upgrades |
| **Pricing Engine** | Prices the upgrades (buy vs craft) |
| **Crafting Engine** (next) | Generates crafting recipes for target items |
| **League Context** | Ensures we're looking at the right league's meta |

---

## MCP Tool Exposure

| MCP Tool | Description |
|----------|-------------|
| `analyse_build_gaps(goal?)` | Run full gap analysis against meta |
| `get_upgrade_plan(budget, goal?)` | Concrete prioritised upgrade plan within budget |
| `what_do_top_players_use(slot)` | Query: "What do top players use in X slot?" |
| `show_progression_tiers(slot)` | Show budget → mid → high-end progression |
| `find_similar_builds(filters?)` | Find specific top characters to reference |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| poe.ninja builds API changes/breaks | No meta data | Cache aggressively (24h+); fall back to GGG ladder API for basic data |
| poe.ninja blocks automated access | No meta data | Respect rate limits, proper User-Agent, cache heavily |
| Small sample size for niche builds | Bad recommendations | Show sample size to user; warn if < 20 characters; widen filter (drop level req) |
| Meta is wrong (top by XP ≠ top by DPS) | Bad advice | Sort by DPS where available (skillDetails.dps), not just ladder rank |
| Player gear is private | Missing data | poe.ninja already handles this (only shows public characters) |
| Outdated meta after balance patch | Stale advice | Show fetch timestamp; user can force refresh; new league = new snapshot |

---

## Sprint Estimate

| Task | Effort |
|------|--------|
| poe.ninja builds API client + parsing | 2 days |
| Character filtering (by skill/class/level) | 1 day |
| Gear pattern analysis (mod frequency per slot) | 2 days |
| Gap analysis (compare user vs meta) | 1-2 days |
| Progression tiers derivation | 1 day |
| PoB simulation for estimated DPS gain | 1 day |
| Upgrade plan generation (combining all modules) | 1-2 days |
| Caching layer | 0.5 day |
| **Total** | **10-12 days** |
