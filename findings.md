# Diving Foot Placement Tracker — Findings

## Project Concept
Foot placement on the board during takeoff matters for consistency and coaching feedback. Goal: sensor hardware that measures foot placement per attempt, syncing to a mobile app that tracks history and shows consistency against the coach's recommended target.

## Sensing Approach — Evolution

1. **Foot-worn sensors (pressure insoles / IMUs)** — ruled out. They measure pressure/motion but have no way to know *where* the foot is relative to the board.
2. **Board-mounted pressure mat / touch sensor (FSR array)** — ruled out. Lower accuracy, physical wear on the board, and prone to noise from water/texture.
3. **Industrial ultrasonic distance sensor (SICK UC20 / UM18-217, IO-Link)** — considered, then dropped. Reasonable accuracy and good noise-averaging (beam cone smooths out board texture/water droplets), but the IO-Link interface master is built for Windows/industrial PC software stacks — Raspberry Pi (ARM Linux) driver/SDK support is uncertain, adding real integration risk.
4. **Single-axis laser displacement sensor (Micro-Epsilon optoNCDT, Keyence IL series)** — viable fallback if only forward/back distance from the board tip matters. Micron-level accuracy, avoids IO-Link risk by using analog or RS-485/Modbus RTU output instead (well-supported on Pi via `pymodbus` or an ADC HAT). Limitation: single axis only — no side-to-side centering data.
5. **Overhead camera + ArUco fiducial markers (chosen)** — full foot placement is a 2D problem (forward/back **and** side-to-side centering), which a coach actually critiques. A calibrated 2D vision approach suits a flat, known plane like the board better than a 3D depth camera (stereo/ToF depth error is typically 2–5mm). Achievable precision is set by camera resolution — see Accuracy Budget.

## Related Research: DiveNet (DFKI, 2023)
Paper: *DiveNet: Dive Action Localization and Physical Pose Parameter Extraction for High Performance Training* (Murthy, Taetz, Lekhra, Stricker — IEEE Access 2023).
- Validates the core strategy: single static camera (60Hz) + homography between the board plane and image, computed **per-dive without manual calibration**, extracts physical parameters (center of mass, joint keypoints, dive height) in metric scale.
- Reported accuracy: COM error ~6px, dive peak height sensitivity ~20cm, mean joint angle error ~10°, pose regression ~70% PCK — good for research-grade action/form analysis, but too coarse for ±1° foot angle.
- **No open-source code or usable dataset found.** A GitHub search led to an unrelated repo (`aim-uofa/Poseur`) with no connection to DiveNet — a bad search association, verified and discarded. The dataset host (`divenet.kl.dfki.de`) now redirects to an unrelated DFKI project; the dataset appears inaccessible as of 2026-09. The full paper PDF is blocked from automated fetch (403 on DFKI and ResearchGate).
- Takeaway: can't reuse DiveNet's code/data directly, but its per-dive automatic homography idea and its published accuracy numbers are a useful reference for our markerless cross-checks.
- **Comparing DiveNet's numbers to our task (2026-09-18):** they split into two kinds of task, and only one transfers.
  - *COM trajectory (~6px) and peak height (~20cm)* come from tracking the **entire dive** and fitting a projectile-motion curve — a multi-frame, compounding-error task, harder than ours. Not a fair ceiling for our accuracy.
  - *2D joint keypoints (~70% PCK, ~10° joint angle error)* is **per-frame keypoint localization** — the same task category as our markerless cross-check. We should do better, since we control calibration and camera placement far more tightly, but the bench accuracy test (below) gives our real number.

## Requirements (updated 2026-09-25)

- **Both feet** are tracked on every attempt.
- **Position** of each foot on the board: forward/back and side-to-side.
- **Orientation** (toe-pointing angle) of each foot, to within **±1°**.
- **Divers are barefoot**, so anything attached to the foot goes directly on skin.

**Why ±1° is hard:** orientation isn't measured directly — it's derived from two points (heel and toe) by computing the angle of the line between them. For a ~270mm foot, 1° of rotation moves the toe sideways relative to the heel by only ~5mm, and reliably detecting that needs each point accurate to roughly 1mm. Noise in two independently-estimated points compounds when they're subtracted.

