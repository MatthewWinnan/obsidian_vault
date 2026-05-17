# Gas Sensor Fun Projects — MQ Kit + Fermion Niche Sensors

Novelty, safety, and curiosity projects for the 9-piece MQ sensor kit and the Fermion MEMS sensors not assigned to serious projects.

---

## Sensor Inventory

### Additional MQ sensors (outside the 9-piece kit)

| Sensor | Target gas | Application |
|--------|-----------|-------------|
| **MQ-136** | H2S (hydrogen sulfide) | Fridge spoilage (primary), sewage safety, sour gas — see [[Fridge Spoilage Detector (ZMOD4450 + MQ)]] |

### MQ Kit (9 sensors — all analog MOX, output 0–5 V proportional to gas concentration)

| Sensor | Primary gas | Also detects |
|--------|------------|-------------|
| MQ-2 | Combustible gas (broad) | LPG, propane, H2, methane, smoke, alcohol |
| MQ-3 | Ethanol / alcohol vapour | Benzene, hexane (poor selectivity) |
| MQ-4 | Methane / natural gas | LPG, propane |
| MQ-5 | LPG / natural gas / coal gas | H2 |
| MQ-6 | LPG / butane / propane / LNG | — |
| MQ-7 | Carbon monoxide | H2 (significant cross-sensitivity) |
| MQ-8 | Hydrogen | Alcohol (some) |
| MQ-9 | CO + combustible gas (dual mode) | Alternating heater voltage needed for dual mode |
| MQ-135 | NH3, benzene, sulfide, "harmful gases" | Smoke, alcohol, CO2 (broad) |

### Fermion MEMS niche sensors (assigned to fun projects, not main deployments)

| Sensor | Gas |
|--------|-----|
| EtOH | Ethanol |
| H2 | Hydrogen |
| CH4 | Methane |
| Odor | Broad odour (skip for serious use — not a physical quantity) |

---

## Lifetime — Storage vs. Active Use

**MOX sensors (all 9 MQ sensors + Fermion EtOH/H2/CH4/Odor/VOC/HCHO/NO2/Smoke):**
Storage does **not** count against lifetime. The oxide sintering that causes drift only occurs when the heater is running (200–500°C). Two years in a drawer = essentially zero ageing. Keep dry, avoid silicone vapours. Requires 24–48h burn-in on first power-up for the oxide surface to stabilise.

**Fermion CO (electrochemical cell, SEN0465):**
Storage **does** count — electrolyte oxidises ~2–5%/year even unpowered. Buy/deploy this one closer to when you actually need it. Check manufacture date on packaging.

---

## Project Ideas

### 1. Breathalyzer
**Sensors:** MQ-3 (primary) + MQ-135 (breath odour corroboration)
**Display:** SSD1306 OLED + single button to trigger timed reading

Trigger a 10-second sampling window, display bar graph + category:
- Sober (baseline)
- Suspicious
- Guilty
- Call a cab

MQ-3 is the classic breathalyzer element. **Not legally accurate** — cross-sensitivity to acetone (diabetics read falsely high), benzene, hexane. Fun at braais, not for driving decisions. Adding Fermion EtOH alongside MQ-3 gives a second reading for comparison.

---

### 2. Smelliness Meter / Stink-O-Meter
**Sensors:** MQ-135 (NH3, H2S, benzene) + MQ-2 (broad combustible/smoke) + MQ-9 (CO + combustible)
**Display:** SSD1306 or WS2812 RGB LED strip for visual "smell level"

Combine weighted sensor readings into a single "stink score." Display categories:
- Fresh Air
- Questionable
- Suspicious
- Evacuate Immediately

MQ-135 catches flatulence compounds (NH3 + H2S), cooking smells, cleaning products. MQ-2 adds smoke and combustible vapours. Fun party prop — hold near food, people, pets.

---

### 3. Fart Detector
**Sensors:** MQ-135 (NH3 + H2S — primary odour compounds) + MQ-9 (methane — ~7% of flatulence)
**Bonus:** Multiple units + WiFi → HA → triangulate source by signal strength

The classic maker project. MQ-135 detects hydrogen sulfide and ammonia (the smell); MQ-9 detects methane (the volume). Combine into a single "event" trigger with timestamp logged to HA. With two or more units placed around a room you can determine proximity to the source.

