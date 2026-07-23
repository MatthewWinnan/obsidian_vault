# Module Card: Item Data Format & Internal Structure

## Overview

Defines the canonical item format used throughout the MCP server — how items are represented internally, how they map to PoB's format, how they're stored in the DB, and how to convert between the various representations (in-game text, PoB raw, trade API JSON, internal dataclass).

---

## PoB Raw Text Format (The Source of Truth)

This is the format PoB uses internally and what `CreateDisplayItemFromRaw()` accepts. It's also what the game produces when you press `Ctrl+C` on an item (with minor differences).

### Example: Rare Gloves (from Whirling Trinity build)

```
Rarity: RARE
Dusk Grasp
Fists of Stone
Unique ID: 64e9cac136195f086390354a4dda392fd33ff285ad972beb504062cc0ecc19c7
Item Level: 82
Quality: 20
Sockets: S
Rune: Emergent Possibility
LevelReq: 60
Implicits: 5
{enchant}{rune}Bonded: 20% increased Elemental Damage
{enchant}{rune}Gain 1% of Damage as Extra Damage of a random Element per
{enchant}{rune}Rune Socketed in Equipped Items
Has +3 to Evasion Rating per player level
Has +1 to maximum Energy Shield per player level
Attacks Gain 21% of Damage as Extra Fire Damage
Attacks Gain 17% of Damage as Extra Cold Damage
+2% to Maximum Lightning Resistance
+39% to Lightning Resistance
+2.5% to Critical Hit Chance
7% chance to gain a Power Charge on Critical Hit
Attacks Gain 21% of Damage as Extra Lightning Damage
```

### Format Rules

```
Line 1:  "Rarity: NORMAL|MAGIC|RARE|UNIQUE"
Line 2:  Item name (for rares/uniques) OR base name (for normal/magic)
Line 3:  Base type name (only if Line 2 is a custom name)
         Optional metadata lines (any order):
           "Unique ID: <hash>"
           "Item Level: <int>"
           "Quality: <int>"
           "Sockets: S S S" (space-separated, S = socket)
           "Rune: <rune name>"
           "LevelReq: <int>"
           "Implicits: <count>"    ← number of implicit lines that follow
         Implicit mod lines (count from Implicits header):
           Can be prefixed with {tags} like {enchant}{rune}
         Explicit mod lines (everything after implicits):
           Plain text, one mod per line
         Status lines (at end):
           "Corrupted"
           "Mirrored"
           "Sanctified"
```

### Minimal White Item (No Mods)

```
New Item
Fists of Stone
```

Just the name marker + base type. PoB fills in defaults.

### Item with Mods

```
New Item
Fists of Stone
+100 to maximum Life
+40% to Fire Resistance
+30% to Cold Resistance
```

---

## PoB Internal Object Fields

When PoB parses an item, it creates a Lua table with these fields:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | "Dusk Grasp, Fists of Stone" |
| `baseName` | string | "Fists of Stone" |
| `rarity` | string | "NORMAL", "MAGIC", "RARE", "UNIQUE" |
| `type` | string | Slot type: "Gloves", "Body Armour", "Helmet", etc. |
| `itemLevel` | int | Item level (affects mod pool) |
| `quality` | int | Quality percentage (0-20) |
| `id` | int | Internal PoB item ID |
| `corrupted` | bool/nil | True if corrupted |
| `fractured` | bool/nil | True if fractured |
| `implicitModLines` | table | Array of {line=str} |
| `explicitModLines` | table | Array of {line=str, modTags=table, crafted=bool, fractured=bool} |
| `enchantModLines` | table | Array of {line=str} (rune bonuses, enchants) |
| `sockets` | table | Array of socket definitions |
| `runes` | table | Array of rune names (e.g., "Emergent Possibility") |
| `raw` | string | The original raw text used to create the item |
| `base` | table | Reference to base type data (base.type = slot) |

---

## Our Internal Python Dataclass