**This rules out markerless BlazePose landmarks as the primary method** — a general-purpose pose model isn't reliable to ~1mm per point from a steep, oblique camera angle. It stays as a position cross-check.

## Accuracy Budget (added 2026-10-01)

- **Resolution:** the Arducam IMX296 is 1440×1080 effective pixels. If the frame covers the front ~1.6m of a standard 4.8m springboard, that's ~1.1mm per pixel along the board. Sub-pixel corner refinement does better than one pixel, but sub-mm precision isn't guaranteed — the bench test decides.
- **One marker per foot is not enough for the angle.** A ~25mm marker is only ~23px wide, so a 0.3px corner error is already ~0.7°, and a marker glued 2° off the foot's axis adds a fixed 2° error that no software can see.
- **Two markers per foot fixes both problems.** With one marker at the heel and one at the base of the big toe, the angle comes from the line between the two marker centers, ~250mm (~230px) apart. A 0.5px center error then costs only ~0.1°, and how each marker is rotated on the skin no longer matters — only where it sits.
- **The angle is computed in board coordinates.** Both points are first corrected for lens distortion and mapped through the homography to millimeters; only then is the angle calculated. An angle measured in raw pixels is skewed by perspective.
- **Lens distortion must be corrected.** M12 lenses (especially the optional 90° lens) have barrel distortion, and a homography assumes none. A one-time lens calibration with a printed checkerboard (`cv2.calibrateCamera`) comes before the board calibration.
- **Board deflection:** the board dips under the diver's weight at the exact moment being measured; a steep camera angle and calibrating under a static load keep this error small (see Mounting).
- **Bench accuracy test before going poolside:** a printed foot template with heel and toe markers on a flat surface, rotated in known 1° steps (protractor or jig) and moved to known positions. Pass criteria: angle error under 1°, position error within the coach's tolerance.

## Recommended Solution

### Primary: two ArUco markers per foot (decided 2026-10-01)
- **Four markers total** from `DICT_4X4_50`: left heel `10`, left toe `11`, right heel `12`, right toe `13`.
- **Placement:** center of the heel and base of the big toe, recorded in the athlete's reference photo so placement is repeatable.
- **Size:** ~20–25mm. The angle comes from the heel–toe line, so markers only need enough pixels to be detected reliably.
- **Material:** waterproof, matte (to avoid glare) printed markers — e.g. waterproof vinyl or temporary-tattoo paper — with a skin-safe adhesive. They must survive a water entry on every attempt and pool chlorine; test durability before relying on them.
- **Output per foot:** heel and toe positions on the board in millimeters, and the foot angle in degrees.

### Camera
- **Arducam IMX296 color global-shutter module** with its bundled M12 lens (~45–55° horizontal FOV) and Pi 5 cable. Global shutter is essential: a rolling shutter smears the fast-moving foot at takeoff.
- **Pi 5 setup on Bookworm:** in `/boot/firmware/config.txt`, change `camera_auto_detect=1` to `camera_auto_detect=0` and add `dtoverlay=imx296` under `[all]` (append `,cam0` if the camera is on the CAM0 port).
- The Arducam 90° M12 lens is optional — buy it only if the bundled lens can't fit the whole takeoff zone in frame at the real mounting distance.

### Calibration
- **Lens (one-time):** ~15–20 photos of a printed checkerboard at different angles → `camera_matrix` and `dist_coeffs`.
- **Board:** reference markers at known board positions → homography from image pixels to board millimeters. Use temporary markers for calibration, or place them on the board's side edges — never leave them on the non-slip walking surface during dives (slip and wear risk). Calibrate under a static test load (someone standing on the board).
- **Recalibrate the board whenever the camera mount moves.** Lens calibration only needs redoing if the lens changes.
- Both results are saved by `CalibrationStore` and loaded at startup.

