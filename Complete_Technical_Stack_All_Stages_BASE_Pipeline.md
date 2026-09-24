# Complete Technical Stack Document
# Single-Pass Drone Video to Accurate 3D Model Generation System
## BASE PIPELINE — All 12 Stages Locked

**Document type:** Master Technical Stack Reference  
**Scope:** Base pipeline only (Video + GPS + Flight Metadata + Barometric Altitude)  
**RTK/PPK:** Excluded (see separate RTK extension document)  
**Status:** All 12 stages locked with full technical choices

---

# How to Read This Document

For every stage you will find:

1. **Purpose** — what the stage does and why it exists  
2. **Where it sits in the pipeline** — input it receives and output it produces  
3. **Full Technical Stack table** — Recommended vs Options with technical notes  
4. **Locked Configuration table** — the final decisions only  
5. **Why each major choice is used** — justification  
6. **Mathematical components** (where relevant)  
7. **What the stage explicitly does not contain**

At the end of the document you will find the complete software stack summary and the end-to-end technical pipeline.

---

# Overall Pipeline (Reminder)

```
Stage 1   Input Ingestion
Stage 2   Telemetry Validation, Cleaning & Synchronization
Stage 3   Hybrid Frame Selection
Stage 4   Keyframe Preparation
Stage 5   Dynamic Object Handling / Masking
Stage 6   Pose Estimation (Structure-from-Motion)
Stage 7   Depth Estimation and Dense Fusion
Stage 8   Metric Alignment / Georeferencing
Stage 9   Texturing and Confidence Map Generation
Stage 10  Final Model Cleanup and Optimization
Stage 11  Export and Output Generation
Stage 12  Validation and Accuracy Evaluation
```

---

# STAGE 1 — Input Ingestion

## Purpose
Receive the raw drone video and any available telemetry, detect which sensors are present, associate video with telemetry by content/time rather than filename, and produce one unified data package plus a mode flag.

## Where it is used
- First stage of the entire system  
- Output is consumed by Stage 2 (telemetry validation) and later by every stage that needs video frames or telemetry

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Content/time-based association | Filename similarity; Manual pairing only | Robust when names differ |
| **Models** | None | — | Classical stage |
| **Methods / Techniques** | Multi-source telemetry discovery (SRT → Embedded → CSV) | Single-source only | Ordered fallback |
| | Automatic sensor-presence detection | Hard-coded assumptions | Sets mode flag |
| | Unified data-package construction | Separate streams | One structure for all later stages |
| **Mathematical Components** | Timestamp difference minimization | — | \( i^* = \arg\min_i |t_v - t_i| \) |
| | Linear interpolation (gated) | Nearest only | Used only inside max gap |
| **Error Correction / Robust Methods** | Ordered source fallback | Fail-fast | SRT → Embedded → CSV |
| | Mode-flag degradation | Force full telemetry | Continues even with video only |
| **Libraries / Tools** | OpenCV (primary video I/O) | FFmpeg/ffprobe, PyAV | Frame access + properties |
| | FFmpeg/ffprobe | — | Container/stream metadata |
| | Custom SRT parser | pysrt | DJI-style and similar |
| | pandas (preferred) / csv | — | Tabular logs |
| | Custom ingestion module | — | Orchestration |
| **Other Required Processors** | Video property extractor | — | Resolution, FPS, duration, frame count |
| | Telemetry record normalizer | — | Common schema |
| | Mode-flag generator | — | VideoOnly / GPS / GPS+Baro |
| | Ingestion report writer | — | Human-readable summary |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Video I/O | OpenCV (primary) + FFmpeg/ffprobe for metadata |
| SRT parsing | Custom lightweight pure-Python parser |
| CSV / LOG | pandas preferred; csv as lightweight fallback |
| Association | Content / timestamp-based matching |
| User fallback | Manual file selection always available |
| Sensor detection | Automatic rule-based mode flag |
| Interpolation | Only inside configurable maximum telemetry gap; otherwise mark unavailable |
| Error handling | Ordered fallback + graceful degradation |

## Why these choices are used
- OpenCV + FFmpeg/ffprobe are the industry standard for reliable video and metadata access.  
- Content/time matching is far more robust than filename matching (widely reported as fragile).  
- Graceful degradation ensures the system never completely fails when optional sensors are missing.  
- Gated interpolation prevents inventing positions across large telemetry gaps.

## What this stage does not contain
No neural networks, no geometry, no Bundle Adjustment, no depth estimation.

---

# STAGE 2 — Telemetry Validation, Cleaning & Synchronization

## Purpose
Validate telemetry integrity, remove or flag unreliable measurements, temporally align every video frame with the best telemetry sample, assign a quality level (A–D), and produce a clean synchronized stream.

