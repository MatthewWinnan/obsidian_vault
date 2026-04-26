# GPS - DFRobot TEL0137 Setup & Troubleshooting

## Device

- **Product**: [DFRobot USB GPS Receiver Module (TEL0137)](https://www.dfrobot.com/product-2216.html)
- **Chip**: U-blox 7 (`hwVersion 00070000`)
- **Firmware**: `1.00 (59842)`, protocol version 14.00
- **Supported constellations** (per `MON-VER`): GPS, SBAS, GLONASS (GLO), QZSS
- **USB IDs**: `1546:01a7` (U-blox AG, product 01a7)
- **Interface**: USB CDC ACM — appears as `/dev/ttyACM0` on Linux; stable symlink configured at `/dev/ttyGPS` via udev rule
- **Script**: [DFRobotdl/USB_GPS_EN on GitHub](https://github.com/DFRobotdl/USB_GPS_EN)
- **Antenna**: Built-in ceramic patch — must face skyward

## Current Status

Integrated into **fr3yr** (home assistant server) via a 5m USB 2.0 extension cord run outdoors. The TEL0137 is managed by `gpsd` and GPS data is published to Home Assistant via MQTT auto-discovery. The USB cable run is temporary — rain will damage the exposed cable. The goal is to incorporate a **wireless GPS chip** as part of the permanent weather station build.

### NixOS Integration (fr3yr)
- `gpsd` reads from `/dev/ttyGPS` (stable udev symlink matching `1546:01a7`)
- A bridge service polls `gpspipe -w`, throttles to one publish every 10 seconds, and pushes JSON to `gps/fr3yr/tpv` on Mosquitto
- Home Assistant auto-discovers sensors: latitude, longitude, altitude, speed, fix mode, satellites in view/used, fix binary sensor, bridge status
- A `device_tracker` entity (`device_tracker.fr3yr_gps`) feeds the HA map card
- Configuration lives in `nix/services/gpsd.nix` in the NIX_REPO

## What "Invalid GPS Data" Actually Means

The script labels output as invalid when the GPS has no satellite fix. This is **not a script or hardware fault** — it's the GPS reporting that it hasn't locked onto satellites yet.

Key NMEA indicators of no fix:

| Field | Value | Meaning |
|-------|-------|---------|
| `$GPRMC` status | `V` | Void — no valid fix |
| `$GPGGA` fix quality | `0` | No fix |
| `$GPGGA` satellites | `00` | No satellites tracked |
| `$GPGSA` mode | `1` | Fix not available |

Once working, `$GPRMC` status changes to `A` (Active) and coordinates/time populate.

## Known Issues & Tips

### USB 3.0 Interference (Most Common Cause of No Fix Indoors)
USB 3.0 ports emit RF noise in the ~2.4GHz band that drowns out the weak GPS signal.
- **Fix**: Plug into a USB 2.0 port
- If only USB 3.0 is available, use a long USB extension cable to physically distance the dongle
- Keep away from USB 3.0 hubs, SSDs, and other USB 3 devices

### Cold Start — Be Patient
- **Cold start** (first use, or after long power-off): up to **15 minutes** outdoors to get a fix
- The module's internal backup battery stores almanac/ephemeris data — subsequent starts are much faster (~1 min warm start)
- If the backup battery has drained (device unused for extended period), treat as a cold start again

### Must Have Clear Sky View
- GPS signals cannot penetrate roofs or most walls
- A window may not be enough
- The ceramic patch antenna must point **face-up toward the sky**
- Avoid large metal surfaces nearby

### Linux Setup
- No driver needed — plug and play on Linux
- Verify detection: `dmesg | grep tty`
- For a more robust real-time view than raw serial: install `gpsd` and use `cgps`
- To interact with the U-blox chip directly: `ubxtool`

### GLONASS Cannot Be Enabled Concurrently — Hardware Limitation (Single-Constellation Chip)

The U-blox 7 is built on the **UBX-G7020 chip**, which is a **single-constellation receiver by hardware design** — per the [NEO-7 Data Sheet (UBX-13003830)](https://content.u-blox.com/sites/default/files/products/documents/NEO-7_DataSheet_(UBX-13003830).pdf), the receiver can track GPS, GLONASS, or Galileo, but only one at a time. This is a fundamental silicon limitation confirmed via `MON-VER`:

```
ubxtool -f /dev/ttyGPS -P 14 -p MON-VER
# swVersion 1.00 (59842) / hwVersion 00070000 / PROTVER 14.00
# extension: GPS;SBAS;GLO;QZSS
```

The `GLO` in the extension list means the chip *supports* GLONASS mode in principle, but whether it actually works depends on whether the module's RF hardware was designed for it (see below). Attempting to enable GLONASS via `CFG-GNSS` while GPS is active returns `UBX-ACK-NAK` — as detailed in the [NEO-7 Hardware Integration Manual (UBX-13003704)](https://content.u-blox.com/sites/default/files/products/documents/MAX7-NEO7_HardwareIntegrationManual_(UBX-13003704).pdf).

Concurrent multi-constellation support (GPS + GLONASS + Galileo/BeiDou simultaneously) only arrived with the **U-blox M8 platform** (UBX-M8030 chip, used in NEO-M8N and similar modules) — see [u-blox M8 Multi-GNSS Platform Offers Concurrent Tracking (GPS World)](https://www.gpsworld.com/u-blox-m8-multi-gnss-platform-offers-concurrent-tracking/). This is a hardware generational difference; no firmware update to the U-blox 7 can add concurrent constellation support.

#### Why the TEL0137 Likely Can't Do GLONASS Even in Switch Mode — RF Hardware

The datasheet note about "dedicated hardware preparation during the design-in phase" refers to three specific RF components that must be designed for **both** GPS L1 (1575.42 MHz) and GLONASS L1 (~1602 MHz) — frequencies that are 27 MHz apart. Per the [NEO-7 Hardware Integration Manual (UBX-13003704)](https://content.u-blox.com/sites/default/files/products/documents/MAX7-NEO7_HardwareIntegrationManual_(UBX-13003704).pdf) and the [GLONASS & GPS HW Design Application Note (GPS.G6-CS-10005)](https://content.u-blox.com/sites/default/files/products/documents/GLONASS-HW-Design_AppNote_(GPS.G6-CS-10005).pdf):

| Component | GPS-only design | GLONASS-ready design |
|---|---|---|
| **SAW filter** | Narrowband ~1575 MHz — physically blocks 1602 MHz GLONASS signals | Wideband or dual-band filter covering both 1575 and 1602 MHz |
| **Ceramic patch antenna** | Tuned to 1575 MHz, narrow bandwidth | Must cover 1602 MHz; u-blox recommends minimum 25×25×4mm for dual-band |
| **Active antenna LNA** | Internal bandpass filter passes GPS only | Filter must be wide enough to pass both bands |

The TEL0137 uses a small, cheap ceramic patch antenna that is almost certainly tuned for GPS L1 only. Even if the chip accepted a GLONASS-only `CFG-GNSS` configuration, the antenna and SAW filter would severely attenuate GLONASS signals before they reach the chip. The consistent `NAK` response is most likely the chip detecting via hardware configuration straps that the module was not prepared for GLONASS reception.

**Practical impact for a static installation in RSA**: GPS-only gives a solid 3D fix with 7–9 satellites and HDOP ~1.1, which is perfectly adequate for a stationary weather station. A newer NEO-M8N based module would give better DOP, faster cold starts, and improved reliability in marginal sky-view conditions — and would have the correct wideband RF frontend by design.

#### Switching to GLONASS-Only Mode (Future Reference — May Not Work Due to RF Hardware)

Although not currently needed (GPS is fine in RSA), it is theoretically possible to switch the chip to GLONASS-only by sending a `CFG-GNSS` message that disables GPS and enables GLONASS, followed by a save and cold reset. The chip must have GPS disabled in the same message — enabling GLONASS while GPS is active is what causes the NAK, per the [NEO-7 Hardware Integration Manual](https://content.u-blox.com/sites/default/files/products/documents/MAX7-NEO7_HardwareIntegrationManual_(UBX-13003704).pdf). **Note**: even if the chip accepts this config, actual GLONASS reception may be poor due to the GPS-only antenna and filter on the TEL0137 module.

Stop gpsd first, then run:

```python
import serial, struct, time

dev = '/dev/ttyGPS'

def ubx_msg(msg_class, msg_id, payload):
    length = len(payload)
    ck_a = ck_b = 0
    for b in bytes([msg_class, msg_id, length & 0xFF, (length >> 8) & 0xFF]) + payload:
        ck_a = (ck_a + b) & 0xFF
        ck_b = (ck_b + ck_a) & 0xFF
    return bytes([0xB5, 0x62, msg_class, msg_id, length & 0xFF, (length >> 8) & 0xFF]) + payload + bytes([ck_a, ck_b])

def send(s, msg, label):
    s.write(msg)
    time.sleep(0.5)
    resp = s.read(100)
    if b'\x05\x01' in resp:
        print(f"ACK — {label}")
        return True
    elif b'\x05\x00' in resp:
        print(f"NAK — {label}")
        return False
    else:
        print(f"No ACK/NAK for {label}: {resp.hex()}")
        return False

with serial.Serial(dev, 9600, timeout=2) as s:
    # CFG-GNSS: GPS/SBAS/QZSS disabled, GLONASS enabled
    # Source: NEO-7 Hardware Integration Manual (UBX-13003704), CFG-GNSS message spec
    header = struct.pack('<BBBB', 0, 22, 22, 4)
    blocks = [
        (0, 4,  255, 0, 0x00000000),  # GPS     - DISABLED
        (1, 1,  3,   0, 0x00000000),  # SBAS    - DISABLED (GPS-specific, useless without GPS)
        (5, 0,  3,   0, 0x00000000),  # QZSS    - DISABLED
        (6, 8,  255, 0, 0x00000001),  # GLONASS - ENABLED
    ]
    payload = header
    for gnssId, resTrkCh, maxTrkCh, reserved, flags in blocks:
        payload += struct.pack('<BBBBI', gnssId, resTrkCh, maxTrkCh, reserved, flags)

    ok = send(s, ubx_msg(0x06, 0x3E, payload), "CFG-GNSS GLONASS-only")
    if not ok:
        print("Aborting — chip rejected GNSS config")
        exit(1)

    # CFG-CFG: save to flash
    save_payload = struct.pack('<III', 0x00000000, 0xFFFFFFFF, 0x00000000)
    send(s, ubx_msg(0x06, 0x09, save_payload), "CFG-CFG save to flash")

    # CFG-RST: cold start + controlled software reset
    print("Sending reset...")
    reset_payload = struct.pack('<HBB', 0xFFFF, 0x02, 0x00)
    s.write(ubx_msg(0x06, 0x04, reset_payload))
    print("Reset sent — chip restarting in GLONASS-only mode")
```

To revert to GPS-only, swap the flags: GPS `0x00000001`, GLONASS `0x00000000`, and run the same sequence.

### Counterfeit Chipsets (Applies to Third-Party VK-172 Units)
Many VK-172-style dongles sold on Amazon have fake U-blox chips. The DFRobot TEL0137 should be genuine. Signs of a counterfeit:
- Doesn't respond to U-blox configuration commands
- Incorrect `GPTXT` messages in U-center software
- Test with [u-blox u-center](https://www.u-blox.com/en/product/u-center) on Windows

## What Worked

- Connecting the TEL0137 via a **5m USB 2.0 extension cord** and placing the module outside with clear sky view. Got a fix after waiting for the cold start to complete.
- Stable device path via udev rule matching `ATTRS{idVendor}=="1546", ATTRS{idProduct}=="01a7"` → symlink `/dev/ttyGPS`
- `gpsd` with `-n` flag (start reading immediately without waiting for a client) ensures a satellite fix is acquired on boot
- Typical fix quality on fr3yr: **3D DGPS FIX**, 7–9 satellites used, HDOP ~1.1, EPX/EPY ~2.5–3.5m

## Future Plan — Wireless GPS Integration

The USB cable run outdoors is not weatherproof long-term. Options to make GPS a permanent part of the weather station:

- **ESP32 + GPS module** (e.g., NEO-6M or NEO-M8N): reads GPS over UART, transmits data via WiFi/MQTT — fully wireless and weatherproof-enclosable
- **Separate weatherproof GPS enclosure** with the antenna mounted outdoors and data sent wirelessly to the main controller
- GPS data adds accurate timestamps and location to the MQTT payload — useful if the station is ever published publicly

## Research Links

### DFRobot TEL0137 / VK-172 Community Name
- [DFRobot TEL0137 Product Page](https://www.dfrobot.com/product-2216.html)
- [DFRobotdl USB GPS Script (GitHub)](https://github.com/DFRobotdl/USB_GPS_EN)
- [VK-172 GPS Review - John's Tech Blog](https://hagensieker.com/2024/01/02/vk-172-gps-review/)
- [VK-172 USB GPS on the Raspberry Pi - TeelSys](https://teelsys.com/vk-172-usb-gps-on-the-raspberry-pi/)
- [VK-172 GPS dongle on Mac - Jeff Geerling (2025)](https://www.jeffgeerling.com/blog/2025/trying-out-cheap-usb-vk-172-gps-dongle-on-mac/)
- [Cheap VK-172 GPS/GLONASS USB module - INDI Forum](https://indilib.org/forum/general/2799-cheap-vk-172-gps-glonass-usb-module-u-blox-chip-works-well.html)
- [u-blox VK-172 support community](https://portal.u-blox.com/s/topic/0TO2p000000Hrh3GAC/vk172)
- [DFRobot Forum - USB GPS not working](https://www.dfrobot.com/forum/topic/315788)

### U-blox 7 Official Documentation
- [NEO-7 Data Sheet (UBX-13003830)](https://content.u-blox.com/sites/default/files/products/documents/NEO-7_DataSheet_(UBX-13003830).pdf) — confirms single-constellation hardware design of the UBX-G7020 chip
- [NEO-7 Hardware Integration Manual (UBX-13003704)](https://content.u-blox.com/sites/default/files/products/documents/MAX7-NEO7_HardwareIntegrationManual_(UBX-13003704).pdf) — CFG-GNSS message spec, constellation switching behaviour, and RF hardware requirements
- [GLONASS & GPS HW Design Application Note (GPS.G6-CS-10005)](https://content.u-blox.com/sites/default/files/products/documents/GLONASS-HW-Design_AppNote_(GPS.G6-CS-10005).pdf) — detailed guidance on SAW filter, antenna, and LNA requirements for GLONASS-ready designs
- [NEO-7 Product Summary (UBX-13003342)](https://content.u-blox.com/sites/default/files/products/documents/NEO-7_ProductSummary_(UBX-13003342).pdf)

### UBX-G7020 Chip-Level Documentation (for posterity)
The UBX-G7020-KT is the bare silicon inside the TEL0137 module. These documents are chip-level (targeting PCB designers, distributed under NDA) and reveal the same conclusions as the NEO-7 module datasheet for end-use purposes — single constellation, GPS/GLONASS switch-mode, Galileo/BeiDou listed as hardware-ready but not implemented in firmware 1.00.
- [UBX-G7020-Kx Data Sheet — Confidential (GPS.G7-HW-12001)](http://innovictor.com/pdf/UBX-G7020-Kx_DataSheet_(GPS%20G7-HW-12001)_Confidential.pdf) — chip datasheet covering KT (commercial) and KA (automotive) QFN40 variants; confirms integrated TCXO and LNA, and that the RF frontend covers both L1 (1575 MHz) and G1 (1602 MHz) bands
- [UBX-G7020 Product Summary (UBX-13003349)](https://content.u-blox.com/sites/default/files/products/documents/UBX-G7020_ProductSummary_(UBX-13003349).pdf)
- [UBX-G7020 series — u-blox product page](https://www.u-blox.com/en/product/ubx-g7020-series) — end-of-life notice confirmed here

### Multi-Constellation Reference
- [u-blox M8 Multi-GNSS Platform Offers Concurrent Tracking — GPS World](https://www.gpsworld.com/u-blox-m8-multi-gnss-platform-offers-concurrent-tracking/) — explains the generational difference between U-blox 7 (single) and M8 (concurrent) platforms

---
[[Weather Station Project|← Back to Weather Station Project]]
