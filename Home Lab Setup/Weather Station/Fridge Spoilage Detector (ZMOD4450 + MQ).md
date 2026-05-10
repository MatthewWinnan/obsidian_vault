# Fridge Spoilage Detector

A smart fridge monitor combining the ZMOD4450 refrigerant sensor with MQ gas sensors and CO2/temperature monitoring to detect food spoilage early and flag fridge failures.

---

## Science — What Spoilage Actually Produces

Microbial and chemical food spoilage releases a characteristic cocktail of gases:

| Gas | Biological source | Food types | Sensor |
|-----|------------------|-----------|--------|
| **H2S** | Sulfur amino acid breakdown (cysteine, methionine) | Meat, eggs, seafood, dairy | **MQ-136** |
| **NH3** | Amino acid deamination, microbial protein catabolism | Meat, fish, dairy, aged cheese | **MQ-135** |
| **Trimethylamine (TMA)** | Bacterial TMAO reduction (fish-specific) | Seafood | MQ-135 (partial) |
| **Ethanol** | Yeast fermentation | Fruit, bread, carbs, soft drinks | **MQ-3** |
| **Esters / aldehydes** | Fat oxidation (rancidity), overripe fruit | Oils, nuts, fruit | ZMOD4450 VOC |
| **Acetic acid** | Aerobic bacteria | Dairy, produce | MQ-135 (partial) |
| **CO2** | All microbial respiration — universal spoilage marker | Everything | **SCD40** (NDIR) |

**H2S is the most reliable primary indicator** — it is specific to protein decomposition and appears early. CO2 is the most universal — produced by all microbial activity regardless of food type, does not suffer cold-temperature calibration issues.

---

## MQ-136 — Hydrogen Sulfide Sensor

Target gas: **H2S (hydrogen sulfide)** — "rotten egg" compound.

- Detection range: 1–200 ppm
- Cross-sensitive to: mercaptans, SO2, other organic sulfides (all spoilage-relevant)
- Standard MQ module — 5V heater, analog voltage output
- More selective for H2S than MQ-135; the two complement each other
- Also relevant for: sewage safety, petroleum sour gas, volcanic monitoring, agricultural slurry pits
- H2S is acutely toxic: 50 ppm causes olfactory fatigue, 150+ ppm life-threatening within minutes — secondary use as a safety sensor in enclosed spaces

---

## ZMOD4450 — Dual Role

The ZMOD4450 MOX micro-heater responds to the VOC cocktail of spoilage regardless of the refrigerant algorithm layer — the oxide reacts; the algorithm only classifies. Two useful outputs:

1. **Raw resistance baseline shift** — broad VOC indicator; rises with esters, aldehydes, alcohols
2. **Refrigerant mode** — if triggered, indicates the fridge is leaking coolant, explaining *why* food is spoiling (compressor failure → temperature rise → accelerated microbial growth)

---

## Temperature Problem — MQ Sensors in a Cold Fridge

MQ sensors are characterised at 20°C ±2°C, 65% RH. At 3–5°C inside a fridge the heater still maintains oxide operating temperature but **baseline resistance increases significantly** and sensitivity curves shift. Absolute ppm readings will be wrong.

**Strategy: relative change from personal baseline, not absolute concentration.**
After a clean-out with no food for 24 hours, record each sensor's clean-air reading as baseline. Alert when channels rise substantially above that reference. Never rely on absolute ppm thresholds from the datasheet at fridge temperatures.

---

## Sensor Placement Options

| Option | Pros | Cons |
|--------|------|------|
| Inside fridge, rear wall | Direct air sampling | Cold → shifted baselines; WiFi through metal body |
| Inside fridge door seal | Less cold (~8–12°C) | Partial air sampling only |
| **Small tube through door seal, sensors outside** | Sensors at room temp (correct calibration), WiFi easy | Gas diffusion delay; tube may ice if poorly sealed |
| Dedicated drawer/crisper compartment | Concentrates odours | Only covers one compartment |

**Recommended: small silicone tube (4–6 mm) through the door seal with a small 5V fan** to actively pull air past the sensors on a timed cycle. Sensors sit at room temperature; sampled air is drawn from inside the fridge. Fan runs for 60–90 s per sample cycle then stops.

---

## Recommended Sensor Stack

```
Detection:
  MQ-136    H2S — primary spoilage marker (meat/eggs/seafood/dairy)
  MQ-135    NH3, sulfides — secondary (meat/fish/dairy)
  MQ-3      Ethanol — fermentative spoilage (fruit/carbs)
  ZMOD4450  Broad VOC baseline shift + refrigerant leak detection

Context:
  SCD40     CO2 (NDIR) — universal microbial respiration marker
            Works correctly at fridge temperature, no calibration shift
  SHT40     Internal fridge T/RH (or TMP117 + SHT45 if WMO-grade)
  TMP117    External ambient T — differential monitoring
```