### Markerless cross-checks (secondary)
- **BlazePose (`PoseDetector`):** MediaPipe's 33-point model includes `heel` and `foot_index` (near the big toe) for both feet — position cross-check only. MediaPipe installs on Raspberry Pi OS **Bookworm** (Python 3.11), not Trixie (Python 3.13), which is why the Pi runs Bookworm. MoveNet is not an option: its keypoints stop at the ankle.
- **Classical CV:** background-subtract the static board to isolate each foot's silhouette, then fit an ellipse or principal axis (`cv2.fitEllipse` / PCA) for position and orientation. Purpose-built for this one scene; no training data needed.
- **Custom-trained keypoint model (added 2026-09-18):** fine-tune a small keypoint model on our own board/camera footage instead of BlazePose's general-purpose training.
  - Needs a few hundred labeled frames (foot at known positions, heel/toe hand-annotated), fine-tuned from a pretrained backbone.
  - Training happens on a GPU (laptop or cloud); only the small exported model runs on the Pi.
  - This is a **vision/keypoint model**, not an LLM — it determines the tooling (TensorFlow/PyTorch fine-tuning).
  - A legitimate independent-study extension in its own right (data collection, labeling, fine-tuning, deployment) — worth including in the funding narrative if the study takes it on.

## Athlete Profile (added 2026-09-25)
Lightweight per-athlete profile, stored in Supabase:
- **Foot length and width.**
- **Heel-to-toe marker distance**, measured once. It should be the same on every reading, so the Pi can flag a detection whose measured distance differs as a probable error.
- **Reference photo of marker placement** (both markers on both feet), so markers go in the same spot every session and measured inconsistency reflects the diver, not the gluing.
- **Scope:** no 3D scan or shape model.

## Mounting & Power

### Mounting
- Mount on the board's **handrails**, not the plank — handrails are bolted to the fixed stand and don't move with the diver's bounce.
- Use a rigid boom/clamp arm from the handrail near the **fulcrum (fixed) end**, aimed down at the **front third of the board** (the takeoff zone).
- Keep the camera angle as **steep/overhead as possible** — a shallow angle turns the board's vertical deflection into horizontal position error (parallax). Handrails sit toward the fixed end, so a steep view of the front third may need a longer boom; measure on site.
- **The Pi rides on the boom next to the camera.** The bundled camera cable is short (check its length on arrival), and CSI cable runs beyond ~1m become unreliable.
- The boom must stay clear of the diver's walk-out, hurdle, and arm swing. **Get pool and coach approval before installing.**
- Camera and Pi both need **weatherproof enclosures** (splash, humidity, chlorine).

### Power: PoE (decided 2026-10-01)
- **Why PoE:** a USB-C cable from a wall outlet means mains power near the water, and the Pi 5 browns out on long USB-C runs because it needs a full 5A. With PoE, only low-voltage DC reaches the deck.
- **Setup:** an **802.3at (PoE+)** injector plugs into a GFCI outlet away from the pool edge. One outdoor-rated Ethernet cable (up to 100m) carries power to the PoE HAT on the Pi. Plain 802.3af can't power a Pi 5.
- **Data** can still go over Wi-Fi — the injector doesn't need a network connection — or over the same cable if the facility allows a wired network port.
- Don't connect the USB-C supply while running on PoE.
- **Check on arrival:** the PoE HAT fits alongside the Active Cooler and inside the Sixfab enclosure. The HAT is rated 25W against the Pi 5's 27W recommendation, which is fine for this load (no USB peripherals).
- The 27W USB-C supply is for bench development indoors.

## Purchase Plan

### Ordered (confirmed 2026-09-14, prices as of 2026-09-25)

**PiShop — $227.85**
| Item | Price |
|---|---|
| Official microSD 64GB w/ Raspberry Pi OS 64-bit | $29.95 |
| Raspberry Pi 5/8GB + 27W USB-C PSU + Active Cooler bundle | $197.90 |

**Amazon — $192.06**
| Item | Price |
|---|---|
| PoE HAT for Raspberry Pi 5/CM5 (802.3af/at) | $22.07 |
| Sixfab Outdoor IP65 Project Enclosure | $75.00 |
| Arducam IMX296 Color Global Shutter Camera, M12 lens + Pi 5 cable included | $94.99 (was $84.99; low stock — buy first) |

