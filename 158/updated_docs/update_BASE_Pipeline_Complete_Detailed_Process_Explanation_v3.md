# Single-Pass Drone Video to Accurate 3D Model Generation System
## BASE PIPELINE — COMPLETE DETAILED PROCESS EXPLANATION (v3, corrected)
### From First Input to Final Self-Validation (No RTK/PPK)

This document explains every process in the v3 architecture in plain language and at high technical depth. It replaces the earlier process explanation. It is written so that someone new to photogrammetry can understand what happens, why, and what each step contributes.

**Scope:** Base pipeline only — a video plus its SRT/CSV telemetry (GPS, flight metadata, and altitude / focal length / gimbal angles when present). RTK/PPK belongs to a separate extension.

**What is different from the earlier version (short list):** georeferencing now happens *before* dense fusion; there is a new Stage 0 that adapts the pipeline to unknown hardware; camera intrinsics are self-calibrated; "up" is solved explicitly; depth is fitted per frame to metric sparse points; validation works without ground truth; the build plan is vertical-slice-first. Numbers marked *(start value)* are starting points to tune on real clips, not proven results.

---

# SECTION A — SYSTEM PURPOSE AND DESIGN PHILOSOPHY

## A.1 What the system is trying to achieve

The organisers give us a drone video and its telemetry file. The system turns them into:

- a 3D representation of terrain, roads, roofs, building shapes (as far as they are visible) and vegetation,
- a textured mesh and/or point cloud in real-world metric coordinates,
- a confidence map showing which parts are reliable,
- the required export formats, a browser viewer with a measuring tool, and an honest report.

## A.2 Why single-pass reconstruction is hard

In normal photogrammetry a drone flies many overlapping paths, so every surface is seen from many angles. In one pass:

- many building sides are never seen,
- baselines between views are short compared with the distance to the scene, so depth from triangulation is weak,
- occlusions are severe,
- the flight path is nearly a straight line, which leaves part of the georeferencing mathematically unconstrained,
- scale and absolute position come almost entirely from telemetry.

## A.3 The design answer

The architecture therefore: selects only useful frames; uses learned features for robust matching; recovers camera poses with structure-from-motion; places everything in a metric frame **before** densifying; densifies with a monocular depth model that is fitted to the metric sparse points; fuses conservatively; marks uncertain regions instead of inventing geometry; and tests itself.

## A.4 Performance philosophy

- **Absolute georeferencing** is limited by GPS quality and bias (typically a few metres for consumer drones without RTK/PPK). The system reports its own held-out error instead of assuming a value.
- **Relative measurement accuracy** (distances and heights inside the model) can be considerably better and is the honest route toward the problem's 1 m target.
- No accuracy figure is claimed unless a measurement supports it, and each figure states what it measures.

---

# SECTION B — INPUTS AND OPERATING MODES

## B.1 What is guaranteed and what is not

| Input | Status |
|---|---|
| Drone video (1080p/4K) | Guaranteed |
| GPS coordinates + flight metadata (SRT / CSV) | Guaranteed |
| Barometric / relative altitude | Use if present |
| Camera focal length, gimbal or attitude angles | Read from telemetry if present (DJI-style SRT often has them) |
| IMU data, camera intrinsics file, RTK/PPK | Optional; never required |
| Ground truth / surveyed points | Not guaranteed |

## B.2 Mode flag

The system detects what exists and sets one of: `video_only`, `gps`, `gps+alt`, `gps+alt+attitude`. The mode and the telemetry quality flag (good / degraded / none) are written to the report and steer later stages.

- **Video only:** a relative, unscaled model is produced and clearly labelled "not georeferenced".
- **GPS present:** metric scale and heading are recovered.
- **Attitude present:** the vertical direction ("up") comes from the gimbal/attitude data.

## B.3 Ways to load data

1. **Smart mode:** point the tool at a folder; it finds the video and telemetry and pairs them by content and timestamps, not filenames.
2. **Manual mode:** the user picks the video and the telemetry file.

## B.4 Graceful degradation

A missing optional input never crashes the run. The system continues with the best available reconstruction and states which mode was used.

