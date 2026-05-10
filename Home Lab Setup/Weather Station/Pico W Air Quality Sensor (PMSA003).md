# Pico W Air Quality Sensor (PMSA003)

## Hardware

| Item | Detail |
|------|--------|
| Board | Raspberry Pi Pico W (RP2040, CYW43439 WiFi) |
| Sensor | Plantower PMSA003 — laser-scattering optical particle counter |
| Display | SSD1306 128×32 OLED (I2C0, SDA=GP20, SCL=GP21, addr 0x3C) |
| UART | UART0, GP0(TX)/GP1(RX), 9600 baud 8N1 |
| Power | USB (display enabled) or battery (display disabled) |

Firmware: `projects/pico-w-air-sensor/` in pico_nix repo.

---

## What the PMSA003 Measures

The PMSA003 is a low-cost Optical Particle Size Spectrometer (OPSS). It draws air over a 650 nm laser diode; a photodetector counts pulses of scattered light, bins them by size, and derives:

**Mass concentrations** (µg/m³) — PM standard atmospheric model:
- PM1.0, PM2.5, PM10

**Particle counts** (per 0.1 L air):
- >0.3 µm, >0.5 µm, >1.0 µm, >2.5 µm, >5.0 µm, >10 µm

The sensor outputs one frame per second. The PMSA003 internally averages over ~0.1 L of sampled air per reading, which at the sensor's ~0.1 L/s flow rate is approximately 1 second of sampling.

---

## Measurement Standards

### Regulatory framework (PM2.5 / PM10)

The primary standard for reporting PM health risk is **WHO Air Quality Guidelines (AQG), 2021**:

| Pollutant | Annual mean | 24-hour mean |
|-----------|-------------|--------------|
| PM2.5 | 5 µg/m³ | 15 µg/m³ |
| PM10 | 15 µg/m³ | 45 µg/m³ |

These are *exposure* averages — the health effect of fine particles accumulates over time. Spot readings (seconds/minutes) are only meaningful for detecting transient events (smoke, cooking, dust). The health-relevant quantity is the **24-hour mean**.

**US EPA AQI** is also defined on 24-hour PM2.5 averages (NowCast uses a weighted 12-hour window for real-time approximation).

### WMO guidance on OPSSs

**WMO-No. 8 (CIMO Guide), 2024, §16.6.3** covers particle number concentration and size distribution instruments:

> "OPSSs include a number of relatively small, **low-cost instruments utilizing laser diodes** that have to be operated carefully to deliver reliable quantitative measurements. Both APSSs and OPSSs need regular QA sizing checks on site. Additionally, annual or biannual calibrations against traceable standards and reference instruments are required. If OPSSs performance is not controlled, both instrument types might drift with time, causing an unnoticed bias of **up to several tens of per cent**."

Reference: WMO-No. 8, Vol. I, §16.6.3, p. 566–567.

**Key implications for the PMSA003:**
- The PMSA003 is an OPSS — WMO guidance applies directly
- Absolute accuracy is limited (factory calibration only); relative changes and trend detection are reliable
- The sensor is suitable for indoor air quality monitoring, pollution events, and trend logging — not for regulatory-grade PM mass measurements
- Chamber clearing on first power-on after storage is a known behaviour (see experiment below)

### Sampling frequency recommendations

**WMO-No. 8, §16.6.5** (AOD measurements) notes 1-minute sampling as standard for continuous instruments with automated QC. For low-cost sensors used in air quality networks, the EPA and EU directives typically require:
- **1-minute** raw sample rate minimum
- **Hourly** averages for short-term monitoring
- **24-hour** averages for health standard comparison

---

## Current Firmware Behaviour

| Parameter | Value | Notes |
|-----------|-------|-------|
| Sample rate | ~1 Hz (PMSA003 native) | One frame per second |
| Publish interval | 10-sample average (~10 s) | `PUBLISH_INTERVAL = 10` |
| Averaging window | 10 seconds | Too short for health-relevant reporting |
| WiFi | Always-on | Not battery-optimised |

**MQTT topics:**
- State: `pico/air-sensor/state`
- Status: `pico/air-sensor/status`

