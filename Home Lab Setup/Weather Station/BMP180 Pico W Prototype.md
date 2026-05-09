# Pico W Auxiliary Weather Station (BMP180 + BME280 + INA219)

The **first of a series of auxiliary Pico W sensors** that complement the main
[[Weather Station Project]] (Pimoroni Enviro Weather Pico W, MicroPython, wind/rain).

Built on a bare **Raspberry Pi Pico W** with three sensors: BMP180 (pressure/temp),
BME280 (pressure/temp/humidity), and INA219 via Pico-UPS-A (battery monitoring).
Streams all readings to Home Assistant via a single MQTT state topic.

## Role in the wider system

```
┌─────────────────────────────────┐     ┌──────────────────────────────────┐
│  Main Station (MicroPython)     │     │  Aux Sensors (C/C++ Pico SDK)    │
│  Pimoroni Enviro Weather Pico W │     │  projects/pico-w-ha-sensor/       │
│  • Wind speed / direction       │     │    BMP180 + BME280 + INA219       │
│  • Rain gauge                   │     │  projects/<future>/  ──── ...     │
│  • Onboard temp/humidity/press  │     │                                   │
│  • Light sensor                 │     │  All share:                       │
└───────────────┬─────────────────┘     │  • Flash credential provisioning  │
                │ WiFi / MQTT           │  • MQTT → HA auto-discovery       │
                ▼                       │  • GPS altitude subscription      │
       ┌────────────────┐               └────────────────┬─────────────────┘
       │  MQTT Broker   │◀───────────────────────────────┘
       │  (fr3yr:1883)  │
       └────────┬───────┘
                ▼
       ┌────────────────┐
       │ Home Assistant │
       └────────────────┘
```

---

## Hardware

| Part | Notes |
|------|-------|
| Raspberry Pi Pico W | RP2040 + CYW43439 WiFi |
| BMP180 | I2C0, addr 0x77 — pressure + temperature (no humidity) |
| BME280 | I2C0, addr 0x76 (SDO → GND) — pressure + temperature + humidity |
| Waveshare Pico-UPS-A | INA219 on I2C1, addr 0x43 — battery current/voltage monitor |
| I2C0 | SDA → GP4, SCL → GP5, 200 kHz — shared by BMP180 + BME280 |
| I2C1 | SDA → GP6, SCL → GP7, 400 kHz — INA219 only |

---

## Firmware

**Repo:** `~/PICO/pico_nix` — project at `projects/pico-w-ha-sensor/`
**Build:** `just build-pico-w-ha-sensor` (root Justfile) or `just build` inside the project dir.
**Board:** `pico_w` — flash with picotool or drag-drop `.uf2`.

### Source layout

```
projects/pico-w-ha-sensor/
├── CMakeLists.txt        # pico_w board, links CYW43 + lwIP MQTT + hardware_flash
├── lwipopts.h            # lwIP config: LWIP_MQTT=1, threadsafe background mode
├── Justfile
├── package.nix           # Nix derivation (board = pico_w)
└── src/
    ├── main.c            # Boot flow + sensor loop
    ├── provisioning.h/c  # Flash credential storage + serial provisioning
    └── mqtt_ha.h/c       # lwIP MQTT client + HA auto-discovery

libs/
├── bmp180/               # BMP180 driver (I2C0)
├── bme280/               # BME280 driver (I2C0)
├── ina219/               # INA219 driver (I2C1)
├── i2c0/                 # I2C0 init (GP4/GP5, shared by BMP180 + BME280)
└── i2c1/                 # I2C1 init (GP6/GP7, INA219)
```

---

## Boot Flow

