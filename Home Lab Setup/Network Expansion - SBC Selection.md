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

## Recommendation

| Role | Device | Reasoning |
|------|--------|-----------|
| **Access Point** | Radxa Zero | No Ethernet means WiFi is its primary interface anyway; good 802.11ac chip |
| **Media Box** | Orange Pi Zero 3 | Gigabit Ethernet critical for reliable media streaming; better mainline Linux support |

### Alternative Consideration
If the Radxa Zero were used as media box with USB Ethernet adapter, the Orange Pi could serve as AP with wired backhaul. However, this adds complexity and cost for marginal benefit.

## Operating Systems

### For Access Point (Radxa Zero)
- **OpenWrt** - Purpose-built for routing/AP, lightweight
- **Armbian** + hostapd - More flexible but more setup

### For Media Box (Orange Pi Zero 3)
- **LibreELEC** - Kodi-focused, minimal, optimized for media playback
- **Armbian** + Kodi/Jellyfin client - More flexible
- **DietPi** - Lightweight with easy software installer

## Next Steps

- [ ] Verify RAM configurations of owned boards
- [ ] Flash and test OpenWrt on Radxa Zero
- [ ] Flash and test LibreELEC/Armbian on Orange Pi Zero 3
- [ ] Configure AP and test WiFi coverage
- [ ] Set up media client and connect to [[Home Lab Setup/Media Server/index|Media Server]]

###### Tags
- #HomeLabSetup
- #Networking
- #MediaServer
- #SBC

---
[[index|← Back to Home]]