```python
from dataclasses import dataclass, field
from enum import Enum


class Rarity(Enum):
    NORMAL = "NORMAL"
    MAGIC = "MAGIC"
    RARE = "RARE"
    UNIQUE = "UNIQUE"


class SlotType(Enum):
    WEAPON_1 = "Weapon 1"
    WEAPON_2 = "Weapon 2"
    HELMET = "Helmet"
    BODY_ARMOUR = "Body Armour"
    GLOVES = "Gloves"
    BOOTS = "Boots"
    AMULET = "Amulet"
    RING_1 = "Ring 1"
    RING_2 = "Ring 2"
    BELT = "Belt"


@dataclass
class ItemMod:
    """A single modifier on an item."""
    text: str                          # "Attacks Gain 21% of Damage as Extra Fire Damage"
    is_implicit: bool = False
    is_enchant: bool = False
    is_crafted: bool = False
    is_fractured: bool = False
    is_rune: bool = False
    # Parsed values (optional, for crafting engine use)
    mod_group: str | None = None       # e.g., "ExtraFireDamage"
    tier: int | None = None            # e.g., 1
    value: float | None = None         # e.g., 21.0


@dataclass
class Item:
    """
    Internal item representation.
    Can be converted to/from PoB raw text, trade API JSON, or DB row.
    """
    # Identity
    name: str                          # "Dusk Grasp" or "" for unnamed
    base_type: str                     # "Fists of Stone"
    rarity: Rarity = Rarity.NORMAL
    slot_type: SlotType | None = None  # Inferred from base type

    # Metadata
    item_level: int = 1
    quality: int = 0
    level_req: int = 0

    # Mods
    implicit_mods: list[ItemMod] = field(default_factory=list)
    explicit_mods: list[ItemMod] = field(default_factory=list)
    enchant_mods: list[ItemMod] = field(default_factory=list)

    # Sockets & Runes
    socket_count: int = 0
    runes: list[str] = field(default_factory=list)

    # Status
    corrupted: bool = False
    fractured: bool = False
    mirrored: bool = False
    sanctified: bool = False

    # Raw text (for direct PoB interaction)
    raw_text: str | None = None

    def to_pob_text(self) -> str:
        """Convert to PoB raw text format for CreateDisplayItemFromRaw()."""
        lines = []

        # Rarity
        lines.append(f"Rarity: {self.rarity.value}")

        # Name + base
        if self.rarity in (Rarity.RARE, Rarity.UNIQUE) and self.name:
            lines.append(self.name)
        lines.append(self.base_type)

        # Metadata
        if self.item_level > 1:
            lines.append(f"Item Level: {self.item_level}")
        if self.quality > 0:
            lines.append(f"Quality: {self.quality}")
        if self.socket_count > 0:
            lines.append(f"Sockets: {' '.join(['S'] * self.socket_count)}")
        for rune in self.runes:
            lines.append(f"Rune: {rune}")
        if self.level_req > 0:
            lines.append(f"LevelReq: {self.level_req}")

        # Implicits
        all_implicits = self.implicit_mods + self.enchant_mods
        if all_implicits:
            lines.append(f"Implicits: {len(all_implicits)}")
            for mod in self.enchant_mods:
                prefix = "{enchant}"
                if mod.is_rune:
                    prefix += "{rune}"
                lines.append(f"{prefix}{mod.text}")
            for mod in self.implicit_mods:
                lines.append(mod.text)

        # Explicit mods
        for mod in self.explicit_mods:
            prefix = ""
            if mod.is_crafted:
                prefix = "{crafted}"
            if mod.is_fractured:
                prefix = "{fractured}"
            lines.append(f"{prefix}{mod.text}")

        # Status
        if self.corrupted:
            lines.append("Corrupted")
        if self.mirrored:
            lines.append("Mirrored")
        if self.sanctified:
            lines.append("Sanctified")

        return "\n".join(lines)

    @classmethod
    def from_pob_text(cls, raw_text: str) -> "Item":
        """Parse PoB raw text into an Item dataclass."""
        lines = raw_text.strip().split("\n")
        item = cls(name="", base_type="")
        item.raw_text = raw_text

        # Parse rarity
        if lines[0].startswith("Rarity:"):
            rarity_str = lines[0].split(":")[1].strip()
            item.rarity = Rarity(rarity_str)
            lines = lines[1:]

        # Parse name/base
        if item.rarity in (Rarity.RARE, Rarity.UNIQUE):
            item.name = lines[0]
            item.base_type = lines[1] if len(lines) > 1 else ""
            lines = lines[2:]
        else:
            item.base_type = lines[0]
            lines = lines[1:]

        # Parse remaining lines
        implicit_count = 0
        reading_implicits = False
        implicits_read = 0

        for line in lines:
            if line.startswith("Item Level:"):
                item.item_level = int(line.split(":")[1].strip())
            elif line.startswith("Quality:"):
                item.quality = int(line.split(":")[1].strip())
            elif line.startswith("LevelReq:"):
                item.level_req = int(line.split(":")[1].strip())
            elif line.startswith("Sockets:"):
                item.socket_count = line.count("S")
            elif line.startswith("Rune:"):
                item.runes.append(line.split(":", 1)[1].strip())
            elif line.startswith("Implicits:"):
                implicit_count = int(line.split(":")[1].strip())
                reading_implicits = True
            elif line == "Corrupted":
                item.corrupted = True
            elif line == "Mirrored":
                item.mirrored = True
            elif line == "Sanctified":
                item.sanctified = True
            elif reading_implicits and implicits_read < implicit_count:
                mod = ItemMod(text=line.lstrip("{enchant}{rune}"))
                if "{enchant}" in line:
                    mod.is_enchant = True
                if "{rune}" in line:
                    mod.is_rune = True
                if mod.is_enchant:
                    item.enchant_mods.append(mod)
                else:
                    item.implicit_mods.append(mod)
                implicits_read += 1
                if implicits_read >= implicit_count:
                    reading_implicits = False
            elif not line.startswith("Unique ID:"):
                # Explicit mod
                mod = ItemMod(text=line)
                if line.startswith("{crafted}"):
                    mod.is_crafted = True
                    mod.text = line.replace("{crafted}", "")
                if line.startswith("{fractured}"):
                    mod.is_fractured = True
                    mod.text = line.replace("{fractured}", "")
                item.explicit_mods.append(mod)

        return item

    @classmethod
    def white_base(cls, base_type: str, item_level: int = 86) -> "Item":
        """Create a white (normal) base item for crafting."""
        return cls(
            name="New Item",
            base_type=base_type,
            rarity=Rarity.NORMAL,
            item_level=item_level,
        )
```

