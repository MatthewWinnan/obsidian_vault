# Jellyfin Setup on ba1dr

Media server setup using Jellyfin on ba1dr (Lenovo Legion Y530-15ICH) with hardware transcoding.

## Goals

- Self-hosted media streaming (movies, TV shows, music)
- Hardware transcoding via NVIDIA NVENC and/or Intel QSV
- Remote access via Tailscale Funnel (through th0r proxy)
- Mobile app support for offline downloads
- Casting to TV (Chromecast, DLNA, or smart TV app)

## Hardware Overview

### ba1dr Specs (Lenovo Legion Y530-15ICH)

| Component | Details |
|-----------|---------|
| **CPU** | Intel Core i5-8300H or i7-8750H (8th gen Coffee Lake) |
| **iGPU** | Intel UHD Graphics 630 (supports QSV, HEVC 10-bit) |
| **dGPU** | NVIDIA GTX 1050 Ti (supports NVENC) |
| **RAM** | 8-16GB DDR4 |
| **Storage** | BTRFS with LUKS encryption |

### Transcoding Capabilities

| Codec | Intel QSV (decode) | NVIDIA NVENC (encode) |
|-------|-------------------|----------------------|
| H.264 | Yes | Yes |
| HEVC 8-bit | Yes | Yes |
| HEVC 10-bit | Yes (8th gen+) | Yes (Pascal+) |
| VP9 | Yes | Decode only |
| AV1 | No (needs 11th gen+) | No (needs RTX 30+) |

**Recommended setup:** Intel QSV for decoding + NVIDIA NVENC for encoding (best of both worlds)

---

## Current ba1dr Configuration

### NVIDIA Already Configured

**`gaming/ba1dr/video_drivers.nix`:**
```nix
{pkgs, ...}: {
  services.xserver.videoDrivers = ["nvidia"];

  hardware.nvidia.prime = {
    sync.enable = true;
    offload.enable = false;
  };
}
```

### Graphics Already Configured

**`gaming/ba1dr/opengl.nix`:**
```nix
{pkgs, ...}: {
  hardware.graphics = {
    enable = true;
    enable32Bit = true;
  };
}
```

### Storage Layout (BTRFS)

- `/` - root subvolume
- `/home` - home subvolume (user data)
- `/nix` - nix store
- `/persist` - persistent data (good for Jellyfin metadata)
- LUKS encrypted

---

## Implementation Plan

### Phase 1: Jellyfin Service

Create `nix/services/jellyfin.nix`:

```nix
{ config, pkgs, ... }:
{
  services.jellyfin = {
    enable = true;
    openFirewall = true;  # Opens port 8096
  };

  # Grant Jellyfin access to GPU for hardware transcoding
  users.users.jellyfin.extraGroups = [ "video" "render" ];

  # Persist Jellyfin data across rebuilds
  # Metadata, config, cache in /persist/jellyfin
  systemd.tmpfiles.rules = [
    "d /persist/jellyfin 0755 jellyfin jellyfin -"
    "d /persist/jellyfin/config 0755 jellyfin jellyfin -"
    "d /persist/jellyfin/data 0755 jellyfin jellyfin -"
    "d /persist/jellyfin/cache 0755 jellyfin jellyfin -"
  ];

  # Symlink Jellyfin data to persistent storage
  fileSystems."/var/lib/jellyfin" = {
    device = "/persist/jellyfin";
    options = [ "bind" ];
  };
}
```

### Phase 2: Intel QSV Support (Hybrid Transcoding)

Add Intel VA-API drivers for hybrid decode:

```nix
# Add to gaming/ba1dr/opengl.nix or create new file
{
  hardware.graphics.extraPackages = with pkgs; [
    intel-media-driver    # VAAPI driver for Intel (newer)
    vaapiIntel           # VAAPI driver for Intel (legacy, try if above fails)
    vaapiVdpau           # VAAPI to VDPAU bridge
    libvdpau-va-gl       # VDPAU to VAAPI bridge
  ];
}
```

### Phase 3: Tailscale Integration

Add Tailscale to ba1dr for network access:

**`machines/ba1dr/nix/tailscale.nix`:**
```nix
{
  services.tailscale.enable = true;

  networking.firewall = {
    trustedInterfaces = [ "tailscale0" ];
    allowedUDPPorts = [ 41641 ];  # Tailscale
  };
}
```

After deployment: `sudo tailscale up`

### Phase 4: Proxy Integration (th0r)

