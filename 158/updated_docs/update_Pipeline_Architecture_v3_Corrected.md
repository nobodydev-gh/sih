# Single-Pass Drone Video → 3D Model — Architecture v3 (Corrected)

**Input contract:** one video + its SRT/telemetry file. Nothing else is guaranteed (no intrinsics file, no IMU, no ground truth).
**Output:** a georeferenced, textured 3D model viewable and measurable in a browser, plus the required export formats and an honest report.
**Status:** v3 supersedes v2. Thresholds marked *(start value)* are starting points to tune on real clips, not proven numbers.

---

## 0. What changed vs v2, and why

1. **Georeferencing moves BEFORE dense fusion.** v2 derived the TSDF voxel size from altitude but fused in the arbitrary-scale SfM frame, so the voxel size was meaningless. Now: SfM → align to ENU (metric, z-up) → depth fit + fusion in metres.
2. **New Stage 0: runtime profile + time-budget controller.** Hardware is unknown; the pipeline adapts model sizes, keyframe count and resolution to hit the 15-minute target.
3. **"Prefer triangulated depth" is replaced by something buildable.** v1 = fitted monocular depth everywhere, SfM points used only as scale anchors, parallax angle computed per pixel and used as confidence. MVS is an optional later add-on, not a claimed capability.
4. **COLMAP-based SfM (via hloc + pycolmap) is the tested default; GLOMAP is an optional accelerator** that must pass a registration/reprojection gate on your own clips before it is switched on.
5. **Intrinsics are self-calibrated.** The PS lists them as optional. Initialise from the SRT focal length (or a field-of-view guess) and let bundle adjustment refine.
6. **"Up" is solved explicitly.** Gimbal/attitude from telemetry if present, otherwise dominant ground-plane fit. This fixes the roll ambiguity of a near-straight flight.
7. **Stage 12 works without ground truth**: spatially blocked held-out GPS check, reprojection stats, path-length/scale consistency, and a confidence-vs-consistency test.
8. **Telemetry hygiene simplified.** A–D grading, MAD ladders and gated-interpolation logic are replaced by: robust filter → smooth → 3-level flag (good / degraded / none).
9. **Sky segmentation is concrete and conditional** (no sky in nadir footage).
10. **Export details that bite:** local-origin coordinates for GLB (float32), UTM + scale/offset for LAS, FBX via conversion.
11. **Per-stage caching + resume, and one fixed set of coordinate conventions**, because pose-convention bugs and full re-runs are the biggest time sinks.
12. **Optional labelled "inferred geometry" layer** (ground plane / planar extrusions) to recover completeness points without breaking the honesty rule.

---

## 1. Design principles

- **Vertical slice first.** A rough end-to-end run on one real clip beats twelve polished stages. Deepen stages only after the chain runs.
- **Metric scale comes from telemetry, shape from vision.** Monocular depth is relative until fitted to SfM.
- **Confidence is an output, and it must be testable** (see §7).
- **Never invent silently.** Anything inferred goes in a separate, labelled layer.
- **Fail-soft.** A bad frame, missing field or failed sub-model degrades quality and is logged; it never kills the run.
- **Everything cached.** Each stage writes artefacts to disk; a re-run skips stages whose inputs and config are unchanged.

## 2. Conventions (fix once, test once)

- **World frame after Stage 5:** local ENU, right-handed, x = East, y = North, z = Up, metres. Origin = first valid GPS sample (stored in metadata).
- **Cameras:** OpenCV axes (x right, y down, z forward), extrinsics stored as world-to-camera. COLMAP and Open3D's TSDF both use this; verify with one synthetic round-trip test (project a known 3D point, back-project, compare).
- **Depth:** metres along the camera z-axis after fitting.
- **Timestamps:** video time in seconds from first frame; telemetry resampled to it.

---

## 3. Pipeline overview