**Current state payload:**
```json
{
  "pm1_0": 12, "pm2_5": 18, "pm10": 23,
  "cnt_03": 1842, "cnt_05": 621, "cnt_10": 89,
  "cnt_25": 12, "cnt_50": 3, "cnt_100": 0
}
```

---

## Recommended Improvements

To align with WMO/WHO best practice:

### 1. Move to 1-min averaging

Collect ~60 samples (at 1 Hz), compute means, publish once per minute. This matches the minimum recommended sampling period and dramatically reduces noise.

### 2. Track a 60-min rolling mean for PM2.5/PM10

Use a ring buffer of 60 × 1-min PM values to compute the hourly mean. Publish alongside the 1-min reading. This gives the context needed to assess against WHO 24h guidelines (a full 24h mean requires 24 hours of data, but the 1h mean is already useful).

### 3. Report WHO AQI breakpoints

For each 1-min and 1-hour PM2.5 publish, include a categorical descriptor:

| Category | PM2.5 1h (µg/m³) |
|----------|-----------------|
| Good | < 5 |
| Fair | 5–15 |
| Moderate | 15–25 |
| Poor | 25–50 |
| Very poor | > 50 |

### 4. Deep-sleep battery mode

Follow the pico-w-env-sensor pattern: sleep between 1-min sample cycles, only power WiFi every N minutes. This would allow battery deployment.

---

## Experiment: Chamber Clearing & Smoke Response (2026-05-09–10)

### Setup

Sensor stored unpowered in a cupboard for an extended period, then powered on continuously and left indoors. A smoke test was performed on 2026-05-10 at ~12:55 local time.

### Results

**Timespan:** 2026-05-09 20:11 → 2026-05-10 13:55 (~18 hours)

**Downward trend on first power-on:**

| Channel | Start (~20:11) | Settled (~08:00) | Ratio |
|---------|---------------|-----------------|-------|
| cnt_03 (>0.3 µm) | 4,395 | ~560 | 7.8× |
| cnt_05 (>0.5 µm) | 1,309 | ~140 | 9.4× |
| cnt_10 (>1.0 µm) | 234 | ~7 | 33× |

This is **chamber clearing** — normal, documented behaviour. The sensor's internal fan sweeps settled dust through the laser beam on first power-on. Larger particles (cnt_10) take proportionally longer to clear because heavier particles settle faster and accumulate more during storage. Readings converge to true ambient concentration as the chamber flushes.

**Smoke spike at ~12:55:**

| Channel | Baseline | Peak | Multiplier |
|---------|----------|------|-----------|
| cnt_03 | ~847 | 36,344 | **43×** |
| cnt_05 | ~241 | 11,797 | **49×** |
| cnt_10 | ~15 | 7,861 | **524×** |

Recovery to near-baseline within ~1 hour.

**Interpretation:**
Combustion smoke is dominated by fine particles (0.3–1 µm). The absolute count increase is largest for cnt_03/cnt_05. The extreme relative spike in cnt_10 (524×) is physically correct: clean indoor air has very few particles >1 µm, so the same smoke plume produces enormous relative amplification in bins that normally sit near zero. The sensor response is correct and well-behaved.

**Sensor verdict:** Working correctly. The downward trend is expected, not a fault. The smoke response is strong, fast, and recovers appropriately.

---

## Planned Hardware Upgrades

### Sensirion gas sensor stack (primary — add first)

| Sensor | Role | Accuracy |
|--------|------|----------|
| **SHT40** | Primary humidity + temperature for SGP41 compensation | ±1.8% RH, ±0.2°C |
| **SGP41** | VOC Index (0–500) + NOx Index (0–500) | Open-source Sensirion algorithms |

SHT40 and SGP41 are a designed pair — SGP41 requires real-time T/RH from the SHT40 to run its index algorithms. SHT40 also enables humidity correction for PMSA003 PM readings (hygroscopic particle growth inflates readings above ~75% RH).

Wait for SGP41 rather than SGP40 — the NOx channel is a real additional measurement not available from any other sensor in the stack.

