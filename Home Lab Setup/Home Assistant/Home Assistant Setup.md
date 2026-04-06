# Home Assistant Setup

Declarative Home Assistant deployment on Raspberry Pi 4 running NixOS, with configuration managed in Git.

## Goals

- Run Home Assistant on Raspberry Pi 4 with NixOS
- Fully declarative configuration stored in GitHub
- Expose through [[Proxy Server Setup|Caddy reverse proxy]] via Tailscale
- Integrate with MQTT, Zigbee, and other home automation protocols

## Architecture

```
Zigbee Devices → Zigbee Dongle → Pi 4 (NixOS + Home Assistant)
                                         ↓
                              Tailscale → Caddy Proxy → Access
```

---

## Part 1: Prepare the Raspberry Pi 4

### 1.1 Download NixOS AArch64 Image

```bash
# Download the latest NixOS SD image for aarch64
wget https://hydra.nixos.org/job/nixos/release-24.11/nixos.sd_image.aarch64-linux/latest/download/1
# Or use the direct link from releases page
```

**Alternative:** Build a custom image with your config pre-baked (see Part 5).

### 1.2 Flash to SD Card or USB SSD

```bash
# Using dd (Linux) - same command for SD card or SSD
sudo dd if=nixos-sd-image-*.img of=/dev/sdX bs=4M status=progress

# Or use balenaEtcher / Raspberry Pi Imager
```

### 1.3 USB SSD Boot (Recommended)

Booting from USB SSD provides better performance and reliability than SD cards.

#### Prerequisites

**Update Pi 4 Bootloader (one-time setup):**

The Pi 4 bootloader must be configured for USB boot. If you have older firmware:

**Option A: Via Raspberry Pi OS**
1. Flash Raspberry Pi OS Lite to an SD card
2. Boot the Pi and run:
```bash
sudo apt update && sudo apt upgrade -y
sudo rpi-eeprom-update -a
sudo reboot
```
3. Configure boot order:
```bash
sudo raspi-config
# → Advanced Options → Boot Order → USB Boot
```

**Option B: Via Raspberry Pi Imager**
- Use "Misc utility images" → "Bootloader" to update EEPROM directly

#### Power Requirements (Critical)

USB SSDs draw significant power. Without adequate power you'll get random corruption and failures.

| Solution | Notes |
|----------|-------|
| Official Pi 4 PSU (5.1V 3A) | Minimum requirement |
| Powered USB hub | Between Pi and SSD, most reliable |
| Pi 4 with USB-C PD | Some third-party cases support this |

#### Flash and Boot

Once bootloader is configured:

```bash
# Flash NixOS image to SSD (same image as SD card)
sudo dd if=nixos-sd-image-*.img of=/dev/sdX bs=4M status=progress

# Remove SD card, connect SSD, power on
```

#### Hardware Configuration for SSD

Update `hardware-configuration.nix` to use stable device identifiers:

```nix
fileSystems."/" = {
  # Use label (works for both SD and SSD)
  device = "/dev/disk/by-label/NIXOS_SD";
  # Or use UUID for maximum reliability:
  # device = "/dev/disk/by-uuid/your-uuid-here";
  fsType = "ext4";
};
```

Find your UUID with: `lsblk -f`