CO2 via SCD40 is the highest single addition if budget is limited. It is the only sensor that:
- Works correctly at fridge temperature without calibration adjustment
- Responds to all spoilage types regardless of food
- Is not confused by normal food odours (garlic, cheese, fermented foods all smell but CO2 only rises with active microbial growth)

---

## HA Integration

### Spoilage risk score

Combine normalised rise-above-baseline for each channel:

```
spoilage_risk = (
    (H2S_ratio   - 1.0) × 0.40 +   # most specific marker
    (NH3_ratio   - 1.0) × 0.25 +
    (CO2_delta        ) × 0.20 +   # ppm above baseline
    (VOC_ratio   - 1.0) × 0.15
) × 100   # scale to 0–100
```

Where `X_ratio = current_reading / clean_baseline`.

### Automation triggers

| Condition | Alert |
|-----------|-------|
| `spoilage_risk > 60` | "Something may be off — check food" |
| `spoilage_risk > 85` | "High spoilage risk — act now" |
| Fridge temp > 7°C for > 2 h | "Fridge unsafe — check door/compressor" |
| ZMOD4450 refrigerant flag | "Possible refrigerant leak — fridge failing" |
| Ethanol spike, H2S/NH3 flat | "Fruit fermenting or open alcohol — probably fine" |
| CO2 rise only, no gas spike | "Produce aging — check best-before dates" |

The 7°C / 2-hour threshold is the food safety standard danger zone boundary (4–60°C is the danger zone; 2 hours is the standard threshold before significant microbial risk accumulates).

---

## Ventilation — Getting Fridge Air to External Sensors

### Why sensors sit outside the fridge

MOX sensors (MQ series, ZMOD4450, Fermion MEMS) need no laser chamber or controlled airflow — they work by direct chemical contact between gas molecules and a heated oxide surface. However, placing them inside the fridge causes two problems:

1. **Cold-temperature calibration shift** — MQ sensors are characterised at 20°C ±2°C. At 3–5°C inside a fridge the baseline resistance increases significantly and sensitivity curves shift. Absolute ppm readings become unreliable.
2. **WiFi signal attenuation** — the metal fridge body acts as a Faraday cage.

Solution: sensors outside, fridge air piped through a tube.

---

### Ventilation Options

#### Option A — Passive Diffusion Tube (simplest, slowest)

Gas diffuses from inside to outside along the concentration gradient.

```
[Fridge interior]
      │
   [mesh inlet filter — keeps food particles out, mid-height rear wall]
      │
   [silicone tube, 4–6 mm ID, through door seal]
      │
   [PTFE membrane at sensor end — blocks condensation droplets]
      │
[Sensor chamber at room temperature]
```

- No power, no moving parts
- Response time: 5–15 minutes depending on tube length and diameter
- Adequate for spoilage (slow process) — not for fast transient events
- Condensate risk: slope tube so any condensate drains back into the fridge, not onto the sensor

#### Option B — Active Fan Sampling (recommended)

A small 5V fan outside the fridge pulls air through the tube on a timed cycle. Fan runs 60–90 s before each read to flush the tube and sensor chamber with fresh fridge air, then stops.

```
[Fridge interior — mesh inlet, mid-height rear]
      │
   [4–6 mm silicone or PTFE tube, as short as practical]
      │
   [sensor chamber — MQ sensors, ZMOD4450, SCD40]
      │
   [5V 40 mm fan — pulls air through, exhausts to room air]
```

Fan sits outside = motor at room temperature (longer life, no cold-weather lubrication issues). Pulling rather than pushing creates slight negative pressure in the tube — any seal leak draws room air in rather than fridge air escaping.

**Response time: 30–90 seconds per flush cycle.**

Fan control sketch:
```c
gpio_put(FAN_PIN, 1);   // Fan on — flush tube + sensor chamber
sleep_ms(60000);         // 60 s flush
mq_sensors_read();       // Read while fan still running
gpio_put(FAN_PIN, 0);   // Fan off — save power between cycles
```

#### Option C — SCD40 Inside, MQ Sensors Outside

The SCD40 (NDIR CO2) has no heater-temperature calibration problem and is sealed enough for fridge conditions. Place it inside on the rear wall, run only a thin I2C cable through the door seal. MQ and ZMOD sensors remain outside with the fan-tube setup. CO2 gives fast inside readings; MQ sensors give slower but more specific chemical identity.