```
Video + SRT
   │
[0] Runtime profile + time-budget controller (tier A/B/C)
   │
[1] Ingest + telemetry clean/sync (SRT → CSV → video-only)
   │
[2] Keyframe selection + extraction (originals kept)
   │
[3] Masks: sky (conditional) + potentially-dynamic objects
   │
[4] SfM: SuperPoint+LightGlue → COLMAP (default) / GLOMAP (gated)
   │        self-calibrated K, windowed for long videos, quality gate
   │
[5] Georeference NOW: Sim3 to ENU + "up" solve + residual gate
   │        → cameras + sparse points in metric ENU
   │
[6] Depth: Depth Anything V2 → per-frame inverse-depth fit to sparse points
   │        → consistency filter → parallax tag → confidence per pixel
   │
[7] Fusion: chunked TSDF in metric frame, voxel from GSD
   │        → mesh + point cloud + DSM
   │
[8] Vertex-colour texturing + confidence C (+ optional inferred layer)
   │
[9] Conservative cleanup
   │
[10] Export (GLB, PLY, OBJ, LAS, GeoTIFF, FBX) + viewer + report
   │
[11] Self-validation (no GT needed) → report section
```

---

## 4. Stage specifications

### Stage 0 — Runtime profile + time-budget controller (NEW)

**Job:** detect GPU/VRAM/CPU/RAM, choose a tier, set budgets, adapt during the run.

- Tiers *(start values, tune on your hardware)*:
  - **A — GPU ≥ ~8 GB:** DA-V2 Base (or Large if time allows), keyframe cap ~400–600, full-res SfM features, depth fit at moderate resolution.
  - **B — GPU ~4–8 GB:** DA-V2 Small, keyframe cap ~200–300, reduced feature/depth resolution.
  - **C — CPU only:** COLMAP with SIFT, keyframe cap ~100–150, coarse depth (DA-V2 Small at low res) or sparse + DSM only.
- Target: total ≤ 1.5× video duration (15 min for 10 min).
- Rough time split to check against *(start value)*: ingest+keyframes 5%, SfM 35%, depth 25%, fusion 15%, texture+export 20%.
- **Benchmark-then-plan:** time the first ~20 keyframes through the expensive steps, extrapolate, and reduce keyframe count/resolution if the projection exceeds the budget.
- Log: detected hardware, chosen tier, projected vs actual time per stage (goes in the report).

### Stage 1 — Ingest + telemetry clean/sync

**Job:** read video properties and telemetry; produce one unified, time-synced package plus a mode flag.

- Video: ffprobe + OpenCV (resolution, fps, frame count, duration, rotation).
- Telemetry parse order: **DJI-style SRT → other SRT/embedded → CSV with column auto-detection → video-only.** Extract lat, lon, altitude(s), and where present focal length, gimbal yaw/pitch/roll, heading. (In DJI-style SRTs the focal length is often a 35mm-equivalent value in tenths of mm — verify against your clip.)
- Clean: range check → speed/jump reject (robust, MAD-based) → light smoothing (Savitzky–Golay or a simple Kalman) → resample to frame times. Interpolate only across short gaps.
- **Flag (3 levels):** `good` (dense, consistent) / `degraded` (gaps, noisy) / `none`. Later stages use this flag; no A–D ladder.
- Mode flag: `video_only` | `gps` | `gps+alt` | `gps+alt+attitude`.
- **Fail-soft:** unparseable telemetry → next parser → video-only. Never crash.

### Stage 2 — Keyframe selection + extraction

**Job:** pick a sparse, sharp, well-spaced frame set.

- Score = movement × sharpness. Movement from GPS distance when flag is `good`; otherwise optical flow / feature-overlap fallback.
- Aim for **~80–90% overlap between consecutive keyframes** *(start value)*; spacing follows speed and altitude, not a fixed interval.
- Reject blurred/over/under-exposed frames (Laplacian variance, histogram). Gap-escape: if the gap grows too large, accept the best remaining candidate.
- Respect the tier's keyframe cap.
- Extract originals with a seek-and-verify check (frame index/timestamp match). **Originals are never modified**; any conditioning is a derived copy (default off).
- Detect likely **stabilised video** (e.g., near-constant residual warp, crop/zoom inconsistencies) and record it; SfM will refine intrinsics regardless.
- Output: keyframe images + manifest (index, time, sharpness, GPS pose, selection reason).

### Stage 3 — Masks

**Job:** exclude pixels that corrupt features and depth.

- **Sky:** conditional — run only if the horizon/sky is visible (oblique footage). Use a segmentation model that has a sky class (SegFormer/ADE20K-style) or a simple horizon+colour heuristic as a fallback. Nadir footage: skip.
- **Potentially dynamic objects:** YOLO-seg nano (person, car, truck, bus, bicycle, motorcycle, animal). "Potentially" — parked cars get masked too; accept the small holes, and log mask coverage.
- Output: one binary mask per keyframe (1 = usable). Configurable dilation. **No inpainting.**
- **Fail-soft:** model failure → continue with sky heuristic only / no masks, logged.