**Total ordered: $419.91**

⚠️ *Do NOT buy the "6mm Wide Angle Lens for Raspberry Pi HQ Camera CS" ($34.00) — wrong mount (CS, not M12) and redundant with the Arducam's bundled lens. Skip any cart containing the Raspberry Pi HQ Camera (PID 4561): it's a rolling-shutter camera.*

**After the microSD arrives:** reflash it with Raspberry Pi OS **Bookworm** via Raspberry Pi Imager — the pre-loaded image is likely Trixie, which breaks MediaPipe.

### Still to buy (estimates)
| Item | Est. price | Note |
|---|---|---|
| PoE+ injector (802.3at, ≥30W) | $25–35 | search `802.3at poe+ injector` |
| Outdoor-rated Ethernet cable (Cat6) | $20–40 | measure the run to the GFCI outlet first |
| Foot-marker material + skin-safe adhesive | $10–20 | waterproof, matte; test durability |
| Weatherproof camera housing | $20–40 | source once the camera+lens dimensions are known |
| Articulating boom/clamp arm | $30–60 | measure handrail diameter first |
| Arducam 90° Wide Angle M12 Lens | $17.99 | only if the bundled lens's FOV is too narrow |
| Checkerboard + board reference markers | free | print |

**Estimated final total: ~$525–635** against the $600 funding request. The high end slightly exceeds it; the conditional lens and the boom arm are where to economize.

### Camera sourcing history
- The official Raspberry Pi Global Shutter Camera was out of stock at 5 of 6 retailers checked on 2026-09-13 (Adafruit, CanaKit, PiShop, The Pi Hut, Pimoroni; SparkFun backorder only). The Arducam IMX296 uses the same sensor and was in stock.
- Other Arducam options considered: AR0234 (2.3MP color, "Pivariety" driver) and OV9782 (1MP, but that listing is USB, not CSI). Monochrome sensors were avoided because the BlazePose cross-check needs color.
- The Raspberry Pi HQ Camera (IMX477) was ruled out: rolling shutter.

### Alternative solutions considered (not chosen)

**Solution B — Intel RealSense depth camera** (no markers, less precise)
| Item | Purpose | Approx. Price |
|---|---|---|
| Raspberry Pi 5 (8GB) | Runs `librealsense`/`pyrealsense2` + app sync | $200 |
| Intel RealSense D435 (or D455) | Stereo depth camera | $200–350 |
| Articulating boom/clamp arm | Mounts camera to handrail | $30–60 |
| Weatherproof housing w/ IR-transparent window | Must not block IR sensing | $30–50 |
| Weatherproof Pi enclosure | Protects Pi electronics | $25–40 |
| USB 3.0 cable, short run (or active/repeater) | Stable USB3 bandwidth | $15–25 |
| Powered USB hub *(optional)* | Reliable camera power | $15–20 |
| MicroSD card + Pi power supply | Pi OS/storage + power | $20–30 |

**Solution C — single-axis laser displacement sensor** (forward/back only)
| Item | Purpose | Approx. Price |
|---|---|---|
| Laser triangulation sensor (Micro-Epsilon optoNCDT / Keyence IL series), analog or RS-485/Modbus output | Micron-level forward/back distance | $800–1,500 |
| Mounting bracket at board's fixed base/fulcrum end | Stable anchor point | $20–40 |
| ADS1115 I2C ADC HAT *(analog)* or USB-to-RS485 adapter *(Modbus)* | Interfaces sensor to Pi | $15–25 |
| Raspberry Pi 5 (8GB) | Runs read loop + app sync | $200 |
| M12 sensor cable | Connects sensor to interface | $25–50 |
| DC power supply (12–24V, sensor-specific) | Powers the sensor | $20–30 |
| Weatherproof Pi enclosure | Protects Pi electronics | $25–40 |
| MicroSD card + Pi power supply | Pi OS/storage + power | $20–30 |

## Architecture Decision
- No laptop poolside — a **Raspberry Pi bridges** the camera to the phone app (camera → Pi → cloud → phone).
- Target platform: **mobile app** (iOS and Android).

## Data Pipeline & App Architecture

