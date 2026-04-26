# Weather Station Project

Build and deploy a weather station with rain gauge, anemometer, and wind direction sensors to gather meteorological data. Data will be sent to Home Assistant via MQTT.

## Goals

- [ ] Collect local meteorological data (rainfall, wind speed, wind direction)
- [ ] Integrate with Home Assistant via MQTT broker
- [ ] Deploy a weatherproof outdoor installation
- [ ] Optionally publish readings publicly

## Hardware

### Sensors
- **Weather Meter Kit**: [SparkFun SEN-15901](https://www.robotics.org.za/SEN-15901?search=weather)
  - Rain gauge (tipping bucket)
  - Anemometer (wind speed)
  - Wind vane (wind direction)
  - Uses RJ11 connectors

- **GPS Receiver**: [DFRobot K-172 High Accuracy USB GPS Module](https://www.dfrobot.com/product-2216.html)
  - Uses U-blox chipset, GPS + GLONASS dual constellation
  - USB interface (shows up as `/dev/ttyACM0` or `/dev/ttyUSB0` on Linux)
  - Built-in backup battery retains almanac data between power cycles
  - See [[GPS - DFRobot K-172 Setup & Troubleshooting]] for full notes
  - **Current status**: Working temporarily via 5m USB 2.0 extension cord run outdoors — rain exposure risk, needs permanent wireless solution

### Controller
- **Enviro Weather Pico W Aboard**: [Pimoroni Board](https://www.pishop.co.za/store/enviro-weather-pico-w-aboard---board-only?keyword=Weather&category_id=0)
  - Built-in WiFi (Pico W)
  - Designed for weather sensing
  - MicroPython support
  - Additional onboard sensors (temperature, humidity, pressure, light)

### Power
- **18650 Battery Board**: [Single Cell 5V 2A Boost](https://www.robotics.org.za/18650-SINGLE-5V2A)
  - Single 18650 cell holder with built-in charging + boost converter
  - Outputs regulated 5V to power Pico W via USB/VSYS
  - USB charging - can charge while running
  - 18650 cells are cheap, high capacity (2000-3500mAh), replaceable

- **Battery Monitoring**: [INA219 Module](https://www.robotics.org.za/INA219-MOD?search=ina219)
  - I2C current/voltage/power sensor
  - Connect on battery side (before boost converter) for accurate level
  - Measures both voltage and current draw
  - Can calculate remaining capacity based on discharge rate

- **Future**: Solar panel + charge controller for autonomous operation

**Battery Level → Home Assistant:**
- Report voltage/percentage via MQTT
- Set up automation to notify when below threshold (e.g., 20%)
- Track discharge rate to predict when charging needed

## Architecture

```
┌─────────────────┐     RJ11      ┌──────────────────┐
│  Weather Meter  │──────────────▶│  Enviro Weather  │
│  Kit (SEN-15901)│               │  Pico W Aboard   │
└─────────────────┘               └────────┬─────────┘
                                           │
                                  ┌────────┴─────────┐
                                  │  18650 + Boost   │
                                  │  + INA219 Monitor│
                                  └────────┬─────────┘
                                           │ WiFi
                                           ▼
                                  ┌──────────────────┐
                                  │   MQTT Broker    │
                                  │ (Home Assistant) │
                                  └────────┬─────────┘
                                           │
                                           ▼
                                  ┌──────────────────┐
                                  │  Home Assistant  │
                                  │   Dashboard      │
                                  │  • Weather data  │
                                  │  • Battery alert │
                                  └──────────────────┘
```

## Enclosure Options

### Option 1: Stevenson Screen (3D Printed)
A louvered enclosure designed to shield sensors from direct sunlight and precipitation while allowing airflow.

**Pros:**
- Professional meteorological design
- Excellent ventilation for accurate readings
- Protects from direct rain and sun
- Looks great

**Cons:**
- Requires significant 3D printing time
- Multiple parts to assemble
- May need UV-resistant filament (ASA/PETG)

**Printable Designs:**
- [Thingiverse Stevenson Screen](https://www.thingiverse.com/search?q=stevenson+screen)
- [Printables Weather Station](https://www.printables.com/search/models?q=stevenson%20screen)
- Mini Stevenson screens designed for Raspberry Pi weather stations

### Option 2: Junction Box / Electrical Enclosure
An IP-rated waterproof electrical enclosure.

**Pros:**
- Readily available, cheap
- Already weatherproof (IP65/IP66)
- Easy to mount
- Quick deployment

**Cons:**
- Poor ventilation (affects temp/humidity readings)
- Not designed for airflow
- May trap heat

**Improvements:**
- Add ventilation holes with mesh/filters
- Use white box to reduce heat absorption
- Mount in shaded location

### Option 3: Hybrid Approach
Use a junction box for the Pico W board, with sensors mounted externally.

**Pros:**
- Electronics protected in sealed box
- Sensors get proper exposure
- Flexible mounting options

**Cons:**
- More complex cable routing
- Need weatherproof cable glands

### Option 4: Commercial Weather Station Mount
Purchase a proper weather station mounting bracket/shield.

**Pros:**
- Designed for purpose
- Durable

**Cons:**
- Additional cost
- May not fit custom setup

## Recommended Approach

1. **For the Pico W board**: Junction box with cable glands (IP65 rated)
2. **For temp/humidity sensor**: Small 3D printed Stevenson screen or radiation shield
3. **For wind/rain sensors**: Mount on pole as per SparkFun guidelines (these are already weatherproof)

## Planning Steps

### Phase 1: Hardware Setup
- [ ] Order weather meter kit (SEN-15901)
- [ ] Order Enviro Weather Pico W board
- [ ] Order 18650 battery board + 18650 cell
- [ ] Order INA219 battery monitoring module
- [ ] Order enclosure materials (junction box + cable glands)
- [ ] Order mounting hardware (pole, brackets)

### Phase 2: Software Development
- [ ] Set up MicroPython on Pico W
- [ ] Write sensor reading code for weather meter kit
- [ ] Implement battery level monitoring
- [ ] Configure MQTT client
- [ ] Test connection to Home Assistant MQTT broker
- [ ] Implement reading intervals and averaging
- [ ] Add deep sleep between readings to conserve battery

### Phase 3: Enclosure & Mounting
- [ ] Design/print or purchase enclosure
- [ ] Drill holes for cable glands
- [ ] Waterproof all connections
- [ ] Select mounting location (clear of obstructions)
- [ ] Install pole/mount

### Phase 4: Home Assistant Integration
- [ ] Configure MQTT sensors in Home Assistant
- [ ] Create weather dashboard
- [ ] Add battery level sensor/gauge
- [ ] Set up historical data logging
- [ ] Configure alerts (high wind, heavy rain, low battery)

### Phase 5: Optional - Public Publishing
- [ ] Decide on publishing platform (Weather Underground, PWSWeather, etc.)
- [ ] Implement data upload
- [ ] Register station

## Research Links

- [SparkFun Weather Meter Hookup Guide](https://learn.sparkfun.com/tutorials/weather-meter-hookup-guide)
- [Pimoroni Enviro Weather](https://shop.pimoroni.com/products/enviro-weather)
- [Enviro Weather GitHub](https://github.com/pimoroni/enviro)
- [Home Assistant MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)

### Phase 6: Future - Solar Power
- [ ] Research solar panel sizing for Pico W power draw
- [ ] Select charge controller (TP4056 or similar)
- [ ] Design mounting for solar panel
- [ ] Implement autonomous charging

### Phase 7: Future - GPS Wireless Integration
- [ ] Replace USB GPS with wireless GPS solution (ESP32 with GPS module, or similar)
- [ ] Transmit GPS data wirelessly to avoid cable exposure to weather
- [ ] Weatherproof GPS antenna mount with clear sky view
- [ ] Integrate GPS timestamp and location into MQTT data payload

### Phase 8: Future - Additional Sensors
- [ ] Integrate lightning detector
- [ ] Integrate rain/snow sensor with heating
- [ ] Update Home Assistant dashboard with new sensors
- [ ] Add lightning strike alerts

## Future Extensions

### Lightning Detector
- **SparkFun AS3935**: [SEN-15441](https://www.robotics.org.za/SEN-15441)
  - Detects lightning up to 40km away
  - I2C or SPI interface
  - Distinguishes lightning from man-made disturbers
  - Reports estimated distance to storm front

### Industrial Rain/Snow Sensor
- **Rain Sensor with Relay & RS485**: [Robotics.org.za](https://www.robotics.org.za/)
  - IP67 waterproof ABS housing
  - Spiral metal coil sensing element
  - Auto-heating function to prevent icing in winter
  - Relay output for external sound/light alarm
  - RS485 output for data integration
  - More robust than tipping bucket for detecting precipitation start/stop

**Integration notes:**
- AS3935 can connect via I2C alongside INA219
- RS485 sensor may need a MAX485 or similar TTL converter for Pico
- Consider separate alerts for lightning proximity vs precipitation

## Notes

- Wind sensors need to be mounted 10m above ground ideally, or at least clear of obstructions
- Rain gauge needs to be level and away from overhangs
- Use deep sleep aggressively - wake every 5-15 min, read sensors, publish, sleep
- 18650 cells don't like extreme temperatures - consider insulation in enclosure
- Future: Solar panel + charge controller for autonomous operation

---
[[index|← Back to Home]]