## Where it is used
- Immediately after Stage 1  
- Cleaned telemetry is used by Stage 3 (frame selection), Stage 6 (soft constraints), Stage 8 (metric alignment), and Stage 12 (reporting)

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Sequential validation → cleaning → synchronization | Monolithic single-pass | Clear separation improves debugging |
| | Nearest-timestamp association with configurable sync tolerance | Pure nearest only; Spline | \( |t_v - t_i| \le \Delta t_{\rm sync} \) |
| **Models** | None | Optional learned smoother (later) | Classical stage |
| **Methods / Techniques** | Range & physical-plausibility checks | Statistical only | Lat/lon bounds, altitude envelope, max speed |
| | Jump / discontinuity detection | Fixed or adaptive threshold | Impossible movements |
| | Gap detection & multi-level quality scoring | Binary valid/invalid | Produces A–D |
| | Synchronized telemetry available to any later frame | Frame-only processing | Any keyframe can retrieve its sample |
| **Mathematical Components** | Timestamp difference minimization | — | \( i^* = \arg\min_i |t_v - t_i| \) |
| | Gated linear interpolation | Nearest only; Higher-order | Both sync tolerance and max gap required |
| | Median Absolute Deviation (MAD) | Z-score | Robust outlier score |
| **Error Correction / Robust Methods** | MAD inside broader physical + statistical pipeline | Mean ± kσ only | More resistant to GPS spikes |
| | Configurable maximum gap gate | Unlimited interpolation | Prevents invented positions |
| | Soft flagging; hard reject only clearly invalid | Always delete | Flagged samples remain inspectable |
| | Quality-level degradation | Force high quality | System continues with lower confidence |
| **Libraries / Tools** | NumPy | — | Core numerics |
| | pandas | Pure Python | Time-series handling |
| | SciPy (optional) | — | Advanced statistics |
| | Custom validation module | — | Full logic |
| **Other Required Processors** | Trajectory jump detector | — | Speed-plausibility |
| | Gap analyser | — | Gap length and location |
| | Quality level assigner | — | A / B / C / D |
| | Synchronization report writer | — | Human-readable summary |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Validation order | Range → Physical/jump → MAD → Gap → Synchronization → Interpolation → Quality scoring → A/B/C/D |
| Association | Nearest-timestamp with configurable sync tolerance \( |t_v - t_i| \le \Delta t_{\rm sync} \) |
| Interpolation | Linear only when both \( |t_v - t_k| \le \Delta t_{\rm sync} \) and \( t_{k+1}-t_k \le \Delta t_{\max} \); otherwise unavailable |
| Outlier handling | MAD inside broader physical + statistical pipeline |
| Outlier action | Flag first; hard reject only clearly invalid / physically impossible |
| Quality output | Multi-level A–D + textual report |
| Libraries | NumPy + pandas (SciPy optional) + custom module |
| Error handling | Flag + degrade quality rather than hard crash |

## Why these choices are used
- MAD is more robust to extreme GPS spikes than mean±σ (standard robust statistics).  
- Gated interpolation prevents the system from inventing positions across large gaps.  
- Soft flagging preserves data for inspection while still protecting later stages.  
- Multi-level quality (A–D) gives every downstream stage a clear signal of telemetry reliability.

## What this stage does not contain
No learned trajectory models, no geometric reconstruction, no Bundle Adjustment, no IMU fusion (can be added later).

---

# STAGE 3 — Hybrid Frame Selection

## Purpose
Select a sparse, high-quality set of keyframes so that later stages do not process every video frame. Combine spatial movement with image quality and fall back to visual methods when GPS is unreliable. Prevent excessively large gaps with an escape condition.

## Where it is used
- After Stage 2  
- Selected keyframes (with full metadata) are the only frames processed by Stages 4–11

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Hybrid scoring (spatial movement + image quality) | Fixed-interval; Pure visual | Avoids redundancy and blur |
| | GPS distance when Stage-2 quality is acceptable | Pure temporal | Uses cleaned telemetry |
| | Visual fallback: Optical flow (primary) + feature overlap (validation) + image quality | Always GPS; Always visual | Activated when GPS quality is poor |
| | Maximum-gap escape condition | No escape | Force-accept best usable candidate if gap becomes excessive |
| **Models** | None | Optional learned quality net (later) | Classical stage |
| **Methods / Techniques** | Laplacian variance for sharpness | Other blur metrics | Fast and effective |
| | Exposure / brightness check | Histogram thresholds | Rejects extreme exposure |
| | Min / max keyframe spacing/displacement constraints | Unlimited | Prevents under- and over-sampling |
| **Mathematical Components** | Laplacian variance \( S = \operatorname{Var}(\nabla^2 I) \) | — | Sharpness |
| | Euclidean distance \( d = \|\mathbf{p}_i - \mathbf{p}_j\|_2 \) | Haversine | Spatial movement |
| | Optical-flow magnitude + feature-overlap ratio | — | Visual fallback signals |
| **Error Correction / Robust Methods** | GPS-quality gate before using distance | Blind trust of GPS | Automatic visual fallback |
| | Hard rejection of low-quality frames | Soft weighting only | Protects later SfM |
| | Maximum-gap escape | — | Prevents large holes |
| **Libraries / Tools** | OpenCV | — | Laplacian, optical flow, statistics |
| | NumPy | — | Distance and scoring |
| | Custom selection module | — | Hybrid logic + fallback + escape |
| **Other Required Processors** | Sharpness calculator | — | Laplacian variance |
| | Exposure analyser | — | Mean / histogram |
| | Movement estimator | — | GPS or visual |
| | Keyframe record writer | — | Full metadata per keyframe |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Selection strategy | Hybrid (spatial movement + image quality) |
| Sharpness metric | Laplacian variance |
| Spatial cue | GPS distance (when Stage-2 quality is acceptable) |
| Visual fallback | Optical flow (primary) + feature overlap/correspondence (validation) + image quality |
| Maximum-gap escape | Yes — force-accept best usable candidate if spacing becomes excessive |
| Spacing control | Minimum and maximum keyframe spacing/displacement constraints |
| Keyframe record | frame_index, timestamp, sharpness_score, exposure_score, GPS_displacement, visual_motion_score, feature_overlap_score, selection_mode, selection_reason |
| Libraries | OpenCV + NumPy + custom module |