### On-Pi: Capture → Detect → Package
- The camera is aimed at the takeoff zone. Detection runs on the full frame, and the takeoff zone is defined in board millimeters, so pixel coordinates always match the calibration.
- Each frame: detect the markers → heel and toe pixel points per foot → board millimeters and foot angle.
- An attempt is both feet present in the takeoff zone, then gone. The last frame with both feet in contact is the reading — no manual trigger needed.
- **Per-attempt record (Pi → Supabase):** `attempt_id, captured_at, session_id, diver_id, dive_type` (all three null when no session is active — "unassigned"), `left_foot {heel_x_mm, heel_y_mm, toe_x_mm, toe_y_mm, angle_deg, confidence, source}`, `right_foot {…}`, `thumbnail_url`.
- The record holds **no coach targets or deviations** — the Pi only reports where the feet were; the app compares against targets (see Compute Responsibility Split).
- A thumbnail (frame with the detected points and foot lines drawn on) lets the coach sanity-check a reading.
- Short video clips are deferred — they'd need a rolling frame buffer and aren't in the first version.

### Session context (decided 2026-10-01)
- **App:** the coach taps "Start session" and picks the diver and current dive type (changeable during the session), then "End session" at the end. This writes a Session row with `status = active`.
- **Pi:** `CloudSyncClient.fetch_active_session()` runs every few seconds on a background thread; `CaptureService` keeps the latest result. Each attempt is tagged with the session, diver, and dive type that were active when it was captured.
- **Offline:** the Pi keeps using the last session it saw.
- **No active session:** attempts are still captured and saved as unassigned; the app lets the coach assign them to a session afterward. Nothing is lost.

### Sync: Pi → Cloud → App
- Supabase is the cloud store. The Pi saves every attempt to its local buffer first, then uploads when online; anything pending is retried on the next sync pass.

### Mobile App
- **Session control:** start/end a session, choose diver and dive type, assign unassigned attempts.
- **Live view:** attempts appear near real-time during practice.
- **History:** filterable by session, date, and dive type.
- **Consistency dashboard:** each foot's position plotted against the coach's target zone, each foot's angle deviation, and both trends over time.
- **Coach controls:** per dive type, per foot — target position, target angle, and tolerances, plus the reference point (toe for forward-facing takeoffs, heel for backward); annotate attempts.
- Single athlete to start; `diver_id` supports more later.

### Stack
- **App:** React Native (TypeScript), cross-platform — chosen over Flutter for its larger beginner ecosystem and Supabase's JavaScript client.
- **Backend:** Supabase (Postgres) — database, realtime updates, file storage.
- **Pi:** Python — OpenCV, Picamera2, `supabase-py`, and MediaPipe for the cross-check — running as a `systemd` service.

## Raspberry Pi: Prerequisites to Learn First
1. Headless setup: flash **Bookworm** (not Trixie) with Raspberry Pi Imager, with SSH enabled before first boot.
2. Basic Linux command line (Pi OS is Debian-based) — `apt`, file permissions.
3. Camera: the `config.txt` lines for the IMX296, a test with `rpicam-hello`, then Picamera2 from Python.
4. Python `venv` + `pip` for isolated package installs.
5. Why Bookworm: MediaPipe doesn't install on Trixie (Python 3.13); it does on Bookworm (Python 3.11).
6. OpenCV: lens calibration (`cv2.calibrateCamera`, `cv2.undistortPoints`), homography (`cv2.findHomography`, `cv2.perspectiveTransform`), and ArUco detection.
7. `systemd` service files (start on boot, restart on crash) + `journalctl` for reading logs remotely.
8. Pi Wi-Fi config (`raspi-config`/`nmcli`) + `supabase-py` for talking to the backend.
9. SQLite basics, for local buffering when Wi-Fi drops.
10. Security: change the default password, keep the OS updated, use Tailscale (not open SSH) for remote access, keep the Supabase key in a `.env` file.

## Compute Responsibility Split: Pi vs. Cloud vs. Phone
Principle: **Pi senses, Cloud stores, Phone computes/presents.**

