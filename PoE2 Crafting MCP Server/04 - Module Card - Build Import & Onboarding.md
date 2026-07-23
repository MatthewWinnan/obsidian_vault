# Module Card: Build Import & Onboarding

## Overview

Handles the initial onboarding flow: getting the user's current PoE2 character into the system so the crafting advisor can start making recommendations. The primary input is a PoB pastebin code. Future versions will support direct account import.

---

## User Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    FIRST-TIME ONBOARDING                         │
│                                                                 │
│  ┌──────────────────┐     ┌──────────────────┐                 │
│  │  Path of Exile 2 │     │    PoB-PoE2       │                 │
│  │  (game running)  │     │  (desktop app)    │                 │
│  └────────┬─────────┘     └────────┬──────────┘                 │
│           │                        │                            │
│           │  1. User opens PoB     │                            │
│           │     Import tab         │                            │
│           │                        │                            │
│           │  2. Enters account     │                            │
│           │     name, selects      │                            │
│           │     character          ▼                            │
│           │               ┌────────────────────┐                │
│           │               │ PoB imports from    │                │
│           │               │ GGG API (built-in)  │                │
│           │               │                    │                │
│           │               │ • Passive tree     │                │
│           │               │ • Equipment        │                │
│           │               │ • Skills           │                │
│           │               │ • Jewels           │                │
│           │               └────────┬───────────┘                │
│           │                        │                            │
│           │  3. User clicks        │                            │
│           │     "Generate" share   │                            │
│           │     code               │                            │
│           │                        ▼                            │
│           │               ┌────────────────────┐                │
│           │               │ Share code copied   │                │
│           │               │ eNq1Wdtu2zgQf...   │                │
│           │               └────────┬───────────┘                │
│           │                        │                            │
│           │  4. User pastes code   │                            │
│           │     to MCP agent       │                            │
│           │                        ▼                            │
│           │               ┌────────────────────┐                │
│           │               │ MCP Agent loads     │                │
│           │               │ build, ready to     │                │
│           │               │ advise              │                │
│           │               └────────────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Supported Import Methods

### Method 1: PoB Share Code (v1 — Primary)

**What the user does:**
1. Open PoB-PoE2
2. Import/Export tab → enter account name → import character
3. Click "Generate" → copy the share code
4. Paste to the MCP agent

**What the code is:**
- A base64-encoded, deflate-compressed XML document
- Contains: passive tree, items, skills, config, notes
- Self-contained — no external calls needed to load it

**Agent response after loading:**
```
Loaded build: "HellFireWitch" (Level 87 Elementalist)
- Main skill: Fireball (245,312 DPS)
- Life: 4,230 | ES: 1,890
- Resistances: Fire 75% | Cold 75% | Lightning 68% | Chaos 22%
- Issues detected: Lightning res uncapped (-7%), Chaos res low

What would you like to improve?
```

### Method 2: PoB XML Paste (v1 — Fallback)

If the user has raw XML (exported from PoB as file):
- Paste the XML directly
- Useful for sharing builds that aren't uploaded to pastebin

### Method 3: Account Import (v2 — Future)

When GGG OAuth registration reopens:
```
User: "Import my character FrostNova from account MatthewPoE"
Agent: *authenticates via OAuth, pulls character JSON, loads into PoB*
```

### Method 4: Item-Text-Only Mode (v1 — Lightweight)

For users who just want crafting advice without a full build:
```
User: "I need a body armour with +100 life and T1 fire res for my 
       ilvl 86 Vaal Regalia. Budget: 5 divine."
Agent: *doesn't need full build — just runs crafting probability + pricing*
```

This skips DPS simulation but still gives valid crafting advice.

---

## Python API Design