## Why these choices are used
- Pure fixed-interval selection fails when flight speed changes; hybrid selection is proven superior in UAV photogrammetry research.  
- Laplacian variance is a fast, reliable sharpness metric.  
- Visual fallback prevents a bad GPS track from destroying the keyframe set before SfM begins.  
- Maximum-gap escape is a safety net that keeps the sequence usable for reconstruction.  
- Storing the full keyframe record costs almost nothing and makes debugging and reporting far clearer.

## What this stage does not contain
No neural keyframe networks, no SfM, no depth estimation.

---

# STAGE 4 — Keyframe Preparation

## Purpose
Physically extract the selected frames, attach synchronized telemetry and Stage-3 metadata, optionally apply very light conditioning, and organize them for the next stages.

## Where it is used
- After Stage 3  
- Prepared keyframes are the direct input to Stage 5 (masking) and Stage 6 (features)

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Sequential extraction + metadata attachment + optional light conditioning | Heavy pre-processing | Deliberately lightweight |
| **Models** | None | Optional learned denoisers (later) | Classical stage |
| **Methods / Techniques** | Direct seek + decode by index | Full re-encode | Only selected frames |
| | Full metadata attachment | Separate lookup tables | Every keyframe carries its record |
| | Optional light denoising / mild normalization | Strong enhancement | Default = off |
| | Batch organization | Single-frame only | Efficient for later inference |
| **Mathematical Components** | None mandatory | Simple intensity stats if normalization used | — |
| **Error Correction / Robust Methods** | Verify extracted frame matches expected index/timestamp | Blind extraction | Detects seek errors |
| | Preserve original bit-depth / colour space when practical | Forced 8-bit | Avoids information loss |
| | Fail-soft on single-frame decode error | Abort stage | Skip + log warning |
| **Libraries / Tools** | OpenCV or FFmpeg | — | Frame extraction |
| | NumPy | — | Array handling |
| | Custom preparation module | — | Orchestration |
| **Other Required Processors** | Frame extractor | — | Seek + decode |
| | Metadata binder | — | Telemetry + Stage-3 record |
| | Optional conditioner | — | Light denoise / normalization (off by default) |
| | Batch organizer + manifest writer | — | Ready for Stage 5 |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Extraction | Direct seek + decode of selected indices (OpenCV or FFmpeg) |
| Metadata | Full Stage-3 record + synchronized telemetry attached |
| Conditioning | Optional and light only (default = off) |
| Colour / bit-depth | Preserve original when practical |
| Error handling | Fail-soft: skip corrupted frame, log, continue |
| Output | Prepared images + complete manifest |
| Libraries | OpenCV / FFmpeg + NumPy + custom module |

## Why these choices are used
- The stage must stay lightweight so that original radiometry is not heavily altered.  
- Attaching the full metadata record once avoids repeated lookups later.  
- Fail-soft behaviour keeps the pipeline running even if a single frame is corrupt.

## What this stage does not contain
No undistortion, no heavy denoising, no feature extraction, no masking.

---

# STAGE 5 — Dynamic Object Handling / Masking

## Purpose
Detect moving objects and produce binary masks so those regions are excluded from feature extraction and depth estimation, preventing transient objects from becoming permanent geometry.

