# Home Assistant Setup

Declarative Home Assistant deployment on Raspberry Pi 4 (fr3yr) running NixOS, with configuration managed in Git.

## Status: ✅ Complete (2026-04-06)

## Goals

- [x] Run Home Assistant on Raspberry Pi 4 with NixOS
- [x] Fully declarative configuration stored in Git
- [x] Expose through [[Proxy Server Setup|Caddy reverse proxy]] via Tailscale
- [x] Integrate with MQTT, Zigbee, and other home automation protocols

## Architecture

```
Zigbee Devices → Zigbee Dongle → fr3yr (NixOS + Home Assistant)
                                         ↓
                              Tailscale → th0r (Caddy Proxy) → Access
```

## Current Configuration

### Machine: fr3yr
- **Hardware**: Raspberry Pi 4
- **OS**: NixOS (aarch64-linux)
- **Kernel**: linux_rpi4 optimized kernel
- **WiFi**: iwd with encrypted credentials via sops-nix
- **Secrets**: sops-nix with age encryption (SSH host key)

### Services
- **Home Assistant**: Port 8123
- **Mosquitto MQTT**: Port 1883 (anonymous access for local network)
- **Tailscale**: Mesh networking for remote access

### Key Files in NIX_REPO
```
machines/fr3yr/
├── default.nix
├── nix/
│   ├── boot.nix              # Pi 4 kernel & bootloader
│   ├── default.nix           # Main imports
│   ├── hardware-configuration.nix
│   ├── networking.nix        # iwd WiFi config
│   └── sops.nix              # Secrets configuration
└── settings/
    ├── config.nix
    └── default.nix

nix/services/
├── home-assistant.nix        # HA service + Mosquitto
└── tailscale.nix             # Tailscale service

secrets/
└── fr3yr.yaml                # Encrypted secrets (sops)
```

---

## Deployment

### Remote Updates via SSH

From your local machine:

```bash
# Build on fr3yr itself (recommended for cross-arch)
nixos-rebuild switch --flake .#fr3yr \
  --target-host root@fr3yr \
  --build-host root@fr3yr
```

Or via IP:
```bash
nixos-rebuild switch --flake .#fr3yr \
  --target-host root@192.168.101.150 \
  --build-host root@192.168.101.150
```

### Rollback

```bash
# On fr3yr, list generations
sudo nix-env --list-generations -p /nix/var/nix/profiles/system

# Rollback to previous
sudo nixos-rebuild switch --rollback
```

---

## Secrets Management (sops-nix)

### Configuration

Secrets are encrypted with age using fr3yr's SSH host key.

**.sops.yaml** (in repo root):
```yaml
keys:
  - &admin age1...  # Your personal age key
  - &fr3yr age1...  # fr3yr's SSH host key converted to age

creation_rules:
  - path_regex: secrets/fr3yr\.yaml$
    key_groups:
      - age:
          - *admin
          - *fr3yr
```

### Encrypted Secrets (secrets/fr3yr.yaml)

Contains:
- `home-assistant-secrets`: latitude, longitude, elevation for HA
- `iwd-network`: WiFi credentials for iwd

### Converting SSH Host Key to Age

```bash
ssh-keyscan fr3yr 2>/dev/null | nix-shell -p ssh-to-age --run ssh-to-age
```

### Editing Secrets

```bash
sops secrets/fr3yr.yaml
```

---

## Proxy Access (via th0r)

Home Assistant is accessible via Tailscale through th0r's Caddy proxy:

```
https://homeassistant.th0r.<tailnet>.ts.net/
```

See [[Proxy Server Setup]] for th0r configuration details.

---

## Data Persistence & Migration

### What's Stored Where

| Data | Location | In Nix Config? |
|------|----------|----------------|
| Base config | `configuration.yaml` | ✅ Yes (via Nix) |
| Secrets | `/var/lib/hass/secrets.yaml` | ✅ Yes (via sops) |
| Dashboards (UI mode) | `/var/lib/hass/.storage/lovelace*` | ❌ No |
| Automations (UI) | `/var/lib/hass/.storage/core.config_entries` | ❌ No |
| History/States | `/var/lib/hass/home-assistant_v2.db` | ❌ No |
| Device registry | `/var/lib/hass/.storage/core.device_registry` | ❌ No |
| User accounts | `/var/lib/hass/.storage/auth*` | ❌ No |