### DFRobot Fermion MEMS gas sensors (via multi-channel ADC → I2C)

Targeted selection for indoor IAQ — not all 11 types, only:

| Sensor | Target gas | Purpose |
|--------|-----------|---------|
| **CO** | Carbon monoxide | Safety — combustion/incomplete burn detection |
| **HCHO** | Formaldehyde | Indoor off-gassing (furniture, paint, flooring) |
| **NO2** | Nitrogen dioxide | Traffic/combustion pollutant |
| **VOC** | Total VOC | Redundancy check against SGP41 VOC Index |
| **Smoke** | Smoke particles | Corroboration with PMSA003 PM channels |

**Drift — causes and references:**
MOX sensors (VOC, HCHO, NO2, Smoke) drift via sintering of the oxide grain structure at operating temperature (200–500°C continuous). Baseline resistance increases monotonically over months. Silicone compounds (RTV, gaskets, aerosols) permanently poison MOX surfaces — never seal the enclosure with silicone. Lifetime: 2–5 years typical. Electrochemical CO sensors dry out over 1–3 years.

Documented in: EPA Air Sensor Guidebook (EPA/600/R-14/159), Figaro Engineering AN-101/AN-102, WMO-No. 8 §16.6.3 (drift warning directly applicable).

**Drift mitigation strategy (software):**
When SGP41 VOC Index ≤ 110 (clean air confirmed by trusted sensor), log each MEMS channel reading. Maintain a 7-day rolling minimum per channel stored in flash. Subtract as zero correction before publishing. This replicates Sensirion's internal background calibration approach without manual intervention.

**Cross-validation:**
- CO: spot-check against a domestic UL-listed CO alarm monthly
- NO2: compare open-window readings against SAAQIS reference data (same method as PM calibration — see [[Public Data Contribution Plan]])
- VOC: compare against SGP41 VOC Index; systematic divergence indicates MEMS drift

### Niche sensors — separate projects

| Sensor | Measurement | Best use case |
|--------|-------------|---------------|
| **CH4** | Methane | Natural gas leak detector (if gas appliances present) |
| **H2** | Hydrogen | Lead-acid/LiPo battery off-gas safety monitor |
| **EtOH** | Ethanol vapour | Fermentation/brewing monitor |
| Odor | Broad-band | Skip — not a physical quantity, unverifiable |

### Full planned stack

```
pico-w-air-sensor:
  PMSA003   UART0    Particles — PM1.0/2.5/10, counts (existing)
  SHT40     I2C      Primary T/RH — SGP41 compensation + PM humidity correction
  SGP41     I2C      VOC Index + NOx Index (Sensirion open-source algorithms)
  CO MEMS   ADC→I2C  Carbon monoxide safety
  HCHO      ADC→I2C  Formaldehyde (off-gassing)
  NO2       ADC→I2C  Traffic/combustion pollutant
  VOC       ADC→I2C  Redundancy vs SGP41
  Smoke     ADC→I2C  Redundancy vs PMSA003
```

---

## Indoor Air Quality Behaviour — Observations & Science

### Observed behaviour

Two clear patterns confirmed with the sensor operating indoors:

1. **Opening a window raises PM counts** — outdoor aerosol loads infiltrate and raise indoor concentration
2. **Sealed indoors with no source causes PM to trend downward** — passive deposition onto surfaces

### Why this happens — source/sink model

Indoor air quality is governed by the balance between **aerosol sources** and **sinks**:

**Sources:** outdoor infiltration, cooking, candles, occupant activity, dust resuspension
**Sinks:** particle deposition (gravity + electrostatic + impaction), filtration, ventilation

When sealed with no source the sink dominates and concentrations decay exponentially:

```
C(t) = C₀ × e^(−λt)
```

where λ is the first-order deposition rate constant, typically **0.2–1.5 h⁻¹** depending on room size, furnishings, and particle size. Larger particles (PM10) settle faster than fine particles (PM2.5), which can remain airborne for hours.

Reference: Thatcher & Layton (1995), *Deposition, resuspension, and penetration of particles within a residence*, Atmospheric Environment; Wallace (1996), *Indoor particles: A review*, JAPCA.

