# 11 — Capstone Project: MQTT Sensor Aggregator

A command-line tool that subscribes to your MQTT broker, collects sensor readings from all machines, and produces a live terminal dashboard — or exports to a file.

This project is directly relevant to your home lab setup and touches every concept from the curriculum.

---

## What it does

```
$ mqtt-agg --broker fr3yr --topics "telegraf/+/cpu,telegraf/+/mem,telegraf/+/disk"

Live Sensor Dashboard — connected to fr3yr:1883
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  th0r      CPU  0.3%   RAM  9.2%   Disk  7.6%   ● online
  fr3yr     CPU  1.9%   RAM  9.0%   Disk 22.4%   ● online
  h31mda11  CPU 71.3%   RAM 40.9%   Disk 69.0%   ● online
  ba1dr     CPU  4.1%   RAM 31.2%   Disk 55.0%   ● online
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Last update: 2026-04-19 14:58:21  (refreshes every 30s)
```

---

## Skills used

| Concept | Where it appears |
|---------|-----------------|
| Structs & Enums | `SensorReading`, `MachineState`, `Topic` enum |
| Error handling | MQTT connection errors, JSON parse failures |
| Collections & Iterators | Aggregating readings per machine |
| Traits | `Display` for formatting, `Formatter` trait for output modes |
| Closures | MQTT message callbacks, iterator pipelines |
| Concurrency | Async MQTT subscription with `tokio` + `rumqttc` |
| Modules | Split into `mqtt`, `display`, `config`, `error` modules |
| Lifetimes | Zero-copy topic string slicing |

---

## Crates needed

```toml
[dependencies]
rumqttc  = "0.24"   # MQTT client
tokio    = { version = "1", features = ["full"] }
serde    = { version = "1", features = ["derive"] }
serde_json = "1"
clap     = { version = "4", features = ["derive"] }
thiserror = "1"
crossterm = "0.27"  # terminal control for the live dashboard
```

---

## Project structure

```
mqtt-agg/
├── Cargo.toml
└── src/
    ├── main.rs        ← CLI setup, starts async runtime
    ├── lib.rs         ← public API
    ├── config.rs      ← clap CLI args → Config struct
    ├── mqtt.rs        ← MQTT connection and subscription
    ├── parser.rs      ← JSON payload → SensorReading
    ├── state.rs       ← Arc<Mutex<HashMap>> of machine states
    ├── display.rs     ← terminal rendering
    └── error.rs       ← unified error type
```

---

## Build order

**Phase 1 — Config and connection**
1. Define `Config` with `clap` (broker host, port, topics, output mode)
2. Connect to MQTT with `rumqttc` using `tokio`
3. Subscribe to `telegraf/+/cpu`, `telegraf/+/mem`, `telegraf/+/disk`, `telegraf/+/status`
4. Print raw incoming messages to confirm it works

**Phase 2 — Parsing**
5. Define `SensorReading` struct with `serde` to deserialise the telegraf JSON payload
6. Define `MachineState` struct holding latest CPU, RAM, disk, and online status
7. Parse incoming messages and extract the machine name from the topic (`telegraf/th0r/cpu` → `th0r`)

**Phase 3 — State management**
8. Store machine states in `Arc<Mutex<HashMap<String, MachineState>>>` shared between the MQTT task and display task
9. Spawn two async tasks: one receiving MQTT messages and updating state, one refreshing the display

**Phase 4 — Display**
10. Use `crossterm` to clear and redraw the terminal table on each update
11. Colour-code values (green/yellow/red) based on thresholds matching your HA dashboard

**Phase 5 — Export mode**
12. Add a `--export csv` flag that writes readings to a file instead of the live display
13. Implement a `Formatter` trait with `LiveDisplay` and `CsvExport` implementations

---

## Stretch goals

- Alert mode: `--alert cpu:85` prints a warning line when any machine exceeds a threshold
- Historical min/max: track the session high/low for each metric
- Reconnect logic: automatically reconnect if the broker goes away, with exponential backoff
- Config file: load defaults from a TOML file (reuse lesson 04's config loader)