---

# SECTION C — THE PIPELINE (OVERVIEW)

```
Stage 0   Runtime profile + time-budget controller
Stage 1   Ingestion + telemetry cleaning / synchronisation
Stage 2   Keyframe selection + extraction
Stage 3   Masking (sky + potentially dynamic objects)
Stage 4   Pose estimation (Structure-from-Motion, self-calibrated)
Stage 5   Georeferencing (metric ENU frame, "up" solve)      <-- before fusion
Stage 6   Depth estimation + per-frame metric fitting + confidence
Stage 7   Dense fusion (chunked TSDF, metric voxels)
Stage 8   Texturing + confidence map (+ optional inferred layer)
Stage 9   Cleanup
Stage 10  Export + viewer + report
Stage 11  Self-validation (no ground truth needed)
```

Every stage has one responsibility, writes its result to disk, and is skipped on re-run if its inputs and configuration are unchanged.

---

# SECTION D — DETAILED STAGE-BY-STAGE PROCESS

## STAGE 0 — RUNTIME PROFILE AND TIME-BUDGET CONTROLLER

### Purpose
The hardware is unknown until the event. This stage makes the pipeline choose its own settings so it can approach the 15-minutes-for-10-minutes target.

### Steps
1. Detect CPU cores, RAM, GPU model and VRAM.
2. Assign a tier *(start values)*:
   - **A** (GPU ≳ 8 GB): Depth Anything V2 Base (Large if time allows), ~400–600 keyframes, full feature resolution.
   - **B** (GPU ~4–8 GB): Depth Anything V2 Small, ~200–300 keyframes, reduced resolution.
   - **C** (CPU only): COLMAP with SIFT features, ~100–150 keyframes, coarse depth or sparse-plus-DSM output only.
3. Set a time budget of roughly 1.5 × the video duration, split about 5% ingest/keyframes, 35% SfM, 25% depth, 15% fusion, 20% texture/export *(start values)*.
4. **Benchmark-then-plan:** run the first ~20 keyframes through the expensive steps, extrapolate the total time, and lower keyframe count / resolution / model size if the projection is over budget.
5. Record hardware, tier, projected and actual timings in the report.

### Output
A run configuration (tier + budgets) used by every later stage.

---

## STAGE 1 — INGESTION AND TELEMETRY CLEANING

### Purpose
Turn raw files into one clean, time-synchronised data package and decide how far the telemetry can be trusted.

### Steps
1. Accept a video path or a folder; locate the video (.mp4, .mov, .avi, .mkv).
2. Read resolution, frame rate, frame count, duration, rotation (ffprobe + OpenCV).
3. Find telemetry candidates and parse in this order: **DJI-style SRT → other SRT / embedded metadata → CSV/log with automatic column detection → video-only.**
4. Extract per sample: time, latitude, longitude, altitude(s) and, when present, focal length, gimbal yaw/pitch/roll, heading. (In DJI-style SRTs the focal length is often a 35 mm-equivalent value, sometimes in tenths of a millimetre — verify on your clip.)
5. Clean:
   - range checks (latitude ±90, longitude ±180, plausible altitude),
   - reject physically impossible jumps using robust statistics (Median Absolute Deviation),
   - light smoothing (Savitzky–Golay or a simple Kalman filter),
   - synchronise to the video timeline, interpolating only across short gaps.
6. Set the **telemetry flag**: `good` (dense and consistent), `degraded` (gaps or noise), `none`.
7. Set the mode flag (Section B.2) and write a short ingestion summary.

### Failure handling
Unparseable file → next parser → video-only. Time-base mismatch → warning and best-effort alignment. The stage never crashes the run.

### Output
Unified data package + mode flag + telemetry flag + ingestion report.

---

## STAGE 2 — KEYFRAME SELECTION AND EXTRACTION

### Purpose
A 10-minute video at 30 fps has 18 000 frames. Keep a sparse, sharp, well-spaced subset.