## Where it is used
- After Stage 4  
- Masks are applied in Stage 6 (features) and Stage 7 (depth)

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Instance segmentation → binary mask → mask application | Semantic only; Optical-flow motion | Object-level masks |
| **Models** | YOLOv8n-seg or YOLOv11n-seg (nano) | Larger YOLO-seg; other instance models | Nano preferred for speed |
| **Methods / Techniques** | Class filtering (dynamic classes only) | Mask everything | person, car, truck, bus, bicycle, motorcycle, animal |
| | Binary masks (1 = static, 0 = dynamic) | Soft / probabilistic | Simpler and safer for geometry |
| | Small optional dilation | No dilation; Heavy dilation | Covers boundary errors |
| | Exclude masked pixels (no inpainting) | Generative inpainting | Prefer unknown over invented geometry |
| **Mathematical Components** | None mandatory | Simple morphological dilation | — |
| **Error Correction / Robust Methods** | Confidence threshold on detections | Accept all | Ignore low-confidence |
| | Prefer “unknown” over hallucinated geometry | Inpainting | Core philosophy |
| | Fail-soft on model failure | Abort pipeline | Continue without masks + log |
| **Libraries / Tools** | Ultralytics (YOLOv8 / YOLOv11) | Detectron2, MMDetection | Instance segmentation |
| | OpenCV | — | Morphology if needed |
| | NumPy | — | Mask arrays |
| | Custom masking module | — | Orchestration |
| **Other Required Processors** | Segmentation inference | — | YOLO-seg per keyframe |
| | Class filter | — | Dynamic classes only |
| | Binary mask writer | — | One mask per keyframe |
| | Optional mask quality reporter | — | Statistics |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Model | YOLOv8n-seg or YOLOv11n-seg (nano) |
| Output | Binary masks (1 = static, 0 = dynamic) |
| Dynamic classes | person, car, truck, bus, bicycle, motorcycle, animal (configurable) |
| Mask usage | Exclude from feature extraction and depth estimation |
| Inpainting | Not used |
| Dilation | Small / optional (default light or off) |
| Error handling | Fail-soft |
| Libraries | Ultralytics + OpenCV + NumPy + custom module |

## Why these choices are used
- YOLO-seg family is proven in multiple recent SfM / SLAM / UAV papers for dynamic-object removal.  
- Binary exclusion is safer for metric reconstruction than generative inpainting.  
- Nano models give acceptable accuracy at the speed required for a <15 min target.  
- The philosophy “prefer unknown over invented geometry” matches conservative, accuracy-oriented SfM practice.

## What this stage does not contain
No generative inpainting, no pose estimation, no depth estimation.

---

# STAGE 6 — Pose Estimation (Structure-from-Motion)

## Purpose
Recover the 3D position and orientation of every keyframe camera and a sparse set of consistent 3D points. This is the geometric backbone of the system.

## Where it is used
- After Stage 5  
- Poses and sparse points are the foundation for Stage 7 (depth fusion), Stage 8 (alignment), Stage 9 (texturing) and Stage 12 (validation)

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Global SfM pipeline | Incremental SfM | Faster and suited to ordered drone sequences |
| | Feature → Match → Verify → Global pose → Triangulate → BA | Classic incremental only | Clear responsibilities |
| **Models** | SuperPoint | SIFT, DISK, ALIKED, ORB | Strong learned detector |
| | LightGlue | SuperGlue, NN + ratio test | Fast high-quality matcher |
| **Methods / Techniques** | GLOMAP (primary) | COLMAP (incremental) fallback | Speed + competitive accuracy |
| | RANSAC + Essential Matrix / 5-point | Ratio test only | Geometric verification |
| | Sparse triangulation (DLT / mid-point) | — | Initial 3D points |
| | Global Bundle Adjustment (Levenberg-Marquardt) | Local BA only | Joint refinement |
| | Soft GPS / altitude constraints (optional) | Hard constraints only | When telemetry quality is good |
| **Mathematical Components** | Essential Matrix & epipolar geometry | Fundamental Matrix | Relative pose |
| | 5-point algorithm | 8-point | Relative rotation & translation |
| | SVD | — | Many closed-form solvers |
| | Rotation averaging | — | Consistent rotations |
| | Reprojection error | — | BA cost function |
| | Levenberg-Marquardt | Gauss-Newton | Non-linear least squares |
| **Error Correction / Robust Methods** | RANSAC | Simple ratio test | Outlier rejection |
| | Huber or Cauchy robust kernel in BA | Pure L2 | Down-weights outliers |
| | Pose quality check (median/mean reprojection error) | No gate | Usability decision |
| | Stage-5 masks applied | Ignore masks | Dynamic pixels excluded |
| **Libraries / Tools** | GLOMAP | COLMAP | Global SfM |
| | SuperPoint + LightGlue | Kornia, official repos | Features & matching |
| | Ceres (or GLOMAP internal) | g2o, GTSAM | BA backend |
| | OpenCV / PoseLib | — | Essential matrix, RANSAC |
| | Custom orchestration | — | Full pipeline control |
| **Other Required Processors** | Feature extractor | — | SuperPoint (masked) |
| | Matcher | — | LightGlue |
| | Geometric verifier | — | RANSAC + epipolar |
| | SfM engine | — | GLOMAP |
| | BA runner | — | Global BA + optional GPS soft constraints |
| | Quality reporter | — | Reprojection error, registered cameras, point count |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Feature detector | SuperPoint |
| Feature matcher | LightGlue |
| SfM engine | GLOMAP (primary) |
| Fallback | COLMAP (incremental) — later option |
| Geometric verification | RANSAC + Essential Matrix / 5-point |
| Optimization | Global Bundle Adjustment (Levenberg-Marquardt) |
| Robust kernel | Huber or Cauchy |
| GPS / altitude | Soft constraints when telemetry quality is good |
| Masks | Stage-5 binary masks applied |
| Quality gate | Median / mean reprojection error check |
| Libraries | GLOMAP + SuperPoint/LightGlue + Ceres/OpenCV + custom module |