- **Raspberry Pi:** frame capture, marker detection (plus the optional markerless cross-check), lens correction + homography (pixels → board mm), foot angle, attempt detection, thumbnail generation, local SQLite buffering + retry-upload, reading the active session.
- **Cloud (Supabase):** passive data store — attempt records, sessions, athlete profiles, coach targets, thumbnail files, real-time updates to the app. No image processing or math.
- **Phone App:** fetches attempts and targets; computes deviation-from-target **at display time** (not stored), so changing a target recalculates history correctly; owns dashboards, session control, and the coach's target editing.
- Rationale: the Pi doesn't need to know about coaching targets — it only reports where the feet were. That keeps the sensing code simple and testable, and lets coaching logic change without touching the Pi.

## Security & Privacy (added 2026-10-01)
- **Supabase key on the Pi:** use a restricted key with row-level security that only allows inserting attempts, uploading thumbnails, and reading the active session — never the service-role key. Keep it in a `.env` file listed in `.gitignore`; never commit it (this repo is on GitHub).
- **Consent:** get athlete consent for recording and storing images in the cloud, and follow the school's policy for athletes under 18. Only per-attempt thumbnails are stored, not continuous video.

## Software Architecture (Pi side, revised 2026-10-01)

Reviewed against SOLID on 2026-09-25: thumbnail generation was extracted from the data record (SRP); a `FootDetector` interface makes detection methods swappable without touching the orchestrator (OCP/DIP); the orchestrator receives its dependencies through its constructor (DIP, testability).

Fixes from the 2026-10-01 quality check:
- Detectors return **pixel** coordinates (`FootKeypoints`); `CalibrationManager.to_world()` converts them to **board millimeters** (`FootMeasurement`) and computes the angle there.
- Lens calibration added to `CalibrationManager`.
- Calibration loading moved to the composition root (`main.py`); `CaptureService` never touches `CalibrationStore`.
- `CaptureService` gains the missing `camera` field and a cached `SessionContext`.
- `crop_to_roi` removed: cropping shifted pixel coordinates away from the calibrated full frame. The takeoff zone is now defined in board millimeters inside `AttemptEventDetector`.
- `clip_path` removed (nothing produced clips); `Frame` and `AttemptTrigger` are now defined.
- Network calls (sync, session refresh) run on a background thread so they never stall frame capture.

**Visual design doc:** class diagram, call sequences, and build order — `pi-side-design.html` in this repo, published at https://claude.ai/artifact/WgcJtuM2NdzgV3FbFQs9vv

### Entry points (scripts, not classes)
- **`main.py`** — composition root. Reads `.env`/config, loads calibration via `CalibrationStore` into `CalibrationManager.from_data()`, builds every collaborator, injects them into `CaptureService`, and calls `run()`. The `systemd` service runs this script.
- **`calibrate.py`** — operator tool. Runs lens calibration from checkerboard photos and board calibration from reference markers, then saves the result through `CalibrationStore`.

### Pi-Side Class Spec — Draft (17 classes)

*Draft: field/method-level detail with a one-line purpose statement per class. Subject to change once implementation starts.*

**Value objects (data only, no behavior)**

- **`Frame`** — Purpose: one captured image and when it was taken. Fields: `image`, `captured_at`. Methods: none.
- **`FootKeypoints`** — Purpose: one foot's heel and toe points in image pixels — what every detector returns. Fields: `foot_side`, `heel_px {x, y}`, `toe_px {x, y}`, `confidence`, `source` (marker/pose). Methods: none.
- **`FootMeasurement`** — Purpose: one foot's heel, toe, and angle in board millimeters — what everything after calibration uses. Fields: `foot_side`, `heel_mm {x, y}`, `toe_mm {x, y}`, `angle_deg`, `confidence`, `source`. Methods: none.
- **`AttemptTrigger`** — Purpose: hands over one detected attempt: the last-contact frame with both feet in pixels and in board millimeters. Fields: `frame`, `keypoints {left, right}`, `measurements {left, right}`. Methods: none.
- **`SessionContext`** — Purpose: the session the app marked active — who is diving and which dive. Fields: `session_id`, `diver_id`, `dive_type`, `fetched_at`. Methods: none.
- **`AttemptRecord`** — Purpose: one completed attempt as plain data, ready to store or upload. Fields: `attempt_id`, `captured_at`, `session_id`, `diver_id`, `dive_type` (all three null when unassigned), `left_foot`, `right_foot` (`FootMeasurement`), `thumbnail_path`. Methods: `to_dict()`.