#### Resources
- [NixOS Pi 4 SSD Discussion](https://discourse.nixos.org/t/nixos-on-rpi4-ssd/34522)
- [Pi 4 SSD Install Guide (GitHub Gist)](https://gist.github.com/chrisguida/170e4ff6b48aabeb0ab0351f82878403)

### 1.4 Initial Boot and SSH Access

1. Insert SD card and power on Pi 4
2. Connect via HDMI for initial setup, or...
3. Mount the SD card and pre-configure SSH:

```bash
# Mount the SD card's root partition
sudo mount /dev/sdX2 /mnt

# Enable SSH by adding to configuration.nix
sudo nano /mnt/etc/nixos/configuration.nix
```

Add:
```nix
services.openssh.enable = true;
users.users.root.openssh.authorizedKeys.keys = [
  "ssh-ed25519 AAAA... your-key-here"
];
```

### Resources
- [NixOS on Raspberry Pi 4](https://wiki.nixos.org/wiki/NixOS_on_ARM/Raspberry_Pi_4)
- [Installing NixOS on Raspberry Pi](https://nix.dev/tutorials/nixos/installing-nixos-on-a-raspberry-pi.html)
- [NixOS AArch64 Images](https://hydra.nixos.org/job/nixos/release-24.11/nixos.sd_image.aarch64-linux)

---

## Part 2: Create Declarative NixOS Configuration

### 2.1 Repository Structure

Create a GitHub repository for your configuration:

```
home-assistant-nixos/
├── flake.nix              # Flake entry point
├── flake.lock             # Locked dependencies
├── configuration.nix      # Main system config
├── home-assistant.nix     # HA-specific config
├── hardware-configuration.nix  # Pi 4 hardware
└── secrets/               # Encrypted secrets (use agenix/sops)
    └── secrets.yaml
```

### 2.2 Initialize Flake

**flake.nix:**
```nix
{
  description = "Home Assistant on Raspberry Pi 4";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  };

  outputs = { self, nixpkgs }: {
    nixosConfigurations.homeassistant = nixpkgs.lib.nixosSystem {
      system = "aarch64-linux";
      modules = [
        ./configuration.nix
        ./home-assistant.nix
      ];
    };
  };
}
```

### 2.3 Base Configuration

**configuration.nix:**
```nix
{ config, pkgs, ... }:

{
  imports = [
    ./hardware-configuration.nix
  ];

  # Bootloader for Pi 4
  boot.loader.grub.enable = false;
  boot.loader.generic-extlinux-compatible.enable = true;

  # Networking
  networking.hostName = "homeassistant";
  networking.networkmanager.enable = true;

  # Tailscale for remote access
  services.tailscale.enable = true;

  # SSH access
  services.openssh = {
    enable = true;
    settings.PasswordAuthentication = false;
  };

  # User configuration
  users.users.admin = {
    isNormalUser = true;
    extraGroups = [ "wheel" "dialout" ];  # dialout for Zigbee/serial
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAA... your-key"
    ];
  };

  # System packages
  environment.systemPackages = with pkgs; [
    vim
    git
    htop
  ];

  # Firewall - allow Home Assistant
  networking.firewall.allowedTCPPorts = [ 8123 ];

  system.stateVersion = "24.11";
}
```

### 2.4 Hardware Configuration

**hardware-configuration.nix:**
```nix
{ config, lib, pkgs, modulesPath, ... }:

{
  imports = [
    (modulesPath + "/installer/sd-card/sd-image-aarch64.nix")
  ];

  # Pi 4 specific
  hardware.raspberry-pi."4".fkms-3d.enable = true;

  # If using Bluetooth (for some Zigbee adapters)
  hardware.bluetooth.enable = true;

  # File systems (generated, adjust as needed)
  fileSystems."/" = {
    device = "/dev/disk/by-label/NIXOS_SD";
    fsType = "ext4";
  };
}
```

### Resources
- [NixOS Flakes](https://wiki.nixos.org/wiki/Flakes)
- [Nix Flakes Tutorial](https://nix.dev/tutorials/nix-language)

---

## Part 3: Configure Home Assistant

### 3.1 Home Assistant NixOS Module

**home-assistant.nix:**
```nix
{ config, pkgs, ... }:

{
  services.home-assistant = {
    enable = true;

    # Extra components to include
    extraComponents = [
      # Core integrations
      "default_config"
      "met"           # Weather
      "radio_browser" # Media

      # Common integrations
      "mqtt"
      "zha"           # Zigbee Home Automation
      "esphome"
      "mobile_app"

      # Add more as needed
      # Full list: https://github.com/NixOS/nixpkgs/blob/master/pkgs/servers/home-assistant/component-packages.nix
    ];

    # Extra Python packages not in nixpkgs
    extraPackages = python3Packages: with python3Packages; [
      # Add any additional Python dependencies here
    ];

    # Home Assistant YAML config (converted from Nix)
    config = {
      homeassistant = {
        name = "Home";
        unit_system = "metric";
        temperature_unit = "C";
        time_zone = "Africa/Johannesburg";  # Adjust to your timezone
        latitude = "!secret latitude";
        longitude = "!secret longitude";
      };

      # Enable frontend
      frontend = {};

      # Enable mobile app support
      mobile_app = {};

      # HTTP configuration for reverse proxy
      http = {
        use_x_forwarded_for = true;
        trusted_proxies = [
          "127.0.0.1"
          "100.0.0.0/8"  # Tailscale range
        ];
      };

      # Recorder for history (SQLite by default)
      recorder = {};

      # Logbook
      logbook = {};

      # History
      history = {};

      # Automation - can define here or in HA UI
      automation = "!include automations.yaml";
      scene = "!include scenes.yaml";
      script = "!include scripts.yaml";
    };
  };

  # Ensure required YAML files exist
  systemd.tmpfiles.rules = [
    "f /var/lib/hass/automations.yaml 0644 hass hass - []"
    "f /var/lib/hass/scenes.yaml 0644 hass hass - []"
    "f /var/lib/hass/scripts.yaml 0644 hass hass - []"
    "f /var/lib/hass/secrets.yaml 0600 hass hass -"
  ];
}
```

### 3.2 Secrets Management

Use [agenix](https://github.com/ryantm/agenix) or [sops-nix](https://github.com/Mic92/sops-nix) for secrets:

**With agenix:**
```nix
# In flake.nix inputs
agenix.url = "github:ryantm/agenix";

# In configuration
age.secrets.ha-secrets = {
  file = ./secrets/ha-secrets.age;
  path = "/var/lib/hass/secrets.yaml";
  owner = "hass";
};
```

### 3.3 Zigbee Support (Optional)

For Zigbee devices using zigbee2mqtt:

```nix
services.zigbee2mqtt = {
  enable = true;
  settings = {
    homeassistant = true;
    permit_join = false;
    serial = {
      port = "/dev/ttyUSB0";  # Adjust for your dongle
    };
    mqtt = {
      server = "mqtt://localhost:1883";
    };
    frontend = {
      port = 8080;
    };
  };
};

# MQTT broker
services.mosquitto = {
  enable = true;
  listeners = [{
    port = 1883;
    users.homeassistant = {
      acl = [ "readwrite #" ];
      hashedPassword = "$7$101$...";  # Use mosquitto_passwd to generate
    };
  }];
};
```

### Resources
- [NixOS Home Assistant Module](https://wiki.nixos.org/wiki/Home_Assistant)
- [Home Assistant Component Packages](https://github.com/NixOS/nixpkgs/blob/master/pkgs/servers/home-assistant/component-packages.nix)
- [agenix - Age-encrypted secrets](https://github.com/ryantm/agenix)
- [sops-nix - Mozilla SOPS secrets](https://github.com/Mic92/sops-nix)

---

## Part 4: Deploy and Update

### 4.1 Initial Deployment

From your local machine (with Nix installed):

```bash
# Clone your repo
git clone https://github.com/yourusername/home-assistant-nixos
cd home-assistant-nixos

# Build the image
nix build .#nixosConfigurations.homeassistant.config.system.build.sdImage

# Flash to USB SSD (or SD card)
sudo dd if=result/sd-image/*.img of=/dev/sdX bs=4M status=progress
```

**Note:** The image is called "sdImage" but works for both SD cards and USB SSDs.

### 4.2 Remote Updates via SSH

After initial boot, update remotely:

```bash
# From your local machine
nixos-rebuild switch --flake .#homeassistant \
  --target-host admin@homeassistant.tail-name.ts.net \
  --use-remote-sudo
```

### 4.3 Automated Updates with GitHub Actions

**.github/workflows/deploy.yml:**
```yaml
name: Deploy to Home Assistant Pi

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: cachix/install-nix-action@v27
        with:
          extra_nix_config: |
            experimental-features = nix-command flakes

      - name: Build configuration
        run: nix build .#nixosConfigurations.homeassistant.config.system.build.toplevel

      # Add deployment step with Tailscale/SSH
```

### 4.4 Rollback

```bash
# On the Pi, list generations
sudo nix-env --list-generations -p /nix/var/nix/profiles/system

# Rollback to previous
sudo nixos-rebuild switch --rollback

# Or boot into previous generation from bootloader menu
```

### Resources
- [Remote NixOS Deployment](https://wiki.nixos.org/wiki/Nixos-rebuild)
- [deploy-rs](https://github.com/serokell/deploy-rs) - Alternative deployment tool

---

## Part 5: Integrate with Reverse Proxy

Connect Home Assistant to your [[Proxy Server Setup|Caddy reverse proxy]]:

### 5.1 Tailscale on the Pi

Already configured in Part 2. Verify:
```bash
tailscale status
```

### 5.2 Add to Caddy Configuration

On your proxy server, add Home Assistant routing:

```caddyfile
homeassistant.myproxy.tail-name.ts.net {
    reverse_proxy homeassistant.tail-name.ts.net:8123 {
        # WebSocket support for HA frontend
        header_up Host {upstream_hostport}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
}
```

Or path-based:
```caddyfile
myproxy.tail-name.ts.net {
    handle_path /homeassistant/* {
        reverse_proxy homeassistant.tail-name.ts.net:8123
    }
}
```

### 5.3 Home Assistant HTTP Config

Ensure `http` config in home-assistant.nix allows the proxy (already configured in Part 3):

```nix
http = {
  use_x_forwarded_for = true;
  trusted_proxies = [
    "100.0.0.0/8"  # Tailscale range
  ];
};
```

---

## Implementation Checklist

### Phase 1: Hardware & Base OS
- [x] Acquire Raspberry Pi 4 (4GB+ RAM recommended) ✅ 2026-04-06
- [x] Get USB SSD + enclosure (or quality SD card as fallback) ✅ 2026-04-06
- [x] Update Pi 4 bootloader for USB boot (one-time) ✅ 2026-04-06
- [x] Ensure adequate power supply (3A+ or powered USB hub) ✅ 2026-04-06
- [x] Download/build NixOS AArch64 image ✅ 2026-04-06
- [x] Flash to SSD and boot Pi ✅ 2026-04-06
- [x] Verify SSH access ✅ 2026-04-06

### Phase 2: Git Repository
- [x] Create GitHub repository ✅ 2026-04-06
- [x] Initialize flake structure ✅ 2026-04-06
- [ ] Add base configuration.nix
- [ ] Add hardware-configuration.nix
- [ ] Test local build

### Phase 3: Home Assistant
- [ ] Add home-assistant.nix module
- [ ] Configure required integrations
- [ ] Set up secrets management (agenix/sops)
- [ ] Deploy to Pi and verify HA starts
- [ ] Access HA web UI on port 8123

### Phase 4: Integrations
- [ ] Install Zigbee dongle (if using)
- [ ] Configure zigbee2mqtt or ZHA
- [ ] Set up MQTT broker
- [ ] Add devices to Home Assistant

### Phase 5: Remote Access
- [ ] Join Pi to Tailscale network
- [ ] Configure Caddy reverse proxy routing
- [ ] Test access via proxy
- [ ] Set up Tailscale Serve/Funnel if needed

### Phase 6: Automation
- [ ] Set up GitHub Actions for CI
- [ ] Configure automated deployment
- [ ] Test rollback procedure
- [ ] Document any manual steps

---

## Recommended Hardware

| Component | Recommendation | Notes |
|-----------|---------------|-------|
| **Pi 4** | 4GB or 8GB model | 8GB if running many integrations |
| **Boot Drive** | USB SSD (recommended) | Much faster/reliable than SD card |
| **SSD** | Any SATA SSD + USB 3.0 enclosure | Or NVMe with USB adapter |
| **SD Card** | Samsung EVO Plus 32GB A2 | Only if not using SSD |
| **Power Supply** | Official Pi 4 PSU (5.1V 3A) | Critical for SSD stability |
| **Powered USB Hub** | Any quality USB 3.0 hub | Recommended for SSD reliability |
| **Zigbee Dongle** | Sonoff ZBDongle-P or SLZB-06 | Avoid CC2531 (outdated) |
| **Case** | Argon One M.2 | Built-in M.2 SATA slot |

---

## Troubleshooting

### Home Assistant won't start
```bash
# Check service status
systemctl status home-assistant

# View logs
journalctl -u home-assistant -f
```

### Missing integration
Add to `extraComponents` in home-assistant.nix and rebuild.

### Zigbee dongle not found
```bash
# Check USB devices
lsusb

# Check serial devices
ls -la /dev/ttyUSB* /dev/ttyACM*

# Ensure user is in dialout group
```

---

[[Proxy Server Setup|← Proxy Server Setup]] | [[index|← Home]]