## Why these choices are used
- SuperPoint + LightGlue currently offer the best speed/quality balance among learned features and matchers.  
- GLOMAP provides global SfM that is significantly faster than classic incremental COLMAP while remaining competitive in accuracy — ideal for the <15 min target.  
- Soft GPS constraints gently pull the reconstruction toward metric consistency without fighting the visual geometry.  
- Robust kernels and RANSAC are standard, proven defences against outliers.

## What this stage does not contain
No dense depth, no final georeferencing, no texturing, no NeRF/Gaussian Splatting as primary path.

---

# STAGE 7 — Depth Estimation and Dense Fusion

## Purpose
Turn the sparse geometric skeleton into dense geometry that can represent terrain, buildings, roads and vegetation.

## Where it is used
- After Stage 6  
- Dense geometry is the input to Stage 8 (alignment), Stage 9 (texturing) and Stage 10 (cleanup)

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Monocular depth → Filtering → Volumetric fusion | Multi-view stereo only; Gaussian Splatting | Depth densifies; TSDF fuses |
| **Models** | Depth Anything V2 | Depth Anything V1, MiDaS, ZoeDepth | Strong zero-shot monocular depth |
| **Methods / Techniques** | Per-keyframe dense depth prediction | Sparse depth only | Uses Stage-6 poses |
| | Depth filtering (confidence + flying pixel + multi-view consistency) | No filtering | Removes bad depth |
| | Open3D TSDF fusion | Poisson, other volumetric | Integrates filtered depth |
| | Mesh / point-cloud extraction | — | From finished TSDF |
| | Stage-5 masks applied | Ignore masks | Dynamic objects excluded |
| **Mathematical Components** | Back-projection \( \mathbf{X} = \pi^{-1}(\mathbf{x}, d) \) | — | Depth to 3D |
| | Truncated Signed Distance Function (TSDF) | — | Volumetric surface |
| | Multi-view consistency | — | Depth agreement across views |
| **Error Correction / Robust Methods** | Confidence thresholding | No filter | Discard low-confidence depth |
| | Flying-pixel removal | — | Isolated samples |
| | Multi-view consistency check | Single-view only | Improves reliability |
| | Optional morphological cleanup | — | Light |
| **Libraries / Tools** | Depth Anything V2 (official) | Other depth models | Inference |
| | Open3D | — | TSDF + extraction |
| | NumPy / OpenCV | — | Filtering helpers |
| | Custom fusion module | — | Orchestration |
| **Other Required Processors** | Depth inference engine | — | Depth Anything V2 |
| | Depth filter | — | Confidence + flying pixel + consistency |
| | TSDF integrator | — | Multi-view fusion |
| | Geometry extractor | — | Dense cloud / mesh |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Depth model | Depth Anything V2 |
| Fusion | Open3D TSDF |
| Filtering | Confidence + flying-pixel + multi-view consistency |
| Masks | Stage-5 masks applied |
| Output | Dense point cloud and/or mesh |
| Optional later | 3D Gaussian Splatting (visual quality only) |
| Libraries | Depth Anything V2 + Open3D + NumPy/OpenCV + custom module |

## Role clarity (locked)
- GLOMAP = geometric backbone (poses)  
- Depth Anything V2 = densification  
- TSDF = multi-view fusion  
Depth model is not responsible for final metric accuracy.

## Why these choices are used
- Depth Anything V2 provides strong zero-shot dense depth that works well on outdoor drone imagery.  
- TSDF fusion is a proven, stable way to integrate many depth maps into one consistent surface.  
- Explicit filtering before fusion removes the most damaging depth errors.

## What this stage does not contain
No final georeferencing, no texturing, no aggressive hole filling, no NeRF as primary path.

---

# STAGE 8 — Metric Alignment / Georeferencing

## Purpose
Convert the relative reconstruction into a real-world metric coordinate system using GPS and altitude.