### Stage 4 — Structure-from-Motion

**Job:** camera poses, intrinsics and a sparse point cloud.

- Features/matching: **SuperPoint + LightGlue** via `hloc`. Match graph = temporal window (±k keyframes) plus a few spatially-near pairs from GPS; not all-pairs.
- Reconstruction: **`pycolmap` incremental mapper is the default.** Use COLMAP position priors if your installed version supports them; otherwise GPS enters in Stage 5.
- **Camera model:** one shared camera (SIMPLE_RADIAL or OPENCV). Initial focal from telemetry (converted to pixels: `f_px = f35 × image_diagonal_px / 43.27`; check on your clip), else from a field-of-view guess. **Bundle adjustment refines focal + distortion.** Do not require an intrinsics file.
- **GLOMAP:** optional accelerator. Enable only if on your test clips it registers ≥ the COLMAP registration rate with comparable reprojection error.
- Masks applied to feature extraction.
- **Quality gate** *(start values)*: registered cameras ≥ ~80% of keyframes, mean reprojection error ≲ 1 px, enough 3D points per image. On failure: retry with denser keyframes / relaxed matching; if still broken, keep the largest sub-models and align each to GPS independently (Stage 5).
- **Long videos:** process in overlapping windows/chunks, align each to GPS, merge. This is your scalability story.
- Output: COLMAP model (cameras, images, points3D) + stats.

### Stage 5 — Georeferencing (moved before fusion)

**Job:** put cameras and sparse points into metric ENU, z-up.

1. Convert telemetry to ENU (origin = first valid GPS sample; pyproj/UTM math).
2. **Robust Sim3** (Umeyama inside RANSAC/Huber) on camera centres ↔ smoothed GPS positions. This fixes scale, heading and translation.
3. **Solve "up" (roll about the track axis):** a near-straight flight leaves this unobservable.
   - If gimbal/attitude present: use it as the vertical reference.
   - Else: fit the dominant ground plane (RANSAC on sparse points) and rotate so its normal = +Z. **Flag the assumption** in the report (wrong on steep terrain).
4. Vertical: use altitude from telemetry as an additional constraint (baro/relative altitude if available). Note the vertical-datum caveat (relative vs barometric vs ellipsoid) in metadata.
5. **Residual gate:** compute per-camera residuals and RMSE. Excessive residual → flag as `low georef confidence`; still output, clearly labelled.
6. If telemetry flag is `none`: output an **unscaled relative model** with a prominent "not georeferenced" flag. Do not fake scale.
7. Optional light constrained BA with soft GPS terms *(later)*.
- Output: cameras + sparse points in ENU; per-axis residual stats; georef confidence flag.

### Stage 6 — Depth, metric fitting, confidence

**Job:** dense depth per keyframe in metres, with a per-pixel confidence.

