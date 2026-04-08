# Media Stack Setup on ba1dr

Complete guide for the automated media stack (the "arr" suite) running on ba1dr.

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌───────────┐     ┌─────────────┐
│ Jellyseerr  │────▶│   Radarr    │────▶│  Prowlarr │────▶│ qBittorrent │
│   :5055     │     │   :7878     │     │   :9696   │     │   :8085     │
│ (requests)  │     │  (movies)   │     │ (indexers)│     │ (download)  │
└─────────────┘     └──────┬──────┘     └───────────┘     └──────┬──────┘
                           │                                      │
┌─────────────┐     ┌──────▼──────┐                        ┌──────▼──────┐
│   Bazarr    │────▶│  Jellyfin   │◀───────────────────────│   /media/   │
│   :6767     │     │   :8096     │                        │  movies/tv  │
│ (subtitles) │     │  (library)  │                        └─────────────┘
└─────────────┘     └─────────────┘

       ▲                           ┌─────────────┐
       └───────────────────────────│   Sonarr    │
                                   │   :8989     │
                                   │ (TV shows)  │
                                   └─────────────┘
```

## Request Flow

1. **Request**: User requests a movie/show in Jellyseerr
2. **Search**: Radarr/Sonarr searches Prowlarr for the content
3. **Download**: Prowlarr finds torrent, sends to qBittorrent
4. **Organize**: Radarr/Sonarr renames & moves file to `/media/movies/` or `/media/tv/`
5. **Subtitles**: Bazarr auto-downloads matching subtitles
6. **Watch**: Jellyfin picks up the new file automatically

---

## Service Overview

| Service | Port | Purpose | Access |
|---------|------|---------|--------|
| Jellyseerr | 5055 | Request UI for users | `https://th0r.../requests/` |
| Jellyfin | 8096 | Media streaming | `https://th0r.../media/` |
| Radarr | 7878 | Movie management | `http://ba1dr:7878` (internal) |
| Sonarr | 8989 | TV show management | `http://ba1dr:8989` (internal) |
| Prowlarr | 9696 | Indexer manager | `http://ba1dr:9696` (internal) |
| Bazarr | 6767 | Subtitle management | `http://ba1dr:6767` (internal) |
| qBittorrent | 8085 | Torrent client | `http://ba1dr:8085` (internal) |

## Directory Structure

```
/media/
├── movies/      # Radarr root folder (completed movies)
├── tv/          # Sonarr root folder (completed TV shows)
└── downloads/   # qBittorrent download location (temporary)
```

---

## Post-Deployment Configuration

Configure services in this order after `nixos-rebuild switch`:

### 1. qBittorrent (http://ba1dr:8085)

First-time setup:

1. Default login: `admin` / `adminadmin`
2. Go to **Tools → Options → Downloads**
3. Set **Default Save Path**: `/media/downloads`
4. Set **Keep incomplete torrents in**: `/media/downloads/incomplete`
5. Go to **Tools → Options → Web UI**
6. Change default password (important!)
7. **Optional**: Enable "Bypass authentication for clients on localhost"

### 2. Prowlarr (http://ba1dr:9696)

Prowlarr manages indexers (torrent sites) for Radarr/Sonarr.

1. **Initial Setup**:
   - Set authentication method (Forms or Basic)
   - Create username/password

2. **Add Indexers** (Settings → Indexers → Add):
   - Click the `+` button
   - Search for your preferred indexers (e.g., 1337x, RARBG, etc.)
   - Configure each indexer with required credentials/settings
   - Test and save

3. **Add Applications** (Settings → Apps → Add):
   - Add **Radarr**:
     - Prowlarr Server: `http://localhost:9696`
     - Radarr Server: `http://localhost:7878`
     - API Key: (get from Radarr → Settings → General)
   - Add **Sonarr**:
     - Prowlarr Server: `http://localhost:9696`
     - Sonarr Server: `http://localhost:8989`
     - API Key: (get from Sonarr → Settings → General)

### 3. Radarr (http://ba1dr:7878)

Radarr manages movie downloads.

1. **Media Management** (Settings → Media Management):
   - Click "Add Root Folder"
   - Path: `/media/movies`
   - Click "Show Advanced" for more options
   - Enable "Rename Movies" if desired

2. **Download Clients** (Settings → Download Clients → Add):
   - Select **qBittorrent**
   - Host: `localhost`
   - Port: `8085`
   - Username/Password: (from qBittorrent setup)
   - Category: `radarr` (optional, helps organize)
   - Test and save

3. **Indexers** (Settings → Indexers):
   - If Prowlarr is configured correctly, indexers sync automatically
   - Verify indexers appear here

4. **Quality Profiles** (Settings → Profiles):
   - Customize quality preferences (e.g., prefer 1080p, allow 4K)
   - Set upgrade behavior

5. **Get API Key** (Settings → General → Security):
   - Copy API Key for use in Prowlarr and Jellyseerr

### 4. Sonarr (http://ba1dr:8989)

Sonarr manages TV show downloads. Configuration mirrors Radarr:

1. **Media Management** (Settings → Media Management):
   - Add Root Folder: `/media/tv`
   - Enable "Rename Episodes" if desired
   - Configure naming format (e.g., `{Series Title} - S{season:00}E{episode:00} - {Episode Title}`)

