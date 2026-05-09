# Waybar Satellite Weather Widget

A custom Waybar element that displays live weather via `wttr.in` and shows the latest satellite image on click.

## Goal

- Waybar module showing current weather (text/icon) via `wttr.in`
- Clicking the module opens a terminal or popup displaying the latest satellite image for Gauteng, rendered with `chafa`

## Satellite Imagery Sources

### NASA GIBS WMS (primary — easy, no auth)

Free public WMS API, no key required. Daily MODIS/VIIRS passes.

```sh
curl -sL "https://gibs.earthdata.nasa.gov/wms/epsg4326/best/wms.cgi?\
SERVICE=WMS&REQUEST=GetMap&VERSION=1.3.0\
&LAYERS=MODIS_Aqua_CorrectedReflectance_TrueColor\
&CRS=EPSG:4326&BBOX=-28,26,-24,31\
&WIDTH=1200&HEIGHT=900&FORMAT=image/jpeg\
&TIME=$(date -d yesterday +%Y-%m-%d)" -o /tmp/sat.jpg && chafa /tmp/sat.jpg
```

**BBOX** for Gauteng with regional context: `-28,26,-24,31` (minLat, minLon, maxLat, maxLon)

**Available layers:**

| Layer | Resolution | Notes |
|---|---|---|
| `MODIS_Terra_CorrectedReflectance_TrueColor` | 250m | Morning pass |
| `MODIS_Aqua_CorrectedReflectance_TrueColor` | 250m | Afternoon pass — usually best |
| `VIIRS_SNPP_CorrectedReflectance_TrueColor` | 375m | Often clearer |
| `VIIRS_NOAA20_CorrectedReflectance_TrueColor` | 375m | NOAA-20 pass |

- Images lag ~1 day; use `date -d yesterday` for TIME parameter
- Infrared layers also available for night imagery

### EUMETView / EUMETSAT (more live — 15-min Meteosat updates)

Geostationary satellite covering Africa (Meteosat-12). WMS API available.

- GitLab with API examples: `https://gitlab.eumetsat.int/eumetlab/data-services/eumetview`
- May require a free EUMETSAT account for some layers
- 15-minute update frequency — significantly more live than NASA MODIS

### RainViewer (free, no key — ~10 min updates)

Provides infrared satellite + radar tiles via a JSON manifest API.

```sh
curl -s "https://api.rainviewer.com/public/weather-maps.json"
```

Returns timestamped tile URLs. Downside: tile-based (z/x/y) rather than bbox WMS — requires tile coordinate calculation or stitching.

### Copernicus Dataspace

- Available at `https://dataspace.copernicus.eu`
- Requires registration; more complex API
- Better suited for high-resolution historical/analytical use than a live widget

### Sat24

No stable public API — not viable.

## wttr.in Integration

```sh
# Simple one-liner for current conditions
curl -s "wttr.in/Johannesburg?format=3"

# Full ASCII weather map
curl wttr.in/Johannesburg
```

## chafa Tips

```sh
# Scale to terminal size
chafa --size=$(tput cols)x$(tput lines) /tmp/sat.jpg

# Kitty graphics protocol (higher quality — Hyprland terminals likely support this)
chafa --format=kitty /tmp/sat.jpg
```

## Waybar Integration Plan

- **Module text**: `wttr.in` formatted output (temp + condition icon)
- **On-click**: script that fetches satellite image and opens it in a floating terminal via `chafa`
- **NixOS packages needed**: `chafa`, plus whatever terminal emulator is used for the popup

## Status

- [ ] Decide on imagery source (NASA daily vs EUMETView live)
- [ ] Write fetch script
- [ ] Configure Waybar custom module
- [ ] Wire up on-click popup
