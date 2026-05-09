# Sensor Enclosure Plans

Two enclosures for the Pico W sensor nodes. Designed to be 3D printed.
FreeCAD macros live in each project's `enclosure/` directory in pico_nix.

---

## pico-w-env-sensor — Outdoor Stevenson Screen

**Purpose:** Houses the BMP180 + BME280 + INA219/UPS-A stack outside.
Pending battery life tests before committing to a permanent outdoor mount.

### Design Principles (Stevenson Screen)
- **Louvered walls** on all four sides — 45° angled slats block direct solar
  radiation and rain while allowing free horizontal airflow across sensors.
- **Double roof** — air gap between two roof layers insulates interior from
  solar heating from above; assembled with a snap or screw-retained inner layer.
- **White/light colour** — reflects rather than absorbs radiation. Use white
  ASA or PETG filament; paint is a secondary option.
- **Open bottom** — traditional screens are open at the base; cool air rises
  through naturally. Mesh or coarse grid to keep insects out.
- **Raised mount** — 1–2 m off ground, away from reflected heat from concrete
  or tiles. Design a pole bracket or wall bracket into the base.

### Key Constraints
- BMP180 and BME280 must be in the direct airflow path, not tucked against a wall.
- INA219 / UPS-A generates some heat under load — mount downstream of the
  temp sensors in the airflow direction, or physically separate.
- USB-C access port with a drip lip for provisioning / firmware flashing.
- Cable entry for any future sensor additions.

### Dimensions (to finalise after measuring stack)
| Item | Measure before modelling |
|------|--------------------------|
| Pico W + UPS-A assembled height | Tallest component |
| Board footprint (Pico on UPS-A) | Sets minimum floor area |
| USB-C connector position | Port hole X/Y placement |
| Preferred mounting pole diameter | Base bracket bore |

### Material
- **ASA** — best choice; genuinely UV stable outdoors, handles temperatures
  up to ~100 °C. Needs enclosure or dry box on printer.
- **PETG** — fallback; UV degrades slowly, glass transition ~80 °C (fine for
  most climates, marginal in direct African sun on a dark day).
- **NOT PLA** — glass transition ~60 °C, will deform in direct sunlight.

### Print Settings
- 3–4 perimeters (structural walls need strength)
- 30–40 % infill
- White or light grey filament
- Consider UV-resistant clear coat spray if using PETG

### Open Design Questions
- [ ] Louvre angle: 45° standard, or steeper for heavy rain?
- [ ] Snap-fit lid vs screw retention (M3 brass inserts)?
- [ ] Single or double pole mount bracket?
- [ ] Internal standoff positions for Pico + UPS-A PCBs
- [ ] Desiccant pocket? (humidity sensor inside means probably not)

---

## pico-w-air-sensor — Indoor AQI Monitor

**Purpose:** Houses the PMSA003 particulate sensor + SSD1306 OLED indoors.
Always USB-powered. Aesthetics matter — this lives on a desk or wall.

### Design Principles
- **PMSA003 airflow is critical.** The sensor has an internal fan with a
  defined intake face and exhaust face. The enclosure must:
  - Leave ≥ 10 mm clearance and open grilles on both faces.
  - Put intake and exhaust on **opposite sides** to prevent recirculation.
  - Not trap particulates — needs a representative sample of room air.
- **Display window** for the SSD1306 (128 × 32 px, ~26 × 11 mm active area).
  Flush or slightly recessed; optional clear acrylic/PETG window for protection.
- **Landscape orientation** — display on front, grilles on left/right sides,
  USB-C entry at rear.
- **Wall-mount or tabletop stand** — design both options into the base.

### Dimensions (nominal, adjust after measuring)
| Component | Nominal size |
|-----------|-------------|
| PMSA003 body | 38 × 35 × 12 mm |
| SSD1306 board | ~35 × 13 mm |
| Pico W | 51 × 21 × 4 mm |
| Enclosure external | 115 × 70 × 50 mm |
| Wall thickness | 2.5 mm |

### FreeCAD Macro
`projects/pico-w-air-sensor/enclosure/pico_w_air_sensor_enclosure.FCMacro`

Parametric script generates:
- **Body** — hollow shell, hex grilles on left/right, display window on front,
  USB-C cutout on rear, 4 internal PCB mounting bosses (M2 self-tap).
- **Lid** — flat top plate with press-fit engagement flange (0.25 mm clearance).

### Material
- **PLA** — fine for indoor use. Any colour; two-colour print option for
  accent grille/front panel looks sharp.
- White body + coloured grille insert is a clean aesthetic.

### Print Settings
- 3 perimeters
- 20 % infill (aesthetic part, not structural)
- Supports: not needed if printed upright (open top)

### Open Design Questions
- [ ] Confirm PMSA003 intake/exhaust face positions from datasheet before
  finalising left/right grille placement.
- [ ] Acrylic window for display: laser cut or just leave open?
- [ ] Wall bracket: integrated clip or separate printed part?
- [ ] Cable management: channel in base for USB-C cable routing?
- [ ] Snap-fit lid or friction fit? (indoor = friction fine)
- [ ] PMSA003 mounting: datasheet hole positions → matching boss pattern inside

---

## Next Steps
1. Print test pieces for fit checks before committing to full prints.
2. Measure actual stacks (board heights, connector positions) and update
   dimensions in the FreeCAD macros.
3. Battery life test for env-sensor before finalising outdoor enclosure.
4. Consider a shared mounting system if both end up wall-mounted.