2. **Download Clients** (Settings → Download Clients → Add):
   - Select **qBittorrent**
   - Host: `localhost`
   - Port: `8085`
   - Category: `sonarr`
   - Test and save

3. **Indexers**: Should sync from Prowlarr automatically

4. **Get API Key** (Settings → General → Security):
   - Copy API Key for Prowlarr and Jellyseerr

### 5. Bazarr (http://ba1dr:6767)

Bazarr automatically downloads subtitles.

1. **Sonarr Connection** (Settings → Sonarr):
   - Enable Sonarr
   - Address: `localhost`
   - Port: `8989`
   - API Key: (from Sonarr)
   - Test and save

2. **Radarr Connection** (Settings → Radarr):
   - Enable Radarr
   - Address: `localhost`
   - Port: `7878`
   - API Key: (from Radarr)
   - Test and save

3. **Subtitle Providers** (Settings → Providers):
   - Add providers (e.g., OpenSubtitles, Subscene, Addic7ed)
   - Some require free accounts

4. **Languages** (Settings → Languages):
   - Set your preferred subtitle languages
   - Configure language profiles

### 6. Jellyseerr (http://ba1dr:5055)

Jellyseerr is the user-facing request interface.

1. **Initial Setup Wizard**:
   - Select "Jellyfin" as your media server
   - Enter Jellyfin URL: `http://localhost:8096`
   - Login with Jellyfin admin credentials
   - Sync libraries

2. **Configure URL Base** (Settings → General):
   - Set URL Base to `/requests` (for reverse proxy)

3. **Add Radarr** (Settings → Services → Radarr):
   - Click "Add Radarr Server"
   - Default Server: Yes
   - Server Name: `Radarr`
   - Hostname: `localhost`
   - Port: `7878`
   - API Key: (from Radarr)
   - Quality Profile: Select preferred
   - Root Folder: `/media/movies`
   - Test and save

4. **Add Sonarr** (Settings → Services → Sonarr):
   - Click "Add Sonarr Server"
   - Default Server: Yes
   - Server Name: `Sonarr`
   - Hostname: `localhost`
   - Port: `8989`
   - API Key: (from Sonarr)
   - Quality Profile: Select preferred
   - Root Folder: `/media/tv`
   - Test and save

5. **User Management** (Settings → Users):
   - Import Jellyfin users
   - Set permissions (who can request, auto-approve, etc.)

### 7. Jellyfin (http://ba1dr:8096)

Add the media libraries:

1. **Add Movies Library** (Dashboard → Libraries → Add):
   - Content type: Movies
   - Folders: `/media/movies`
   - Configure metadata agents

2. **Add TV Shows Library** (Dashboard → Libraries → Add):
   - Content type: Shows
   - Folders: `/media/tv`
   - Configure metadata agents

3. **Configure URL Base** (Dashboard → Networking):
   - Base URL: `/media`

---

## Reverse Proxy (th0r)

Access via Tailscale Funnel:

| Service | External URL |
|---------|--------------|
| Jellyfin | `https://th0r.tail13fdb.ts.net/media/` |
| Jellyseerr | `https://th0r.tail13fdb.ts.net/requests/` |

### Sops Secret (secrets/th0r.yaml)

```yaml
caddy-env: |
  HA_BACKEND=192.168.101.150:8123
  JELLYFIN_BACKEND=ba1dr:8096
  JELLYSEERR_BACKEND=ba1dr:5055
```

---

## Usage Workflow

### Requesting Content

1. Go to `https://th0r.tail13fdb.ts.net/requests/`
2. Search for a movie or TV show
3. Click "Request"
4. Select quality options if prompted
5. Submit request

### Monitoring Downloads

- **Radarr/Sonarr**: Activity tab shows download progress
- **qBittorrent**: Shows active torrents
- **Jellyseerr**: Shows request status

### Watching Content

1. Go to `https://th0r.tail13fdb.ts.net/media/`
2. Content appears automatically after download completes
3. Subtitles are added by Bazarr

---

## Troubleshooting

### Content not appearing in Jellyfin

1. Check Radarr/Sonarr Activity tab for errors
2. Verify file permissions (`media` group)
3. Trigger library scan in Jellyfin

### Downloads stuck

1. Check qBittorrent for errors
2. Verify indexer is working in Prowlarr
3. Check if torrent has seeders

### Subtitles not downloading

1. Check Bazarr logs
2. Verify provider credentials
3. Manual search in Bazarr

### Permission issues

All services should be in the `media` group:
```bash
ls -la /media/
# Should show group as 'media'
```

---

## API Keys Reference

After setup, note your API keys:

| Service | Location |
|---------|----------|
| Radarr | Settings → General → API Key |
| Sonarr | Settings → General → API Key |
| Prowlarr | Settings → General → API Key |
| Jellyfin | Dashboard → API Keys |

---

## Related

- [[Jellyfin Setup]] - Media server configuration
- [[Proxy Server Setup]] - Caddy reverse proxy on th0r
- [[Tailscale]] - Mesh networking

---

[[index|← Home]]
