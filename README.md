# H3 vs Rasters: Agricultural Data Integration Demo

Side-by-side interactive comparison of **H3 hexagonal grids** and **raster grids** for combining agricultural data layers. Set in Embu County, Kenya — a real smallholder maize-growing region.

## Quick Start

```bash
cd app
npm install
npm run dev        # → http://localhost:5173
```

No API keys needed. All data is synthetic.

---

## The Problem

Agricultural systems must join data from multiple sources to produce per-farm recommendations:

| Data Layer | Source | Native Resolution |
|-----------|--------|-------------------|
| Soil quality | National soil survey | ~5 km² |
| Rainfall | Weather model | ~36 km² |
| Maize health (NDVI) | Satellite imagery | ~0.1 km² |

Each source arrives at a **different resolution, grid alignment, and coordinate system**. To produce a combined planting advisory, these must be joined at a common scale.

---

## How H3 Solves This

H3 is a hierarchical hexagonal grid. Every hex at a fine resolution **nests exactly into one parent** at the next coarser resolution — no overlaps, no gaps.

```
Res 6  (rainfall)     [A]                    ← 1 hex covers ~36 km²
                    /   |   \
Res 7  (soil)     [A1] [A2] [A3] ...         ← 7 children, each ~5 km²
                  / | \
Res 8  (advisory) [A1a][A1b]...[A1g]         ← 7 children, each ~0.7 km²
                  / | \
Res 9  (NDVI)   [A1a₁][A1a₂]...[A1a₇]       ← 7 children, each ~0.1 km²
```

**Every child belongs to exactly one parent.** This makes aggregation mathematically exact.

---

## What the Demo Shows

### Layer Counts (this dataset)

| Layer | H3 Resolution | Hex Count | Area per Hex |
|-------|:---:|---:|---:|
| Rainfall | 6 | **51** | ~36 km² |
| Soil | 7 | **354** | ~5 km² |
| Advisory | 8 | **2,477** | ~0.7 km² |
| Maize NDVI | 9 | **17,335** | ~0.1 km² |

All four layers cover the same geographic area. Their hex counts differ because finer resolutions have smaller hexagons.

### Aggregation to Common Resolution (Res 8)

When you click **SUM**, **MIN**, or **MAX**, all three source layers are brought to Res 8 (farm-plot scale):

#### Soil (Res 7 → Res 8): Upsample

Soil is *coarser* than the target. Each res-7 hex contains ~7 res-8 children. Children inherit the parent's value.

```
Before:  1 soil hex at res 7     value = 92.6
After:   7 child hexes at res 8  each value = 92.6

354 soil hexes → 2,478 advisory hexes
```

No interpolation. No ambiguity. Each child knows its parent.

#### Rainfall (Res 6 → Res 8): Upsample

Rainfall is even coarser. Each res-6 hex contains ~49 res-8 children (7 × 7, two levels up).

```
Before:  1 rainfall hex at res 6   value = 540mm
After:  49 child hexes at res 8    each value = 540mm

51 rainfall hexes → 2,499 advisory hexes
```

#### Maize NDVI (Res 9 → Res 8): Aggregate Down

NDVI is *finer* than the target. Each res-8 parent contains ~7 res-9 children. The selected function (SUM/MIN/MAX) combines them.

```
Before:  7 NDVI hexes at res 9     values = [0.72, 0.68, 0.81, 0.65, 0.77, 0.70, 0.74]
After:   1 advisory hex at res 8

  SUM → 5.07    (all children summed — exact, no double-counting)
  MIN → 0.65    (worst child — unambiguous)
  MAX → 0.81    (best child — unambiguous)
  AVG → 0.724   (mean of all children)

17,335 NDVI hexes → 2,553 advisory hexes
```

### SUM Conservation Proof

When aggregating NDVI down (res 9 → res 8), the sum is **exactly conserved**:

```
Source sum (17,335 hexes): 8,672.8
Target sum  (2,553 hexes): 8,672.8
                    Error: 0.0000%
                     Time: ~23ms
```

This is because every res-9 hex maps to exactly one res-8 parent. No hex is counted twice. No hex is missed.

---

## Why Rasters Fail at This

Raster grids from different sources have:

| Problem | Soil Grid | Rainfall Grid | NDVI Grid |
|---------|:---------:|:------------:|:---------:|
| Cell size | 0.038° × 0.035° | 0.09° × 0.085° | 0.017° × 0.016° |
| Grid offset | +0.01° lat | -0.02° lat | +0.003° lat |
| Rotation | 2.5° | -3° | 0.8° |

These grids **do not align**. When you try to aggregate:

```
┌────────┐
│ Soil   │    ┌──────────────┐
│ pixel  │    │   Rainfall   │     Pixels overlap partially.
│        │    │   pixel      │     Which soil value goes with
└────────┘    │              │     which rainfall cell?
   ┌─────────┐└──────────────┘
   │  NDVI   │
   │  pixel  │  ← Different size, offset, AND rotated
   └─────────┘
```

**SUM**: Edge pixels straddle 2+ coarse cells. Split 50/50? By area? Each rule gives different totals.

| Layer | Simulated Error |
|-------|:---:|
| Soil | **-3.5%** |
| Rainfall | **+4.2%** |
| Maize NDVI | **-3.8%** |

**MIN/MAX**: A boundary pixel belongs to two cells. Its value inflates the MAX in both, or the MIN in both.

**Time**: Requires manual GIS work (reprojection, resampling, clipping) — minutes to hours per run. Cannot automate for millions of farms.

---

## Key Comparisons at a Glance

| | H3 Hexagonal Grid | Raster Grid |
|---|:---:|:---:|
| **Parent-child mapping** | Exact 1:1 | Undefined (pixels don't nest) |
| **SUM conservation** | 0.000% error | 3–5% error per layer |
| **Boundary ambiguity** | None (no shared edges) | ~17% of pixels disputed |
| **Aggregation time** | ~35ms (all 3 layers) | N/A (manual GIS) |
| **Coverage** | 100% (no gaps or overlaps) | Gaps at rotated grid edges |
| **Multi-source join** | Table join on hex ID | Reproject + resample + clip |
| **Coordinate system** | One global grid | Different CRS per source |
| **Farm-level ID** | Stable hex ID forever | Pixel coordinates shift per image |

---

## Tech Stack

| | |
|---|---|
| Frontend | React 18 + TypeScript (Vite) |
| Maps | deck.gl `H3HexagonLayer` + `PolygonLayer` |
| Base tiles | MapLibre GL JS (CARTO dark-matter, free) |
| H3 | h3-js v4 |
| Styling | Tailwind CSS v4 |
| Data | Synthetic (Perlin noise), generated at startup |