---

### Door Seal Tube Routing

Route the tube without permanent modification:

- **Hinge side** — the hinge-side seal compresses less than the latch side; a natural gap to run a small tube without distorting the seal
- **Bottom lip** — where the door seal meets the floor lip; a natural flex point
- A 4 mm silicone tube is small enough to pass without breaking the thermal seal significantly — verify door still seals by checking a strip of paper held against the seal after routing

**Inlet position inside the fridge:** mid-height, rear wall — where cold air circulation from the evaporator passes. Avoid the very bottom (coldest, condensation pool) and the door area (warmest, most affected by door opening).

---

### Condensation Management

Cold humid fridge air (~90% RH, ~3–5°C) travelling into ~22°C room air reaches the dew point somewhere in the tube.

| Fix | How |
|-----|-----|
| Slope tube downward into fridge | Condensate drains back in, not onto sensor |
| PTFE tube instead of silicone | Hydrophobic inner surface — condensate beads and drains |
| PTFE membrane at sensor inlet | Passes gas, blocks liquid droplets reaching the oxide |
| Short tube | Less thermal gradient, less condensation |
| Fan flush before reading | Clears accumulated condensate by airflow |

---

### Do MOX Sensors Need a Chamber Like the PMSA003?

No — the PMSA003 needs its internal fan and laser chamber because **optical particle counting requires controlled laminar flow** to present particles one at a time in a defined beam volume. MOX sensors work by chemical contact only — no precision airflow is needed for the sensor itself.

However, the **sensor enclosure** still needs design for:

| Concern | Solution |
|---------|----------|
| Slow diffusion response in sealed box | Ventilation ports or active fan |
| Dust on oxide surface over time | PTFE membrane or mesh over inlet |
| Liquid water ingress | PTFE membrane |
| Heater self-heating of enclosure air | Vented enclosure — must breathe |
| Post-event recovery flush | Fan flush cycle |
| MQ heater warming adjacent sensors | Space sensors, direct airflow across all |

**Practical enclosure for the external sensor chamber:**
- Small vented box (louvered or with mesh patches)
- PTFE membrane over each sensor inlet — protects the oxide, passes gas
- 5V fan pulling air through, exhausting away from inlets
- Fan runs 60–90 s before each read; stops after reading to save power
- No complex internal geometry needed — just ensure fresh air reaches every sensor

---

## Firmware Architecture

```
Wake every 10–15 minutes
  │
  ├─ Power MQ heater(s) via GPIO → MOSFET
  ├─ Wait 60 s for MQ stabilisation
  ├─ Read MQ-136, MQ-135, MQ-3 via ADC
  ├─ Read ZMOD4450 via I2C
  ├─ Read SCD40 CO2 via I2C (no warm-up needed)
  ├─ Read SHT40 / TMP117 T/RH
  ├─ Power MQ heater OFF
  ├─ Compute rise-above-baseline for each channel
  ├─ Compute spoilage_risk score
  ├─ WiFi on → MQTT publish → WiFi off
  └─ Deep sleep until next cycle
```

MQ heaters draw ~150 mA at 5V during the 60 s warm-up window. At 15-minute cycles with 60 s heater-on time, heater duty cycle is ~6.7% — acceptable for a USB-powered device, possible on battery if sized appropriately.

---

## Calibration Procedure

1. Remove all food from fridge, clean interior surfaces
2. Leave door closed for 24 hours (let residual odours dissipate)
3. Power sensor for 48 h burn-in if sensors are new
4. Record 6-hour baseline for each channel — average these as `clean_baseline`
5. Store baseline in flash (same pattern as QNH EMA in env-sensor)
6. Restock fridge with normal contents in good condition
7. Monitor for 1 week to confirm no false positives from normal foods
8. Adjust thresholds if garlic, aged cheese, or fermented products cause consistent false positives

**Known false positive sources:**
- Garlic, onion: sulfur compounds → H2S and NH3 channels
- Aged/blue cheese: H2S, NH3 naturally elevated
- Kimchi, sauerkraut, kombucha: NH3, CO2, ethanol
- Open wine/beer: ethanol channel
- Strategy: weight ethanol low; use H2S + NH3 + CO2 in combination; require two channels to rise simultaneously before alerting

---

## Related Notes

- [[Gas Sensor Fun Projects (MQ + Fermion Niche)]] — MQ-136 and ZMOD4450 background, MQ-9 cycling
- [[Public Data Contribution Plan]] — broader sensor network goals
- [[Weather Station Project]] — main project context