### Steps
1. Compute movement since the last keyframe: from GPS distance when the telemetry flag is `good`, otherwise from optical flow and feature overlap.
2. Compute quality: sharpness (Laplacian variance), exposure (histogram), simple blur indicators; reject frames that are too blurred, dark or bright.
3. Select frames with enough movement and acceptable quality, aiming for **~80–90% overlap between consecutive keyframes** *(start value)*. Spacing follows speed and altitude, not a fixed interval.
4. **Gap escape:** if no candidate passes and the gap grows too large, accept the best remaining frame.
5. Respect the keyframe cap from Stage 0.
6. Extract the frames with a check that the decoded frame matches the expected index/time. **Originals are never modified**; any conditioning would be a derived copy (default off).
7. Detect likely **stabilised video** (in-camera stabilisation breaks the fixed-pinhole camera model) and record it.
8. Write a manifest: index, timestamp, sharpness, GPS pose, selection reason, file path.

### Output
Keyframe images + manifest.

---

## STAGE 3 — MASKING

### Purpose
Exclude pixels that would corrupt matching or depth.

### Steps
1. **Sky:** if the horizon/sky is visible (oblique footage), mask it with a segmentation model that has a sky class (SegFormer/ADE20K-style) or a horizon-and-colour heuristic. Monocular depth at the horizon is meaningless and would poison fusion. Nadir footage has no sky: skip.
2. **Potentially dynamic objects:** run YOLOv8n-seg / YOLOv11n-seg and mask person, car, truck, bus, bicycle, motorcycle, animal. "Potentially" — parked vehicles get masked too; a few small holes are accepted and mask coverage is logged.
3. Write binary masks (1 = usable, 0 = exclude) with a configurable dilation to cover boundary errors.
4. **No inpainting.** An honest unknown is better than invented geometry.

### Failure handling
If a model fails, continue with the heuristic or without masks and log it.

### Output
One binary mask per keyframe.

---

## STAGE 4 — POSE ESTIMATION (STRUCTURE-FROM-MOTION)

### Purpose
Recover each keyframe's camera position and orientation, the camera's intrinsics, and a sparse 3D point cloud.

### Steps
1. Extract **SuperPoint** features from each keyframe, ignoring masked pixels.
2. Match with **LightGlue** across a candidate graph: a temporal window (±k keyframes) plus a few spatially close pairs from GPS. Not all pairs.
3. Geometric verification (RANSAC with essential-matrix / 5-point) removes wrong matches.
4. Reconstruct with **COLMAP's incremental mapper (via hloc / pycolmap) — the tested default.** If your installed COLMAP supports position priors, use the GPS positions as priors; otherwise GPS enters in Stage 5.
5. **Self-calibrate the camera:** one shared camera model (SIMPLE_RADIAL or OPENCV). Initialise the focal length from telemetry (`f_px = f35 × image_diagonal_px / 43.27` — verify on your clip) or from a field-of-view guess; bundle adjustment refines focal length and distortion. No calibration file is required.
6. Run global bundle adjustment with a robust kernel (Huber/Cauchy).
7. **GLOMAP** is an optional accelerator, switched on only if it registers at least as many cameras as COLMAP with comparable reprojection error on your own clips (global methods can be fragile on near-straight strips).
8. **Quality gate** *(start values)*: registered cameras ≳ 80% of keyframes, mean reprojection error ≲ 1 px. On failure: retry with denser keyframes / relaxed matching; if still broken keep the largest sub-models and align each to GPS separately in Stage 5.
9. **Long videos:** process in overlapping windows, align each window to GPS, merge.

### Output
COLMAP model (cameras, images, sparse points) + registration and reprojection statistics.

---

## STAGE 5 — GEOREFERENCING (BEFORE DENSE FUSION)

### Purpose
Move the sparse model into a real-world metric frame so that everything after this — voxel sizes, depth fitting, measurements — is in metres.