---

### 4. Gas Leak Safety Panel
**Sensors:** MQ-4 (methane/natural gas) + MQ-5 (LPG/natural gas) + MQ-6 (LPG/butane/propane) + MQ-9 (CO + combustible)
**Output:** Buzzer + LED matrix + HA alert

If you have gas appliances (stove, geyser, gas braai), this is genuinely useful safety infrastructure. MQ-4/5/6 overlap but together cover both natural gas AND LPG regardless of which you use. MQ-9 adds CO detection for incomplete combustion. Wall-mount near appliances, integrate with HA for phone notifications.

Fermion CH4 can corroborate MQ-4 for the methane channel if you want a more accurate reference alongside the cheap MQ.

---

### 5. Refuge / Sealed Room Monitor
**Sensors:** MQ-7 (CO) + MQ-9 (CO + combustible) + MQ-135 (NH3, H2S — air quality decay)
**Gap:** CO2 accumulation requires an NDIR sensor (e.g. SCD40) — MQ sensors cannot detect CO2

Monitors a sealed space for hazardous gas build-up:
- CO from fire smoke or a generator running outside
- NH3 and H2S from sanitation breakdown or decomposition
- Combustible gases from structural damage

Would suit a storm shelter, panic room, or generator room. Add an SCD40 (true NDIR CO2) for a complete picture — CO2 accumulation from breathing is the first hazard in a sealed room.

---

### 6. Fermentation / Brewing Monitor
**Sensors:** MQ-3 + Fermion EtOH
**Display:** HA dashboard trend graph over the fermentation cycle

Clamp near (not inside) the airlock of a fermentation vessel. As yeast activity increases, ethanol vapour output rises; as fermentation completes, it falls. Non-invasive fermentation progress tracking. Compare MQ-3 (broad, cheap) against Fermion EtOH (more targeted) — useful for validating the cheaper sensor's behaviour.

---

### 7. Hydrogen Safety Monitor
**Sensors:** MQ-8 + Fermion H2
**Trigger:** HA automation → alert if reading exceeds threshold

Lead-acid batteries produce H2 during charging, particularly on overcharge. Explosive above 4% v/v but detectable at much lower concentrations. Place near any lead-acid UPS, solar battery bank, or vehicle battery on charge. MQ-8 is selective for H2 (reasonably good); Fermion H2 corroborates.

---

### 8. Garage / Car Exhaust Monitor
**Sensors:** MQ-7 (CO) + MQ-9 (CO + combustible) + MQ-2 (broad hydrocarbon)
**Output:** HA automation to trigger ventilation fan or gate opener

Exhaust from a running engine in a closed garage is a CO poisoning risk within minutes. MQ-7 + MQ-9 cover CO; MQ-2 adds unburnt hydrocarbons (petrol/diesel vapour). HA integration triggers ventilation or sends an alert if the garage is occupied.

---

## MQ Sensor to Project Cross-Reference

| Sensor | Primary project | Secondary use |
|--------|----------------|---------------|
| MQ-2 | Gas leak panel | Smelliness meter, garage monitor |
| MQ-3 | Breathalyzer | Fermentation monitor |
| MQ-4 | Gas leak panel | — |
| MQ-5 | Gas leak panel | — |
| MQ-6 | Gas leak panel (LPG) | — |
| MQ-7 | Refuge monitor | Garage/CO safety |
| MQ-8 | Hydrogen safety monitor | Gas leak panel backup |
| MQ-9 | Smelliness / fart detector | Refuge monitor, garage |
| MQ-135 | Smelliness / fart detector | Breathalyzer corroboration, refuge monitor |
| Fermion EtOH | Fermentation monitor | Breathalyzer second channel |
| Fermion H2 | Hydrogen safety monitor | — |
| Fermion CH4 | Gas leak detector | Gas leak panel corroboration |
| Fermion Odor | Skip | Not a calibrated physical quantity |

---

## MQ-9 Dual-Mode Implementation

### At fixed 5V (simpler)
At fixed 5V the heater runs hot continuously — good sensitivity to combustible gases (LPG, methane, propane, butane), poor CO sensitivity. Behaves like a slightly worse MQ-2. Fine for a gas leak alarm; not useful as a CO detector.

### Voltage cycling for dual CO + combustible mode

