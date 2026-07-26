# QGIS → PCG Mask Workflow — Prep Brief
*For the next build session. Project: ArchVizExplorer (UE 5.8, Cesium), level TKOPPARCHVIZ. Written 26 Jul 2026.*

> **This workflow is untried as of writing — it has never been run.** Nothing below has been
> executed or verified in this project.
> Every step is marked either **[confirmed]** — checked against a cited source or this repo — or
> **[to-verify]** — plausible and sourced, but unproven here. Treat every **[to-verify]** line as a
> question to answer in-editor, not as an instruction to follow blindly. Where a node name, setting,
> or number is not cited, it is not stated.

The **goal**: use real Pakipaki GIS vectors (stream centrelines, wetlands, vegetation polygons) to
decide *exactly where* PCG spawns foliage — so planting follows the actual whenua rather than noise.

**Scope of this brief:** from "you have vectors open in QGIS" to "PCG reads a mask and varies spawn
placement". It stops there deliberately.

- Sourcing/downloading LINZ data → **PAK-16**
- Heightmap & proxy landscape import → **PAK-17**
- Which species go where (biome design) → **PAK-18**

---

## What I've already confirmed (you don't need to re-gather this)

- **The PCG nodes that consume a texture exist and are named.** In UE 5.8, `Get Texture Data`
  "loads a texture to a surface data", and `Texture Sampler` samples UV coordinates at each point
  using planar or explicit point-UV mapping. Downstream, `Density Filter` filters points by density
  against a range, `Density Remap` applies a linear transform to point densities, and
  `Static Mesh Spawner` spawns one mesh per surviving point. `Surface Sampler` generates the initial
  point grid. **[confirmed — [UE 5.8 PCG Node Reference](https://dev.epicgames.com/documentation/en-us/unreal-engine/procedural-content-generation-framework-node-reference-in-unreal-engine)]**
- **So the mask→spawn mechanism is real, not hypothetical.** The shape of the graph is
  `Surface Sampler → (Get Texture Data → Texture Sampler) → Density Filter → Static Mesh Spawner`.
  **[to-verify]** — the node list is confirmed; this exact wiring order is not, and node names should
  be re-checked against the installed 5.8 build before relying on them.
- **Cesium does not understand NZTM2000.** `CesiumGeoreference` converts between Unreal World,
  ECEF (Earth-Centered Earth-Fixed), ENU, and Longitude/Latitude/Height. Projected national grids
  are not in that list. **[confirmed — [ACesiumGeoreference reference](https://cesium.com/learn/cesium-unreal/ref-doc/classACesiumGeoreference.html), [Placing Objects on the Globe](https://cesium.com/learn/unreal/unreal-placing-objects/)]**
  This is the single most important fact in this document — see the alignment section.
- **EPSG:2193 is "NZGD2000 / New Zealand Transverse Mercator 2000"** — Transverse Mercator, central
  meridian 173°, scale factor 0.9996, datum NZGD2000 (GRS 1980 ellipsoid), **units metres**, covering
  onshore North/South/Stewart Islands. **[confirmed — [epsg.io/2193](https://epsg.io/2193)]**
- **`Content/` is gitignored in this repo** (~7.5 GB of `.uasset` binaries), so any texture you import
  lives outside version control. Masks are therefore *build inputs you must keep safe yourself* —
  the repo will not remember them. **[confirmed — `.gitignore`]**
- **QGIS rasterises vectors via GDAL, and the algorithm gives you everything this workflow needs.**
  `gdal:rasterize` ("Rasterize (vector to raster)") takes `WIDTH`/`HEIGHT` — settable in **either pixels
  or georeferenced units**, chosen via *Output raster size units* — assigns pixel values from an
  attribute field, a fixed burn value, or Z-values, and can output **Byte (8-bit unsigned)** among 12
  data types. **[confirmed — [QGIS `gdal:rasterize` docs](https://docs.qgis.org/latest/en/docs/user_manual/processing_algs/gdal/vectorconversion.html)]**
  Being able to specify size in *georeferenced units* is the useful part: it means you can set
  metres-per-pixel directly rather than computing pixel counts by hand (see **Q3**).

---

## 1. Exporting the mask from QGIS

**What the export must carry** — four properties, in priority order:

| Property | Value | Status |
|---|---|---|
| Format | **GeoTIFF** (`.tif`) out of QGIS — the standard raster interchange for UE/GIS work | **[confirmed]** — `gdal:rasterize` writes GeoTIFF; the [GeotiffLandscape plugin](https://github.com/iwer/GeotiffLandscape) exists precisely because GeoTIFF is the expected format |
| Georeferencing | Must be **reprojected to EPSG:4326 (WGS 84 lat/lon)** before it is any use to Cesium, *not* left in EPSG:2193 | **[confirmed]** — follows from Cesium's coordinate list above |
| Bit depth | **Byte (8-bit unsigned)** is sufficient for a binary or banded mask. 16-bit is for *heightmaps* — PAK-17's problem, not this one | **[confirmed]** that `gdal:rasterize` can output Byte ([QGIS docs](https://docs.qgis.org/latest/en/docs/user_manual/processing_algs/gdal/vectorconversion.html)); **[to-verify]** which channel PCG's `Texture Sampler` actually reads |
| Resolution | Set `WIDTH`/`HEIGHT` with *Output raster size units* = **Georeferenced units** to specify metres-per-pixel directly | **[confirmed]** the parameter exists ([QGIS docs](https://docs.qgis.org/latest/en/docs/user_manual/processing_algs/gdal/vectorconversion.html)); the *value* is **Q3**, yours to choose |

> **Caveat on the GeotiffLandscape citation:** that plugin is cited only as evidence that GeoTIFF is
> the expected interchange format, and for its statement that "the grayscale values are mapped to a
> range of 0 to 1 that represent the layer weight". **Do not install it** — it targets UE 4.27–5.1 and
> was **archived (read-only) in May 2026**. It is not a 5.8 tool.

**A caution on `Get Texture Data`:** it is documented for "sampling compressed textures and
CPU-available formats". A GeoTIFF is not a UE texture — it must be imported into the project as a
`Texture2D` first, and its compression/sRGB settings will matter. **[to-verify]** — the required
import settings are not documented in the node reference and must be found by experiment.

---

## 2. How PCG consumes it

Confirmed node roles, from Epic's 5.8 reference:

- **`Get Texture Data`** — loads a texture to surface data.
- **`Texture Sampler`** — samples UVs at each point (planar, or explicit point UV).
- **`Density Filter`** — filters points by density against a filter range. This is the node that turns
  "the mask is dark here" into "no point survives here".
- **`Density Remap`** / **`Curve Remap Density`** — reshape density before filtering, if a hard
  cut-off is too abrupt.
- **`Attribute Filter`** / **`Attribute Filter Range`** — filter on attributes rather than density.
- **`Static Mesh Spawner`** — spawns one mesh per point.
- **`Get Landscape Data`** — returns typed landscape data, if you later want slope/height to combine
  with the mask.

**What the mask must be able to distinguish** *(not which species — that is **PAK-18**)*:

- **Distance-from-water bands** — so a riparian zone can be separated from a dry zone. A single
  binary stream mask cannot express this; you need either graded values or several masks.
- **Wet vs dry ground** — wetland polygons as their own channel or value.
- **Exclusion zones** — roads, buildings, marae, urupā, and any area that must stay clear. Worth
  treating as a hard mask applied last, so nothing can spawn there by accident.

Species names such as harakeke or kahikatea appear in PAK-14 only as illustrations of the idea. This
brief deliberately makes no planting recommendations.

---

## 3. Coordinate alignment ⭐ the crux

**This is where the workflow will fail silently if it fails at all.** A mask that is correctly
authored but wrongly aligned produces foliage in the wrong place — possibly out at sea — with no
error message anywhere.

**The problem in one line:** your source data is in **EPSG:2193, a metre-based Transverse Mercator
grid** **[confirmed]**, and Cesium works in **ECEF / lat-lon-height on the WGS 84 ellipsoid**
**[confirmed]**. Those are not the same space, and nothing converts between them automatically.

**Proposed approach** — reproject in QGIS, never in Unreal:

1. Do all cropping, aligning, and rasterising in **EPSG:2193**, where units are metres and distances
   behave sensibly. **[confirmed — units are metres]**
2. As the **final** export step, reproject the finished mask to **EPSG:4326**. **[to-verify]** — this is
   the step most likely to introduce error, because reprojecting a raster resamples it.
3. In Unreal, note the `CesiumGeoreference` origin latitude/longitude/height for TKOPPARCHVIZ and
   treat it as the fixed reference point everything else is measured from. Adjusting the origin shifts
   the whole level; georeferenced objects stay put, non-georeferenced ones move.
   **[confirmed — [Placing Objects on the Globe](https://cesium.com/learn/unreal/unreal-placing-objects/)]**
4. Anchor the PCG volume to the globe with a **`CesiumGlobeAnchor`** component (actor Mobility must be
   **Movable**) so it holds its real-world position rather than drifting when the origin changes.
   **[confirmed — same source]**

**A known limitation worth reading before you rely on this:** Cesium's own documentation warns that a
precise globe position "will not automatically solve every problem" — Unreal gravity still pulls in
the wrong direction far from the origin, and multi-region work needs sublevels.
**[confirmed — [Placing Objects on the Globe](https://cesium.com/learn/unreal/unreal-placing-objects/)]**
For a single site like Pakipaki this should not bite, but it is the reason to keep the georeference
origin *at* the site.

**Open questions this approach depends on** — see Q1–Q4 below. None of them can be answered from
documentation.

---

## Housekeeping before the session

- **Back up the project** (zip) before the first import. Standing rule for this project.
- **Save all** in-editor, and make sure we are **not in PIE** when we start.
- Have the editor **open on TKOPPARCHVIZ**, and the VibeUE MCP server reachable — it was
  **unavailable** during the writing of this brief, which is why the node names here are cited from
  Epic's docs rather than read from your installed build.
- Keep the source `.tif` and the QGIS project file **somewhere outside `Content/`** — the repo will
  not track them.

---

## Decisions I need from you (the real prep)

### Q1. Where is the `CesiumGeoreference` origin for TKOPPARCHVIZ? ⭐ biggest one
Everything in section 3 is measured from it. I cannot read it while the MCP server is down, and it is
a project fact rather than a documented one.
**➜ Your prep:** open the level, select the `CesiumGeoreference` actor, and give me the origin
latitude / longitude / height. If it has been moved since setup, say so.

### Q2. What is the actual extent you want covered?
This sets the mask's resolution, its file size, and whether one mask is enough or you need tiles.
**➜ Your prep:** describe the area in plain terms — "the basin plus 500 m of hill either side", or a
bounding box, or two corner coordinates. Even a rough answer unblocks Q3.

### Q3. What metres-per-pixel do you want?
A trade-off only you can call: finer means individual plants land believably along a stream edge but
files grow quickly; coarser is cheaper but riparian planting will look approximate.
**➜ Your prep:** pick a target, or tell me the largest file size you are willing to keep and I will
work backwards from Q2's extent.

### Q4. Which exclusion zones are non-negotiable?
Named in section 2 as a hard mask. This is a cultural and practical question, not a technical one.
**➜ Your prep:** list what must never have foliage spawned on it — roads, buildings, marae, urupā,
access ways, sightlines you want kept open.

### Q5. Graded mask or several masks?
Distance-from-water bands can be one raster with graded values, or several binary rasters combined in
the graph. Graded is one file and one sampler; separate masks are easier to reason about and to fix
one at a time.
**➜ Your prep:** a preference, or "whichever is simpler to debug" and I will take that as separate
masks.

---

## Locked-in rules I'm already carrying (no action needed — just so you know I've got them)

- **Reproject in QGIS, never in Unreal.** Unreal has no concept of EPSG:2193.
- **Do the metric work in 2193, export in 4326.** Reproject once, at the end.
- **The georeference origin stays at the site.** Moving it to solve something else will silently move
  the planting.
- **Exclusion masks apply last**, as a hard cut, so nothing spawns on wāhi tapu by accident.
- **No species decisions here** — that is PAK-18, and it is yours and mana whenua's call, not mine.
- **Verify node names against the installed 5.8 build** before wiring anything. Everything in
  section 2 is cited from Epic's documentation, not read from your editor.

---

### Fastest path to "yes let's go"
Answer **Q1** (georeference origin) and **Q2** (extent) and the workflow becomes concrete enough to
prototype. Then the smallest useful test is a **single binary stream mask** over a small patch: export
it, import as `Texture2D`, wire `Get Texture Data → Texture Sampler → Density Filter → Static Mesh
Spawner` with any placeholder mesh, and check whether the spawn pattern lands on the Awanui where it
should. That one test proves or breaks the alignment chain before any real planting work goes in.
Ka pai.