Add Jellyfin route to `nix/services/caddy-proxy/default.nix`:

```nix
# Jellyfin media server
handle /media/* {
  uri strip_prefix /media
  reverse_proxy ba1dr:8096 {
    header_up Host {upstream_hostport}
    header_up X-Forwarded-Proto https
  }
}
```

Update landing page to add Jellyfin link:

```html
<a href="/media/" class="service-link">Jellyfin</a>
```

### Phase 5: Media Storage

Options for media library:

| Option | Pros | Cons |
|--------|------|------|
| **Local HDD/SSD** | Fast, simple | Limited space, laptop only |
| **External USB drive** | Easy expansion | USB overhead, must be connected |
| **NAS mount (NFS/SMB)** | Centralized storage | Network dependent |
| **Syncthing from NAS** | Offline capability | Duplicate storage |

**Recommended:** External USB drive or NAS mount depending on your setup.

```nix
# Example: Mount NAS media share
fileSystems."/media/library" = {
  device = "//nas/media";
  fsType = "cifs";
  options = [
    "credentials=/persist/secrets/nas-creds"
    "uid=jellyfin"
    "gid=jellyfin"
    "x-systemd.automount"
    "x-systemd.idle-timeout=60"
  ];
};
```

---

## Jellyfin Configuration (Post-Install)

### Initial Setup

1. Access: `http://ba1dr:8096` or `https://th0r.tail13fdb.ts.net/media/`
2. Create admin account
3. Add media libraries (Movies, TV Shows, Music)

### Enable Hardware Transcoding

In Jellyfin Dashboard → Playback → Transcoding:

1. **Hardware acceleration:** NVIDIA NVENC
2. **Enable hardware decoding for:**
   - [x] H.264
   - [x] HEVC
   - [x] HEVC 10-bit
   - [x] VP9
3. **Hardware encoding:** Enable NVENC
4. **Prefer OS native DXVA or VA-API:** Yes (for Intel QSV decode)

### Recommended Settings

| Setting | Value |
|---------|-------|
| **Transcoding temp path** | `/tmp/jellyfin-transcodes` |
| **Thread count** | Auto |
| **Allow encoding in HEVC** | Yes (smaller files) |
| **Throttle transcodes** | Yes (saves CPU when buffered) |

---

## Laptop Considerations (24/7 Use)

### Power Management

```nix
# Prevent sleep when lid closed (if using as server)
services.logind = {
  lidSwitch = "ignore";
  lidSwitchExternalPower = "ignore";
};

# Optional: Set power profile
powerManagement.cpuFreqGovernor = "ondemand";  # or "powersave"
```

### Heat Management

- Use laptop cooling pad
- Clean fans/vents regularly
- Monitor temps: `nvidia-smi` and `sensors`

### Battery Care

- Remove battery if possible
- Or limit charge to 60-80% (if BIOS supports)
- Consider always-on-AC use

---

## Client Apps

### Official Clients

| Platform | App |
|----------|-----|
| Android/iOS | Jellyfin Mobile (free) |
| Android TV | Jellyfin for Android TV |
| Web | Built-in web UI |
| Desktop | Jellyfin Media Player |

### Third-Party Clients

| Platform | App |
|----------|-----|
| iOS | Swiftfin (native, better UX) |
| Apple TV | Swiftfin |
| Kodi | Jellyfin for Kodi plugin |

### Offline Downloads

Mobile apps support downloading for offline viewing:
1. Open movie/episode
2. Tap download icon
3. Select quality
4. Available offline in app

---

## Checklist

### Setup
- [ ] Add Jellyfin service to ba1dr
- [ ] Configure Intel VA-API drivers
- [ ] Test hardware transcoding
- [ ] Add Tailscale to ba1dr
- [ ] Add route in th0r Caddy proxy
- [ ] Update landing page
- [ ] Configure media storage

### Post-Setup
- [ ] Run initial Jellyfin setup wizard
- [ ] Enable hardware transcoding in Jellyfin
- [ ] Add media libraries
- [ ] Install client apps
- [ ] Test remote streaming
- [ ] Test offline downloads

---

## Related

- [[Media Stack Setup]] - Automated downloading with Radarr/Sonarr/Prowlarr
- [[Proxy Server Setup]] - Caddy reverse proxy on th0r
- [[Tailscale]] - Mesh networking
- [[Home Assistant Setup]] - Smart home integration (media player controls)

---

[[index|← Home]]
