# Network Expansion - SBC Selection

Planning document for adding an access point and media box to the home network using available single-board computers.

## Available Hardware

### Radxa Zero
- **CPU**: Amlogic S905Y2 quad-core Cortex-A53
- **RAM**: Up to 4GB LPDDR4
- **GPU**: Mali-G31 MP2
- **Storage**: Up to 128GB eMMC
- **WiFi**: 802.11ac (2.4GHz + 5GHz) + Bluetooth 5.0
- **Ethernet**: None (USB-C ports only)
- **Video**: Micro HDMI
- **Docs**: [Radxa Wiki](https://wiki.radxa.com/Zero)

### Orange Pi Zero 3
- **CPU**: Allwinner H618 quad-core Cortex-A53 @ 1.5GHz
- **RAM**: 1GB - 4GB LPDDR4
- **GPU**: Mali-G31 MP2
- **Storage**: 16MB SPI flash + microSD
- **WiFi**: WiFi 5 + Bluetooth 5.0
- **Ethernet**: Gigabit Ethernet
- **Video**: Mini HDMI
- **Docs**: [Orange Pi Official](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/details/Orange-Pi-Zero-3.html)

## Decision Space

### Role: Access Point

| Factor | Radxa Zero | Orange Pi Zero 3 |
|--------|-----------|------------------|
| WiFi Quality | 802.11ac dual-band | WiFi 5 |
| Uplink | Must use WiFi bridge or USB Ethernet | Native Gigabit Ethernet |
| AP Mode Support | Good (hostapd) | Good (hostapd) |

**Consideration**: The Radxa lacks Ethernet, so it would need to either:
1. Bridge WiFi (client) to WiFi (AP) - adds latency, halves bandwidth
2. Use USB-to-Ethernet adapter for wired uplink

The Orange Pi's native Gigabit makes wired uplink straightforward.

### Role: Media Box

| Factor | Radxa Zero | Orange Pi Zero 3 |
|--------|-----------|------------------|
| Network | WiFi only | Gigabit Ethernet |
| Video Decode | Mali-G31 (H.264/HEVC) | Mali-G31 (H.264/HEVC) |
| Streaming Stability | Depends on WiFi | Reliable wired connection |
| Linux Support | Amlogic (fair) | Allwinner H618 (good mainline) |

**Consideration**: Media streaming benefits significantly from wired Ethernet - no WiFi congestion, consistent bandwidth for 4K content.

## Recommendation (Updated)

| Role | Device | Reasoning |
|------|--------|-----------|
| **Access Point** | Orange Pi Zero 3 | Gigabit Ethernet for wired backhaul; better mainline Linux support |
| **Media Box** | TBD (dedicated device) | Separate device for media playback |

The Orange Pi Zero 3's native Gigabit Ethernet provides reliable wired backhaul to the main network, making it ideal for an access point.

## Operating Systems

### For Access Point (Orange Pi Zero 3)
- **Armbian** + hostapd - Full control, good community support
- **DietPi** + hostapd - Lightweight, easy software management

### For Media Box (TBD)
- **LibreELEC** - Kodi-focused, minimal, optimized for media playback
- **Armbian** + Kodi/Jellyfin client - More flexible
- **DietPi** - Lightweight with easy software installer

---

## Orange Pi Zero 3 Access Point Setup

### Network Configuration

| Setting | Value |
|---------|-------|
| Main network | 192.168.101.0/24 |
| Router/Gateway | 192.168.101.1 |
| DNS | 192.168.101.1 (or 1.1.1.1) |

**Two AP Modes:**

| Mode | WiFi Subnet | Pros | Cons |
|------|-------------|------|------|
| **Bridged** | Same as LAN (192.168.101.x) | All devices on same network, easy discovery | Slightly more complex setup |
| **Routed (NAT)** | Separate (192.168.102.x) | Isolated WiFi network | Double NAT, devices on different subnets |

**Recommendation:** Use **Bridged** mode for seamless integration with your existing network.

---

### Option A: Armbian (Bridged Mode - Recommended)

#### Step 1: Download and Flash

1. Download Armbian for Orange Pi Zero 3:
   - [Armbian Downloads](https://www.armbian.com/orange-pi-zero-3/)
   - Choose **Bookworm** (Debian 12) minimal/server image

2. Flash to microSD:
   ```bash
   # Identify your SD card (be careful!)
   lsblk

   # Flash the image (replace sdX with your device)
   sudo dd if=Armbian_*.img of=/dev/sdX bs=4M status=progress conv=fsync
   ```

3. Insert SD card, connect Ethernet, and boot

#### Step 2: Initial Setup

1. SSH into the device (default: `root` / `1234`):
   ```bash
   ssh root@<ip-address>
   ```

2. Complete first-run setup:
   - Set root password
   - Create regular user
   - Set locale/timezone

3. Update the system:
   ```bash
   apt update && apt upgrade -y
   ```

#### Step 3: Install Required Packages

```bash
apt install -y hostapd bridge-utils
```

Note: **No dnsmasq needed** - your main router (192.168.101.1) handles DHCP.

#### Step 4: Configure Network Bridge

1. Identify interfaces:
   ```bash
   ip link show
   # Ethernet: eth0 or end0
   # WiFi: wlan0
   ```

2. Stop NetworkManager from managing these interfaces:
   ```bash
   nano /etc/NetworkManager/conf.d/bridge.conf
   ```

   ```ini
   [keyfile]
   unmanaged-devices=interface-name:eth0;interface-name:wlan0;interface-name:br0
   ```

3. Configure the bridge:
   ```bash
   nano /etc/network/interfaces
   ```

   ```
   # Loopback
   auto lo
   iface lo inet loopback

   # Ethernet - part of bridge (no IP)
   auto eth0
   iface eth0 inet manual

   # WiFi - part of bridge (no IP, managed by hostapd)
   auto wlan0
   iface wlan0 inet manual

   # Bridge interface - gets IP from your router
   auto br0
   iface br0 inet dhcp
       bridge_ports eth0
       bridge_stp off
       bridge_fd 0
       # wlan0 added by hostapd
   ```

   Or use a static IP:
   ```
   auto br0
   iface br0 inet static
       address 192.168.101.50
       netmask 255.255.255.0
       gateway 192.168.101.1
       dns-nameservers 192.168.101.1
       bridge_ports eth0
       bridge_stp off
       bridge_fd 0
   ```

#### Step 5: Configure hostapd

1. Create hostapd config:
   ```bash
   nano /etc/hostapd/hostapd.conf
   ```

   ```ini
   # Bridge mode
   bridge=br0
   interface=wlan0
   driver=nl80211

   # WiFi settings
   ssid=YourNetworkName
   hw_mode=a
   channel=36
   country_code=ZA

   # 802.11n/ac settings
   ieee80211n=1
   ieee80211ac=1
   wmm_enabled=1

   # Security
   wpa=2
   wpa_passphrase=YourSecurePassword
   wpa_key_mgmt=WPA-PSK
   rsn_pairwise=CCMP

   # Optional: Hide SSID
   # ignore_broadcast_ssid=1
   ```

2. Point hostapd to config:
   ```bash
   nano /etc/default/hostapd
   ```

   ```
   DAEMON_CONF="/etc/hostapd/hostapd.conf"
   ```

3. Unmask and enable hostapd:
   ```bash
   systemctl unmask hostapd
   systemctl enable hostapd
   ```

#### Step 6: Reboot and Test

```bash
reboot
```

After reboot:
- The AP should broadcast your SSID
- WiFi clients get IPs from your main router (192.168.101.x range)
- All devices on same subnet - no NAT, no double routing

#### Verify Bridge

```bash
# Check bridge status
brctl show

# Should show:
# br0    8000.xxxx  no   eth0
#                        wlan0

# Check IP
ip addr show br0
# Should have 192.168.101.x address
```

---

### Option A2: Armbian (Routed/NAT Mode - Alternative)

Use this if you want WiFi clients on a separate subnet (192.168.102.x).

#### Step 3: Install Required Packages

```bash
apt install -y hostapd dnsmasq iptables-persistent
```

#### Step 4: Configure Static IP for WiFi Interface

```bash
nano /etc/network/interfaces.d/wlan0
```

```
auto wlan0
iface wlan0 inet static
    address 192.168.102.1
    netmask 255.255.255.0
```

#### Step 5: Configure hostapd

```bash
nano /etc/hostapd/hostapd.conf
```

```ini
interface=wlan0
driver=nl80211
ssid=YourNetworkName
hw_mode=a
channel=36
country_code=ZA
ieee80211n=1
ieee80211ac=1
wmm_enabled=1
wpa=2
wpa_passphrase=YourSecurePassword
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

Enable:
```bash
echo 'DAEMON_CONF="/etc/hostapd/hostapd.conf"' > /etc/default/hostapd
systemctl unmask hostapd
systemctl enable hostapd
```

#### Step 6: Configure dnsmasq

```bash
mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
nano /etc/dnsmasq.conf
```

```ini
interface=wlan0
dhcp-range=192.168.102.10,192.168.102.100,255.255.255.0,24h
dhcp-option=3,192.168.102.1
dhcp-option=6,192.168.101.1,1.1.1.1
```

```bash
systemctl enable dnsmasq
```

#### Step 7: Enable NAT

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT

netfilter-persistent save
```

#### Step 8: Add Static Route on Main Router

For devices on 192.168.101.x to reach WiFi clients on 192.168.102.x, add a route on your router:

```
192.168.102.0/24 via <orange-pi-eth-ip>
```

#### Step 9: Reboot

```bash
reboot
```

---

### Option B: DietPi (Bridged Mode)

#### Step 1: Download and Flash

1. Download DietPi for Orange Pi Zero 3:
   - [DietPi Downloads](https://dietpi.com/#downloadinfo)
   - Select **Orange Pi Zero 3** image

2. Flash to microSD:
   ```bash
   sudo dd if=DietPi_*.img of=/dev/sdX bs=4M status=progress conv=fsync
   ```

#### Step 2: Pre-configure (Optional)

Before first boot, mount the SD card and edit:

```bash
# Mount boot partition
mount /dev/sdX1 /mnt

# Edit dietpi.txt for headless setup
nano /mnt/dietpi.txt
```

Set:
```ini
AUTO_SETUP_LOCALE=en_GB.UTF-8
AUTO_SETUP_KEYBOARD_LAYOUT=gb
AUTO_SETUP_TIMEZONE=Africa/Johannesburg
AUTO_SETUP_NET_ETHERNET_ENABLED=1
AUTO_SETUP_NET_WIFI_ENABLED=0
AUTO_SETUP_HEADLESS=1
AUTO_SETUP_AUTOSTART_TARGET_INDEX=0
```

#### Step 3: First Boot and Update

1. Boot and SSH in (default: `root` / `dietpi`):
   ```bash
   ssh root@<ip-address>
   ```

2. Complete DietPi first-run installer
   - Change passwords
   - Skip software selection for now

#### Step 4: Install Bridge Tools and hostapd

```bash
apt install -y hostapd bridge-utils
```

#### Step 5: Configure Bridge

```bash
nano /etc/network/interfaces
```

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet manual

auto wlan0
iface wlan0 inet manual

auto br0
iface br0 inet dhcp
    bridge_ports eth0
    bridge_stp off
    bridge_fd 0
```

Or static:
```
auto br0
iface br0 inet static
    address 192.168.101.50
    netmask 255.255.255.0
    gateway 192.168.101.1
    dns-nameservers 192.168.101.1
    bridge_ports eth0
    bridge_stp off
    bridge_fd 0
```

#### Step 6: Configure hostapd

```bash
nano /etc/hostapd/hostapd.conf
```

```ini
bridge=br0
interface=wlan0
driver=nl80211

ssid=YourNetworkName
hw_mode=a
channel=36
country_code=ZA

ieee80211n=1
ieee80211ac=1
wmm_enabled=1

wpa=2
wpa_passphrase=YourSecurePassword
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

```bash
echo 'DAEMON_CONF="/etc/hostapd/hostapd.conf"' > /etc/default/hostapd
systemctl unmask hostapd
systemctl enable hostapd
```

#### Step 7: Reboot and Verify

```bash
reboot
```

After reboot:
```bash
brctl show
# br0 should list eth0 and wlan0

ip addr show br0
# Should have 192.168.101.x address from your router
```

WiFi clients will get IPs from your main router (192.168.101.1).

---

## Kodi + Jellyfin Media Client Setup

Run Kodi as a media client alongside the access point. The AP services run in the background while Kodi displays on the Mini HDMI output.

```
┌─────────────────────────────────────┐
│      Orange Pi Zero 3               │
│  ┌─────────────┬─────────────────┐  │
│  │   hostapd   │      Kodi       │  │
│  │  (WiFi AP)  │   (Jellyfin)    │  │
│  └──────┬──────┴────────┬────────┘  │
│         │               │           │
│    [wlan0]         [Mini HDMI]      │
│         │               │           │
└─────────┼───────────────┼───────────┘
          │               │
     WiFi Clients        TV
     (192.168.101.x)
```

Both services run simultaneously - AP in background, Kodi on display.

### Prerequisites

- Orange Pi Zero 3 connected to TV/monitor via Mini HDMI
- Access point setup completed (bridged mode recommended)
- Jellyfin server running on your network (e.g., ba1dr)

### Step 1: Install Kodi and Dependencies

```bash
# Install Kodi and required packages
apt install -y kodi kodi-inputstream-adaptive kodi-peripheral-joystick

# For hardware video acceleration (Mali-G31)
apt install -y mesa-utils libmali-valhall-g610-g13p0-x11
```

### Step 2: Create Kodi User (Optional but Recommended)

Running Kodi as a dedicated user is cleaner than root:

```bash
# Create kodi user with video/audio groups
useradd -m -G video,audio,input,dialout,plugdev -s /bin/bash kodi

# Set password (for SSH access if needed)
passwd kodi
```

### Step 3: Configure Auto-Start Kodi on Boot

Create a systemd service:

```bash
nano /etc/systemd/system/kodi.service
```

```ini
[Unit]
Description=Kodi Media Center
After=network-online.target sound.target graphical.target
Wants=network-online.target

[Service]
User=kodi
Group=kodi
Type=simple
Environment=DISPLAY=:0
ExecStart=/usr/bin/kodi-standalone
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
systemctl daemon-reload
systemctl enable kodi
```

### Step 4: Configure X11 Auto-Start (for Kodi Display)

Kodi needs an X server. Install and configure:

```bash
# Install minimal X server
apt install -y xserver-xorg xinit x11-xserver-utils
```

Create X auto-start:

```bash
nano /etc/systemd/system/x11.service
```

```ini
[Unit]
Description=X11 Display Server
After=systemd-user-sessions.service

[Service]
Type=simple
ExecStart=/usr/bin/X :0 -nolisten tcp vt7
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable x11
```

### Step 5: Install Jellyfin for Kodi Add-on

After Kodi starts, install the Jellyfin add-on:

**Method A: From Kodi UI**

1. Go to **Settings** → **Add-ons** → **Install from repository**
2. Select **Kodi Add-on repository**
3. **Video add-ons** → **Jellyfin**
4. Install

**Method B: Install Repository Manually (Headless)**

```bash
# Download Jellyfin Kodi repository
wget -O /tmp/jellyfin-repo.zip \
  https://kodi.jellyfin.org/repository.jellyfin.kodi.zip

# Install as kodi user
su - kodi -c "mkdir -p ~/.kodi/addons"
unzip /tmp/jellyfin-repo.zip -d /home/kodi/.kodi/addons/
chown -R kodi:kodi /home/kodi/.kodi
```

Then from Kodi UI:
1. **Add-ons** → **Install from repository**
2. **Jellyfin Kodi Add-ons** → **Video add-ons** → **Jellyfin** → Install

### Step 6: Configure Jellyfin Connection

On first launch, Jellyfin add-on will prompt for server details:

| Setting | Value |
|---------|-------|
| Server Address | `http://ba1dr:8096` or `http://192.168.101.x:8096` |
| Username | Your Jellyfin username |
| Password | Your Jellyfin password |

**Sync Options (Recommended):**
- Play Mode: **Add-on (default)** - streams directly, no sync needed
- Enable **Native** playback mode for better performance

### Step 7: Configure Kodi Settings

Recommended settings for Orange Pi Zero 3:

**Settings → Player → Videos:**
- Allow hardware acceleration: **On**
- Render method: **Auto detect**

**Settings → System → Display:**
- Resolution: **1920x1080** (or match your TV)
- Refresh rate: **60Hz**

**Settings → System → Audio:**
- Audio output device: **HDMI**
- Passthrough: Enable for surround sound

### Step 8: Remote Control Options

**Option A: CEC (TV Remote)**

If your TV supports HDMI-CEC, Kodi can be controlled with your TV remote:

```bash
apt install -y libcec6 cec-utils
```

Enable in Kodi: **Settings → System → Input → Peripherals → CEC Adapter**

**Option B: Kodi Remote App**

Use smartphone apps:
- **Kore** (Android) - Official Kodi remote
- **Official Kodi Remote** (iOS)

Enable remote control in Kodi:
1. **Settings → Services → Control**
2. Enable **Allow remote control via HTTP**
3. Enable **Allow remote control from applications on this system**
4. Set port (default 8080) and credentials if desired

**Option C: SSH + kodi-send**

Control Kodi via SSH:

```bash
apt install -y kodi-eventclients-kodi-send

# Example commands
kodi-send --action="Play"
kodi-send --action="Pause"
kodi-send --action="Stop"
kodi-send --action="PlayerControl(Next)"
```

### Step 9: Verify Everything Works

```bash
# Reboot to start all services
reboot

# After reboot, check services
systemctl status x11
systemctl status kodi
systemctl status hostapd

# All three should be running
```

Your Orange Pi Zero 3 should now:
1. Broadcast your WiFi network (AP mode)
2. Display Kodi on the connected TV/monitor
3. Connect to your Jellyfin server for media playback

### Performance Tips

**Reduce Memory Usage:**
```bash
# Disable unused services
systemctl disable bluetooth  # If not using Bluetooth
systemctl disable ModemManager
```

**GPU Memory Split:**

Edit `/boot/armbianEnv.txt` or `/boot/orangepiEnv.txt`:
```
extraargs=cma=256M
```

This allocates 256MB for GPU/video (adjust as needed).

**Check Hardware Decoding:**
```bash
# In Kodi, enable debug logging, then check:
cat ~/.kodi/temp/kodi.log | grep -i "hardware"
```

---

## Troubleshooting

### WiFi interface not found
```bash
# Check if WiFi is blocked
rfkill list
rfkill unblock wifi

# Load WiFi driver
modprobe <driver_name>
```

### hostapd fails to start
```bash
# Check logs
journalctl -u hostapd -n 50

# Common issues:
# - Interface already in use: stop NetworkManager or wpa_supplicant
# - Channel not allowed: check country_code and channel
```

### Clients can't get IP
```bash
# Check dnsmasq
systemctl status dnsmasq
journalctl -u dnsmasq -n 50

# Verify interface has correct IP
ip addr show wlan0
```

### No internet for clients
```bash
# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward
# Should be 1

# Check iptables rules
iptables -t nat -L -n
iptables -L FORWARD -n
```

## Next Steps

- [ ] Verify RAM configuration of Orange Pi Zero 3 (2GB+ recommended for Kodi)
- [ ] Flash Armbian or DietPi to microSD
- [ ] Follow bridged access point setup guide
- [ ] Configure AP and test WiFi coverage
- [ ] Install Kodi + Jellyfin add-on
- [ ] Connect to TV via Mini HDMI and configure display
- [ ] Test media playback from Jellyfin server
- [ ] Configure remote control (CEC or phone app)

###### Tags
- #HomeLabSetup
- #Networking
- #MediaServer
- #SBC

---
[[index|← Back to Home]]
