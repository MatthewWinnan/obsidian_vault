# Music Stack Setup (Desktop)

Local music acquisition stack for desktop machines (gaming/personal profiles).

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Aurral  │────▶│  Prowlarr   │────▶│   Lidarr    │────▶│ qBittorrent │────▶│    Beets    │────▶│     MPD     │
│   :3001     │     │   :9696     │     │   :8686     │     │   :8085     │     │    (CLI)    │     │   :6600     │
│ (requests)  │     │ (indexers)  │     │  (manager)  │     │ (download)  │     │ (organize)  │     │   (play)    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘     └──────┬──────┘
                                                                                       │                    │
                                                                                       ▼                    ▼
                                                                                ~/Media/Music ◀────────── rmpc
                                                                                                         (control)
```

## Request Flow

1. **Request**: Browse and request artists/albums via Aurral's web UI
2. **Index**: Prowlarr manages indexers and syncs them to Lidarr automatically
3. **Search**: Lidarr searches indexers via Prowlarr for requested content
4. **Download**: Lidarr sends torrent to qBittorrent
5. **Organize**: Run `beet import` to tag, rename, and move to library
6. **Play**: Beets notifies MPD, music appears in rmpc

---

## Service Overview

| Service | Port | Purpose | Access |
|---------|------|---------|--------|
| Aurral | 3001 | Music request UI | http://localhost:3001 |
| Prowlarr | 9696 | Indexer manager | http://localhost:9696 |
| Lidarr | 8686 | Music collection manager | http://localhost:8686 |
| qBittorrent | 8085 | Torrent client | http://localhost:8085 |
| Beets | CLI | Tag/organize music | `beet import <path>` |
| MPD | 6600 | Music playback daemon | via rmpc |
| rmpc | - | TUI MPD client | Terminal |

## Directory Structure

```
~/Media/
├── Music/              # Organized library (MPD watches this)
└── Downloads/
    └── music/          # qBittorrent download location (temporary)
```

---

## Post-Deployment Configuration

Configure services in this order after `nixos-rebuild switch`:

### 1. qBittorrent (http://localhost:8085)

First-time setup:

1. Default login: `admin` / `adminadmin`
2. **Change the password immediately!**
3. Go to **Tools → Options → Downloads**
4. Set **Default Save Path**: `~/Media/Downloads/music`
5. Set **Keep incomplete torrents in**: `~/Media/Downloads/music/incomplete`

### 2. Prowlarr (http://localhost:9696)

Prowlarr manages all your indexers centrally and syncs them to Lidarr automatically.

1. **Initial Setup**:
   - Set authentication method
   - Create username/password

2. **Add Indexers** (Indexers → Add Indexer):
   - Click `+` button
   - Search for your preferred music indexers
   - Configure credentials/API keys as needed
   - Test and save each indexer

3. **Connect to Lidarr** (Settings → Apps → Add Application):
   - Select **Lidarr**
   - Prowlarr Server: `http://localhost:9696`
   - Lidarr Server: `http://localhost:8686`
   - API Key: Copy from Lidarr (Settings → General → API Key)
   - Sync Level: **Full Sync** (recommended)
   - Test and save

4. **Sync Indexers**:
   - Click **Sync App Indexers** button
   - All configured indexers will appear in Lidarr automatically

### 3. Lidarr (http://localhost:8686)

1. **Initial Setup**:
   - Set authentication method
   - Create username/password

2. **Verify Indexers** (Settings → Indexers):
   - Indexers should appear automatically from Prowlarr
   - If not, check Prowlarr app connection

3. **Download Client** (Settings → Download Clients → Add):
   - Select **qBittorrent**
   - Host: `localhost`
   - Port: `8085`
   - Username/Password: (from qBittorrent setup)
   - Category: `lidarr` (optional)
   - Test and save

4. **Root Folder** (Settings → Media Management):
   - Click "Add Root Folder"
   - Path: `~/Media/Music` (or `/home/<user>/Media/Music`)
   - Enable "Use Hardlinks instead of Copy" for same-filesystem efficiency

5. **Quality Profiles** (Settings → Profiles):
   - Customize quality preferences (FLAC, 320kbps MP3, etc.)

6. **Copy API Key** (Settings → General):
   - Copy the API Key (needed for Aurral)

### 4. Aurral (http://localhost:3001)

Aurral provides a Jellyseerr-like request UI for music. It runs as a Podman container.

1. **Access UI**: http://localhost:3001

2. **Configure Lidarr** (Settings → Integrations → Lidarr):
   - URL: `http://localhost:8686`
   - API Key: (paste from Lidarr Settings → General)
   - Click Test, then Save

3. **Configure MusicBrainz** (Settings → Integrations → MusicBrainz):
   - Contact Email: (your email, required for API policy)

4. **Optional - Last.fm** (Settings → Integrations → Last.fm):
   - API Key from https://www.last.fm/api/account/create
   - Enables artist recommendations based on listening history

