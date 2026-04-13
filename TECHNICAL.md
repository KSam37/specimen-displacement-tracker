# Technical Documentation

## Overview

This tracker automates displacement measurement from Instron tensile test videos. Each video shows an elastic specimen clamped in a tensile tester, being stretched vertically. Two black Sharpie marker dots are placed on the specimen; the tracker measures the distance between them over time and outputs displacement relative to the first frame.

The pipeline has five main stages:

1. Specimen region detection
2. Initial dot detection
3. Frame-by-frame template matching
4. Centroid refinement
5. Output and calibration

---

## File Structure

```
Python openCV Tracker/
├── app.py              — GUI application (tkinter)
├── tracker_core.py     — Core tracking engine (OpenCV)
├── Launch Tracker.bat  — Windows one-click launcher
├── input_videos/       — Place .MOV files here
├── output_data/        — Auto-saved CSV results
└── validation/
    ├── build_comparison.py              — Generates comparison Excel sheet
    └── Tracker vs openCV Comparison.xlsx
```

---

## tracker_core.py

### `extract_initial_distance_mm(filename)`

Parses the filename to extract the initial dot separation in mm using a regex pattern matching `<number>mm` (e.g., `49.9mm`). This value is used as the pixel-to-mm calibration reference.

---

### `find_specimen_region(gray)`

Detects the bright specimen region between the two dark Instron jaw clamps.

**How it works:**
1. Counts dark pixels (intensity < 60) per row across the full frame width
2. Smooths the count with a 10-row moving average
3. Classifies any row where >30% of pixels are dark as a "jaw row"
4. Finds contiguous jaw bands taller than 30px
5. Identifies the gap between adjacent jaw bands that has the highest average brightness in its center strip — this is the specimen

**Why this matters:** Without jaw detection, the tracker could latch onto dark rig hardware (bolts, clamp edges) that score higher contrast than the actual dots, especially in zoomed-out video frames.

**Returns:** `(y_min, y_max)` of the specimen region, or `None` if fewer than 2 jaw bands are found.

---

### `find_initial_dots(gray)`

Detects exactly two Sharpie marker dots in the first frame.

**How it works:**

**Step 1 — Restrict search area:**  
Limits detection to the specimen region (from `find_specimen_region`) plus a small margin. Also excludes the leftmost and rightmost quarters of the frame, since dots are always placed near the specimen centerline.

**Step 2 — Multi-threshold blob detection:**  
Iterates threshold values from 90 to 170 (step 5). At each threshold, inverts the image (dark pixels become white), applies morphological opening to remove noise, and extracts contours. Contours are filtered by:
- Area: 15–500 px²
- Aspect ratio: 0.15–6.0 (excludes extreme elongation)

**Step 3 — Annular contrast filter:**  
For each candidate blob centroid, computes:
- `surround_mean`: mean intensity in an annulus from radius 8–25px around the centroid
- `center_mean`: mean intensity in a 7×7px patch at the centroid
- `contrast = surround_mean - center_mean`

Only candidates where `surround_mean > 140` (bright surroundings) **and** `contrast > 40` (clearly darker than surroundings) pass. This rejects dark hardware features that sit on dark backgrounds.

**Step 4 — Clustering:**  
Candidates within 15px of each other (the same dot detected at multiple thresholds) are merged into a single cluster. Each cluster's score is `max_contrast × detection_count`.

**Step 5 — Pair selection:**  
Among the top 8 clusters by score, selects the pair with the highest combined score where the two dots are at least 50px apart vertically. Returns them sorted top-to-bottom.

**Returns:** `[(x1, y1), (x2, y2)]` or `None`.

---

### `track_dot_template(gray, template, last_pos, search_radius=60)`

Tracks a single dot in a new frame using Normalized Cross-Correlation (NCC).

**How it works:**
1. Defines a search window centered on the dot's last known position, extended by `search_radius` in each direction
2. Runs `cv2.matchTemplate` with `TM_CCOEFF_NORMED` (values range −1 to 1; 1 = perfect match)
3. Rejects the result if the peak correlation score is below 0.25

**Why NCC:** Normalized cross-correlation is invariant to uniform brightness changes, which matters because the specimen stretches and the dot appearance changes (it elongates and may fade as the specimen deforms).

**Returns:** `(cx, cy), score` or `(None, score)` on failure.

---

### `refine_centroid(gray, pos, patch_size=30)`

Snaps the template-matched position to the true centroid of the dark blob, sub-pixel accurate.

**Problem it solves:** As the specimen stretches, dots elongate and the template match may land slightly off-center. Simple intensity thresholding also fails because overall frame brightness varies. This function finds the dark anomaly relative to the local background regardless of absolute intensity.

**How it works:**
1. Extracts a patch of size `2×patch_size` around the template position
2. Computes a heavy Gaussian blur (σ=12) of the patch — this estimates the local background intensity at each pixel, as if the dot weren't there
3. Computes `contrast_map = blurred − patch` — positive values indicate pixels darker than their surroundings (i.e., the dot)
4. Thresholds to keep only pixels with contrast > 35% of the peak contrast in the patch
5. Finds connected components in the mask and selects the component closest to the patch center
6. Computes an intensity-weighted centroid within that component