The MQ-9 heater is ~33Ω, draws ~150mA at 5V — Pico GPIO cannot drive it directly. Use a MOSFET to switch between voltages.

**Circuit — one MOSFET + one resistor:**
```
5V ──── 82Ω ──── H+
                  │
                 33Ω heater
                  │
                 H─ ──── GND

Pico GPIO ──── MOSFET gate (2N7000 or IRLZ44N)
5V ──────────── MOSFET drain
                MOSFET source ──── H+ node (same node as 82Ω output)
```
- MOSFET ON: H+ = 5V (bypasses 82Ω) → combustible gas mode
- MOSFET OFF: H+ = 5 × 33/(33+82) ≈ **1.43V** → CO mode

**Cycle timing (per datasheet):**

| Phase | Voltage | Duration | Stabilise before reading |
|-------|---------|----------|--------------------------|
| High heat | 5V | 60 s | 30 s |
| Low heat | 1.4V | 90 s | 60 s (CO needs longer) |

CO reading only available every 150 s (60+90). Fine for a safety monitor.

**Pico implementation sketch:**
```c
#define MQ9_HEATER_PIN   15     // Controls MOSFET gate
#define MQ9_ADC_CHANNEL   2     // ADC2 = GP28
#define HIGH_HEAT_MS  60000u
#define LOW_HEAT_MS   90000u
#define HIGH_STAB_MS  30000u    // wait before reading combustible
#define LOW_STAB_MS   60000u    // wait before reading CO

typedef enum { PHASE_HIGH, PHASE_LOW } mq9_phase_t;
static mq9_phase_t s_phase;
static uint32_t    s_phase_start_ms;

void mq9_init(void) {
    gpio_init(MQ9_HEATER_PIN);
    gpio_set_dir(MQ9_HEATER_PIN, GPIO_OUT);
    gpio_put(MQ9_HEATER_PIN, 1);   // Start high-heat
    s_phase = PHASE_HIGH;
    s_phase_start_ms = to_ms_since_boot(get_absolute_time());
}

// Call every loop iteration. Fills whichever ratio is valid, NAN otherwise.
// Returns true when a fresh reading is available.
bool mq9_update(float *combustible_ratio, float *co_ratio) {
    uint32_t now     = to_ms_since_boot(get_absolute_time());
    uint32_t elapsed = now - s_phase_start_ms;
    *combustible_ratio = NAN;
    *co_ratio          = NAN;

    if (s_phase == PHASE_HIGH) {
        if (elapsed >= HIGH_HEAT_MS) {
            gpio_put(MQ9_HEATER_PIN, 0);   // Switch to low-heat
            s_phase = PHASE_LOW;
            s_phase_start_ms = now;
        } else if (elapsed >= HIGH_STAB_MS) {
            adc_select_input(MQ9_ADC_CHANNEL);
            *combustible_ratio = (float)adc_read() / 4095.0f;
            return true;
        }
    } else {
        if (elapsed >= LOW_HEAT_MS) {
            gpio_put(MQ9_HEATER_PIN, 1);   // Switch back to high-heat
            s_phase = PHASE_HIGH;
            s_phase_start_ms = now;
        } else if (elapsed >= LOW_STAB_MS) {
            adc_select_input(MQ9_ADC_CHANNEL);
            *co_ratio = (float)adc_read() / 4095.0f;
            return true;
        }
    }
    return false;
}
```

ADC gives raw ratio (0–1). Full ppm conversion requires the datasheet Rs/Ro sensitivity curve and a clean-air baseline calibration. For alarm use, a threshold on the raw ratio is sufficient.

**Practical notes:**
- First use: 48h burn-in at 5V before trusting any reading
- No flyback diode needed — heater is resistive, not inductive
- If only the gas leak alarm is needed: fix at 5V, skip cycling entirely. Simpler, faster response.

---

## ZMOD4450 — SparkFun Refrigerant Gas Sensor (SPX-16677)

**Chip:** Renesas ZMOD4450. MOX MEMS micro-heater tuned specifically for refrigerant gases: R134a, R410A, R32, HFCs, HFOs.
**Interface:** Qwiic (I2C). **Purchased:** 2023.