**Features:**
- Artist search powered by MusicBrainz
- View release groups (albums, EPs, singles)
- Smart discovery based on existing library
- Grid view of your Lidarr library

### 5. Beets (CLI)

Beets is pre-configured via home-manager. Basic usage:

```bash
# Import and organize downloaded music
beet import ~/Media/Downloads/music

# Interactive mode (default) - confirms matches
# Use -q for quiet mode (auto-accept good matches)
beet import -q ~/Media/Downloads/music

# List library
beet ls

# Search library
beet ls artist:Beatles

# Show track info
beet info ~/Media/Music/Artist/Album/track.flac

# Update library after manual changes
beet update
```

**Configured plugins:**
- `fetchart` - Downloads album artwork
- `embedart` - Embeds art in audio files
- `lastgenre` - Fetches genres from Last.fm
- `replaygain` - Calculates replay gain
- `mpdupdate` - Notifies MPD after import
- `mpdstats` - Tracks play statistics

**Config location:** `~/.config/beets/config.yaml`
**Library database:** `~/.config/beets/library.db`
**Import log:** `~/.config/beets/import.log`

### 6. MPD & rmpc

MPD is pre-configured and starts automatically. Control with rmpc:

```bash
rmpc           # Launch TUI
```

**rmpc keybindings:**
- `1-7` - Switch tabs (Queue, Directories, Artists, etc.)
- `j/k` - Navigate up/down
- `Enter` - Play/confirm
- `p` - Toggle pause
- `>/<` - Next/previous track
- `,/.` - Volume down/up
- `q` - Quit

---

## Usage Workflow

### Adding New Music

1. **Via Lidarr** (recommended for albums):
   - Go to http://localhost:8686
   - Search for artist
   - Add artist/album to collection
   - Lidarr monitors and downloads automatically

2. **Manual download + Beets**:
   - Download music to `~/Media/Downloads/music`
   - Run `beet import ~/Media/Downloads/music`
   - Confirm matches or correct if needed
   - Music moves to `~/Media/Music`, MPD updates

### Playing Music

1. Open terminal, run `rmpc`
2. Press `2` for Directories or `3` for Artists
3. Navigate with `j/k`, add to queue with `a`
4. Press `1` for Queue, `Enter` to play

---

## Troubleshooting

### Music not appearing in rmpc

1. Check if file is in `~/Media/Music`
2. Run `beet update` to rescan library
3. In rmpc, MPD should auto-update, or restart MPD:
   ```bash
   systemctl --user restart mpd
   ```

### Beets not matching correctly

1. Use interactive mode: `beet import ~/path` (no `-q`)
2. Type `e` to enter search manually
3. Check MusicBrainz for correct release

### No sound from MPD

1. Verify PipeWire is running: `systemctl --user status pipewire`
2. Check MPD status: `systemctl status mpd`
3. Check audio output in `~/.config/mpd/mpd.conf`

### qBittorrent connection refused

1. Check if service is running: `systemctl status qbittorrent`
2. Verify port 8085 is correct
3. Check firewall settings

### Prowlarr not syncing to Lidarr

1. Verify Prowlarr app connection (Settings → Apps)
2. Check API key is correct (copy fresh from Lidarr)
3. Test connection in Prowlarr
4. Click "Sync App Indexers" manually
5. Check Prowlarr logs: `journalctl -u prowlarr -f`

### Indexers not appearing in Lidarr

1. In Prowlarr: Settings → Apps → Edit Lidarr connection
2. Verify Sync Level is set to "Full Sync"
3. Click Test, then Save
4. Click "Sync App Indexers"
5. Refresh Lidarr page

### Aurral not connecting to Lidarr

1. Go to Settings → Integrations → Lidarr in Aurral UI
2. URL should be `http://localhost:8686`
3. Verify API key is correct (copy fresh from Lidarr Settings → General)
4. Click Test to verify connection
5. Restart container if needed: `sudo systemctl restart podman-aurral`
6. Check logs: `sudo journalctl -u podman-aurral -f`

### Aurral container not starting

1. Check container status: `sudo systemctl status podman-aurral`
2. View logs: `sudo journalctl -u podman-aurral -f`
3. Verify data directory exists: `ls ~/.local/share/aurral/`
4. Verify Podman is working: `podman ps -a`

---

## NixOS Configuration

**Service files:**
- `nix/services/music-stack.nix` - Aurral + Prowlarr + Lidarr + qBittorrent
- `nix/services/mpd.nix` - MPD daemon
- `home/programs/beets/default.nix` - Beets config
- `home/programs/rmpc/default.nix` - rmpc TUI client

**Container:**
- Aurral runs as a Podman container via `virtualisation.oci-containers`
- Managed by systemd unit: `podman-aurral.service`

**Enabled on profiles:** `gaming`, `personal`

---

## Related

- [[Media Stack Setup]] - Video media stack on ba1dr
- [[Jellyfin Setup]] - Media server

---

[[index|← Home]]
