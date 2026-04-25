# Library Stack Setup on ba1dr

Complete guide for the automated book, audiobook and research paper stack running on ba1dr.

## ⚠ Outstanding Issues

> Work through these in order — each depends on the previous being solved.

- [ ] **Indexers** — current indexers are low quality, not finding enough content. See [[#Indexer Notes]] below. Consider whether manual download + Calibre import is a better workflow until this is resolved. Check how Rachelle set hers up.
- [ ] **Readarr ↔ Calibre sync** — do not configure until indexers are working and downloads are confirmed. See [[#Readarr Calibre Integration]] below.
- [ ] **KOReader sync** — do not set up until searching and downloading is confirmed working. See [[#KOReader Setup]] below.

---

## Indexer Notes

Current problem: Prowlarr indexers for books are either low quality, require invites, or are unreliable.

**Options to investigate:**

| Option | Notes |
|--------|-------|
| Anna's Archive | No Prowlarr indexer — manual download only via annas-archive.org |
| Library Genesis | Prowlarr indexer exists but hit or miss — check if Libgen Fiction vs Non-Fiction variant matters |
| MyAnonamouse | Best quality, requires free account registration at myanonamouse.net |
| Z-Library | No reliable Prowlarr indexer, manual only |

**Manual download workflow (fallback):**
If indexers remain unreliable, bypass Readarr entirely:
1. Find book on Anna's Archive or Z-Library
2. Download epub/pdf manually
3. Import into Calibre via the desktop UI (`calibre`) or upload via Calibre-Web
4. Calibre-Web picks it up automatically

Check how Rachelle set up her indexers — she may have working credentials or a better source.

---

## Readarr Calibre Integration

> **Do not configure until indexers are working and a test download has succeeded end-to-end.**

Once downloads are confirmed working, connect Readarr to Calibre so imported books get proper metadata:

1. **Readarr** (Settings → Media Management):
   - Enable "Use Calibre"
   - Calibre Host: `localhost`
   - Calibre Port: `8083`
   - Calibre Library: `/media/books`
2. Test by adding a book in Readarr and confirming it appears in Calibre-Web with correct metadata

---

## KOReader Setup

> **Do not configure until searching and downloading is confirmed working.**

KOReader can sync with Calibre-Web via OPDS or the KOReader sync server plugin.

**OPDS (simplest — read only):**
1. In KOReader: Search → OPDS Catalogue → Add
2. URL: `http://ba1dr:8083/opds`
3. Browse and download books directly to device

**KOReader Sync (reading position + highlights):**
1. In Calibre-Web: Admin → Edit Calibre-Web → Enable KOReader Sync
2. In KOReader: Tools → Progress sync → Custom sync server
3. URL: `http://ba1dr:8083`
4. Register with your Calibre-Web username/password

---

## Architecture

```
┌─────────────┐     ┌───────────┐     ┌─────────────┐
│   Readarr   │────▶│  Prowlarr │────▶│ qBittorrent │
│   :8787     │     │   :9696   │     │   :8085     │
│   (books)   │     │ (indexers)│     │ (download)  │
└──────┬──────┘     └───────────┘     └──────┬──────┘
       │                                      │
       │                               ┌──────▼──────┐
       │                               │   /media/   │
       │                               │    books/   │
       │                               └──────┬──────┘
       │                                      │
       ▼                                      ▼
┌─────────────┐                       ┌─────────────┐
│ Calibre-Web │◀──────────────────────│   Calibre   │
│   :8083     │                       │  (library)  │
│ (reading UI)│                       └─────────────┘
└─────────────┘
```

## Request Flow

1. **Search**: Find a book in Readarr by title or author
2. **Download**: Readarr searches Prowlarr indexers (Libgen, Audiobookbay) and sends to qBittorrent
3. **Organise**: Readarr renames and moves the file to `/media/books`
4. **Library**: Calibre-Web picks up the new file automatically
5. **Read**: Access via Calibre-Web in browser or send to a reading device

---

## Service Overview

| Service | Port | Purpose | Access |
|---------|------|---------|--------|
| Readarr | 8787 | Book/audiobook management | `http://ba1dr:8787` (internal) |
| Calibre-Web | 8083 | Reading UI and library browser | `http://ba1dr:8083` (internal) |
| Prowlarr | 9696 | Indexer manager (shared with media stack) | `http://ba1dr:9696` (internal) |
| qBittorrent | 8085 | Torrent client (shared with media stack) | `http://ba1dr:8085` (internal) |

## Directory Structure

```
/media/
├── books/       # Readarr root folder / Calibre library
└── downloads/   # qBittorrent download location (shared, temporary)
```

---

## Post-Deployment Configuration

Configure services in this order after `nixos-rebuild switch`.

> qBittorrent and Prowlarr should already be configured from the media stack.
> If starting fresh see [[Media Stack Setup]] steps 1 and 2 first.

### 1. Prowlarr — Add Book Indexers (http://ba1dr:9696)

Add indexers specifically for books and research papers.

1. Go to **Indexers → Add Indexer** (`+`)
2. Add **Library Genesis (Libgen)**:
   - Search for `libgen`
   - Select the Libgen Fiction or Libgen Non-fiction variant as needed
   - Test and save
3. Add **Audiobookbay** (for audiobooks):
   - Search for `audiobookbay`
   - Test and save
4. Add **MyAnonamouse** (high quality books/audiobooks, requires free account):
   - Search for `myanon`
   - Enter credentials
   - Test and save
5. **Add Readarr as an Application** (Settings → Apps → Add):
   - Select **Readarr**
   - Prowlarr Server: `http://localhost:9696`
   - Readarr Server: `http://localhost:8787`
   - API Key: (get from Readarr → Settings → General after next step)
   - Sync Level: Full Sync
   - Test and save

### 2. Readarr (http://ba1dr:8787)

Readarr manages book and audiobook downloads.

1. **Initial Setup**:
   - Set authentication (Settings → General → Authentication)
   - Create username/password

2. **Get API Key** (Settings → General → Security):
   - Copy API Key — go back and add it to Prowlarr now (step 1.5 above)

3. **Media Management** (Settings → Media Management):
   - Click "Add Root Folder"
   - Path: `/media/books`
   - Enable "Rename Books" — recommended for clean library structure
   - Book naming format: `{Author Name}/{Book Title} ({Release Year})`

4. **Download Client** (Settings → Download Clients → Add):
   - Select **qBittorrent**
   - Host: `localhost`
   - Port: `8085`
   - Username/Password: (from qBittorrent setup)
   - Category: `readarr`
   - Test and save

5. **Indexers** (Settings → Indexers):
   - Should sync automatically from Prowlarr
   - Verify Libgen and Audiobookbay appear

6. **Quality Profiles** (Settings → Profiles):
   - Default profile covers epub, pdf, mobi
   - For audiobooks: create a separate profile prioritising mp3/m4b formats

7. **Metadata** (Settings → Metadata):
   - Enable Calibre metadata so downloaded books get proper tags

### 3. Calibre-Web (http://ba1dr:8083)

Calibre-Web provides the reading interface over the Calibre library.

1. **First Login**:
   - Default credentials: `admin` / `admin123`
   - Change password immediately (Admin → Edit User)

2. **Library Path**:
   - Should be pre-configured to `/media/books` via NixOS config
   - If not: Admin → Edit Calibre-Web → Calibre Library Path → `/media/books`

3. **Enable Uploads** (Admin → Edit Calibre-Web):
   - Enable "Allow Uploading"
   - This allows Readarr to import books into the library

4. **Readarr Integration** (Admin → Edit Calibre-Web):
   - Enable the Calibre Content Server if you want Readarr to use Calibre directly for imports
   - Readarr → Settings → Media Management → Calibre Host: `localhost:8083`

5. **Reading Features**:
   - Built-in reader supports epub in browser
   - "Send to Kindle/Kobo" can be configured under user settings with your device email
   - OPDS feed available at `http://ba1dr:8083/opds` for e-reader apps (KOReader, Moonreader)

---

## Finding Content

### Books and Research Papers

| Source | Type | Notes |
|--------|------|-------|
| Library Genesis | Ebooks, textbooks, papers | Largest collection, add via Prowlarr |
| Anna's Archive | Aggregator | Mirrors Libgen + Sci-Hub, browse at annas-archive.org |
| Sci-Hub | Research papers | Use DOI to find specific papers |
| Audiobookbay | Audiobooks | Add via Prowlarr |
| MyAnonamouse | Books + audiobooks | Requires free account, high quality |

### Searching in Readarr

- Search by **author name** to monitor all future releases automatically
- Search by **book title** for one-off downloads
- Add an author to "Monitored" and Readarr will grab new releases automatically

---

## Usage Workflow

### Downloading a Book

1. Go to `http://ba1dr:8787`
2. Click **Add Book** → search by title or author
3. Select the book → choose quality profile → **Add Book**
4. Readarr searches indexers and sends to qBittorrent
5. File lands in `/media/books` and appears in Calibre-Web

### Downloading a Research Paper

1. Find the DOI or title on Anna's Archive or Sci-Hub
2. Download the PDF manually
3. Upload directly to Calibre-Web (top menu → Upload)
4. Calibre-Web adds metadata automatically

### Reading

- **Browser**: `http://ba1dr:8083` → click any epub to read in browser
- **E-reader app**: Point KOReader/Moonreader at `http://ba1dr:8083/opds`
- **Download**: Click the download button on any book for the file

---

## Troubleshooting

### Book not found by Readarr

1. Check Prowlarr → Indexers — verify Libgen is working (Test button)
2. Try searching manually in Prowlarr → Search
3. Some textbooks are only on Libgen Non-fiction — add both variants

### File downloaded but not in Calibre-Web

1. Check Readarr Activity tab for import errors
2. Verify `/media/books` is writable by the `media` group:
   ```bash
   ls -la /media/books
   ```
3. Trigger a manual library scan in Calibre-Web (Admin → Tasks → Reconnect Database)

### Calibre-Web shows empty library

1. Confirm library path is `/media/books` in Admin settings
2. Check that a `metadata.db` file exists in `/media/books` — Calibre creates this on first use
3. If missing, initialise it:
   ```bash
   nix run nixpkgs#calibre -- calibredb add /media/books --empty
   ```

### Permission issues

All services run under the `media` group:
```bash
ls -la /media/books
# Should show group 'media' with write permission
```

---

## API Keys Reference

| Service | Location |
|---------|----------|
| Readarr | Settings → General → API Key |
| Prowlarr | Settings → General → API Key |

---

## Related

- [[Media Stack Setup]] — Movies and TV setup, qBittorrent and Prowlarr configuration
- [[Jellyfin Setup]] — Media server

---

[[index|← Home]]
