# Single-Pass Drone Video to Accurate 3D Model Generation System
## BASE PIPELINE — COMPLETE DETAILED PROCESS EXPLANATION
### From First Input to Final Validation (No RTK/PPK)

This document explains every process in the locked Base Architecture in plain language and at high technical depth.  
It is written so that a person who is new to photogrammetry can understand what happens, why it happens, and what each small step contributes.

**Scope of this document:** Base pipeline only (Video + GPS + Flight Metadata + Barometric Altitude when available).  
RTK/PPK is intentionally excluded and belongs to the separate RTK extension document.

---

# SECTION A — SYSTEM PURPOSE AND DESIGN PHILOSOPHY

## A.1 What the system is trying to achieve

The system receives a video recorded by a drone that flies over a target area only once. From that single continuous video (and the standard telemetry that normally accompanies it) the system produces:

- A 3D representation of the terrain
- Building shapes (as far as they are visible)
- Roads and infrastructure
- Vegetation and obstacles
- A textured 3D mesh and/or point cloud
- A confidence map that shows which parts of the model are reliable
- Georeferenced outputs that can be used for measurement and analysis

The model is intended to be useful for visualization, approximate measurement, and rapid situational awareness.

## A.2 Why single-pass reconstruction is difficult

In classical photogrammetry a drone flies many overlapping paths so that every surface is seen from many different angles. In a single-pass flight:

- Many building sides are never seen
- There is less geometric constraint
- Occlusions are more severe
- Scale and absolute position must come almost entirely from telemetry

Because of these limitations the architecture deliberately:

- Selects only the most useful frames
- Uses modern learned features and matching
- Uses a fast global Structure-from-Motion engine
- Densifies geometry with a monocular depth model
- Fuses depth conservatively
- Aligns the model to the real world using GPS and altitude
- Marks uncertain regions instead of inventing geometry
- Measures real accuracy against ground truth at the end

## A.3 Official performance philosophy

The system targets sub-2-meter horizontal accuracy under good standard-GPS conditions.  
Actual accuracy is determined by ground-truth validation, not assumed from the quality of the telemetry alone.

---

# SECTION B — INPUTS AND OPERATING MODES

## B.1 Target Georeferenced Mode (recommended)

| Input | Role |
|-------|------|
| Drone video (1080p or 4K) | Visual observations of the scene |
| GPS coordinates | Absolute horizontal position |
| Timestamps / flight metadata | Time alignment between video and telemetry |
| Barometric altitude (preferred) | Vertical constraint |

## B.2 Minimum Mode

| Input | Result |
|-------|--------|
| Video only | Relative (non-metric) 3D reconstruction is still possible |

## B.3 Optional inputs that improve quality when present

| Input | Benefit |
|-------|---------|
| IMU orientation | More stable camera orientation estimates |
| Camera intrinsic parameters | More accurate projection model |
| Higher-quality altitude source | Better vertical georeferencing |

## B.4 Two supported ways to load data

1. **Smart Automatic Mode**  
   The system is given a folder. It searches for a video file and any telemetry file, then attempts to match them by time content rather than by file name alone.

2. **Manual Selection Mode**  
   The user explicitly chooses the video file and the telemetry file (SRT or CSV).

## B.5 Graceful degradation principle

The system never completely fails just because one optional sensor is missing.  
It detects what is available, sets a mode flag, and continues with the best possible reconstruction. Later stages and the final report clearly state which mode was used.

---

# SECTION C — THE TWELVE-STAGE PIPELINE (OVERVIEW)