## Where it is used
- After Stage 7  
- Georeferenced model is the input to Stage 9–12 and to all final deliverables

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Hybrid Sim3 + optional constrained BA | Direct GPS assignment only | Closed-form first, then optional refinement |
| **Models** | None | — | Classical stage |
| **Methods / Techniques** | GPS + altitude → local Cartesian (ENU/UTM) | Keep geographic | Required for metric scale |
| | Camera-centre ↔ GPS correspondences | Sparse points only | Most reliable |
| | Umeyama / Horn Sim3 | Rigid only | Scale + rotation + translation |
| | Apply transform to all cameras and points | — | Real-world frame |
| | Optional light GPS + altitude constrained BA | No refinement | Soft residuals when quality is good |
| **Mathematical Components** | Umeyama / Horn Sim3 (SVD) | Kabsch | Closed-form similarity |
| | Soft GPS / altitude residuals | Hard constraints | Inside BA |
| | Levenberg-Marquardt | — | Optional constrained BA |
| **Error Correction / Robust Methods** | Residual analysis after Sim3 | Blind accept | Detects poor alignment |
| | Quality-dependent soft constraints | Always force | Only when Stage-2 quality is acceptable |
| **Libraries / Tools** | NumPy / SciPy | — | SVD |
| | Open3D or custom | — | Alignment helpers |
| | Ceres (optional) | — | Constrained BA |
| | Custom alignment module | — | Full process |
| **Other Required Processors** | Coordinate converter | — | Lat/Lon/Alt → ENU/UTM |
| | Correspondence builder | — | Cameras ↔ GPS |
| | Sim3 estimator | — | Umeyama |
| | Optional BA runner | — | Soft-constrained |
| | Alignment quality reporter | — | Residuals, scale |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Transform | Hybrid Sim3 (Umeyama) |
| Coordinate system | Local Cartesian (ENU or UTM) |
| Refinement | Optional light GPS + altitude constrained BA |
| Soft constraints | Only when telemetry quality is good |
| Libraries | NumPy + Open3D/custom + Ceres (optional) |

## Why these choices are used
- Umeyama Sim3 is the standard closed-form solution for similarity alignment and is numerically stable.  
- Soft constraints gently improve metric consistency without overriding good visual geometry.  
- Performing alignment after dense fusion still works because the camera centres remain the primary correspondences.

## What this stage does not contain
No texturing, no cleanup, no export.

---

# STAGE 9 — Texturing and Confidence Map Generation

## Purpose
Colour the 3D surface with real imagery and produce a confidence map that honestly shows which parts of the model are reliable.

## Where it is used
- After Stage 8  
- Textured + confidence-annotated geometry is the input to Stage 10 and Stage 11

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Best-view selection + weighted blending | Single best view; Average all | Reduces seams |
| **Models** | None | — | Classical stage |
| **Methods / Techniques** | Visibility determination | — | Which keyframes see each surface |
| | Best-view (angle + resolution + occlusion) | Random | Prefers frontal high-res views |
| | Weighted blending | No blending | Softens seams |
| | Confidence = view count + reprojection error + viewing angle | Other combinations | Multi-factor reliability |
| **Mathematical Components** | Viewing-angle quality | — | Near-frontal preferred |
| | Reprojection-error contribution | — | From Stage 6 |
| | View-count support | — | More observations → higher confidence |
| **Error Correction / Robust Methods** | Discard high-occlusion / extreme-angle views | Use all | Better texture |
| | Colour coding (Green / Yellow / Red) | Continuous scalar only | Clear visualisation |
| **Libraries / Tools** | Open3D / trimesh | — | Mesh texturing |
| | Custom texturing module | — | Best-view + blending |
| | Custom confidence module | — | Score combination |
| **Other Required Processors** | Visibility computer | — | Per-face / per-point |
| | Best-view selector | — | Optimal source image(s) |
| | Blender | — | Weighted colour |
| | Confidence calculator | — | Per-vertex / per-face score |
| | Texture & confidence writer | — | Attach to geometry |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Texturing | Best-view selection + weighted blending |
| Confidence factors | View count + Reprojection error + Viewing angle |
| Visualisation | Green (high) / Yellow (medium) / Red (low) |
| Libraries | Open3D / trimesh + custom modules |

## Why these choices are used
- Best-view + blending produces cleaner textures than naïve averaging while still reducing seams.  
- The three-factor confidence map is one of the strongest engineering features for single-pass reconstruction: it communicates uncertainty instead of hiding it.  
- Colour coding makes the confidence map immediately usable by non-experts.

## What this stage does not contain
No geometry modification, no hole filling, no export.

---

# STAGE 10 — Final Model Cleanup and Optimization

## Purpose
Remove noise and unreliable geometry while deliberately refusing to invent missing surfaces.

## Where it is used
- After Stage 9  
- Cleaned model is the final geometric product passed to Stage 11

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Conservative multi-step cleanup | Aggressive hole filling | Honesty over visual completeness |
| **Models** | None | — | Classical stage |
| **Methods / Techniques** | Moderate confidence filtering | Aggressive filtering | Only clearly low-confidence geometry |
| | Statistical Outlier Removal (conservative) | — | — |
| | Radius Outlier Removal | — | Isolated points |
| | Small connected-component removal | — | Floating fragments |
| | Light topology repair | Heavy repair | Safe fixes only |
| | Normal recomputation | — | After cleanup |
| | Feature-preserving decimation (Quadric) | Uniform decimation | Protects important edges |
| | Hole filling optional and strictly limited | Fill everything | Large holes left empty |
| **Mathematical Components** | Statistical outlier score | — | Mean distance + std |
| | Quadric Error Metrics | — | Feature-preserving simplification |
| **Error Correction / Robust Methods** | Conservative thresholds | Aggressive cleaning | Protects metric honesty |
| | Prefer visible holes over invented geometry | Fill everything | Core design rule |
| **Libraries / Tools** | Open3D | — | SOR, radius, components, decimation |
| | trimesh | — | Topology utilities |
| | Custom cleanup module | — | Full sequence |
| **Other Required Processors** | Confidence filter | — | Stage-9 threshold |
| | Outlier remover | — | SOR + radius |
| | Component filter | — | Tiny fragments |
| | Decimator | — | Feature-preserving |
| | Normal computer | — | Updated normals |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Strategy | Conservative confidence-aware cleanup |
| Outlier removal | Statistical + Radius (conservative) |
| Components | Remove small disconnected pieces |
| Decimation | Feature-preserving (Quadric) |
| Hole filling | Optional and strictly limited |
| Core rule | Visible incompleteness preferred over invented geometry |
| Libraries | Open3D + trimesh + custom module |

