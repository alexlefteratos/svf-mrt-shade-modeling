# Urban Thermal Comfort Analysis
### Sky View Factor · Shade · Mean Radiant Temperature · UTCI

A Google Colab notebook for computing pixel-level outdoor thermal comfort metrics from a normalized Digital Surface Model (nDSM). Point it at any urban site's GeoTIFF, pick a date and time, and the notebook handles the rest — coordinate detection, weather data, sun position, and rendering.

---

## Table of Contents

- [What It Computes](#what-it-computes)
- [Repository Structure](#repository-structure)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Configuration Reference](#configuration-reference)
- [Daily Animation Mode](#daily-animation-mode)
- [Adapting to a New Site](#adapting-to-a-new-site)
- [Output Interpretation](#output-interpretation)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## What It Computes

| Output | Description |
|--------|-------------|
| **Sky View Factor (SVF)** | Fraction of sky visible from each ground-level pixel (0–1) |
| **Shade** | Binary sun/shadow state at a given date and time |
| **Mean Radiant Temperature (MRT)** | Radiant heat load (°C) combining longwave and shortwave radiation |
| **UTCI** | Universal Thermal Climate Index (°C) — a single thermal comfort score |

---

## Repository Structure

```
.
├── README.md                          ← You are here
├── SVF_shade_MRT_calculation.ipynb    ← Main analysis notebook
└── data/
    └── your_nDSM_file.tif            ← Your input raster (not tracked in git)
```

> **Note:** nDSM raster files are typically large and should be added to `.gitignore`. Upload them to Google Drive and reference via path in the notebook.

---

## Requirements

**Platform:** Google Colab (recommended)

**Input:** A single-band GeoTIFF (`.tif`) containing a normalized Digital Surface Model (nDSM) in any projected CRS (e.g., UTM, State Plane). Values should represent above-ground heights with ground level near zero.

**Internet:** Required for weather data from the [Open-Meteo API](https://open-meteo.com/) (free, no API key needed).

**Python packages** (installed automatically by the notebook):

| Package | Purpose |
|---------|---------|
| `numba` | JIT-compiled ray-tracing kernel |
| `rasterio` | GeoTIFF reading and CRS handling |
| `requests` | Open-Meteo API calls |
| `timezonefinder` / `pytz` | Automatic timezone detection |
| `numpy` / `matplotlib` / `seaborn` | Computation and plotting |
| `mplcursors` | Interactive plot inspection |
| `Pillow` | GIF generation |

---

## Quick Start

### 1. Upload your nDSM to Google Drive

Place your `.tif` file in a folder on your Drive. Note the full path.

### 2. Open the notebook in Colab

Upload `SVF_shade_MRT_calculation.ipynb` to Colab, or open it directly from GitHub:

```
File → Open notebook → GitHub → paste the repo URL
```

### 3. Set your file path (Cell 6)

```python
path_file = '/content/drive/MyDrive/your-folder/your_nDSM_file.tif'
```

### 4. Set your date and time (Cell 10)

```python
SIMULATION_DATETIME = datetime(2025, 7, 1, 14, 0, 0)  # Year, Month, Day, Hour, Min, Sec
```

### 5. Run all cells

`Runtime → Run all`. The notebook will:
- Mount your Google Drive
- Load and preview the nDSM
- Auto-detect coordinates and timezone from the raster CRS
- Fetch hourly weather (temperature, humidity, radiation, wind) from Open-Meteo
- Compute SVF, shade, MRT, and UTCI
- Display heatmap plots for each output

---

## Configuration Reference

All parameters are in **Cell 10**. This is the only cell you need to edit for a standard analysis.

### Date and Time

```python
SIMULATION_DATETIME = datetime(2025, 7, 1, 14, 0, 0)
```

Local date and time to simulate. Format: `(year, month, day, hour, minute, second)`.

### Wind Speed

```python
WIND_SPEED_MPS = 2.0
```

Wind speed in m/s at ~10 m height. Used for UTCI. Overridden by API data when weather fetch succeeds.

### SVF Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `N_AZIMUTH` | `72` | Number of ray directions (72 = every 5°). Lower = faster, higher = smoother. |
| `MAX_DISTANCE_M` | `200.0` | Max ray search distance in meters. Increase for sites near distant tall buildings. |
| `GROUND_THRESHOLD_M` | `1.0` | Heights below this are treated as ground for SVF. Pixels above are buildings (SVF = NaN). |

### Coordinate Detection

```python
AUTO_DETECT_COORDS = True
MANUAL_LAT = 40.74
MANUAL_LON = -74.03
```

When `True`, coordinates and cell size are read from the raster's CRS metadata. Set `False` and fill in manual values only if your raster lacks CRS information.

---

## Daily Animation Mode

Cells 24–27 generate 24-hour animated GIFs independently of the single-timestep analysis.

### Configuration (Cell 24)

```python
ANIMATION_DATE = datetime(2025, 7, 1)  # Date to animate (year, month, day)
GIF_FPS = 2                             # Frames per second
GIF_DPI = 100                           # Image resolution
```

### Output Directory (Cell 26)

```python
GIF_OUTPUT_DIR = '/content/drive/MyDrive/your-folder/'
```

### Generated Files

| File | Description |
|------|-------------|
| `shade_YYYYMMDD.gif` | Shadow animation (black = shaded, white = sunlit) |
| `mrt_YYYYMMDD.gif` | Mean Radiant Temperature animation |
| `utci_YYYYMMDD.gif` | UTCI thermal comfort animation |
| `svf_YYYYMMDD.png` | Static SVF image (geometry-only, doesn't change hourly) |

### Performance

SVF is computed once (geometry only). Shade, MRT, and UTCI are recomputed for each of the 24 hours. For a ~200×300 pixel raster, expect roughly 5–10 minutes total. Larger rasters or higher `N_AZIMUTH` values scale proportionally.

---

## Adapting to a New Site

To analyze a completely different location, change only two things:

1. **Cell 6** — Update `path_file` to your new nDSM GeoTIFF
2. **Cell 10** — Update `SIMULATION_DATETIME` to the date/time of interest

Everything else adapts automatically: coordinates are extracted from the raster CRS, timezone is detected from the coordinates, and weather is fetched from Open-Meteo for the detected location and chosen datetime.

### nDSM Requirements

| Property | Requirement |
|----------|-------------|
| Format | Single-band GeoTIFF (`.tif`) |
| Values | Above-ground heights (nDSM). Ground ≈ 0. |
| CRS | Any projected CRS with linear units (meters or feet). UTM, State Plane, etc. |
| Nodata | Large negative values (e.g., `-3.4e+38`) are handled automatically as NaN. |

---

## Output Interpretation

### Sky View Factor (SVF)

Values range 0–1. A value of 1 means fully open sky; values near 0 indicate enclosed locations like narrow street canyons. Only ground-level pixels (below `GROUND_THRESHOLD_M`) receive values; building pixels are NaN.

### Shade

Binary per pixel: `1` = shaded by a nearby structure, `0` = in direct sunlight. All pixels are shaded when the sun is below the horizon.

### Mean Radiant Temperature (MRT)

The temperature (°C) of a uniform enclosure producing the same radiant heat exchange as the real environment. Combines longwave radiation (from buildings and sky, weighted by SVF) with shortwave solar radiation (weighted by shade state). Higher MRT = greater radiant heat stress.

### UTCI

A single index (°C) integrating air temperature, humidity, wind, and MRT:

| UTCI (°C) | Stress Category |
|-----------|----------------|
| > 46 | Extreme heat stress |
| 38 – 46 | Very strong heat stress |
| 32 – 38 | Strong heat stress |
| 26 – 32 | Moderate heat stress |
| 9 – 26 | No thermal stress |
| 0 – 9 | Slight cold stress |
| < 0 | Moderate to extreme cold stress |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| **Blank SVF plot** | `GROUND_THRESHOLD_M` may be too low for your nDSM. Check ground-level values with `np.percentile(ar_ndsm[ar_ndsm > -1e30], [1, 5, 10, 50])` and increase the threshold accordingly. |
| **Weather fetch failed** | Requires internet. Offline fallback uses defaults (32°C, 60% RH, 800 W/m²). Override `Ta`, `RH`, `K_global`, `K_diffuse` manually in Cell 11 after the `except` block. |
| **Slow computation** | Reduce `N_AZIMUTH` (72 → 36) or `MAX_DISTANCE_M`. Same parameters apply to each animation frame. |
| **CRS not detected** | Set `AUTO_DETECT_COORDS = False` in Cell 10. Fill in `MANUAL_LAT`, `MANUAL_LON`, and set `CELLSIZE` manually in Cell 11. |
| **Negative nDSM values** | Small negatives are normal (LiDAR noise) and clamped to zero automatically. Very large negatives (e.g., `-3.4e+38`) are nodata and handled as NaN. |

---

## License

<!-- Replace with your chosen license -->
This project is licensed under the [MIT License](LICENSE).