```
Power on
  │
  ▼
USB serial connected?  ──(wait)──▶ yes
  │
  ▼
Load credentials from flash
  │ magic == 0xC0FFEE01?
  ├── NO  ──▶ Serial provisioning (prompt SSID / pass / MQTT host / port)
  │            Write to flash → continue
  └── YES ──▶ continue
  │
  ▼
CYW43 init + WiFi connect (30 s timeout)
  │ failed? ──▶ creds_invalidate() + watchdog_reboot() → re-provisioning on next boot
  │
  ▼
I2C0 init → BMP180 init → BME280 init
I2C1 init → INA219 init
  │
  ▼
MQTT connect (DNS resolution + lwIP MQTT + 15 s timeout)
  │ failed? ──▶ creds_invalidate() + watchdog_reboot() → re-provisioning on next boot
  │
  ▼
Publish HA auto-discovery (retained, 12 entities)
Subscribe gps/fr3yr/tpv
Publish status "online" (retained)
  │
  ▼
Sensor loop (every 10 s) ─────────────────────────────────────────────────────┐
  │                                                                            │
  ├─ Trigger BME280 forced-mode conversion (~40 ms to complete)               │
  ├─ BMP180: blocking read — 3 samples × (10ms temp + 24ms press) = ~102 ms  │
  ├─ Read GPS altitude from mqtt.gps_altitude_m (lwip_begin/end)             │
  ├─ bmp180_compute_sea_pressure(p, station_alt) → QNH                       │
  ├─ EMA update: qnh_ref = 0.2 × QNH + 0.8 × qnh_ref  (if GPS valid)       │
  ├─ bmp180_compute_altitude(p, qnh_ref) → barometric altitude               │
  ├─ Read BME280 results (conversion finished ~62 ms ago, instant read)       │
  ├─ Compensate BME280 T / P / H                                             │
  ├─ Compute BME280 QNH + barometric altitude inline                         │
  ├─ INA219 read (voltage, current, battery %)                               │
  ├─ Publish state JSON                                                       │
  └─ sleep_ms(10000) → loop ───────────────────────────────────────────────────┘
```

---

## Credential Storage (Flash)

Last 4 KB sector of the 2 MB flash (`PICO_FLASH_SIZE_BYTES - FLASH_SECTOR_SIZE`).
Validated by magic word `0xC0FFEE01`.

```c
typedef struct {
    uint32_t magic;          // 0xC0FFEE01 when valid
    char     wifi_ssid[64];
    char     wifi_pass[64];
    char     mqtt_host[128]; // IP or hostname
    uint16_t mqtt_port;
    uint8_t  _pad[2];
} creds_t;
```

- Written with `flash_range_erase` + `flash_range_program` (interrupts disabled).
- `creds_invalidate()` erases the sector — next boot enters provisioning mode.
- Re-provisioning is triggered automatically on WiFi or MQTT connect failure.

---

## Measurements

### BMP180 (`libs/bmp180/`)

`bmp180_get_measurement()` takes `BMP_180_SS = 3` averaged samples at
oversampling mode `BMP_180_OSS = 1`. Returns:
- `T` — temperature in 0.1 °C units (divide by 10 for °C)
- `p` — absolute pressure in Pa (QFE)

Two compute-only functions (no I²C):

```c
bmp180_compute_sea_pressure(&sensor, station_alt_m);  // → p_relative (Pa, QNH)
bmp180_compute_altitude(&sensor, qnh_ref_pa);          // → altitude (m)
```

**Note on temperature accuracy:** BMP180 draws ~0.9 mA during measurement (older design),
causing die self-heating. It consistently reads ~1.7–1.9 °C *higher* than the BME280.
The BME280 is closer to actual ambient. BMP180 temperature exists primarily to compensate
its own pressure reading, not as a precision thermometer.

**Note on thermal lag:** The BMP180 responds to ambient temperature changes ~10 s
(one full cycle) slower than the BME280. This is a physical difference in thermal
mass between the two packages, not a code issue — both sensors are measured within
the same ~102 ms window each cycle. The BME280's smaller die equilibrates with
surrounding air faster; the BMP180's metal cap retains heat longer.

---

### BME280 (`libs/bme280/`)