## Why these choices are used
- Aggressive cleanup or heavy hole filling can destroy metric honesty and hide the limitations of single-pass data.  
- Feature-preserving decimation keeps important edges while reducing size.  
- The explicit philosophy gives a strong, defensible answer when judges ask what happens to unseen surfaces.

## What this stage does not contain
No texturing, no georeferencing changes, no export.

---

# STAGE 11 — Export and Output Generation

## Purpose
Write all practical deliverables required by the problem statement and normal geospatial workflows.

## Where it is used
- After Stage 10  
- Final files delivered to the user / evaluator

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Multi-format export pipeline | Single format only | Mesh + point cloud + DSM + viewer |
| **Models** | None | — | Classical stage |
| **Methods / Techniques** | Textured mesh (OBJ) | — | With materials/textures |
| | Point cloud with colour + confidence (PLY, LAS) | — | — |
| | Modern real-time mesh (GLB/GLTF) | — | Web & real-time |
| | DSM rasterisation (GeoTIFF) | — | Height map |
| | Web viewer packaging | — | Simple browser viewer |
| | Metadata + processing report | — | Mode, quality, statistics, CRS |
| **Mathematical Components** | None mandatory | Rasterisation | Height-map generation |
| **Error Correction / Robust Methods** | Validate georeferencing metadata | Ignore CRS | GIS compatibility |
| **Libraries / Tools** | trimesh | — | OBJ, PLY, GLB |
| | Open3D | — | Point cloud / mesh I/O |
| | laspy | — | LAS |
| | rasterio | — | GeoTIFF |
| | pygltflib (optional) | — | GLTF helpers |
| | Three.js / model-viewer | — | Web viewer |
| | Custom export module | — | Orchestration + report |
| **Other Required Processors** | Mesh exporter | — | OBJ + textures |
| | Point-cloud exporter | — | PLY + LAS |
| | DSM generator | — | Rasterise height |
| | Viewer packager | — | GLB + HTML |
| | Report writer | — | Processing + quality summary |

## Locked Configuration

| Format | Library | Priority |
|--------|---------|----------|
| OBJ + textures | trimesh / Open3D | Core |
| PLY (colour + confidence) | Open3D / trimesh | Core |
| LAS | laspy | Core |
| GLB / GLTF | trimesh / pygltflib | Core |
| GeoTIFF DSM | rasterio | Core |
| Web viewer | Three.js / model-viewer | Core |
| Processing report + CRS metadata | Custom | Core |

## Why these choices are used
- The set of formats covers the needs stated in the problem statement (mesh, point cloud, GIS, visualisation).  
- Open libraries keep the system fully reproducible and free of proprietary lock-in.  
- The processing report makes the run auditable and supports the SIH evaluation criteria.

## What this stage does not contain
No further geometric processing, no validation against ground truth (that is Stage 12).

---

# STAGE 12 — Validation and Accuracy Evaluation

## Purpose
Measure the real geometric accuracy of the finished model against independent ground-truth data and produce a quantitative report. This is what makes the system scientifically defensible.

## Where it is used
- Final stage  
- Accuracy report is part of the deliverable shown to evaluators

## Full Technical Stack

| Category | Recommended | Options | Technical Notes |
|----------|-------------|---------|-----------------|
| **Algorithms** | Ground-truth comparison pipeline | Visual inspection only | Required for SIH defensibility |
| **Models** | None | — | Classical stage |
| **Methods / Techniques** | Load surveyed / reference points | — | Must lie inside reconstructed area |
| | Establish correspondence with model | Nearest-neighbour or projection | Links GT to reconstruction |
| | Compute horizontal, vertical, 3D errors | — | Core metrics |
| | Aggregate RMSE, MAE, Max error | — | Summary |
| | Coverage / completeness | — | Fraction of area reconstructed |
| | Confidence vs actual error analysis | — | Validates the confidence map |
| **Mathematical Components** | \( E_h = \sqrt{(x_r-x_g)^2+(y_r-y_g)^2} \) | — | Horizontal |
| | \( E_v = |z_r - z_g| \) | — | Vertical |
| | \( E_{3D} = \sqrt{E_h^2 + E_v^2} \) | — | 3D |
| | RMSE / MAE | — | Standard |
| **Error Correction / Robust Methods** | Optional robust statistics on residuals | Use all points | — |
| **Libraries / Tools** | NumPy | — | Error calculations |
| | Custom validation module | — | Full evaluation |
| **Other Required Processors** | GT loader | — | Reference points |
| | Correspondence finder | — | GT ↔ model |
| | Metric calculator | — | All statistics |
| | Report generator | — | Human-readable accuracy report |