### Migration to New Machine

```bash
# On old machine
sudo tar -czvf hass-backup.tar.gz /var/lib/hass

# On new machine (after deploying NixOS config)
sudo systemctl stop home-assistant
sudo tar -xzvf hass-backup.tar.gz -C /
sudo chown -R hass:hass /var/lib/hass
sudo systemctl start home-assistant
```

### Declarative Dashboards (TODO)

Dashboards can be made fully reproducible using YAML mode instead of UI:

**In `configuration.yaml`:**
```yaml
lovelace:
  mode: yaml
  dashboards:
    lovelace-home:
      mode: yaml
      filename: dashboards/home.yaml
      title: Home
      icon: mdi:home
```

**Dashboard file (`dashboards/home.yaml`):**
```yaml
views:
  - title: Home
    cards:
      - type: entities
        title: Lights
        entities:
          - light.living_room
          - light.bedroom

      - type: weather-forecast
        entity: weather.home
```

This would allow dashboards to be stored in the Nix repo and deployed declaratively.

### Backup Strategy (TODO)

- [ ] Set up Home Assistant's built-in backups (Settings → System → Backups)
- [ ] Configure automated backup to external storage
- [ ] Consider declarative dashboards for full reproducibility

---

## Declarative MQTT Sensors

MQTT sensors can be configured declaratively in NixOS via `services.home-assistant.config`.

**In `nix/services/home-assistant.nix`:**

```nix
services.home-assistant.config = {
  # MQTT integration
  mqtt = {
    sensor = [
      {
        name = "Living Room Temperature";
        state_topic = "home/livingroom/temperature";
        unit_of_measurement = "°C";
        device_class = "temperature";
      }
      {
        name = "Living Room Humidity";
        state_topic = "home/livingroom/humidity";
        unit_of_measurement = "%";
        device_class = "humidity";
      }
    ];

    binary_sensor = [
      {
        name = "Front Door";
        state_topic = "home/frontdoor/contact";
        device_class = "door";
        payload_on = "open";
        payload_off = "closed";
      }
    ];

    switch = [
      {
        name = "Garden Light";
        command_topic = "home/garden/light/set";
        state_topic = "home/garden/light/state";
      }
    ];
  };
};
```

### Available MQTT Platforms

| Platform | Use Case |
|----------|----------|
| `sensor` | Read values (temperature, humidity, etc.) |
| `binary_sensor` | On/off states (doors, motion, etc.) |
| `switch` | Controllable on/off devices |
| `light` | Controllable lights (brightness, color) |
| `climate` | HVAC / thermostats |
| `cover` | Blinds, garage doors |
| `fan` | Fan control |
| `lock` | Smart locks |

### Resources
- [Home Assistant MQTT Sensor](https://www.home-assistant.io/integrations/sensor.mqtt/)
- [Home Assistant MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)

---

## Troubleshooting

### Home Assistant won't start
```bash
systemctl status home-assistant
journalctl -u home-assistant -f
```

### Missing integration
Add to `extraComponents` in `nix/services/home-assistant.nix` and rebuild.

### Secrets not loading
```bash
# Check if sops decrypted the file
cat /var/lib/hass/secrets.yaml

# Check sops service
systemctl status sops-nix
```

### WiFi not connecting
```bash
# Check iwd status
systemctl status iwd
iwctl station wlan0 show

# Check if network profile exists
ls -la /var/lib/iwd/
```

### Test MQTT
```bash
# Subscribe (terminal 1)
mosquitto_sub -h localhost -t "test/#" -v

# Publish (terminal 2)
mosquitto_pub -h localhost -t "test/topic" -m "hello"
```

---

## Resources
- [NixOS Home Assistant Module](https://wiki.nixos.org/wiki/Home_Assistant)
- [Home Assistant Component Packages](https://github.com/NixOS/nixpkgs/blob/master/pkgs/servers/home-assistant/component-packages.nix)
- [sops-nix](https://github.com/Mic92/sops-nix)
- [NixOS on Raspberry Pi 4](https://wiki.nixos.org/wiki/NixOS_on_ARM/Raspberry_Pi_4)

---

[[Proxy Server Setup|← Proxy Server Setup]] | [[index|← Home]]