### Indoor/outdoor (I/O) ratio

Studies consistently show I/O ratios of **0.5–0.8 for PM2.5** in ventilated buildings without indoor sources (EPA, UK AQEG, WHO). When a window opens, indoor concentration rises toward outdoor levels within minutes. The building envelope provides passive filtration — typically 20–50% attenuation.

Outdoor suburban/urban background PM2.5 is typically **10–25 µg/m³** at ground level, higher near roads.

### Health assessment implications

| Condition | Interpretation |
|-----------|---------------|
| Window closed, low PM | Clean indoor air; building acting as filter |
| Window open, PM rises | Normal infiltration; partial outdoor attenuation |
| PM spike, no window change | Indoor source (cooking, candles, resuspension) |
| PM elevated all day | High outdoor background or persistent indoor source |

The WHO 24-hour mean standard (PM2.5 ≤ 15 µg/m³) applies to **outdoor ambient exposure**. Clean indoor air with sealed windows typically sits well below this — the sensor showing very low values indoors is the correct and expected result for a clean environment.

### Value for sensor calibration

This I/O ratio behaviour provides informal calibration evidence. When the window is open and indoor readings have stabilised, compare against the nearest SAAQIS reference monitor. A consistent ratio (indoor ÷ reference ≈ 0.6–0.8) supports sensor plausibility. Systematic deviation indicates sensor bias or a local source. Document these comparisons before submitting data to any public network.

---

## Goal: Contribute Data to Public Networks

### Target networks

| Network | Data | Priority | Status |
|---------|------|----------|--------|
| **Sensor.Community** | PM1.0, PM2.5, PM10, particle counts | High | Not started |
| **Weather Underground PWS** | Temperature, humidity, pressure (QFE + QNH) | High | Not started |
| **SAWS** (SA Weather Service) | Pressure, temperature, humidity | Medium | Contact required |
| **SAAQIS** (SA Air Quality IS) | PM2.5, PM10 | Medium | Contact required |
| **CWOP** | Pressure, temperature, humidity | Low | Requires CWOP ID |

### Implementation plan

**Phase 1 — HA bridge (no firmware changes needed)**

Both Sensor.Community and Weather Underground accept data via simple HTTP APIs. An HA `rest_command` triggered by MQTT state updates is the cleanest approach — the Pico stays battery-efficient and HA (always-on) handles the external reporting.

- Sensor.Community: HTTP POST to their data ingest endpoint every ~2.5 min, identified by sensor hardware ID (WiFi MAC). JSON schema maps directly to our PM fields.
- Weather Underground: HTTP GET with temperature, humidity, barometric pressure (station + MSL). Requires registering a PWS station and obtaining an API key + station ID.

**Phase 2 — calibration evidence**

Before approaching SAWS or SAAQIS, accumulate several months of clean data and compare pressure readings against the nearest official SAWS station. Establish a documented bias correction. This is the evidence needed for a formal data-sharing conversation.

### Known limitations to document when submitting

- **PMSA003**: Low-cost OPSS (WMO §16.6.3). Uncalibrated against reference instrument. Positive PM bias in high-humidity conditions (>85% RH). No humidity correction currently applied.
- **BMP180 / BME280**: Consumer-grade MEMS pressure sensors. QNH normalised via GPS altitude (EMA). No traceability to national pressure standards.
- **Tendency**: 3-hour WMO-aligned window, 10-min intervals (per WMO §3.5.1 recommendation). Fed by BME280 MSL mean.

---

## WMO Reference Pages

| Topic | Reference |
|-------|-----------|
| Low-cost OPSSs, calibration, drift | WMO-No. 8 (CIMO Guide 2024), Vol. I, §16.6.3, p. 566–567 |
| Aerosol mass concentration measurement | WMO-No. 8, Vol. I, §16.6.1, p. 560–566 |
| Aerosol variables and units | WMO-No. 8, Vol. I, §16.1.2, p. 538–540 |
| WHO AQG 2021 (PM2.5/PM10 targets) | WHO Global Air Quality Guidelines, 2021 |