**Interface & implementations**

- **`FootDetector`** *(interface)* — Purpose: the one contract every detection method satisfies, so methods swap without touching the orchestrator. Fields: none. Methods: `detect(frame) → list[FootKeypoints]` — returns an empty list when no foot is found, never raises for "not found", so implementations are interchangeable.
- **`MarkerDetector`** *(implements FootDetector)* — Purpose: finds the four ArUco markers and reports each foot's heel and toe points — the primary method. Fields: `dictionary`, `marker_ids` (`{left: {heel: 10, toe: 11}, right: {heel: 12, toe: 13}}`), `detector_params` (sub-pixel corner refinement on). Methods: `detect(frame)` — each marker's center (the mean of its four refined corners) becomes the heel or toe point; a foot is reported only when both of its markers are found.
  - **`dictionary`:** the ArUco marker dictionary (`cv2.aruco`) — a predefined family of binary square patterns, e.g. `DICT_4X4_50` (4×4-bit grid, 50 unique IDs), not a Python dict. More bits (6×6, 7×7) give more IDs and better false-positive rejection but need more pixels per marker; fewer bits let a smaller marker decode reliably. **Use `DICT_4X4_50`:** markers on bare skin should be small, and the project needs ~10 IDs (4 foot markers + up to 6 board calibration markers), well under 50.
- **`PoseDetector`** *(implements FootDetector)* — Purpose: BlazePose heel/foot_index landmarks as a markerless position cross-check. Fields: `model`, `confidence_threshold`. Methods: `detect(frame)` — `heel` becomes the heel point, `foot_index` the toe point.

**Geometry & calibration**

- **`CalibrationManager`** — Purpose: maps image pixels to board millimeters (lens correction + homography) — pure math, no file I/O. Fields: `camera_matrix`, `dist_coeffs`, `homography_matrix`. Methods: `calibrate_lens(checkerboard_images)`, `calibrate_board(points_px, points_mm)`, `pixel_to_world(x_px, y_px) → (x_mm, y_mm)`, `to_world(keypoints) → FootMeasurement` (converts heel and toe, then computes `angle_deg` in board coordinates), `to_data() → dict`, `from_data(dict)` *(classmethod)*.
- **`CalibrationStore`** — Purpose: saves and loads calibration data on disk, so `CalibrationManager` never touches files. Fields: `file_path`. Methods: `save(calibration_data)`, `load() → calibration_data`.

**Capture & event logic**