**Returns:** `(cx, cy)` in full-frame coordinates.

---

### `VideoTracker` class

Stateful, frame-by-frame tracker designed for GUI integration.

**Key state:**
- `pos_top`, `pos_bot`: current dot positions (full-frame float coordinates)
- `template_top`, `template_bot`: rolling templates (updated every 15 frames)
- `ref_template_top`, `ref_template_bot`: original first-frame templates (fallback)
- `results`: list of `(time_s, distance)` tuples
- `positions`: list of `(pos_top, pos_bot)` tuples, parallel to results
- `frame_indices`: list of frame numbers, parallel to results

**`open()`:**  
Opens the video, reads the first frame, runs `find_initial_dots` + `refine_centroid`, stores templates, records the first data point at t=0, and returns an annotated preview frame.

**`step()`:**  
Processes one step (skipping `frame_skip − 1` frames):
1. Tries current rolling template → falls back to original reference template if score < 0.25
2. Expands `search_radius` by 20px per consecutive failure
3. Rejects jumps > 80% of search radius (treated as false matches)
4. On success: refines both centroids, updates rolling templates every 15 frames, appends result
5. On failure: repeats the last known distance and position; terminates after 60 consecutive failures

**Template rolling:** The rolling template is updated every 15 frames to track gradual appearance changes (dot elongation, fading). The original reference template is kept as a fallback to recover from brief tracking loss.

**`save_csv(path)`:**  
Writes `time_s` and `displacement_mm` (or `displacement_px`). Displacement is distance minus the first-frame distance, so it starts at 0.

---

### `annotate_frame(frame, pos_top, pos_bot, dist_val, unit)`

Draws tracking overlays on a frame for display:
- Green crosshairs (±18px) and circles (r=12) at each dot position
- Cyan line connecting the two dots
- Distance label in mm or px

All drawn on an overlay copy, then blended at 50% opacity using `cv2.addWeighted` so the dot remains visible under the crosshair.

---

## app.py

### Threading model

Processing runs in a background `daemon` thread (`_worker`). The worker sends messages to the UI via a `queue.Queue`. The UI polls the queue every 30ms using `tk.after`. This prevents the GUI from freezing during processing.

Message types:
- `MsgProgress(vid_idx, frame_idx, total_frames, frame_bgr)` — progress update; frame sent every 20 processed frames to limit UI load
- `MsgDone(vid_idx, tracker)` — video finished; full tracker object with all results
- `MsgError(vid_idx, error_msg)` — detection or file error
- `MsgAllDone` — all videos finished

### Video review

Completed videos can be scrubbed frame-by-frame. A `cv2.VideoCapture` is held open for the currently-reviewed video. The scrub bar (`ttk.Scale`) calls `_show_review_frame(frame_idx)` which:
1. Seeks to the requested frame with `cv2.CAP_PROP_POS_FRAMES`
2. Finds the closest tracked position using `np.searchsorted` on the stored `frame_indices`
3. Calls `annotate_frame` with that position and re-renders

A `_scrub_blocked` flag prevents the scrub callback from firing during `configure(to=...)` calls (which would otherwise seek to frame 0 spuriously).

### Outlier cleaning (`clean_data`)

Two-pass MAD-based filter:

**Pass 1 — Velocity outliers:**  
Computes frame-to-frame velocity `dd/dt`. In a rolling window of 51 frames, computes the median velocity and MAD (Median Absolute Deviation). Points where velocity deviates by more than 5× MAD are removed.

**Pass 2 — Position outliers:**  
On the velocity-filtered data, computes a rolling window median of the distance values. Points deviating more than 5× MAD from the local median are removed.

MAD is preferred over standard deviation because it is robust to the outliers being removed — a single large spike does not inflate the threshold.

### Pixel-to-mm calibration

On `VideoTracker.open()`, the pixel distance between the two detected dot centroids is computed. If the filename contains an initial distance in mm (e.g., `49.9mm`), then:

```
px_per_mm = initial_pixel_distance / initial_distance_mm
```

All subsequent distances are divided by `px_per_mm`. If no distance is found in the filename, distances are reported in pixels.

---

## validation/build_comparison.py

Generates `Tracker vs openCV Comparison.xlsx` comparing manual Tracker measurements against OpenCV output.

**Process:**
1. Loads manual Tracker CSV (2 header rows, columns: time, length in meters)
2. Loads OpenCV CSV (time, pixel distance)
3. Calibrates OpenCV pixel values to mm using `tracker_mm[0] / opencv_px[0]`
4. Time-matches: for each Tracker timestamp, finds the nearest OpenCV timestamp
5. Computes % difference column
6. Writes a formatted table, statistics panel (RMSE, R², correlation, mean/median/max % difference), and two embedded Excel charts (distance overlay + % difference over time)
