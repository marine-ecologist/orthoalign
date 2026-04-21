# Drone orthomosaic co-registration for time-series change detection

<img width="800" height="488" alt="orthoalign1" src="https://github.com/user-attachments/assets/9f0f1e71-8194-4898-b7f7-0c7e1746f430" />


With timeseries analysis of drone orthomosaics, artefacts in orthomosaic creation or image processing may lead to spatial differences in features that require alignment for paired time-points. `orthoalign` is a desktop Python/tkinter tool for co-registering (aligning two or more orthomosaics so that identical geographic features occupy the same coordinates) designed for time-series change detection in drone orthomosaics. 

`orthoalign` presents a geographically synchronised split-screen view of a reference and target raster allowing the user to place matching Ground Control Point (GCP[^1]) pairs on each panel. Once all required features are matched, `orthoalign` warps the target orthomosaic into alignment with the reference to produce a corrected GeoTIFF. `orthoalign` supports affine, polynomial, and thin-plate spline transforms with auto-selection based on GCP count.

Plenty of excellent tools exist for manual or automated co-registration of satellite imagery (e.g. [AROSICS](https://github.com/GFZ/arosics) for automatic subpixel co-registration of two satellite image datasets). In the case of drone orthomosaics, the magnitude of change between time periods or significant shift between images meant AROSICS and other tools struggled to co-register identifiable features. `orthoalign` is similar to [QGIS Georeferencer](https://docs.qgis.org/3.44/en/docs/user_manual/managing_data_source/georeferencer.html) (manual GCP placement, multiple warp methods, GeoTIFF output) but overcomes the limitation of working on one image at a time in a single-panel view by visualising paired orthomosaics with synchronised zoom and geographic alignment between reference and target[^2].

Additional features include a toggleable metric grid overlay, `.points` export, and memory-efficient tiled processing for large rasters.

[^1]: Matched pixel pairs in this tool differ from *true* physical GCPs used in drone surveys — physical markers placed on the ground before a flight whose exact position is surveyed with a GPS receiver.
[^2]: There may be more efficient or easier to use tools in this space that I'm unaware of, but this approach worked for me!

---

## Installation / requirements

```bash
pip install rasterio numpy Pillow scipy
```

| Package | Role |
|---|---|
| `rasterio` | Raster I/O, GCP reprojection, affine transforms |
| `numpy` | Array maths, tile processing |
| `Pillow` | Display rendering and resampling |
| `scipy` | Thin-plate spline interpolation (`RBFInterpolator`) |

Python 3.9+ recommended. Tested on macOS (Apple Silicon and Intel) and Linux.

---

```bash
# Clone or download the script, then install dependencies
pip install rasterio numpy Pillow scipy

# Run directly
python3 gcp_calibrator_v5.py <reference> <target>
```

---

## Usage

Simple use:

```bash
python3 "~/droneorthomosaics/gcp_calibrator_v5.py" "~/droneorthomosaics/before_ortho.tiff" "~/droneorthomosaics/after_ortho.tiff" --output "~/droneorthomosaics/after_coreferenced_ortho.tiff"
```

Or generally:

```
python3 gcp_calibrator_v5.py <reference> <target> [options]
```

| Argument | Description |
|---|---|
| `reference` | Path to the reference (base) raster — the image to align **to** |
| `target` | Path to the target raster — the image to be warped and corrected |


| Flag | Default | Description |
|---|---|---|
| `-o`, `--output <path>` | `<target>_corrected.tif` | Output path for the corrected GeoTIFF |
| `-t`, `--transform <method>` | `auto` | Transform method: `auto`, `affine`, `polynomial`, or `tps` |
| `--tile-pixels <N>` | `500000` | TPS only — maximum query points per RBF tile. Lower values reduce peak RAM at the cost of speed. e.g. `--tile-pixels 100000` for systems with limited memory |
| `--tps-smoothing <S>` | `0.0` | TPS only — smoothing factor for the radial basis function. `0` = exact interpolation through all GCPs. Values of 1–10 are useful for noisy drone-to-satellite co-registration |

---

## Transform methods

| Method | Description | Use case |
|---|---|---|
| `affine` | 6-parameter linear transform (translation, rotation, scale, shear). ≥ 3 GCPs | Simple offset, rotation or scale difference between orthomosaics |
| `polynomial` | Rubber-sheet warp via rasterio GCP reprojection. ≥ 3 GCPs | Moderate non-linear distortion from lens curvature or terrain relief |
| `tps` | Thin-plate spline fitted through all GCPs via `RBFInterpolator`. Tiled processing controls peak RAM | Severe non-linear distortion across multi-date drone surveys |
| `auto` | Selects method by GCP count: ≤4 → affine, 5–8 → polynomial, ≥9 → tps | Default — recommended for most use cases |

---

## Workflow

### 1. Launch and load

Open the tool with your reference and target rasters. Both are loaded at display resolution (downsampled to ≤6144 px on the longest axis) and rendered side by side. The panels are geographically synchronised from the start — if both rasters share a CRS their extents will overlap automatically. Zoom on either map with  `+` / `-` keys to locate features, move the maps with two-finger drag (or Pan | Shift + drag).

### 2. Place GCPs

Click a recognisable feature on the **Reference panel** (left, green markers), then click the same feature on the **Target panel** (right, red markers). Each pair is numbered. The status bar shows how many paired GCPs exist and how many more are needed.

Minimum GCPs required by method:

| Method | Minimum GCPs |
|---|---|
| Affine | 3 |
| Polynomial | 3 (more improves accuracy) |
| TPS | 3 (accuracy improves significantly with 9+) |

### 3. Adjust the grid (optional)

Use the **Grid** checkbox and spinbox in the bottom panel to overlay a metric grid on both images. The cell size is in the raster's map units (typically metres). The grid is a visual aid only — it has no effect on GCPs or output.

### 4. Run calibration or export

- **Calibrate** — warps the target raster to the reference coordinate system and writes the output GeoTIFF. Progress is shown in the status bar; the UI remains responsive during processing.
- **Export GCPs (.points)** — saves the current paired GCPs as a QGIS Georeferencer `.points` file. Load it in QGIS via *Georeferencer → File → Load GCP Points*.

---

## Controls

| Action | Input |
|---|---|
| Place GCP | Left-click on either panel |
| Pan | Shift + drag, or right/middle-click drag |
| Zoom | Two-finger scroll (pan) / Shift + scroll (zoom) |
| Zoom in / out | `+` / `-` keys |
| Undo last point | `Ctrl+Z` |
| Clear all points | `Ctrl+Shift+Z` |
| Run calibration | `Ctrl+R` |
| Reset view | `H` |

---


## Output

The corrected raster is a GeoTIFF with:

- The same pixel grid and coordinate system as the reference raster
- All bands from the target raster, resampled with bilinear interpolation
- LZW compression (BigTIFF format if the uncompressed size would exceed 3 GB)
- Exported `.points` files follow the QGIS Georeferencer format

---

## License

GPL v3 