#### Oversampling profile (Bosch "weather station" recommendation)

| Parameter | Setting | Value |
|-----------|---------|-------|
| `osrs_h` | 1 | ×1 humidity |
| `osrs_t` | 2 | ×2 temperature |
| `osrs_p` | 5 | ×16 pressure |
| Mode | forced | single-shot, returns to sleep (~0.1 µA) |
| Conversion time | ~40 ms | triggered at top of 10 s loop, read after sleep |

#### Non-blocking read pattern

BME280 forced-mode conversion (~40 ms) is triggered at the top of the loop, before
the BMP180 blocking read. The BMP180 takes ~102 ms, so by the time it finishes the
BME280 has been done for ~62 ms — no polling or sleep needed.

```c
// Top of loop: trigger BME280 (conversion takes ~40 ms)
bme_sensor.settings->mode = 0b01;
bme280_start_measurements(&bme_sensor);

// BMP180 blocking read (~102 ms: 3 × (10ms temp + 24ms press))
bmp180_get_measurement(&sensor);

// ... QNH / altitude computation (~0 ms) ...

// BME280 is already done — instant read, no wait
bme280_get_uncompensated_measurements(&bme_sensor);
bme280_compensate_temp(&bme_sensor);
bme280_compensate_press(&bme_sensor);
bme280_compensate_hum(&bme_sensor);

// ... INA219, publish ...

sleep_ms(10000);  // sleep is LAST, after publishing
```

**Note on BMP180 blocking time:** `bmp180_get_ut()` waits `BMP_180_TMP_TIME × 2 = 10 ms`
and `bmp180_get_up()` waits `pressure_time[OSS] × 3 = 24 ms` — both use conservative
multipliers over the datasheet minimums. With `BMP_180_SS = 3` averaged samples the
total blocking time is ~102 ms, not "a few ms" as older comments suggest.

#### Compensation output formats

| Quantity | Type | Units | Conversion |
|----------|------|-------|------------|
| Temperature | `int32_t T` | 0.01 °C | divide by 100 |
| Pressure | `uint32_t P` | Q24.8 Pa | divide by 256 → Pa |
| Humidity | `uint32_t H` | Q22.10 %RH | divide by 1024 |

Pressure Q24.8 means the value is Pa × 256. Valid range: 7680000–28160000
(corresponding to 300–1100 hPa × 256).

#### Sea pressure and altitude (computed inline in main.c)

```c
float bme_press_pa     = (float)bme_sensor.measure->P / 256.0f;
float bme_press_msl_pa = bme_press_pa / powf(1.0f - (station_alt / 44330.0f), 5.255f);
float bme_altitude_m   = 44330.0f * (1.0f - powf(bme_press_pa / mqtt.qnh_ref_pa, 1.0f / 5.255f));
```

Both sensors share the same `qnh_ref_pa` EMA (derived from GPS + BMP180 QNH).

#### Known driver bugs fixed

**osrs clobber (root cause of 691 hPa output):**
`bme280_set_config` called `bme280_read_ctrl_meas` to preserve the current mode bits
via read-modify-write. But `bme280_read_ctrl_meas` had the side effect of writing the
entire register back to the struct — including `osrs_p` and `osrs_t`. At init time the
chip is at power-on-reset (`0xF4 = 0x00`), so both fields were overwritten with 0
("skip measurement"). The chip then returned the sentinel `0x80000` for every pressure
reading, which the compensation formula turned into ~691 hPa garbage.
Fixed by reading only the mode bits directly without touching the struct's osrs fields.

**Pressure Q24.8 format mishandled:**
Original code divided by 256 × 100 (treating output as Pa × 100) then divided again,
producing ~270 hPa. Fixed by storing raw Q24.8 from the Bosch formula and dividing by
256 at display/publish time.

---

### QFE vs QNH

| Symbol | Meaning | Use |
|--------|---------|-----|
| QFE | Absolute station pressure (Pa) | Raw sensor output |
| QNH | Pressure normalised to sea level (Pa) | Weather comparison, aviation |