### Storage sensitivity
Low but not zero. The ZMOD4450 is a precision MEMS device (finer than crude MQ ceramic elements). Renesas storage spec: <40°C, <85% RH non-condensing. Hermetically sealed package protects the oxide surface. Stored indoors in SA since 2023 (~2–3 years by deployment) — within safe shelf life. Requires a conditioning/burn-in cycle on first power-up per Renesas application firmware spec.

### Run lifetime
Heater runs in **low-power pulsed mode** (not always-on like MQ sensors) — significantly extends heater life. Typical ZMOD run lifetime: **5–10+ years**.

### Algorithm dependency
Like the BME680/BSEC situation, the ZMOD4450 needs a **proprietary Renesas algorithm library** to convert raw resistance to refrigerant concentration. SparkFun's Arduino library wraps this blob. Porting to bare-metal Pico C will require sourcing the Renesas ZMOD library for ARM Cortex-M — check availability before committing.

### Project: Refrigerant Leak Detector
Mount near A/C compressor, fridge compressor, or car aircon service port. Refrigerant leaks are expensive and HFCs are potent greenhouse gases (R410A GWP ~2000). A HA-integrated alert on leak detection is genuinely useful.

---

## Gas Sensor Enclosure Design

### MOX sensors vs PMSA003 — different requirements

The PMSA003 needs a precision fan + laser chamber because optical particle counting requires controlled laminar airflow to present particles one at a time in a defined beam volume. MOX gas sensors (MQ series, Fermion MEMS, BME680, ZMOD4450) work purely by chemical contact — gas molecules diffuse to and react with a heated oxide surface. No fan or precision chamber is required for the sensor to function.

However, the enclosure around the sensors still needs design:

| Concern | Why | Solution |
|---------|-----|----------|
| Slow diffusion in sealed box | Concentration changes outside take minutes to reach sensor | Ventilation ports or active fan |
| Dust on oxide surface | Accumulates over months, blocks gas access, causes drift | PTFE membrane or mesh over sensor inlets |
| Liquid water ingress | Outdoor/humid — condensation on hot oxide causes damage | PTFE membrane (passes gas, blocks liquid) |
| Heater self-heating | MQ heaters at 5V warm air in sealed box, shifting readings | Enclosure must breathe |
| Post-event flush | After high-concentration event, sensor needs fresh air to recover | Fan flush cycle or passive venting |
| Multiple sensors cross-heating | Adjacent MQ heaters warm each other's local air | Space sensors; direct airflow across all uniformly |

### Practical enclosure (indoor multi-sensor station)

- Louvered or vented box — same principle as a Stevenson screen but smaller
- PTFE membrane patch over each individual sensor inlet (protects oxide, passes gas freely)
- 5V 40 mm fan pulling fresh air in from one side, exhausting from the other
- Fan runs 60–90 s before each read cycle, off between reads (saves power, reduces heater load)
- No precision internal geometry needed — just ensure fresh air reaches every sensor

### Practical enclosure (outdoor use)

- IP54 or better rated box with louvered vents on underside (rain cannot enter from below)
- PTFE membrane over each vent (Goretex-style)
- Sensor inlets face downward or sideways — never upward
- Stevenson screen style radiation shield if temperature sensor is co-located
- For extreme environments: heated enclosure to keep sensors above dew point

### Fridge spoilage — tube-based remote sampling

See [[Fridge Spoilage Detector (ZMOD4450 + MQ)]] for full details on routing fridge air to external sensors via silicone tube + 5V fan. Summary: sensors outside, 4–6 mm tube through door seal, fan pulls air through on a 60–90 s flush cycle before each read.

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

## Notes on MQ Wiring

All MQ sensors output an analog voltage (0–5V typically, or 0–3.3V on 3.3V modules). The Pico ADC is 12-bit, 3.3V max. Options:
- Use 3.3V MQ modules (most breakout boards have a 3.3V-compatible output or include a voltage divider)
- Use a multi-channel ADC (ADS1115, GP8413) for more channels than the Pico's 3 ADC pins
- The DFRobot multi-channel ADC board used for Fermion sensors works here too

MQ-9 requires alternating heater voltage (5V for 60s, 1.4V for 90s) for dual CO/combustible mode — needs a PWM-controlled heater circuit. Running it at fixed 5V gives combustible gas detection only.

---

[[Weather Station Project|← Back to Weather Station Project]]
