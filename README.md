# DAPI Nuclei Tools

![Python](https://img.shields.io/badge/Python-3.9+-3776AB?style=flat&logo=python&logoColor=white)
![OpenCV](https://img.shields.io/badge/OpenCV-4.x-5C3EE8?style=flat&logo=opencv&logoColor=white)
![scikit-image](https://img.shields.io/badge/scikit--image-0.19+-F7931E?style=flat)
![Built with ChatGPT 5.2](https://img.shields.io/badge/Built%20with-ChatGPT%205.2-74aa9c?style=flat&logo=openai&logoColor=white)

Two Python GUI applications for **automated DAPI-stained nuclei analysis** in fluorescence microscopy images. This project was developed with the assistance of **ChatGPT 5.2**.

---

## Context & Origin

This toolset was developed as part of a **Bachelor's thesis** at the **Hochschule Mittweida (HSMW)**, investigating:

> **Zellvitalität und Nährstoffversorgung in 3D-Hydrogelen unter dem Einfluss perfundierter und zellbesiedelter Kollagenkanäle**
>
> *(Cell viability and nutrient supply in 3D hydrogels under the influence of perfused and cell-seeded collagen channels)*

The DAPI Analyze tool was specifically designed to quantify nuclei distribution and density as a function of distance to perfused collagen channels within hydrogel constructs.

> [!IMPORTANT]
> **Regarding default parameters:** The default segmentation parameters (e.g. `gaussian_sigma=0.57`, `local_radius_px=5`, `min_size_px2=40`, `max_size_px2=500`, `watershed_h_px=2.2`) were determined through systematic validation using the DAPI Calibrator tool on images from **my specific microscopy setup and sample preparation**. These parameters are optimized for my acquisition conditions (microscope type, magnification, camera, staining protocol, hydrogel samples) and **may not be optimal for other imaging setups or sample types**. It is strongly recommended to run your own calibration using the DAPI Calibrator with your own ground-truth counts before applying these tools to a different dataset.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
  - [DAPI Analyze](#dapi-analyze)
  - [DAPI Calibrator](#dapi-calibrator)
- [Segmentation Algorithm](#segmentation-algorithm)
- [Distance Classification](#distance-classification)
- [Output Files](#output-files)
- [CSV Format](#csv-format)
- [Project Structure](#project-structure)
- [Examples](#examples)
- [Parameter Validation Notice](#parameter-validation-notice)
- [Development](#development)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## Overview

| Tool | Purpose |
|---|---|
| **DAPI Analyze** (`dapi_analyze.py`) | Segment, count, classify nuclei by distance to tissue boundaries, generate heatmaps and zone maps |
| **DAPI Calibrator** (`dapi_calibrator.py`) | Find optimal segmentation parameters via grid search against manual ground-truth counts |

Both tools provide **tkinter-based GUIs** and require no command-line interaction.

---

## Features

### DAPI Analyze

- **Nuclei segmentation** using Phansalkar local thresholding + optional watershed splitting
- **Distance-based classification** of nuclei relative to inner/outer tissue boundaries
- **Batch processing** with manual queue and per-image µm/px scale
- **Grid-based heatmap** of nuclei counts (turbo/jet colormap, fixed scale)
- **Color-coded zone visualization** with overlap shading between boundaries
- **Comprehensive CSV export** (per-object data, summary statistics, density tiles)
- **German-locale CSV** (`;` separator, `,` decimal) for direct Excel compatibility
- **Line artifact removal** for cleaner segmentation on stitched images
- **Abort support** for long-running analyses

#### Analysis Modes

| Mode | Boundaries needed | CSV output | Image output |
|---|---|---|---|
| **Count only** | ❌ | ✅ | Heatmap only |
| **Distance CSV** | ✅ | ✅ | ❌ |
| **Distance + Images** | ✅ | ✅ | ✅ All |
| **Batch (Queue)** | Per job | ✅ | ✅ Configurable |

### DAPI Calibrator

- **Parameter grid search** defined via simple CSV file
- **Two search modes**: Staged pruning (fast) or brute force (exhaustive)
- **Ground truth UI** with per-image count entry, CSV import/export
- **Live CSV streaming** — results written to disk during calibration
- **Visual verification** — top-N parameter sets get overlay images saved
- **Smart image ordering** — zigzag by ground truth for maximum early pruning

#### Staged Pruning Schedule

| Stage | Keep fraction | Minimum survivors |
|---|---|---|
| 1 | 50% | 100 |
| 2 | 50% | 100 |
| 3+ | 20% | 100 |

---

## Installation

### Prerequisites

- Python ≥ 3.9
- A system with `tkinter` support (included in most Python distributions)

### Install Dependencies

```bash
pip install -r requirements.txt
```

Or manually:

```bash
pip install numpy pandas tifffile opencv-python scipy scikit-image
```

### Supported Image Formats

`.tif`, `.tiff`, `.png`, `.jpg`, `.jpeg`, `.bmp`

---

## Usage

### DAPI Analyze

```bash
python dapi_analyze.py
```

#### Single Image Analysis

1. **Select mode** — Count only, Distance CSV, or Distance + Images
2. **Choose DAPI image** — fluorescence microscopy image (blue channel extracted automatically)
3. **Choose boundary images** (distance mode only) — Outer and Inner boundary line images
4. **Set Pixel per µm** — must be > 0 (determines all physical measurements)
5. **Set distance thresholds** — e.g. `200,500` for three zones
6. **Select output folder**
7. **Click ▶ Run**

#### Batch Mode

1. Switch to **Batch (Queue)** mode
2. For each image:
   - Select DAPI (+ optional boundary images)
   - Set the correct **px/µm** for that specific image
   - Click **Schnitt hinzufügen** (Add section)
3. Review jobs in the queue table
4. Click **▶ Run** — all jobs are processed sequentially
5. Each job uses its own px/µm scale

#### Key Parameters

| Parameter | Default | Description | Note |
|---|---|---|---|
| Pixel per µm | — | Image scale (required) | Setup-specific |
| Distance thresholds | `200,500` | Zone boundaries in µm | Application-specific |
| Gaussian sigma | `0.57` | Pre-smoothing strength | ⚠️ Validated for my setup |
| Local radius | `5` px | Phansalkar window radius | ⚠️ Validated for my setup |
| Min size | `40` px² | Minimum nucleus area | ⚠️ Validated for my setup |
| Max size | `500` px² | Maximum nucleus area | ⚠️ Validated for my setup |
| Watershed | `enabled` | Split touching nuclei | — |
| Watershed h | `2.2` px | h-maxima seed height | ⚠️ Validated for my setup |

> ⚠️ Parameters marked as "Validated for my setup" were optimized for specific microscopy conditions at HSMW. See [Parameter Validation Notice](#parameter-validation-notice).

### DAPI Calibrator

```bash
python dapi_calibrator.py
```

1. **Prepare a parameter range CSV** (see [examples](#examples))
2. **Select image folder** — containing DAPI images
3. **Enter ground-truth counts** per image (or load from CSV)
4. **Choose mode** — Staged pruning (recommended) or Brute force
5. **Click Start calibration**
6. Monitor progress in the log panel
7. Results and top-N visualizations are saved to the output folder

#### Parameter Range CSV Format

```csv
parameter,min,max,step
gaussian_sigma,0.3,1.0,0.1
local_radius_px,3,7,2
min_size_px2,20,60,10
max_size_px2,300,700,100
do_watershed,0,1,1
watershed_h_px,1.0,4.0,0.5
exclude_edge,1,1,1
```

**All 7 parameters are required:**

| Parameter | Type | Description |
|---|---|---|
| `gaussian_sigma` | float | Gaussian pre-smoothing sigma |
| `local_radius_px` | int | Phansalkar local window radius (≥ 1) |
| `min_size_px2` | int | Minimum nucleus area in px² |
| `max_size_px2` | int | Maximum nucleus area in px² |
| `do_watershed` | bool (0/1) | Enable watershed splitting |
| `watershed_h_px` | float | h-maxima height for watershed seeds |
| `exclude_edge` | bool (0/1) | Remove border-touching nuclei |

---

## Segmentation Algorithm

The nuclei segmentation pipeline consists of the following steps:

### 1. Blue Channel Extraction

The DAPI fluorescence signal is extracted from the blue channel. The tool handles both RGB and BGR (OpenCV) channel ordering automatically.

### 2. Gaussian Smoothing

Optional pre-smoothing with configurable sigma ($\sigma$) to reduce noise.

### 3. Phansalkar Local Thresholding

Adaptive local threshold computed as:

$$T(x,y) = \mu \left(1 + p \cdot e^{-q\mu} + k \cdot \left(\frac{\sigma_{local}}{R} - 1\right)\right)$$

where:
- $\mu$ = local mean intensity
- $\sigma_{local}$ = local standard deviation
- $p = 2.0$, $q = 10.0$ (exponential weighting)
- $k = 0.25$ (sensitivity parameter)
- $R = 0.5$ (dynamic range normalization)

### 4. Global Noise Floor

An additional global threshold at $0.3 \times T_{Otsu}$ removes background noise that may pass the local threshold.

### 5. Morphological Cleanup

- Small object removal (area < `min_size_px2`)
- Binary hole filling
- Line artifact removal (for stitched images): removes components with height ≤ 2 px, width ≥ 80 px, aspect ratio ≥ 15

### 6. Watershed Splitting (Optional)

- Euclidean distance transform of the binary mask
- h-maxima extraction with configurable height $h$
- Marker-controlled watershed on the inverted distance map

### 7. Area Filtering

Final filter: $min\_size\_px^2 \leq A \leq max\_size\_px^2$

---

## Distance Classification

When boundary images (outer + inner) are provided:

### Boundary Detection

Boundary images are automatically binarized using multiple strategies:
- **Otsu** on grayscale
- **Canny** edge detection
- **HSV blue** color detection
- **Non-black** pixel detection

Each candidate is scored and the best is selected automatically.

### Distance Computation

For each nucleus centroid:
- $d_{outer}$ = distance to nearest outer boundary pixel
- $d_{inner}$ = distance to nearest inner boundary pixel
- $d_{min} = \min(d_{outer}, d_{inner})$

Distances are computed using **scipy cKDTree** for performance.

### Classification

Nuclei are classified based on $d_{min}$ and configurable thresholds.

For thresholds `200,500` µm → three classes:

| Class | Range | Label (DE) | Label (EN) | Color |
|---|---|---|---|---|
| 1 | $d < 200\,\mu m$ | nah | near | 🟢 Green |
| 2 | $200 \leq d \leq 500\,\mu m$ | mittel | medium | 🟡 Yellow |
| 3 | $d > 500\,\mu m$ | fern | far | 🔴 Red |

### Subzones

Each class is further divided into subzones:
- **outer_only** (Aussengrenze-nah) — closer to outer boundary
- **inner_only** (Innengrenze-nah) — closer to inner boundary
- **overlap** (Überlappung) — equidistant (same class from both boundaries)

Overlap regions are rendered with a darkened color (factor 0.65) in the zone map.

---

## Output Files

### DAPI Analyze Output

```
dapi_results/
└── <image_name>_<timestamp>/
    ├── <name>_all_data.csv                    # Per-object measurements
    ├── <name>_summary.csv                     # Aggregated statistics
    ├── <name>_analysis_info.txt               # Full run parameters
    ├── <name>_classes_filled.png              # Nuclei colored by distance class
    ├── <name>_distance_zones_ds8.png          # Zone map with overlap shading
    ├── <name>_heatmap_counts_turbo_*.png      # Grid-based count heatmap
    ├── <name>_local_density_tiles_*.csv       # Per-tile density data
    ├── <name>_outer_BEST_*.png                # Debug: outer boundary mask
    ├── <name>_inner_BEST_*.png                # Debug: inner boundary mask
    ├── run_log.txt                            # Runtime log
    ├── run_info.txt                           # Summary info
    └── used_code.txt                          # Copy of script used
```

### DAPI Calibrator Output

```
calibration_<timestamp>/
├── run_info.txt                              # Calibration settings
├── top10_results.csv                         # Best 10 parameter sets
├── finalists_results.csv                     # All surviving candidates (staged)
├── live_results_long.csv                     # Per-stage per-candidate results (staged)
├── live_results_wide.csv                     # All results per combo (brute force)
├── stage_survivors.csv                       # Pruning log per stage (staged)
├── all_results.csv                           # Full grid results (brute force, ≤100k)
└── top01_absErrSum*_countTotal*/             # Visualization per top candidate
    ├── params_used.txt
    ├── per_image_counts.csv
    ├── <image>_nuclei_mask_ds4.png
    ├── <image>_overlay_dots_ds4.png
    └── <image>_label_edges_ds4.png
```

---

## CSV Format

### Object CSV (`_all_data.csv`)

| Column | Description |
|---|---|
| `ObjectID` | Unique nucleus ID |
| `Included` | 1 = inside ring region, 0 = excluded |
| `X_px`, `Y_px` | Centroid in pixels |
| `X_um`, `Y_um` | Centroid in µm |
| `Area_px2`, `Area_um2` | Nucleus area |
| `MeanIntensity` | Mean DAPI intensity (normalized) |
| `DistOuterRaw_um` | Distance to outer boundary |
| `DistInnerRaw_um` | Distance to inner boundary |
| `DistMinDirected_um` | Minimum distance (classification basis) |
| `DistanceClass` | Class code (1, 2, 3, ...) |
| `DistanceClassLabel` | Human-readable label |
| `DistanceSubzone` | `outer_only`, `inner_only`, `overlap`, or `none` |

> **Note:** CSV uses `;` as separator and `,` as decimal point (German Excel format).

### Summary CSV (`_summary.csv`)

| Column | Description |
|---|---|
| `Bildname` | Image filename |
| `Zeilentyp` | Row type (Gesamt / Distanzklasse_Gesamt / Distanzklasse_Teilzone) |
| `Distanzklasse` | Distance class name (nah / mittel / fern) |
| `Teilzone` | Subzone name (Aussengrenze-nah / Innengrenze-nah / Überlappung) |
| `Zellkerne_Anzahl` | Nuclei count |
| `Anteil_Zellkerne_Prozent` | Percentage of total nuclei |
| `Fläche_mm2_geschätzt` | Estimated area in mm² |
| `Dichte_Zellkerne_pro_mm2` | Density (nuclei per mm²) |

---

## Project Structure

```
dapi-nuclei-tools/
├── dapi_analyze.py              # Main analysis GUI application
├── dapi_calibrator.py           # Parameter calibration GUI application
├── requirements.txt             # Python dependencies
├── .gitignore                   # Git ignore rules
├── README.md                    # This file
└── examples/
    └── parameter_ranges.csv     # Example calibrator input CSV
```

---

## Examples

### Example: Count-Only Analysis

```bash
python dapi_analyze.py
# 1. Select "Nur zählen" (Count only)
# 2. Pick a DAPI .tif image
# 3. Set px/µm (e.g. 1.535)
# 4. Pick output folder
# 5. Click ▶ Run
```

### Example: Distance Analysis with Batch

```bash
python dapi_analyze.py
# 1. Select "Batch (Queue)"
# 2. For each section:
#    - Pick DAPI, Outer boundary, Inner boundary
#    - Set px/µm for that specific image
#    - Click "Schnitt hinzufügen"
# 3. Click ▶ Run
```

### Example: Parameter Calibration

```bash
python dapi_calibrator.py
# 1. Select parameter_ranges.csv
# 2. Select folder with DAPI images
# 3. Enter ground truth counts (or load GT CSV)
# 4. Select "Staged pruning"
# 5. Click "Start calibration"
```

#### Ground Truth CSV Format

```csv
filename,ground_truth
sample_001.tif,142
sample_002.tif,89
sample_003.tif,215
```

---

## Parameter Validation Notice

> [!WARNING]
> **The default segmentation parameters in this repository are NOT universal.**

The current defaults were determined through a systematic calibration process using the DAPI Calibrator tool:

| Parameter | Default value | Validated on |
|---|---|---|
| `gaussian_sigma` | 0.57 | DAPI-stained cryosections of collagen-hydrogel constructs |
| `local_radius_px` | 5 | Images acquired at HSMW with specific microscope/camera setup |
| `min_size_px2` | 40 | HDF/HEK293 nuclei in Kollagen-I / Fibrin hydrogels |
| `max_size_px2` | 500 | Same sample type and preparation protocol |
| `watershed_h_px` | 2.2 | Same imaging conditions |

**These parameters may need re-calibration if you use:**
- A different microscope, objective, or camera
- Different magnification or pixel size
- Different cell types or staining protocols
- Different sample preparation (e.g. paraffin sections vs. cryosections)
- Different hydrogel materials or tissue types

**Recommended workflow for new setups:**
1. Manually count nuclei in 5–10 representative images
2. Save counts as a ground-truth CSV
3. Run the **DAPI Calibrator** with appropriate parameter ranges
4. Use the top-ranked parameters in the **DAPI Analyze** tool

---

## Development

This project was developed with the assistance of **ChatGPT 5.2**.
The AI was used for:
- Code architecture and design
- Algorithm implementation
- Debugging and optimization
- Documentation and README writing

The toolset was created for and applied in a **Bachelor's thesis** at the **Hochschule Mittweida (HSMW)**, investigating cell viability and nutrient supply in 3D hydrogels under the influence of perfused and cell-seeded collagen channels.

The scientific methodology (Phansalkar thresholding, watershed segmentation, distance-based classification) is based on established bioimage analysis techniques.

---

## License

*Add your license here (e.g. MIT, GPL-3.0, Apache-2.0).*

---

## Acknowledgments

- [scikit-image](https://scikit-image.org/) — Image processing algorithms
- [OpenCV](https://opencv.org/) — Computer vision library
- [SciPy](https://scipy.org/) — Scientific computing (cKDTree, ndimage)
- [NumPy](https://numpy.org/) — Numerical computing
- [pandas](https://pandas.pydata.org/) — Data handling and CSV export
- [tifffile](https://github.com/cgohlke/tifffile) — TIFF image I/O
- [ChatGPT 5.2](https://openai.com/chatgpt) — AI-assisted development
- **Hochschule Mittweida (HSMW)** — University context for the Bachelor's thesis