- **Depth Anything V2** per keyframe (model size from tier). Its output is relative, disparity-like (inverse depth).
- **Per-frame affine fit in inverse-depth space** to that frame's visible sparse points (now metric, from Stage 5): `inv_depth_metric ≈ a · pred + b`, robust fit (RANSAC/Huber). Convert back: `depth = 1 / (a·pred + b)`, clamp to valid range.
- **Fit quality gate:** require a minimum point count and spatial spread in the image, and a residual below threshold *(start value)*. Frames that fail the gate get **no depth** (or low weight) rather than a wrong one.
- **Filters:** sky/dynamic masks, edge/flying-pixel removal, **multi-view consistency** (reproject depth into neighbouring keyframes, keep pixels that agree within a tolerance).
- **Per-pixel parallax angle** from geometry (baseline to neighbouring views vs depth). Low parallax = weak triangulation support.
- **Per-pixel confidence** = f(fit quality, consistency, parallax). Pixels below threshold are dropped before fusion (Open3D's TSDF has no per-pixel weights).
- **Not in v1:** claiming triangulated/MVS dense depth. Optional later: patch-match MVS where parallax is good; if added, tag the source per surface.
- Output: metric depth + confidence per keyframe.

### Stage 7 — Fusion

**Job:** merge depth maps into one surface, in metric ENU.

- **Chunked/scalable TSDF** (Open3D `ScalableTSDFVolume`, or the GPU `VoxelBlockGrid` if available), tiled along the flight track to bound memory.
- **Voxel size from GSD** *(start value)*: `GSD ≈ altitude / f_px` (m/pixel); at depth-map resolution scaled by the downsample factor, `voxel ≈ 2 × GSD_eff`. Cap the total voxel count to fit RAM.
- **Downscale depth** for integration (e.g., ~640–960 px wide) — full-res CPU TSDF integration is a common time sink.
- Extract mesh + dense point cloud; rasterise a **DSM** (2.5D height grid) from the same geometry (also feeds GeoTIFF).
- Output: raw mesh, point cloud, DSM.

### Stage 8 — Texturing + confidence C (+ optional inferred layer)

**Job:** colour the surface and attach a testable support score.

- **Texturing v1:** per-vertex colour by projecting into keyframes, visibility via depth test against rendered depth, best-view weighted blending (angle, resolution, occlusion). Full UV texturing (e.g., OpenMVS TextureMesh) is optional later.
- **Support score C ∈ [0,1]** per vertex/face — weighted combination of: view count, local reprojection error, viewing-angle quality, **depth-fit residual, multi-view consistency residual, parallax angle**. Weights start as heuristics *(start value)* and are checked in §7. Colour map: green / yellow / red.
- **Optional inferred layer** (build after v0): fill the ground plane and extrude planar building shells from visible roof/edge geometry into a **separate mesh** with `class = inferred`. Default off in any accuracy statement, toggle in the viewer, always visually distinct.

### Stage 9 — Cleanup

- Confidence filter (conservative), statistical + radius outlier removal, small-component removal, light topology repair, normals, feature-preserving decimation to a viewer-friendly size.
- Hole filling only for tiny holes. Large gaps stay empty (or are covered only by the labelled inferred layer).
- Log every threshold in the report.

### Stage 10 — Export + viewer + report

| Format | Notes |
|---|---|
| **GLB** | Local ENU coordinates **relative to a stored origin** (float32-safe). Colour + confidence variants. |
| **PLY** | Vertex colour + confidence attribute. |
| **OBJ** | Via trimesh/Open3D (+ material/texture or vertex colours). |
| **LAS** | `laspy`, projected CRS (UTM zone of the scene centre) with scale/offset set properly; CRS + vertical-datum note in header/metadata. |
| **GeoTIFF DSM** | `rasterio`, CRS + resolution + nodata set. |
| **FBX** | No good pure-Python writer: convert from GLB with Blender headless or `assimp`. Do last. |

- **Viewer:** static Three.js page served by a local HTTP server. Orbit/pan/zoom, **two-point distance measuring tool** (shows metres), confidence toggle, layer toggle (model / inferred), stats panel.
- **Report (auto-generated):** mode flag, telemetry flag, tier + hardware, keyframe count, per-stage timing vs budget, SfM stats, georef residuals, and the §7 self-validation results. Every claim states whether it is measured or assumed.

### Stage 11 — Self-validation (no ground truth needed)

See §7.

---

## 5. Runtime, caching and failure handling

- One `config.yaml` (tier defaults + overrides), one CLI: `run --video X --telemetry Y --out DIR`, plus a thin web UI wrapper (file pick, progress bar, timer, link to viewer).
- Each stage writes to `DIR/stageN/` with a hash of its inputs + config; unchanged stages are skipped on re-run.
- Every stage returns a status (`ok` / `degraded` / `failed`) and the orchestrator decides whether to continue, retry with a fallback, or stop with a partial output. Partial outputs (e.g., sparse cloud + DSM only) are still exported and labelled.

## 6. Data contracts

| From → To | Contract |
|---|---|
| 1 → 2, 4, 5 | Synced telemetry table (ENU-ready), mode flag, focal/attitude if present |
| 2 → 3, 4 | Keyframe images + manifest |
| 3 → 4, 6 | Binary masks per keyframe |
| 4 → 5 | COLMAP model (cameras, images, points3D) + stats |
| 5 → 6, 7 | Same model in ENU/metric, up-vector, georef flag + residuals |
| 6 → 7, 8 | Metric depth + per-pixel confidence per keyframe |
| 7 → 8, 9 | Mesh/point cloud/DSM in ENU |
| 8 → 9, 10 | Coloured geometry + C (+ inferred layer, separate) |
| all → report | Timings, statuses, thresholds, metrics |

---

## 7. Self-validation without ground truth

Report **absolute** and **relative** separately, and label what each actually measures.

1. **Held-out GPS check (absolute georeferencing):** split camera positions into spatially *blocked* folds (not random — neighbouring cameras are correlated), fit Sim3 + up on the rest, report horizontal/vertical RMSE on the held-out block. This measures georeferencing consistency, **not** surface accuracy.
2. **SfM quality:** mean/median reprojection error, registration rate, track lengths.
3. **Scale sanity:** model path length vs GPS path length; fitted scale factor vs expectation from altitude/focal.
4. **Geometric plausibility:** flatness residual of detected road/roof planes (a flat road that renders as a bowl signals depth-fit problems); verticality of detected walls.
5. **Confidence test:** correlate C with an empirical error proxy — leave-one-view-out depth discrepancy (predict a view's depth from the others and compare). A useful C should rank high-error regions low.
6. **Timing:** measured per-stage and total runtime on the detected hardware.
7. **If ground truth is provided** (unlikely): add the standard horizontal/vertical/3D RMSE, MAE, max error and coverage on top.
8. **Do not state "≤ 1 m" unless a measurement supports it.** State what was measured and its scope.

---

## 8. Build order (vertical slice first)

| Milestone | Contents | Done when |
|---|---|---|
| **M0 (first hours)** | Stage 1 (SRT parse) + Stage 2 (simple keyframes) + Stage 4 (hloc + COLMAP) on ONE real clip | Poses register, reprojection ≲ 1 px, sparse cloud looks right |
| **M1** | Stage 5 (Sim3 + up) + Stage 6 (depth fit) + Stage 7 (TSDF) → GLB | A metric mesh appears in a bare Three.js page |
| **M2** | Viewer with measure tool, PLY, report, Stage 0 tiers/controller, held-out check | Full demo flow: drop files → progress → 3D view → measure → report |
| **M3** | Masks, confidence C (extended), texturing quality, cleanup, caching/resume | Robust on 3+ different clips |
| **M4** | LAS, GeoTIFF, OBJ, FBX, inferred layer, GLOMAP trial, windowed long-video mode | Checklist of PS output formats complete |

**Team split (3 people):** A = ingest/telemetry/keyframes/exports/viewer; B = SfM + georeferencing; C = depth/fusion/texturing/confidence. Agree §2 conventions and §6 contracts on day one.

## 9. Test protocol (build before the event)

- 3+ real single-pass clips with SRT (different altitude, terrain, one nasty: vegetation-heavy / low texture / oblique with sky).
- One public UAV dataset with LiDAR or surveyed ground truth for a real accuracy number (verify availability first).
- Your own tape-measured distances/heights on at least one clip.
- Regression checks: reprojection error, held-out GPS RMSE, road-flatness, runtime — tracked per commit.
- Dry runs on the weakest hardware you can find, with the network off and all model weights pre-downloaded.

## 10. Out of scope for now

RTK/PPK mode; Gaussian Splatting/NeRF; generative inpainting; A–D telemetry grading; full MVS; UV texture atlases. Revisit only after M3.

## 11. Known risks and tripwires

| Risk | Tripwire | Response |
|---|---|---|
| SfM fails on near-collinear single-pass footage | Registration < ~80% or reproj > ~1 px | Denser keyframes, relaxed matching, sub-model merge via GPS |
| GLOMAP unstable on strips | Lower registration than COLMAP | Keep COLMAP default |
| Depth fit fails (low texture / few points) | Fit residual or point count gate | Drop those frames' depth; rely on neighbours; flag low coverage |
| Model tilted after georef | Ground-plane normal off vertical after solve; large vertical residual | Use gimbal data; else re-fit plane; flag |
| Slow on weak hardware | Projected time > budget after benchmark | Drop tier: fewer keyframes, smaller depth model, lower resolution |
| Stabilised video breaks pinhole | High reprojection with shared camera | Let BA refine intrinsics; try per-segment cameras |
| Float precision jitter in viewer | Model far from origin | Local-origin GLB with stored offset |
| Overclaiming accuracy | Any accuracy statement without a measurement | Use §7 wording only |

*End of v3.*