### Steps
1. Convert telemetry to a local **East-North-Up** frame (x East, y North, z Up, metres); origin = first valid GPS sample (stored in the metadata).
2. Estimate a **robust similarity transform** (Umeyama/Horn Sim3 inside RANSAC or a robust loss) between reconstructed camera centres and the smoothed GPS track. This fixes scale, heading and translation.
3. **Solve "up".** A near-straight flight leaves the roll about the flight direction unconstrained; without a fix the model tilts sideways (1° of tilt at 100 m to the side ≈ 1.7 m of vertical error).
   - If gimbal/attitude is in the telemetry: use it as the vertical reference.
   - Otherwise: fit the dominant ground plane (RANSAC on sparse points) and rotate so its normal is +Z. This assumption is stated in the report and is wrong on steep terrain.
4. Use the altitude channel to constrain the vertical, noting the datum caveat (relative vs barometric vs ellipsoidal height).
5. **Residual gate:** compute per-camera residuals and RMSE; flag poor alignment as `low georeferencing confidence` (the model is still output, clearly labelled).
6. If the telemetry flag is `none`: output an unscaled relative model labelled "not georeferenced". Do not fake a scale.
7. *(Later)* light constrained bundle adjustment with soft GPS terms.

### Output
Cameras and sparse points in metric ENU, the up-vector, residual statistics and a georeferencing flag.

---

## STAGE 6 — DEPTH ESTIMATION, METRIC FITTING AND CONFIDENCE

### Purpose
Turn the sparse skeleton into dense depth per keyframe, in metres, with a per-pixel confidence.

### Why the fit matters
Depth Anything V2 predicts *relative*, disparity-like (inverse) depth with an unknown scale and shift for every frame. Back-projecting it directly with the SfM poses makes frames disagree and produces thick, ghosted surfaces.