```
Stage 1   Input Ingestion
Stage 2   Telemetry Validation, Synchronization and Quality
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

Each stage has a single clear responsibility. This separation makes the system easier to implement, debug and explain.

---

# SECTION D — DETAILED STAGE-BY-STAGE PROCESS

---

## STAGE 1 — INPUT INGESTION

### Purpose
Turn raw files into one clean, unified data package that the rest of the system can use.

### Why this stage exists
Drones and flight apps produce many different file formats and naming conventions. If the system depends only on file names, it will frequently fail. This stage therefore focuses on content and structure rather than names.

### Detailed intermediate and small steps

1. Accept either a single video path or a folder path.
2. Locate the primary video file (common extensions: .mp4, .mov, .avi, .mkv).
3. Open the video and read its basic properties:
   - Resolution
   - Frame rate
   - Total number of frames
   - Duration
   - Whether timestamps are available inside the container
4. Search for telemetry candidates in the same folder or in user-provided paths.
5. Attempt to parse telemetry in the following priority order:
   - SRT (subtitle-style telemetry commonly written by DJI and similar systems)
   - Embedded metadata inside the video
   - CSV or plain text flight log
6. From each successful telemetry source extract, for every available sample:
   - Timestamp
   - Latitude
   - Longitude
   - Altitude (barometric if present, otherwise any available height)
   - Optional extra fields (speed, heading, etc.)
7. Attempt to match the video timeline with the telemetry timeline using time information rather than file names.
8. Build a single internal data structure that holds:
   - Video access handle
   - Ordered list of telemetry samples
   - Mapping information between video time and telemetry time
9. Detect which sensors are actually present and set a mode flag:
   - Video only
   - GPS available
   - GPS + Barometer available
10. Write a short ingestion summary (what was found, what was missing, which mode was selected).

### What can go wrong and how the stage handles it
- Missing telemetry → system continues in video-only mode.
- Corrupted SRT → tries the next source.
- Time base mismatch → records a warning and attempts the best possible alignment.

### Output of Stage 1
A unified data package + mode flag + ingestion report.

---

## STAGE 2 — TELEMETRY VALIDATION, SYNCHRONIZATION AND QUALITY

### Purpose
Decide whether the telemetry is trustworthy enough to influence later metric alignment, clean obvious errors, and align every video frame with the best telemetry sample.

### Why this stage exists
Raw GPS often contains jumps, gaps and noise. Using dirty telemetry directly can pull the entire 3D model into the wrong place or give it the wrong scale. Validation is performed early so that expensive later stages are not wasted on bad data.

### Detailed intermediate and small steps

1. Perform basic range checks:
   - Latitude must lie between –90 and +90
   - Longitude must lie between –180 and +180
   - Altitude must be within a plausible flight envelope
2. Detect sudden spatial jumps that are physically impossible for the drone’s speed.
3. Detect long gaps where telemetry samples are missing.
4. Flag or remove statistical outliers (points that deviate strongly from the local trajectory).
5. For every video frame (or every candidate keyframe) find the temporally closest telemetry sample.
6. When necessary, interpolate position and altitude between surrounding telemetry samples.
7. Compute simple quality indicators:
   - Number of valid samples
   - Size of largest gap
   - Magnitude of residual jumps after cleaning
8. Assign an overall telemetry quality level (A = excellent, B = good, C = usable, D = poor).
9. Generate a human-readable telemetry quality report.
10. Pass the cleaned and synchronized telemetry forward together with the quality level.

### Mathematical ideas used
- Time difference minimization for nearest-sample matching
- Linear interpolation between telemetry samples
- Statistical outlier detection (mean + standard deviation or median absolute deviation)
- Optional later improvement: light Kalman smoothing

### Output of Stage 2
Clean synchronized telemetry, quality level, and quality report.

---

## STAGE 3 — HYBRID FRAME SELECTION

### Purpose
Select a sparse but high-quality set of frames (keyframes) from the continuous video so that the rest of the pipeline does not have to process every frame.

### Why this stage exists
A ten-minute video at 30 frames per second contains 18 000 frames. Processing all of them is unnecessary and slow. The system therefore keeps only frames that both move the camera enough and are visually usable.

### Detailed intermediate and small steps

1. Read the timestamp of every frame (or of a dense candidate set).
2. When GPS quality is acceptable, compute the spatial distance travelled since the last selected keyframe.
3. For each candidate frame compute image quality scores:
   - Sharpness (commonly Laplacian variance)
   - Exposure / brightness
   - Simple motion-blur indicators
4. Reject frames that are too dark, too bright, or too blurred.
5. Combine spatial movement score with image quality score into a hybrid score.
6. Select a frame when it has both sufficient movement and acceptable quality.
7. If GPS quality is poor or missing, activate the visual fallback:
   - Estimate image motion (optical flow magnitude) or
   - Estimate visual overlap / feature persistence between frames
8. Produce the final ordered list of keyframe indices.
9. Record how many frames were kept and which selection mode (GPS-based or visual) was used.

### Design notes
- Purely fixed-interval selection (every Nth frame) is avoided because it ignores both motion and quality.
- The visual fallback prevents a bad GPS track from destroying the keyframe set before reconstruction even begins.

### Output of Stage 3
Ordered list of keyframe indices + selection statistics.

---

## STAGE 4 — KEYFRAME PREPARATION

### Purpose
Physically extract the chosen frames and prepare them for the computer-vision models that follow.

### Detailed intermediate and small steps

1. Seek to each selected frame index inside the video.
2. Decode and store the image (in memory or as temporary files).
3. Preserve the original timestamp of each frame.
4. Attach the already-synchronized telemetry values to each keyframe.
5. Optionally apply very light denoising or contrast normalization (kept optional so that original radiometry is not heavily altered).
6. Organize the frames into batches of suitable size for later neural-network inference.
7. Write a simple keyframe manifest (index, timestamp, telemetry, file path).

### Output of Stage 4
Prepared keyframe images + manifest linking each image to its telemetry.

---

## STAGE 5 — DYNAMIC OBJECT HANDLING / MASKING

### Purpose
Detect moving objects and prevent them from being treated as permanent scene geometry.

### Why this stage exists
In a single-pass flight a moving person or vehicle appears in different places in different frames. If those pixels are allowed to generate 3D points, the model contains ghost surfaces or floating fragments.

### Detailed intermediate and small steps

1. Load a lightweight instance-segmentation model (YOLOv8n-seg or YOLOv11n-seg).
2. Run the model on every keyframe.
3. Keep only the classes that are considered dynamic (person, car, truck, bus, bicycle, motorcycle, animal, etc.).
4. Convert the detected instances into a binary mask for each keyframe (1 = static scene, 0 = dynamic object).
5. Store the masks in the same order as the keyframes.
6. Ensure that later feature extraction ignores masked pixels.
7. Ensure that later depth estimation ignores masked pixels.

### Design philosophy
Binary masking is preferred over generative inpainting.  
The system would rather leave a region marked as unknown than invent geometry that never existed.

### Output of Stage 5
One binary mask per keyframe.

---

## STAGE 6 — POSE ESTIMATION (STRUCTURE-FROM-MOTION)

### Purpose
Recover the three-dimensional position and orientation of every keyframe camera and a sparse set of 3D points that are consistent with all images.

### Why this stage exists
All later dense reconstruction and georeferencing depend on knowing where each camera was when the image was taken. This stage is the geometric backbone of the entire system.

### Detailed intermediate and small steps

1. Extract learned visual features from every keyframe using SuperPoint.
2. Match features between overlapping keyframes using LightGlue.
3. Build a graph of verified feature correspondences.
4. Apply geometric verification (RANSAC + epipolar geometry / Essential Matrix) to remove wrong matches.
5. Feed the verified matches into GLOMAP (global Structure-from-Motion).
6. GLOMAP estimates all camera rotations and positions in a globally consistent way.
7. Triangulate sparse 3D points from the verified multi-view observations.
8. Run Global Bundle Adjustment:
   - Jointly optimize all camera poses and all 3D points
   - Minimize total reprojection error
   - Optionally add soft constraints from GPS and altitude
   - Use a robust kernel (Huber or Cauchy) so that remaining outliers have limited influence
9. Compute quality statistics (average and median reprojection error, number of registered cameras, number of 3D points).
10. Decide whether the pose reconstruction is good enough to continue.

### Core technical choices

| Component | Choice |
|-----------|--------|
| Feature detector | SuperPoint |
| Feature matcher | LightGlue |
| SfM engine | GLOMAP |
| Optimization | Global Bundle Adjustment |
| Robust kernel | Huber / Cauchy |

### Important mathematical ideas (plain language)

- **Reprojection error** — After a 3D point is projected back into an image, how many pixels away is it from the originally detected feature? Lower is better.
- **Bundle Adjustment** — A large non-linear optimization that gently moves cameras and points until the total reprojection error is as small as possible.
- **RANSAC** — A method that repeatedly tries random subsets of matches so that a correct geometric model can be found even when many matches are wrong.

### Output of Stage 6
Camera pose for every keyframe + sparse 3D point cloud + pose quality metrics.

---

## STAGE 7 — DEPTH ESTIMATION AND DENSE FUSION

### Purpose
Turn the sparse geometric skeleton into a dense surface that can represent terrain, buildings, roads and vegetation.

### Why this stage exists
Sparse points alone are not sufficient for a usable 3D model. Dense depth is required.

### Role of each component
- GLOMAP supplies accurate camera poses (the geometric backbone).
- Depth Anything V2 supplies a dense depth map for every keyframe (densification).
- TSDF fusion combines many depth maps into one consistent volumetric model.

### Detailed intermediate and small steps

1. Run Depth Anything V2 on each prepared keyframe to obtain a dense depth map.
2. Associate each depth map with the corresponding camera pose from Stage 6.
3. Apply the dynamic-object masks so that moving objects do not contribute depth.
4. Filter each depth map:
   - Discard low-confidence depth values
   - Remove flying pixels (isolated depth samples floating in space)
   - Enforce multi-view consistency where the same surface is seen from several cameras
   - Apply light morphological cleanup if needed
5. Back-project the surviving depth values into 3D space using the camera poses.
6. Integrate the back-projected points into a Truncated Signed Distance Function (TSDF) volume using Open3D.
7. Extract a mesh and/or a dense point cloud from the finished TSDF volume.

### Optional later path
3D Gaussian Splatting can be offered as an alternative high-visual-quality representation, but the primary metric path remains TSDF-based.

### Output of Stage 7
Dense point cloud and/or mesh of the observed scene.

---

## STAGE 8 — METRIC ALIGNMENT / GEOREFERENCING

### Purpose
Transform the reconstruction from an arbitrary local coordinate system into a real-world, metric coordinate system.

### Why this stage exists
Structure-from-Motion alone recovers shape only up to scale and orientation. GPS and altitude supply the missing absolute position and scale.

### Detailed intermediate and small steps

1. Convert the GPS latitude/longitude (and altitude) into a local Cartesian coordinate system (ENU or UTM).
2. Collect the reconstructed camera centers from Stage 6.
3. Form correspondences between reconstructed camera centers and the corresponding GPS positions.
4. Estimate a Similarity transformation (Sim3) that best aligns the two sets of points. The closed-form solution is the Umeyama method (based on Singular Value Decomposition).
5. Apply the recovered scale, rotation and translation to every camera and every 3D point.
6. Optionally run a light Bundle Adjustment that includes soft GPS and altitude residual terms to refine the alignment.
7. Compute residual errors between the aligned model and the telemetry.
8. Record georeferencing quality statistics.

### Mathematical core
- Umeyama / Horn Sim3 (scale + rotation + translation)
- Soft GPS / altitude constraints inside Bundle Adjustment
- Levenberg-Marquardt optimization

### Output of Stage 8
Georeferenced cameras and 3D model expressed in a real-world coordinate system.

---

## STAGE 9 — TEXTURING AND CONFIDENCE MAP GENERATION

### Purpose
1. Colour the 3D surface with real imagery from the original video.
2. Produce a confidence (uncertainty) map that shows which parts of the model are trustworthy.

### Why this stage exists
A colourless geometric model is hard to interpret.  
A confidence map is essential because single-pass capture leaves many surfaces poorly observed or completely unseen.

### Detailed intermediate and small steps

**Texturing**
1. For each mesh face (or surface region) determine which keyframes can see it.
2. Among the visible keyframes choose the best view according to viewing angle, resolution and occlusion.
3. Project the chosen image(s) onto the surface.
4. When several good views exist, blend them with weights so that seams are reduced.

**Confidence**
5. Count how many independent views support each point or face.
6. Average the reprojection error of the observations that support it.
7. Evaluate the quality of the viewing angles.
8. Combine the three factors into a single confidence score.
9. Store the confidence value (as a vertex attribute or separate layer).
10. Visualize confidence with a simple colour scale (green = high, yellow = medium, red = low/incomplete).

### Output of Stage 9
Textured mesh + confidence information attached to the geometry.

---

## STAGE 10 — FINAL MODEL CLEANUP AND OPTIMIZATION

### Purpose
Remove noise and unreliable geometry while deliberately refusing to invent missing surfaces.

### Why this stage exists
Dense fusion still produces floating points, tiny disconnected fragments and excess triangles. Cleanup improves usability without destroying metric honesty.

### Detailed intermediate and small steps

1. Apply moderate confidence-based filtering (remove only clearly low-confidence geometry).
2. Perform statistical outlier removal with conservative parameters.
3. Perform radius-based outlier removal.
4. Detect and delete very small connected components (floating fragments).
5. Perform light topology repair (non-manifold edges, duplicate vertices) only where safe.
6. Recompute surface normals.
7. If the model is still too large, apply feature-preserving decimation (Quadric Error Metrics that protect important edges).
8. Hole filling is kept optional and is used only for tiny holes when justified; large missing regions are left empty.

### Core rule
Visible incompleteness is preferable to invented geometry that looks complete but is metrically wrong.

### Output of Stage 10
Clean, usable textured mesh and/or point cloud ready for export.

---

## STAGE 11 — EXPORT AND OUTPUT GENERATION

### Purpose
Write all practical deliverables required by the problem statement and by normal geospatial workflows.

### Detailed intermediate and small steps

1. Load the final cleaned geometry, textures and confidence attributes.
2. Ensure georeferencing metadata (coordinate system, units) is attached.
3. Export textured mesh as OBJ (with material and texture files).
4. Export mesh or point cloud as PLY (including colour and confidence when possible).
5. Export point cloud as LAS.
6. Export modern real-time mesh as GLB/GLTF.
7. Rasterize a Digital Surface Model and write it as GeoTIFF.
8. Package a simple web viewer (Three.js or model-viewer) that can load the GLB.
9. Write a processing report that records:
   - Which input mode was used
   - Number of keyframes
   - Telemetry quality level
   - Basic reconstruction statistics
10. Write coordinate-system metadata so downstream GIS tools can place the model correctly.

### Output of Stage 11
Complete set of files (mesh, point cloud, DSM, confidence, report, viewer).

---

## STAGE 12 — VALIDATION AND ACCURACY EVALUATION

### Purpose
Measure the real geometric accuracy of the finished model against independent ground-truth data.

### Why this stage exists
Telemetry quality alone cannot prove that the 3D model is accurate. Only comparison with surveyed or otherwise trusted reference points can do that.

### Detailed intermediate and small steps

1. Load a set of surveyed or reference ground-truth points that lie inside the reconstructed area.
2. For each ground-truth point find the corresponding location on the reconstructed model.
3. Compute horizontal error:  
   \( E_h = \sqrt{(x_r - x_g)^2 + (y_r - y_g)^2} \)
4. Compute vertical error:  
   \( E_v = |z_r - z_g| \)
5. Compute three-dimensional error:  
   \( E_{3D} = \sqrt{E_h^2 + E_v^2} \)
6. Aggregate the errors into RMSE, MAE and maximum error.
7. Compute coverage / completeness (fraction of the area that received a valid reconstruction).
8. Compare the confidence values of the evaluated points with the actual measured errors (to verify that the confidence map is meaningful).
9. Write a clear accuracy report that can be shown to evaluators.

### Output of Stage 12
Quantitative accuracy report (the final scientific statement of how well the system performed).

---

# SECTION E — HOW THE STAGES WORK TOGETHER

No single neural network is responsible for the whole result.

- Stages 1–5 prepare clean, masked, well-spaced observations.
- Stage 6 (GLOMAP + SuperPoint + LightGlue) builds the geometric backbone.
- Stage 7 densifies that backbone.
- Stage 8 gives the model real-world scale and position.
- Stage 9 adds appearance and an honest uncertainty map.
- Stage 10 removes artefacts without inventing geometry.
- Stage 11 produces usable files.
- Stage 12 tells the truth about accuracy.

This clear separation of responsibilities is what makes the architecture implementable and debuggable.

---

# SECTION F — REALISTIC EXPECTATIONS (BASE PIPELINE)

## Accuracy

| Condition | Horizontal | Vertical |
|-----------|------------|----------|
| Video only | Relative only | Relative only |
| GPS only | approximately 2–5 m | approximately 3–8 m |
| GPS + Barometer (normal/good) | approximately 1–3 m | approximately 1.5–4 m |
| Very good controlled acquisition | approximately 0.5–2 m | approximately 1–3 m |

The SIH target wording remains: sub-2-meter horizontal accuracy under defined good conditions, proven by Stage 12.

## Processing time

Target: less than 15 minutes for a 10-minute video on a defined GPU configuration.  
Actual time must be measured on the demonstration hardware.

---

# SECTION G — LOCKED BASE TECHNOLOGY STACK (SUMMARY)

| Stage | Primary Choice |
|-------|----------------|
| Features + Matching | SuperPoint + LightGlue |
| Pose Estimation | GLOMAP |
| Depth | Depth Anything V2 |
| Fusion | Open3D TSDF |
| Metric Alignment | Hybrid Sim3 (Umeyama) + constrained Bundle Adjustment |
| Texturing | Best-view selection + weighted blending |
| Confidence | View count + reprojection error + viewing angle |
| Cleanup | Conservative confidence-aware pipeline |
| Export | trimesh, Open3D, laspy, rasterio, Three.js |
| Validation | Ground-truth RMSE / MAE / coverage |

---

# SECTION H — ONE-LINE MENTAL MODEL

**READ → CHECK → SYNC → SELECT → MASK → RECONSTRUCT CAMERA MOTION → ESTIMATE DEPTH → FUSE → GEOREFERENCE → TEXTURE + CONFIDENCE → CLEAN → EXPORT → VALIDATE**

---

**Document status:** Complete detailed process explanation of the locked Base Pipeline  
**Intended audience:** Developers, reviewers and anyone who needs to understand or implement every step  
**Companion documents:** Base Architecture, RTK Extension Architecture, Implementation Guide

*End of detailed Base Pipeline process explanation.*