Formula: `QNH = QFE / (1 - alt_m / 44330)^5.255`

### Why GPS altitude → QNH is not circular

GPS altitude provides the station height to normalise QFE → QNH.
Using that same QNH to compute barometric altitude would just return GPS altitude —
circular and useless.

Instead, a **slowly-adapting EMA reference** (`qnh_ref_pa`) is maintained:

```
α = 0.20  →  ~5-reading window  →  ~50 s convergence at 10 s intervals
qnh_ref = α × QNH_new + (1-α) × qnh_ref
```

`qnh_ref` is then used to compute barometric altitude independently of the current GPS fix.

**Before first GPS fix:** initialised to standard atmosphere (101325 Pa), giving
"pressure altitude" — accurate to ±0–300 m depending on weather.

**After GPS calibration:** converges to the site's true QNH. Subsequent altitude
readings track *weather-driven pressure deviations* (~±30–100 m with fronts).

---

## MQTT

**Broker:** Mosquitto on `fr3yr`, port 1883, anonymous auth.

### Topics

| Topic | Direction | Retain | Content |
|-------|-----------|--------|---------|
| `pico/weather-aux/state` | Publish | No | JSON state payload (all sensors) |
| `pico/weather-aux/status` | Publish | Yes | `"online"` / `"offline"` (LWT) |
| `gps/fr3yr/tpv` | Subscribe | Yes | GPS JSON (extract `alt` field) |
| `homeassistant/sensor/<id>/config` | Publish | Yes | HA auto-discovery (12 entities) |

### State payload

```json
{
  "bmp180_temperature": 22.5,
  "bmp180_pressure": 843.25,
  "bmp180_pressure_msl": 1012.34,
  "bmp180_altitude": 1457.1,
  "bme280_temperature": 20.7,
  "bme280_pressure": 842.10,
  "bme280_pressure_msl": 1011.05,
  "bme280_altitude": 1459.3,
  "bme280_humidity": 38.4,
  "battery": 91,
  "voltage": 4.08,
  "current": -5.0
}
```

Pressures in **hPa**, altitude in **metres**, temperature in **°C**,
humidity in **%RH**, current in **mA** (negative = discharging).

### HA auto-discovery entities

All entities belong to device `pico_weather_aux` ("Pico W Aux Weather Station").

| Entity ID | Name | Device class | Unit |
|-----------|------|-------------|------|
| `sensor.bmp180_temperature` | BMP180 Temperature | `temperature` | °C |
| `sensor.bmp180_pressure` | BMP180 Pressure (QFE) | `atmospheric_pressure` | hPa |
| `sensor.bmp180_pressure_msl` | BMP180 Pressure MSL (QNH) | `atmospheric_pressure` | hPa |
| `sensor.bmp180_altitude` | BMP180 Altitude | *(none)* | m |
| `sensor.bme280_temperature` | BME280 Temperature | `temperature` | °C |
| `sensor.bme280_pressure` | BME280 Pressure (QFE) | `atmospheric_pressure` | hPa |
| `sensor.bme280_pressure_msl` | BME280 Pressure MSL (QNH) | `atmospheric_pressure` | hPa |
| `sensor.bme280_altitude` | BME280 Altitude | *(none)* | m |
| `sensor.bme280_humidity` | BME280 Humidity | `humidity` | % |
| `sensor.pico_w_battery` | Battery | `battery` | % |
| `sensor.pico_w_voltage` | Battery Voltage | `voltage` | V |
| `sensor.pico_w_current` | Battery Current | `current` | mA |

### GPS altitude subscription

`gps/fr3yr/tpv` is the retained topic published by the gpsd bridge on `fr3yr`
(see `NIX_REPO/nix/services/gpsd.nix`). Payload format:

```json
{"lat": -25.755, "lon": 28.232, "alt": 1457.8, "speed": 0.1, "mode": 3, "nSat": 12, "uSat": 8}
```

The `alt` field is extracted with a simple `strstr` search — no JSON library.
Parsed in the lwIP MQTT incoming-data callback (IRQ context) and written to
`mqtt_ha_t.gps_altitude_m`.

---

## lwIP / CYW43 Threading Notes

The CYW43 driver runs in **threadsafe background** mode: the WiFi interrupt drives
lwIP from IRQ context. Any lwIP API call from the main thread must be wrapped:

```c
cyw43_arch_lwip_begin();
mqtt_publish(...);
cyw43_arch_lwip_end();
```

The MQTT callbacks (`connection_cb`, `inpub_request_cb`, `inpub_data_cb`) fire from
IRQ context where the lock is already held — do not call `lwip_begin` inside them.

Shared state between IRQ and main (`gps_altitude_m`, `gps_alt_valid`) is read inside
`cyw43_arch_lwip_begin/end`. `qnh_ref_pa` is only touched by main context so no
locking is needed.

---

## Home Assistant

**Config file:** `NIX_REPO/nix/services/home-assistant.nix`
**Lovelace view:** "Weather Station" (`/weather-station`)

Cards:
1. Entities — BMP180 current readings (T, QFE, QNH, altitude)
2. Entities — BME280 current readings (T, QFE, QNH, altitude, humidity)
3. Entities — Pico UPS-A battery (%, voltage, current)
4. History graph — Temperature & Humidity 24 h (BMP180 + BME280 temp, BME280 humidity)
5. History graph — Pressure 24 h (BMP180 + BME280 QFE and QNH)
6. History graph — Altitude 24 h (BMP180 + BME280)
7. History graph — Battery 24 h (% and voltage)

Lovelace is fully declarative (`lovelaceConfigWritable = false`), managed by Nix.
Deploy with `nixos-rebuild switch` on `fr3yr`.

---

## Waveshare Pico-UPS-A Battery Monitor

**Hardware (from schematic `docs/datasheets/pico-ups-a-schematic.pdf`):**

| Detail | Value |
|--------|-------|
| IC | INA219 (TI current/power monitor) |
| I2C bus | I2C1 — GP6 (SDA), GP7 (SCL) |
| I2C address | 0x43 (A0 = VS, A1 = GND — confirmed in hardware) |
| Shunt resistor | 10 mΩ (R1, 1%, 2512) |
| Charger IC | ETA6003 |
| Protection | S8261 + FS8205 |

**Driver:** `libs/ina219/`

**Calibration (Cal = 4096):**
- Current_LSB = 0.04096 / (4096 × 0.01 Ω) = **1 mA / LSB**
- Power_LSB = 20 × 1 mA = **20 mW / LSB**

**Battery % from voltage:** piecewise-linear LiPo curve (4.20 V = 100%, 3.00 V = 0%).

#### INA219 current reading is a point-in-time sample, not an average

The INA219 is read immediately before `mqtt_ha_publish_state()` — during the active
phase when the RP2040 is running and the CYW43 is transmitting. This captures peak
current (~50–60 mA), not nominal current.

Actual duty cycle per 10-second loop:

| Phase | Duration | Approx current |
|-------|----------|----------------|
| Active (measure + WiFi TX) | ~150 ms | ~60 mA |
| Sleep (RP2040 WFE, CYW43 PM) | ~9850 ms | ~20 mA |
| **Average** | | **~21 mA** |

CYW43 in `CYW43_DEFAULT_PM` wakes every ~100 ms for WiFi beacon; RP2040 `sleep_ms()`
uses WFE (Wait For Event) reducing core current but not to zero.
To measure true average current, sample continuously and average, or use a coulomb counter.

---

## Related Notes

- [[Weather Station Project]] — full project with wind/rain sensors
- [[GPS - DFRobot TEL0137 Setup & Troubleshooting]] — GPS module that provides altitude
