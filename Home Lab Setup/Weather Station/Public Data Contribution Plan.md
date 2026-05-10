# Public Data Contribution Plan

Goal: contribute local sensor data to public meteorological and air quality networks, improving open data coverage for our area and subjecting our measurements to peer scrutiny.

Related devices: [[Pico W Auxiliary Weather Station (BMP180 + BME280 + INA219)]] (pressure, temperature, humidity) and [[Pico W Air Quality Sensor (PMSA003)]] (PM2.5, PM10, particle counts).

---

## Target Networks

| Network | Data | Priority | Status |
|---------|------|----------|--------|
| **Sensor.Community** | PM1.0, PM2.5, PM10, particle counts | High | Not started |
| **Weather Underground PWS** | Temperature, humidity, pressure (QFE + QNH) | High | Not started |
| **CWOP** (Citizen Weather Observer Programme) | Pressure, temperature, humidity | Medium | Requires CWOP ID |
| **SAWS** (SA Weather Service) | Pressure, temperature, humidity | Medium | Formal contact required |
| **SAAQIS** (SA Air Quality IS) | PM2.5, PM10 | Medium | Formal contact required |

---

## Phase 1 — HA Bridge (no firmware changes)

Both Sensor.Community and Weather Underground accept data via simple HTTP APIs. The cleanest approach is an HA `rest_command` triggered by MQTT state updates — the Pico stays efficient, HA (always-on) handles external reporting.

### Sensor.Community

- HTTP POST to `https://api.sensor.community/v1/push-sensor-data/`
- Identified by sensor hardware ID (WiFi MAC address of the Pico W)
- Required fields: `P1` (PM10 µg/m³), `P2` (PM2.5 µg/m³)
- Optional: particle counts
- Publish interval: every 2.5 minutes (their recommended cadence)
- No registration required — first POST auto-creates the sensor; add description on their map

**HA implementation:**
```yaml
rest_command:
  sensor_community_push:
    url: "https://api.sensor.community/v1/push-sensor-data/"
    method: POST
    headers:
      X-PIN: "1"
      X-Sensor: "raspi-<MAC>"   # replace with Pico W MAC
      Content-Type: "application/json"
    payload: >
      {"software_version": "pico_nix-1.0",
       "sensordatavalues": [
         {"value_type": "P1", "value": "{{ states('sensor.air_pm10') }}"},
         {"value_type": "P2", "value": "{{ states('sensor.air_pm2_5') }}"}
       ]}
```

### Weather Underground PWS

- HTTP GET to `https://weatherstation.wunderground.com/weatherstation/updateweatherstation.php`
- Requires station ID + API key (register at wunderground.com/member/stations/signup)
- Required fields: `tempf` (°F), `baromin` (inHg), `humidity` (%)
- Optional: `dewptf`, `rainin`, `windspeedmph`, `winddir`
- Publish interval: every 5 minutes recommended

**HA implementation:**
```yaml
rest_command:
  wunderground_push:
    url: >
      https://weatherstation.wunderground.com/weatherstation/updateweatherstation.php
      ?ID=<STATION_ID>
      &PASSWORD=<API_KEY>
      &dateutc=now
      &tempf={{ ((states('sensor.bme280_temperature') | float) * 9/5 + 32) | round(1) }}
      &humidity={{ states('sensor.bme280_humidity') }}
      &baromin={{ ((states('sensor.bme280_pressure_msl') | float) * 0.02953) | round(4) }}
      &action=updateraw
    method: GET
```

---

## Phase 2 — Calibration Evidence (before SAWS/SAAQIS contact)

Before approaching official agencies, accumulate several months of clean data and build a calibration record:

### Pressure calibration (for SAWS)
- Compare BME280 QNH against the nearest SAWS published station reading
- Log the offset and check its stability over seasons
- Target: < ±1 hPa systematic bias with documented drift < 0.5 hPa/month

### PM calibration (for SAAQIS)
- Use the indoor/outdoor (I/O) ratio method: with window open and indoor concentrations stabilised, compare against the nearest SAAQIS continuous monitor
- Expected I/O ratio: 0.6–0.8 for PM2.5 (building attenuation)
- Systematic deviation from this range indicates sensor bias
- Also document humidity sensitivity (PMSA003 reads high above ~85% RH)

### What to include in a submission to SAWS/SAAQIS
1. Sensor specifications and WMO classification (OPSS per WMO §16.6.3)
2. Calibration comparison dataset (minimum 3 months)
3. Known limitations (no humidity correction, no traceable calibration)
4. Proposed data sharing format (CSV, MQTT, API)
5. Station metadata: location, elevation, enclosure type, exposure

---

## Phase 3 — CWOP Registration

CWOP (Citizen Weather Observer Programme) is maintained by NOAA/NWS and the amateur radio community. Data feeds directly into NWS assimilation and is used by Weather Underground and others.

- Register at `wxqa.com` or via the CWOP website
- Assigned a station ID (format: CW + 4 digits, or amateur call sign)
- Transmit using APRS protocol (Automatic Packet Reporting System)
- HA can submit via a Python script or the `aprs` integration

CWOP data is ingested by MADIS (Meteorological Assimilation Data Ingest System) — this is the highest-value public contribution short of a formal SAWS affiliation.

---

## Known Limitations to Disclose

| Sensor | Limitation |
|--------|-----------|
| PMSA003 (OPSS) | Low-cost laser scatter; uncalibrated vs reference; positive bias at RH > 85%; no humidity correction |
| BMP180 | Consumer MEMS; QNH normalised via GPS altitude EMA; no traceability |
| BME280 | Same as BMP180; higher accuracy (±1 hPa typical) |
| GPS altitude | Used for QNH normalisation; accuracy ±3–5 m, which introduces ~0.4 hPa QNH error |
| Tendency | 3-hour WMO-aligned window (WMO §3.5.1), 10-min intervals, fed by BME280 |

---

## References

- Sensor.Community API: https://github.com/opendata-stuttgart/meta/wiki/EN-APIs
- Weather Underground PWS API: https://support.weather.com/s/article/PWS-Upload-Protocol
- CWOP: http://wxqa.com/
- SAWS: https://www.weathersa.co.za
- SAAQIS: https://saaqis.environment.gov.za
- WMO-No. 8 §16.6.3: Low-cost OPSSs, calibration requirements
- WHO AQG 2021: PM2.5/PM10 health targets

---

[[Weather Station Project|← Back to Weather Station Project]]