- **`CameraCapture`** — Purpose: owns the camera through Picamera2 — the only class that talks to the hardware. Fields: `resolution` (1440×1080), `frame_rate`, `exposure_us` (short, to freeze motion). Methods: `start()`, `stop()`, `get_frame() → Frame`.
- **`AttemptEventDetector`** — Purpose: watches the per-frame measurements and decides when one attempt happened, keeping the last frame with both feet in contact. Fields: `zone_mm` (takeoff-zone rectangle on the board), `presence_frames` (frames both feet must be in the zone to start tracking), `absence_frames` (frames gone to end the attempt), `state` (idle/tracking), `last_contact`. Methods: `process(frame, keypoints, measurements) → Optional[AttemptTrigger]`.
- **`ThumbnailGenerator`** — Purpose: draws a reviewable image per attempt so a coach can sanity-check a reading instead of trusting a bare number. Fields: `overlay_style`, `output_dir`. Methods: `generate(trigger) → thumbnail_path` (draws heel/toe points and each foot's line in pixels, labels the angles).

**Storage & sync**

- **`LocalBuffer`** — Purpose: keeps attempt records on local disk (SQLite) so nothing is lost if Wi-Fi drops, until they're confirmed uploaded. Fields: `db_path`. Methods: `save(record)`, `get_unsynced(limit) → list[AttemptRecord]`, `mark_synced(attempt_id)`.
- **`CloudSyncClient`** — Purpose: the only class that talks to Supabase. Fields: `supabase_url`, `supabase_key` (restricted key from `.env`). Methods: `is_online() → bool`, `upload_thumbnail(path) → url`, `upload(record, thumbnail_url) → bool`, `fetch_active_session() → Optional[SessionContext]`.

**Orchestrator**

- **`CaptureService`** — Purpose: runs the main loop and coordinates every other class through injected dependencies — the one place that knows the full sequence. Fields: `camera`, `detector: FootDetector`, `calibration`, `event_detector`, `thumbnail_generator`, `buffer`, `sync_client` (all injected), `session` (cached `SessionContext` or none), `sync_interval_s`, `session_refresh_s`. Methods: `run()`, `stop()`. Internal steps: `_process_frame`, `_handle_attempt`, and, on a background thread, `_sync_pending` and `_refresh_session`.

### Backend data model (Supabase)

| Entity | Fields |
|---|---|
| **Athlete** | `diver_id, name, foot_length_mm, foot_width_mm, heel_toe_marker_distance_mm, marker_placement_photo_url, created_at` |
| **Session** | `session_id, diver_id, status (active/ended), current_dive_type, started_at, ended_at, location, coach_notes` |
| **CoachTarget** | `target_id, diver_id, dive_type, reference_point (toe/heel), left_target {x_mm, y_mm, angle_deg}, right_target {x_mm, y_mm, angle_deg}, tolerance_radius_mm, tolerance_angle_deg, active_from` |
| **Attempt** | mirror of the Pi's `AttemptRecord`, with `thumbnail_url` instead of a local path; `session_id` null means unassigned |

**DRY note:** `AttemptRecord` (Pi) and `Attempt` (backend) are independently defined mirrors — legitimate, since one holds a local path and the other a URL — so define them from one shared schema deliberately rather than letting them drift.

### Mobile app (client-side)

| Class | Functionality |
|---|---|
| **SessionRepository** | `startSession(diverId, diveType)`, `setDiveType(sessionId, diveType)`, `endSession(sessionId)`, `getActive()` |
| **AttemptRepository** | `fetchHistory(diverId, filters)`, `subscribeLive(sessionId, onUpdate)`, `assign(attemptIds, sessionId)` |
| **AthleteProfileRepository** | `getProfile(diverId)`, `updateProfile(profile)` |
| **CoachTargetRepository** | `getTarget(diverId, diveType)`, `setTarget(target)` |
| **DeviationCalculator** | `computeDeviation(attempt, target) → per foot {deviation_mm, deviation_angle_deg, within_tolerance}` — position measured at the target's reference point (toe or heel); computed **at display time**, not stored |
| **SessionControlViewModel** | Start/end a session, pick diver and dive type, assign unassigned attempts |
| **ConsistencyDashboardViewModel** | `refresh()`, `filterByDiveType()` — feeds position and angle charts; mean deviation, % within tolerance, std dev |
| **LiveSessionViewModel** | Real-time feed during practice, subscribes to new synced attempts |
| **CoachTargetEditorViewModel** | Create/edit target position, angle, tolerances, and reference point per dive type |
| **AthleteProfileViewModel** | View/edit foot dimensions, heel-to-toe marker distance, and marker-placement reference photo |

## Reading List
See `reading-list.md` for the step-by-step reading order.

## Open / Not Yet Decided
- Attempt-detection thresholds (`presence_frames`, `absence_frames`) — tune against real footage.
- Dive type is set by the coach per session for now; automatic detection is deferred.
- Marker durability on wet skin, and glare under pool lighting — test before relying on the markers.
- Lighting: a short global-shutter exposure needs enough light; check the pool's lighting and reflections off the wet board.
- Mount geometry: handrail position vs. a steep viewing angle — measure on site.
- On arrival: PoE HAT + Active Cooler fit, camera cable length, camera housing dimensions.
- Budget: the remaining items may push the total slightly past $600.
- Short video clips per attempt (deferred).
- One shared schema for `AttemptRecord` and `Attempt`.