## Locked Configuration

| Item | Locked Choice |
|------|---------------|
| Metrics | Horizontal, Vertical, 3D error + RMSE + MAE + Max + Coverage |
| Confidence check | Compare confidence values against measured errors |
| Output | Quantitative accuracy report |
| Libraries | NumPy + custom validation module |

## Why these choices are used
- Telemetry quality alone cannot prove 3D accuracy. Only comparison with independent ground truth can.  
- Reporting RMSE/MAE/coverage and linking confidence to actual error makes the system honest and evaluable against the SIH criteria.

## What this stage does not contain
No further model modification; it only measures and reports.

---

# END-TO-END TECHNICAL PIPELINE (Summary View)

```
Raw Video + Telemetry
        │
        ▼
[1] Input Ingestion (OpenCV, FFmpeg/ffprobe, SRT/CSV parsers)
        │
        ▼
[2] Telemetry Validation & Sync (NumPy, pandas, MAD, gated interpolation)
        │
        ▼
[3] Hybrid Frame Selection (OpenCV Laplacian, GPS distance, optical-flow fallback)
        │
        ▼
[4] Keyframe Preparation (extract + metadata attachment)
        │
        ▼
[5] Dynamic Object Masking (YOLOv8n/11n-seg → binary masks)
        │
        ▼
[6] Pose Estimation (SuperPoint → LightGlue → GLOMAP → Global BA)
        │
        ▼
[7] Depth + Fusion (Depth Anything V2 → filter → Open3D TSDF)
        │
        ▼
[8] Metric Alignment (Umeyama Sim3 + optional soft GPS BA)
        │
        ▼
[9] Texturing + Confidence (best-view blending + view-count/reproj/angle score)
        │
        ▼
[10] Cleanup (confidence filter → SOR/radius → components → feature-preserving decimation)
        │
        ▼
[11] Export (OBJ, PLY, LAS, GLB, GeoTIFF, web viewer, report)
        │
        ▼
[12] Validation (ground-truth RMSE / MAE / coverage / confidence analysis)
```

---

# COMPLETE SOFTWARE STACK (All Libraries & Tools)

| Category | Libraries / Tools |
|----------|-------------------|
| **Video & Image I/O** | OpenCV, FFmpeg/ffprobe |
| **Telemetry** | Custom SRT parser, pandas, Python csv |
| **Numerics** | NumPy, SciPy (optional) |
| **Frame selection** | OpenCV (Laplacian, optical flow) |
| **Instance segmentation** | Ultralytics (YOLOv8n-seg / YOLOv11n-seg) |
| **Features & Matching** | SuperPoint, LightGlue (official / Kornia) |
| **Structure-from-Motion** | GLOMAP (primary), COLMAP (fallback) |
| **Bundle Adjustment** | Ceres Solver (or GLOMAP internal) |
| **Depth** | Depth Anything V2 |
| **Dense fusion** | Open3D (TSDF) |
| **Geometry & Mesh** | Open3D, trimesh |
| **Point-cloud export** | laspy |
| **Raster / DSM** | rasterio |
| **glTF** | pygltflib (optional) |
| **Web viewer** | Three.js / model-viewer |
| **Custom modules** | Ingestion, validation, selection, masking, orchestration, alignment, texturing, confidence, cleanup, export, validation |

---

# FINAL LOCKED TECHNOLOGY SUMMARY (One-page view)

| Stage | Primary Technical Choices |
|-------|---------------------------|
| 1 | OpenCV + FFmpeg/ffprobe, custom SRT, pandas, content/time matching |
| 2 | MAD, gated interpolation, quality A–D, NumPy/pandas |
| 3 | Hybrid (GPS + Laplacian), optical-flow + feature-overlap fallback, gap escape |
| 4 | Direct extract + full metadata attachment, light conditioning optional |
| 5 | YOLOv8n/11n-seg, binary masks, no inpainting |
| 6 | SuperPoint + LightGlue + GLOMAP + Global BA (Huber/Cauchy) |
| 7 | Depth Anything V2 + Open3D TSDF + depth filtering |
| 8 | Umeyama Sim3 + optional soft GPS/altitude BA |
| 9 | Best-view texturing + confidence (view count + reproj + angle) |
| 10 | Conservative confidence-aware cleanup, feature-preserving decimation |
| 11 | OBJ, PLY, LAS, GLB, GeoTIFF, web viewer, report |
| 12 | Ground-truth RMSE / MAE / coverage / confidence analysis |

---

**Document status:** Complete Technical Stack — All 12 Stages Locked  
**Companion documents:** Base Architecture, RTK Extension, Detailed Process Explanation, Implementation Guide

*End of document.*