### Steps
1. Run Depth Anything V2 on each keyframe (model size from the tier).
2. **Per-frame affine fit in inverse-depth space:** find `a, b` so that `a·prediction + b` matches the metric inverse depth of the sparse points visible in that frame (robust fit: RANSAC or Huber). Convert back: `depth = 1 / (a·prediction + b)`, clamped to a valid range.
3. **Fit-quality gate:** require enough points, decent spatial spread and a small residual *(start values)*. A frame that fails contributes **no depth** rather than wrong depth.
4. Apply sky and dynamic masks; remove flying pixels and depth edges.
5. **Multi-view consistency:** reproject each depth map into neighbouring keyframes and keep pixels that agree within a tolerance.
6. **Parallax angle per pixel** from the baseline to neighbouring views versus depth. Small angle = weak triangulation support.
7. **Per-pixel confidence** from fit quality, consistency and parallax. Pixels below a threshold are dropped before fusion (Open3D's TSDF has no per-pixel weights).
8. **Not in v1:** dense multi-view stereo. It can be added later (patch-match MVS where parallax is good); if added, the source of each surface is tagged.

### Output
Metric depth maps + confidence for each keyframe.

---

## STAGE 7 — DENSE FUSION

### Purpose
Merge the depth maps into one consistent surface in the metric frame.

### Steps
1. Downscale depth maps for integration (about 640–960 px wide) — full-resolution CPU integration is a common time sink.
2. Choose the voxel size from the **ground sample distance** *(start value)*: `GSD ≈ altitude / f_px` (metres per pixel), scaled by the depth downsample factor; `voxel ≈ 2 × effective GSD`. Cap the total voxel count to fit RAM.
3. Integrate into a **chunked / scalable TSDF** (Open3D `ScalableTSDFVolume`, or the GPU `VoxelBlockGrid` if available), tiled along the flight track to bound memory.
4. Extract a mesh and a dense point cloud; rasterise a **DSM** (2.5D height grid) from the same geometry.

### Output
Raw mesh, point cloud and DSM in ENU.

---

## STAGE 8 — TEXTURING AND CONFIDENCE MAP

### Purpose
Colour the surface with real imagery and attach a *testable* support score.

### Texturing (v1)
1. For each vertex, find keyframes that see it (depth test against rendered depth).
2. Rank views by viewing angle, resolution and occlusion; blend the good ones with weights to soften seams.
3. Store vertex colours. (A full UV atlas, e.g., OpenMVS TextureMesh, is an optional later upgrade.)

### Confidence score C ∈ [0,1]
Weighted combination of: view count, local reprojection error, viewing-angle quality, **depth-fit residual, multi-view consistency residual, parallax angle**. The last three stop a smooth-but-wrong surface from scoring as trustworthy. Weights start as heuristics *(start value)* and are checked in Stage 11. Display: green / yellow / red.

### Optional inferred layer (build after the first end-to-end run)
Generate ground-plane fill and planar building shells as a **separate mesh** with `class = inferred`, visually distinct, toggleable, and excluded from any accuracy statement.

### Output
Coloured geometry with C attached (+ inferred layer as a separate object).

---

## STAGE 9 — CLEANUP

### Steps
1. Remove clearly low-confidence geometry (conservative threshold).
2. Statistical and radius outlier removal; delete tiny floating components.
3. Light topology repair only where safe; recompute normals.
4. Feature-preserving decimation (quadric error metrics) to a viewer-friendly size.
5. Hole filling only for tiny holes; large gaps stay empty.
6. Log every threshold in the report.

**Core rule:** visible incompleteness is preferable to invented geometry that looks complete but is metrically wrong. (The labelled inferred layer is the only sanctioned exception.)

---

## STAGE 10 — EXPORT, VIEWER AND REPORT

| Format | Details |
|---|---|
| **GLB / glTF** | Local ENU coordinates relative to a stored origin, so float32 keeps precision. |
| **PLY** | Vertex colour + confidence attribute. |
| **OBJ** | Via trimesh / Open3D (material + texture or vertex colours). |
| **LAS** | `laspy`; projected CRS (UTM zone of the scene centre) with correct scale/offset; CRS and vertical-datum note included. |
| **GeoTIFF DSM** | `rasterio`; CRS, resolution and nodata set. |
| **FBX** | Converted from GLB with Blender headless or `assimp` (no good pure-Python writer). Do this last. |

- **Viewer:** static Three.js page with orbit/pan/zoom, a **two-point distance-measuring tool** (metres), confidence toggle, layer toggle (measured / inferred) and a stats panel.
- **Report:** mode and telemetry flags, hardware and tier, keyframe count, per-stage timing vs budget, SfM statistics, georeferencing residuals, all thresholds, and the Stage 11 results. Each claim is marked measured or assumed.

---

## STAGE 11 — SELF-VALIDATION (NO GROUND TRUTH NEEDED)

### Purpose
The dataset may contain no surveyed points, so the system tests itself and says exactly what each number means.

### Checks
1. **Held-out GPS (absolute georeferencing):** split camera positions into spatially *blocked* folds (random splits are misleading because neighbours are correlated), fit the georeference on the rest, report horizontal and vertical RMSE on the held-out block. This measures georeferencing consistency, **not** surface accuracy.
2. **SfM quality:** mean/median reprojection error, registration rate, track lengths.
3. **Scale sanity:** model path length versus GPS path length; fitted scale versus what altitude and focal length predict.
4. **Geometric plausibility:** flatness of detected road/roof planes (a flat road rendering as a bowl signals depth-fit problems); verticality of detected walls.
5. **Confidence test:** correlate C with a leave-one-view-out depth discrepancy (predict a view's depth from the others). A useful C ranks high-error regions low.
6. **Timing:** measured per stage and in total, with the detected hardware.
7. **If ground truth is supplied** (unlikely): add horizontal / vertical / 3D error (E_h = √((x_r−x_g)²+(y_r−y_g)²), E_v = |z_r−z_g|, E_3D = √(E_h²+E_v²)), RMSE, MAE, maximum error and coverage.

**Rule:** no claim of "≤ 1 m" unless a measurement supports it, with its scope stated.

---

# SECTION E — HOW THE STAGES WORK TOGETHER

No single neural network produces the result.

- Stages 0–3 prepare a well-spaced, masked, hardware-appropriate set of observations.
- Stage 4 builds the geometric backbone.
- Stage 5 gives that backbone real-world scale, position and orientation.
- Stage 6 densifies it with depth that is fitted, gated and scored.
- Stage 7 fuses it in metres.
- Stage 8 adds appearance and an honest, testable uncertainty map.
- Stages 9–10 clean and deliver.
- Stage 11 tells the truth about accuracy.

---

# SECTION F — REALISTIC EXPECTATIONS

## Accuracy (to be measured, not assumed)

| Quantity | Expectation |
|---|---|
| Absolute position | Limited by GPS quality and bias; a few metres is typical without RTK/PPK. Reported from the held-out check. |
| Relative distances/heights | Can be considerably better than absolute position; measured on test clips with tape-measured distances. |
| Facades and vegetation | Weakest areas — single-pass views and monocular depth; expect low confidence and gaps. |
| Roofs, roads, terrain | Strongest areas. |

## Processing time
Target ≤ ~1.5 × video duration on the detected hardware. Actual time is measured and reported; Stage 0 adapts settings to try to hit it.

---

# SECTION G — BUILD PLAN (VERTICAL SLICE FIRST)

Do **not** implement Stage 1 → 12 one at a time and polish each. Build one thin end-to-end path on a real clip, then deepen.

| Milestone | Contents | Done when |
|---|---|---|
| **M0** (first hours) | SRT parse + simple keyframes + hloc/COLMAP on ONE real clip | Poses register; reprojection ≲ 1 px |
| **M1** | Georeferencing + up-solve, depth fit, TSDF, GLB, bare Three.js page | A metric mesh appears in the browser |
| **M2** | Viewer with measuring tool, PLY, report, tier controller, held-out check | Full demo flow works |
| **M3** | Masks, extended confidence, better texturing, cleanup, caching | Robust on ≥ 3 different clips |
| **M4** | LAS, GeoTIFF, OBJ, FBX, inferred layer, GLOMAP trial, windowed long videos | All PS formats exported |

**Team split (3 people):** A = ingestion, telemetry, keyframes, exports, viewer; B = SfM + georeferencing; C = depth, fusion, texturing, confidence. Agree the coordinate conventions and stage contracts on day one.

**Conventions (fix once, test once):** ENU right-handed z-up in metres; cameras use OpenCV axes with world-to-camera extrinsics (COLMAP and Open3D's TSDF both use this — verify with a synthetic round-trip test); depth in metres along the camera z-axis.

---

# SECTION H — TEST PROTOCOL BEFORE THE EVENT

- At least three real single-pass clips with SRT (different altitude and terrain; one hard case: vegetation-heavy, low texture, or oblique with sky).
- One public UAV dataset with LiDAR or surveyed ground truth for a real accuracy number (verify availability first).
- Your own tape-measured distances/heights on at least one clip.
- Regression tracking per commit: reprojection error, held-out GPS RMSE, road-flatness, runtime.
- Dry runs on the weakest hardware available, offline, with all model weights pre-downloaded.

---

# SECTION I — KNOWN RISKS AND RESPONSES

| Risk | Tripwire | Response |
|---|---|---|
| SfM fails on near-straight footage | Registration < ~80% or reprojection > ~1 px | Denser keyframes, relaxed matching, sub-model merge via GPS |
| GLOMAP unstable | Fewer registered cameras than COLMAP | Keep COLMAP default |
| Depth fit fails on low-texture frames | Fit gate fails | No depth for those frames; flag coverage loss |
| Model tilted after georeferencing | Ground-plane normal off vertical; large vertical residual | Use gimbal data; refit; flag |
| Slow hardware | Projected time > budget after benchmark | Drop a tier |
| Stabilised video | High reprojection with shared camera | Let BA refine intrinsics; try per-segment cameras |
| Overclaiming accuracy | Any figure without a measurement | Use Stage 11 wording only |

---

# SECTION J — ONE-LINE MENTAL MODEL

**PROFILE → READ → CLEAN → SELECT → MASK → RECONSTRUCT CAMERAS → GEOREFERENCE → DEPTH (FITTED) → FUSE → TEXTURE + CONFIDENCE → CLEAN → EXPORT → SELF-VALIDATE**

---

**Document status:** v3 — corrected; to be revised after the first end-to-end run on real footage.
**Companion documents:** Master Idea & Architecture Document (v3, Word), Technical Stack / Architecture v3 (Markdown).

*End of detailed Base Pipeline process explanation (v3).*