```python
from dataclasses import dataclass
from enum import Enum


class ImportMethod(Enum):
    POB_CODE = "pob_code"       # Base64 share code
    POB_XML = "pob_xml"         # Raw XML text
    API_JSON = "api_json"       # GGG API JSON (future)
    ITEM_ONLY = "item_only"     # No build, just item/crafting context


@dataclass
class BuildSummary:
    """Quick overview returned after successful import."""
    name: str
    level: int
    class_name: str
    ascendancy: str
    main_skill: str
    total_dps: float
    life: float
    energy_shield: float
    mana: float
    fire_res: float
    cold_res: float
    lightning_res: float
    chaos_res: float
    issues: list[str]  # Auto-detected problems


class BuildImporter:
    """
    Handles importing builds from various formats into the PoB engine.
    """

    def __init__(self, pob_engine: "PoBEngine"):
        self.pob = pob_engine

    def import_pob_code(self, code: str) -> BuildSummary:
        """
        Import from a PoB share code (base64-encoded compressed XML).
        
        Args:
            code: The share code string (e.g., "eNq1Wdtu2zgQ...")
        
        Returns:
            BuildSummary with key stats and any detected issues.
        """
        xml = self._decode_pob_code(code)
        return self.import_pob_xml(xml)

    def import_pob_xml(self, xml: str, name: str = "Imported Build") -> BuildSummary:
        """
        Import from raw PoB XML.
        """
        stats = self.pob.load_build_xml(xml, name)
        return self._create_summary(stats)

    def import_from_api(self, items_json: str, passives_json: str) -> BuildSummary:
        """
        Import from GGG API character JSON (future — requires OAuth).
        """
        stats = self.pob.load_build_from_api(items_json, passives_json)
        return self._create_summary(stats)

    def _decode_pob_code(self, code: str) -> str:
        """
        Decode PoB share code → XML.
        
        Format: base64url encoded → zlib decompress → XML string
        """
        import base64
        import zlib

        # PoB uses URL-safe base64 with possible padding
        # Handle both with and without padding
        code = code.strip()
        padding = 4 - len(code) % 4
        if padding != 4:
            code += "=" * padding

        raw = base64.urlsafe_b64decode(code)
        xml = zlib.decompress(raw).decode("utf-8")
        return xml

    def _create_summary(self, stats: "BuildStats") -> BuildSummary:
        """Create a user-friendly summary with auto-detected issues."""
        issues = []

        # Check resistance caps (75% for ele, varies for chaos)
        if stats.fire_res < 75:
            issues.append(f"Fire resistance uncapped ({stats.fire_res:.0f}%, need 75%)")
        if stats.cold_res < 75:
            issues.append(f"Cold resistance uncapped ({stats.cold_res:.0f}%, need 75%)")
        if stats.lightning_res < 75:
            issues.append(f"Lightning resistance uncapped ({stats.lightning_res:.0f}%, need 75%)")
        if stats.chaos_res < 0:
            issues.append(f"Chaos resistance negative ({stats.chaos_res:.0f}%)")

        # Check life/ES
        if stats.total_life < 3000 and stats.total_es < 3000:
            issues.append("Very low effective HP — consider defensive upgrades")

        # TODO: More heuristics (accuracy, hit chance, etc.)

        return BuildSummary(
            name=self._get_build_name(),
            level=self._get_character_level(),
            class_name=self._get_class(),
            ascendancy=self._get_ascendancy(),
            main_skill=self._get_main_skill_name(),
            total_dps=stats.total_dps,
            life=stats.total_life,
            energy_shield=stats.total_es,
            mana=stats.total_mana,
            fire_res=stats.fire_res,
            cold_res=stats.cold_res,
            lightning_res=stats.lightning_res,
            chaos_res=stats.chaos_res,
            issues=issues,
        )

    def _get_build_name(self) -> str:
        """Extract build/character name from loaded PoB data."""
        # Access from PoB's internal build object
        ...

    def _get_character_level(self) -> int:
        ...

    def _get_class(self) -> str:
        ...

    def _get_ascendancy(self) -> str:
        ...

    def _get_main_skill_name(self) -> str:
        ...
```

---

## Build State Management

The agent needs to track the build across a session (multiple questions/comparisons):

```python
class BuildSession:
    """
    Manages the current build state across an advisor session.
    
    Supports:
    - Loading a build
    - Snapshotting before item swaps
    - Reverting to previous state
    - Tracking what the user has already tried
    """

    def __init__(self, pob_engine: "PoBEngine"):
        self.pob = pob_engine
        self.importer = BuildImporter(pob_engine)
        self._original_xml: str | None = None
        self._current_summary: BuildSummary | None = None
        self._history: list[str] = []  # Previous build XMLs for undo

    @property
    def is_loaded(self) -> bool:
        return self._current_summary is not None

    @property
    def summary(self) -> BuildSummary:
        if not self.is_loaded:
            raise ValueError("No build loaded. Please import a build first.")
        return self._current_summary

    def load(self, code_or_xml: str) -> BuildSummary:
        """
        Load a build. Auto-detects format (share code vs raw XML).
        """
        if code_or_xml.strip().startswith("<?xml") or code_or_xml.strip().startswith("<PathOfBuilding"):
            summary = self.importer.import_pob_xml(code_or_xml)
        else:
            summary = self.importer.import_pob_code(code_or_xml)

        self._original_xml = code_or_xml
        self._current_summary = summary
        self._history = []
        return summary

    def snapshot(self) -> str:
        """Save current state for later restore."""
        # Export current build as XML from PoB
        ...

    def restore(self, snapshot: str) -> BuildSummary:
        """Restore to a previous snapshot."""
        ...

    def reset_to_original(self) -> BuildSummary:
        """Revert all changes, back to the originally imported build."""
        return self.load(self._original_xml)
```

---

## MCP Tool Exposure

| MCP Tool | Description | Requires Build? |
|----------|-------------|-----------------|
| `import_build(code)` | Load a PoB share code or XML | No (this creates the build) |
| `get_build_summary()` | Show current build stats & issues | Yes |
| `reset_build()` | Revert to originally imported state | Yes |

---

## Error Handling

| Error | Cause | User Message |
|-------|-------|-------------|
| Invalid share code | Corrupted/truncated paste | "That doesn't look like a valid PoB share code. Make sure you copied the full string from PoB's Import/Export tab." |
| Decode fails | Wrong encoding/compression | "Couldn't decode the build. Try exporting fresh from PoB-PoE2 (not the PoE1 version)." |
| No main skill | Build has no active skills configured | "Build loaded but no main skill is selected. DPS will show as 0. You can still get crafting advice." |
| PoE1 build | User pasted a PoE1 build by mistake | "This looks like a PoE1 build. This tool works with PoE2 — make sure you're using Path of Building PoE2." |

---

## Sprint Estimate

| Task | Effort |
|------|--------|
| PoB code decode (base64 + zlib) | 0.5 day |
| BuildImporter + BuildSummary | 1 day |
| Auto-issue detection heuristics | 0.5 day |
| BuildSession state management | 1 day |
| MCP tool wiring | 0.5 day |
| Error handling + user messages | 0.5 day |
| **Total** | **4 days** |