---

## DB Schema for Items

```sql
CREATE TABLE items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    -- Identity
    name TEXT,
    base_type TEXT NOT NULL,
    rarity TEXT NOT NULL DEFAULT 'NORMAL',
    slot_type TEXT,
    -- Metadata
    item_level INTEGER DEFAULT 1,
    quality INTEGER DEFAULT 0,
    level_req INTEGER DEFAULT 0,
    -- Status
    corrupted INTEGER DEFAULT 0,
    fractured INTEGER DEFAULT 0,
    mirrored INTEGER DEFAULT 0,
    sanctified INTEGER DEFAULT 0,
    -- Sockets
    socket_count INTEGER DEFAULT 0,
    runes TEXT,                  -- JSON array of rune names
    -- Mods (stored as JSON arrays)
    implicit_mods TEXT,          -- JSON: [{"text": "...", "mod_group": "...", "tier": 1}]
    explicit_mods TEXT,          -- JSON: [{"text": "...", "crafted": false, "fractured": false}]
    enchant_mods TEXT,           -- JSON: [{"text": "...", "is_rune": true}]
    -- Raw text (for PoB interaction)
    raw_text TEXT,
    -- Source tracking
    source TEXT,                 -- 'build_import', 'trade_search', 'crafted', 'manual'
    created_at REAL
);
```

---

## Format Conversions

```
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│  In-Game Text    │──parse──▶  Item dataclass   │──to_pob──▶  PoB Raw Text   │
│  (Ctrl+C)       │◀─────────│                  │◀────────│  (for engine)    │
└──────────────────┘        └────────┬─────────┘        └──────────────────┘
                                     │
                              to_json / from_json
                                     │
                            ┌────────▼─────────┐
                            │  SQLite / JSON   │
                            │  (for storage)   │
                            └──────────────────┘
                                     │
                              from_trade_api
                                     │
                            ┌────────▼─────────┐
                            │  Trade API JSON  │
                            │  (GGG format)    │
                            └──────────────────┘
```

### In-Game Text → Item

The in-game Ctrl+C format is nearly identical to PoB raw text, with minor differences:
- In-game adds `--------` separator lines between sections
- In-game shows computed properties (Armour: 534, Energy Shield: 230)
- PoB strips separators and computed props, keeps only mods

### Trade API JSON → Item

The GGG trade API returns items as JSON with fields like:
```json
{
  "name": "Dusk Grasp",
  "typeLine": "Fists of Stone",
  "ilvl": 82,
  "rarity": "Rare",
  "explicitMods": ["Attacks Gain 21% of Damage as Extra Fire Damage", ...],
  "implicitMods": ["Has +3 to Evasion Rating per player level", ...]
}
```

---

## Slot Mapping

| PoB Slot Name | Item Base Categories |
|---------------|---------------------|
| `Weapon 1` | Swords, Axes, Maces, Bows, Staves, Wands, Sceptres, Daggers |
| `Weapon 2` | Shields, Quivers, or dual-wield second weapon |
| `Helmet` | Helmets |
| `Body Armour` | Body Armours |
| `Gloves` | Gloves |
| `Boots` | Boots |
| `Amulet` | Amulets |
| `Ring 1` | Rings |
| `Ring 2` | Rings |
| `Belt` | Belts |

---

## Usage in the MCP Server

```python
# Creating an item for PoB simulation:
item = Item.white_base("Fists of Stone", item_level=86)
item.explicit_mods = [
    ItemMod(text="+100 to maximum Life"),
    ItemMod(text="+40% to Fire Resistance"),
]
pob_text = item.to_pob_text()
# Feed to PoB: lua.execute(f'build.itemsTab:CreateDisplayItemFromRaw([[{pob_text}]])')

# Parsing an item from a user's Ctrl+C paste:
user_paste = "Rarity: RARE\nDusk Grasp\nFists of Stone\n..."
item = Item.from_pob_text(user_paste)
print(item.explicit_mods)  # [ItemMod(text="Attacks Gain 21%..."), ...]

# Storing in DB:
db.insert_item(item)

# Retrieving and feeding to PoB:
item = db.get_item(id=42)
lua.execute(f'build.itemsTab:CreateDisplayItemFromRaw([[{item.to_pob_text()}]])')
```
