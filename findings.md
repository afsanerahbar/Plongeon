# Diving Foot Placement Tracker — Findings

## Project Concept
Foot placement on the board during takeoff matters for consistency and coaching feedback. Goal: a camera system that measures both feet's placement and angle on every attempt, and an app that tracks history and shows consistency against the coach's recommended target.

## Sensing Approach — Evolution

1. **Foot-worn sensors (pressure insoles / IMUs)** — ruled out. They measure pressure/motion but have no way to know *where* the foot is relative to the board.
2. **Board-mounted pressure mat / touch sensor (FSR array)** — ruled out. Lower accuracy, physical wear on the board, and prone to noise from water/texture.
3. **Industrial ultrasonic distance sensor (SICK UC20 / UM18-217, IO-Link)** — considered, then dropped. Reasonable accuracy and good noise-averaging, but the IO-Link interface master is built for Windows/industrial PC software stacks — Raspberry Pi (ARM Linux) driver/SDK support is uncertain, adding real integration risk.
4. **Single-axis laser displacement sensor (Micro-Epsilon optoNCDT, Keyence IL series)** — viable only if forward/back distance from the board tip were enough. Micron-level accuracy over analog or RS-485/Modbus RTU (well-supported on Pi). Limitation: single axis — no side-to-side centering, no angle.
5. **Overhead camera + ArUco fiducial markers (chosen)** — full foot placement is a 2D problem (forward/back **and** side-to-side) plus an angle. A calibrated 2D vision approach suits a flat, known plane like the board better than a 3D depth camera (stereo/ToF depth error is typically 2–5mm). Achievable precision is set by camera resolution — see Accuracy Budget.
6. **Phone camera at 240fps (considered 2026-10-01, not chosen)** — cheaper and no extra hardware, but rolling shutter, not a dedicated device, and it drops most of the embedded-systems work the independent study is built around.

## Related Research: DiveNet (DFKI, 2023)
Paper: *DiveNet: Dive Action Localization and Physical Pose Parameter Extraction for High Performance Training* (Murthy, Taetz, Lekhra, Stricker — IEEE Access 2023).
- Validates the core strategy: single static camera (60Hz) + homography between the board plane and image, computed **per-dive without manual calibration**, extracts physical parameters (center of mass, joint keypoints, dive height) in metric scale.
- Reported accuracy: COM error ~6px, dive peak height sensitivity ~20cm, mean joint angle error ~10°, pose regression ~70% PCK — good for research-grade form analysis, too coarse for a ±1° foot angle.
- **No open-source code or usable dataset found.** A GitHub search led to an unrelated repo (`aim-uofa/Poseur`) — a bad search association, verified and discarded. The dataset host (`divenet.kl.dfki.de`) now redirects to an unrelated DFKI project. The full paper PDF is blocked from automated fetch (403 on DFKI and ResearchGate).
- **Comparing DiveNet's numbers to our task (2026-09-18):** the COM and height numbers come from tracking the whole dive and fitting a projectile curve — harder than our task, so not a fair ceiling. The keypoint numbers (~70% PCK, ~10° joint angle error) are per-frame localization — the same task category as our optional markerless cross-check (Phase 5). Our own bench test gives the real number.

## Requirements (updated 2026-09-25)

- **Both feet** are tracked on every attempt.
- **Position** of each foot on the board: forward/back and side-to-side.
- **Orientation** (toe-pointing angle) of each foot, to within **±1°**.
- **Divers are barefoot**, so anything attached to the foot goes directly on skin.

**Why ±1° is hard:** orientation is derived from two points (heel and toe) by computing the angle of the line between them. For a ~270mm foot, 1° of rotation moves the toe sideways relative to the heel by only ~5mm, and reliably detecting that needs each point accurate to roughly 1mm. Noise in two independently-estimated points compounds when they're subtracted. A general-purpose pose model like BlazePose isn't reliable to ~1mm per point from a steep camera angle, so markers are the primary method.

## Accuracy Budget (added 2026-10-01)

- **Resolution:** the Arducam IMX296 is 1440×1080 effective pixels. If the frame covers the front ~1.6m of a standard 4.8m springboard, that's ~1.1mm per pixel along the board. Sub-pixel corner refinement does better than one pixel, but sub-mm precision isn't guaranteed — the bench test decides.
- **Two markers per foot.** One marker per foot fails: a ~25mm marker is only ~23px wide, so a 0.3px corner error is already ~0.7°, and a marker glued 2° off the foot's axis adds a fixed 2° error. With a marker at the heel and one at the base of the big toe, the angle comes from the line between the two marker centers, ~250mm (~230px) apart: a 0.5px error costs ~0.1°, and how each marker is rotated on the skin no longer matters.
- **The angle is computed in board coordinates.** Both points are corrected for lens distortion and mapped to millimeters first; only then is the angle calculated. An angle measured in raw pixels is skewed by perspective.
- **Lens distortion is corrected** with a one-time checkerboard calibration (`cv2.calibrateCamera`) before the board calibration — a homography assumes an undistorted image.
- **Recorded clips must be high quality.** Heavy video compression blurs marker corners and costs accuracy (see Capture).
- **Board deflection:** the board dips under the diver's weight at the measured moment; a steep camera angle and calibrating under a static load keep this error small.
- **Bench accuracy test (Phase 0 exit):** a printed foot template with heel and toe markers, rotated in known 1° steps and moved to known positions. Pass: angle error under 1°, position error within the coach's tolerance.

## Recommended Solution

### Markers: two ArUco markers per foot (decided 2026-10-01)
- **Four markers** from `DICT_4X4_50`: left heel `10`, left toe `11`, right heel `12`, right toe `13`.
- **Placement:** center of the heel and base of the big toe, recorded in the athlete's reference photo so placement is repeatable.
- **Size:** ~20–25mm — the angle comes from the heel–toe line, so markers only need enough pixels to be detected.
- **Material:** waterproof, matte (to avoid glare) printed markers — e.g. waterproof vinyl or temporary-tattoo paper — with a skin-safe adhesive. They must survive a water entry on every attempt and pool chlorine (tested in Phase 0).
- **Output per foot:** heel and toe positions on the board in millimeters, and the foot angle in degrees.

### Camera
- **Arducam IMX296 color global-shutter module** with its bundled M12 lens (~45–55° horizontal FOV) and Pi 5 cable. Global shutter keeps the fast-moving foot sharp.
- **Pi 5 setup on Bookworm:** in `/boot/firmware/config.txt`, set `camera_auto_detect=0` and add `dtoverlay=imx296` under `[all]` (append `,cam0` if using the CAM0 port).
- The Arducam 90° M12 lens is optional — buy it only if Phase 0 shows the bundled lens can't fit the takeoff zone.

### Capture: video, not single pictures
- The camera records a continuous stream at ~60 frames per second. The foot is planted for only a fraction of a second, and there's no way to fire a single photo at exactly that moment; at 60fps the system gets a few dozen planted frames to choose from.
- **Offline mode (Phases 1–2):** the Pi saves a few-second clip of each attempt during practice and analyzes the clips afterward.
- **Live mode (Phase 3+):** frames are analyzed as they arrive and discarded; each attempt keeps one still image (the last-contact frame, with measurements drawn on).
- **The Pi 5 has no hardware video encoder**, unlike earlier Pis. Plan: keep the last few seconds of raw frames in memory (8GB RAM holds several seconds at 1440×1080), and when an attempt ends, write that clip at high quality. Recording a whole practice continuously is likely too much; Phase 0 confirms the approach.

### Calibration
- **Lens (one-time):** ~15–20 photos of a printed checkerboard at different angles → `camera_matrix` and `dist_coeffs`.
- **Board:** reference markers at known board positions → homography from pixels to board millimeters. Use temporary markers, or place them on the board's side edges — never on the non-slip walking surface during dives. Calibrate under a static test load (someone standing on the board).
- **Recalibrate the board whenever the camera moves** — for the temporary mount in Phases 0–2, that means at the start of every session.

## Project Breakdown (risk-first, offline-first — adopted 2026-10-01)

Answer the riskiest questions first (do markers survive water? is ±1° achievable at the pool? does the mount work?), deliver something the coach can use early, and add live results and the mobile app only once the measurement is proven. Each phase ends with a usable result and an exit test.

| Phase | What gets built | Exit test |
|---|---|---|
| **0. Feasibility** (1–2 weeks) | Pi + camera + printed markers on a temporary mount. Record raw clips at the bench and in one pool session; analyze them in a Jupyter notebook on a laptop. | Bench: angle error under 1°, position within tolerance. Pool: markers survive 10+ water entries; markers detected in pool lighting on nearly every planted frame; the mount shows the whole takeoff zone at a steep angle. |
| **1. Offline pipeline** | The Pi records a short clip per attempt during practice. After practice, the clips run through the measurement pipeline; results go to a CSV plus thumbnails and a one-page session report. | The coach gets per-attempt numbers after each practice; repeated readings of the same planted foot agree within 0.5°. |
| **2. Cloud + simple dashboard** | Supabase tables; results upload after processing; a simple web dashboard (Streamlit) shows history and consistency against coach targets; the coach edits targets there. | The coach views trends from anywhere and sets targets. |
| **3. Real-time on the Pi** | The same pipeline on the live camera; results upload during practice; permanent handrail mount with PoE power; runs as a `systemd` service. The dashboard gains Start/End session. | Results appear within seconds of each dive; runs a whole practice unattended. |
| **4. Mobile app** | React Native app: live view, session control, target editor, history. | The full product. |
| **5. Optional: markerless** | BlazePose cross-check, classical-CV orientation, or a custom keypoint model trained on clips collected in Phases 0–3. | Accuracy close to the markers, without markers. |

**What this changes:** the coach gets results after practice rather than live until Phase 3. In return, the hard questions are answered in the first two weeks, each phase gives the independent study a working deliverable, and the remaining purchases wait until Phase 0 shows they're needed.

## Athlete Profile
Lightweight per-athlete profile (a local file in Phase 1, Supabase from Phase 2):
- **Foot length and width.**
- **Heel-to-toe marker distance**, measured once. It should match every reading, so a different measured distance flags a probable bad detection.
- **Reference photo of marker placement** (both markers on both feet), so markers go in the same spot every session.
- **Scope:** no 3D scan or shape model.

## Mounting & Power

### Phases 0–2: temporary setup
- **Mount:** a temporary clamp on the handrail, or a tall stand beside the board, set up and removed each practice — **with pool and coach approval**, and clear of the diver's walk-out, hurdle, and arm swing.
- **Power:** a USB-C power bank (5V/3A, Power Delivery). The Pi 5 runs fine at 3A; it only limits current to USB accessories, which this setup doesn't use. No wiring near the water, no installation.
- **The Pi sits next to the camera** in its enclosure: the bundled camera cable is short (check its length on arrival), and camera-cable runs beyond ~1m become unreliable.

### Phase 3: permanent installation
- **Mount** on the board's **handrails** (bolted to the fixed stand, so they don't move with the diver's bounce) with a rigid boom near the **fulcrum end**, aimed down at the **front third of the board**. Keep the angle as **steep/overhead as possible** — a shallow angle turns the board's dip into position error. Handrails sit toward the fixed end, so a steep view may need a longer boom; Phase 0 tells us.
- **Power: PoE.** An **802.3at (PoE+)** injector plugs into a GFCI outlet away from the pool edge; one outdoor-rated Ethernet cable (up to 100m) carries power to the PoE HAT on the Pi, so only low-voltage DC reaches the deck. Plain 802.3af can't power a Pi 5. Data can go over Wi-Fi or the same cable if the facility allows a wired port. Don't connect USB-C power while on PoE.
- **Check on arrival:** the PoE HAT fits alongside the Active Cooler and inside the Sixfab enclosure. The HAT is rated 25W vs. the Pi 5's 27W recommendation — fine for this load.
- Camera and Pi both need **weatherproof enclosures** (splash, humidity, chlorine).
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
| PoE HAT for Raspberry Pi 5/CM5 (802.3af/at) | $22.07 (not needed until Phase 3) |
| Sixfab Outdoor IP65 Project Enclosure | $75.00 |
| Arducam IMX296 Color Global Shutter Camera, M12 lens + Pi 5 cable included | $94.99 (was $84.99; low stock — buy first) |

**Total ordered: $419.91**

⚠️ *Do NOT buy the "6mm Wide Angle Lens for Raspberry Pi HQ Camera CS" ($34.00) — wrong mount (CS, not M12) and redundant with the Arducam's bundled lens. Skip any cart containing the Raspberry Pi HQ Camera (PID 4561): it's a rolling-shutter camera.*

**After the microSD arrives:** reflash it with Raspberry Pi OS **Bookworm** via Raspberry Pi Imager — the pre-loaded image is likely Trixie.

### Phase 0 (buy now)
| Item | Est. price | Note |
|---|---|---|
| Foot-marker material + skin-safe adhesive | $10–20 | waterproof, matte |
| USB-C power bank, 5V/3A Power Delivery | $25–40 | borrow one if possible |
| Temporary mount (clamp or tall stand) | $0–30 | borrow if possible |
| Checkerboard + markers | free | print |

### After Phase 0 passes
| Item | Est. price | Note |
|---|---|---|
| Articulating boom/clamp arm | $30–60 | sized from the Phase 0 mount test |
| Weatherproof camera housing | $20–40 | sized from the real camera + lens |
| Arducam 90° Wide Angle M12 Lens | $17.99 | only if the bundled lens's FOV is too narrow |

### Phase 3
| Item | Est. price | Note |
|---|---|---|
| PoE+ injector (802.3at, ≥30W) | $25–35 | search `802.3at poe+ injector` |
| Outdoor-rated Ethernet cable (Cat6) | $20–40 | measure the run to the GFCI outlet first |

**Budget:** Phases 0–2 come to about **$505–630** against the $600 request (borrowing the power bank and mount keeps it under). Phase 3 adds **$45–75**, which can come from a follow-on request once Phases 0–2 show results.

### Camera sourcing history
- The official Raspberry Pi Global Shutter Camera was out of stock at 5 of 6 retailers on 2026-09-13 (Adafruit, CanaKit, PiShop, The Pi Hut, Pimoroni; SparkFun backorder only). The Arducam IMX296 uses the same sensor and was in stock.
- Other Arducam options considered: AR0234 (2.3MP color, "Pivariety" driver) and OV9782 (1MP, but that listing is USB, not CSI). Monochrome sensors were avoided because the Phase 5 BlazePose cross-check needs color.
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

## Data Pipeline

### The measurement pipeline (same in every phase)
1. Get a frame — from a recorded clip (offline) or the live camera.
2. Detect the four markers → heel and toe points per foot, in pixels.
3. Correct for lens distortion, map to board millimeters, compute each foot's angle.
4. Track the takeoff zone (defined in board millimeters): both feet present, then gone = one attempt. The last frame with both feet planted is the reading.
5. Draw a thumbnail and store the attempt record locally.

### Offline mode (Phases 1–2)
- **Before practice:** `record.py --diver <id> --dive <type>` creates a session folder with the session details; the operator runs it again if the dive type changes (or fixes dive types in the report afterward).
- **During practice:** the Pi records a short clip of each attempt into the session folder.
- **After practice:** `process_clips.py` runs every clip through the pipeline, writes a CSV, thumbnails, and a one-page session report, and (Phase 2) uploads results to Supabase.
- **Clips are deleted after processing by default**; they're kept only with athlete consent, to build a dataset for Phase 5.

### Live mode (Phase 3+)
- `main.py` runs the same pipeline on the live camera, so results reach the cloud within seconds.
- **Session context:** the coach taps Start session in the dashboard (Phase 3) or app (Phase 4), choosing diver and dive type; this marks a Session row active in Supabase. A background worker on the Pi reads it every few seconds and caches it; each attempt is tagged with the cached session. Offline, the Pi keeps the last session it saw. With no active session, attempts are saved as unassigned and the coach assigns them later.

### Per-attempt record
`attempt_id, captured_at, session_id, diver_id, dive_type` (null = unassigned), `left_foot {heel_x_mm, heel_y_mm, toe_x_mm, toe_y_mm, angle_deg, confidence, source}`, `right_foot {…}`, thumbnail. The record holds **no coach targets or deviations** — the Pi only reports where the feet were.

## Compute Responsibility Split
Principle: **Pi senses, Cloud stores, Dashboard/App computes and presents.**

- **Raspberry Pi:** recording, marker detection, lens correction + homography, foot angles, attempt detection, thumbnails, local storage, uploads, reading the active session (Phase 3).
- **Cloud (Supabase, Phase 2+):** passive store — attempts, sessions, athletes, coach targets, thumbnails. No image processing or math.
- **Dashboard (Phase 2–3) and App (Phase 4):** compute deviation from target **at display time** (not stored), so changing a target recalculates history correctly; own all charts, target editing, and session control.

## Stack
- **Pi:** Python on Raspberry Pi OS **Bookworm** — OpenCV, Picamera2, SQLite, `supabase-py`; MediaPipe only for the Phase 5 cross-check (it doesn't install on Trixie).
- **Phase 0 analysis:** Jupyter notebook on a laptop.
- **Backend (Phase 2+):** Supabase (Postgres) — database, file storage, realtime updates for the app.
- **Dashboard (Phase 2–3):** Streamlit — a web dashboard written in Python, so it uses the language already learned for the Pi.
- **App (Phase 4):** React Native (TypeScript), cross-platform — chosen over Flutter for its larger beginner ecosystem and Supabase's JavaScript client.

## Prerequisites to Learn, by Phase
- **Phase 0:** headless Pi setup (flash **Bookworm**, SSH enabled); basic Linux command line; camera setup (`config.txt` lines, `rpicam-hello`, Picamera2 recording); Python `venv`; Jupyter; OpenCV ArUco detection, lens calibration, and homography.
- **Phase 1:** Python classes and abstract base classes (for the interfaces); SQLite.
- **Phase 2:** Supabase and `supabase-py`; Streamlit.
- **Phase 3:** `systemd` services and `journalctl`; Tailscale for remote access.
- **Phase 4:** React Native and Supabase's JavaScript client.
- **Phase 5:** MediaPipe, classical CV segmentation.
- See `reading-list.md` for links, ordered by phase.

## Security & Privacy
- **Supabase key on the Pi:** a restricted key with row-level security that only allows inserting attempts, uploading thumbnails, and reading the active session — never the service-role key. It lives in a `.env` file listed in `.gitignore`; never commit it (this repo is on GitHub).
- **Consent:** get athlete consent for recording and for storing images in the cloud, and follow the school's policy for athletes under 18. Clips are deleted after processing unless consent covers keeping them.

## Software Architecture (Pi side, revised 2026-10-01)

**Design principles:**
- Three interfaces let the same code run in every phase: `FrameSource` (recorded clip or live camera), `FootDetector` (markers, later markerless), and `SessionProvider` (local session file or the cloud).
- Detectors return **pixel** coordinates; `CalibrationManager.to_world()` converts to **board millimeters** and computes the angle there.
- `CaptureService` has no network code. Uploads and session refresh happen in `SyncWorker`, on its own thread, so they never stall capture.
- Dependencies are passed in through constructors (SOLID review of 2026-09-25), so each class can be tested alone.

**Visual design doc:** class diagram, call sequences, and phase plan — `pi-side-design.html` in this repo, published at https://claude.ai/artifact/WgcJtuM2NdzgV3FbFQs9vv

### Entry scripts
| Script | Phase | What it does |
|---|---|---|
| `calibrate.py` | 0–1 | Lens calibration from checkerboard photos, board calibration from reference markers, saved through `CalibrationStore`. |
| `record.py` | 1 | Creates a session folder with the diver and dive type, then runs `ClipRecorder` during practice. |
| `process_clips.py` | 1 | After practice: runs each clip through `CaptureService` (with `VideoFileSource` + `FixedSession`), then `ReportWriter`; in Phase 2 also `SyncWorker.sync_once()`. |
| `main.py` | 3 | Live: runs `CaptureService` with `CameraCapture` + `CloudSyncClient`, and starts `SyncWorker`. Run by `systemd`. |

### Classes by phase
| Phase | Classes |
|---|---|
| 0 | None yet — notebook prototypes of recording, marker detection, calibration, and angle math. What passes becomes the Phase 1 classes. |
| 1 | `Frame`, `FrameSource`, `CameraCapture`, `VideoFileSource`, `ClipRecorder`, `FootDetector`, `MarkerDetector`, `FootKeypoints`, `FootMeasurement`, `CalibrationManager`, `CalibrationStore`, `AttemptEventDetector`, `AttemptTrigger`, `ThumbnailGenerator`, `SessionContext`, `SessionProvider`, `FixedSession`, `AttemptRecord`, `LocalBuffer`, `ReportWriter`, `CaptureService` |
| 2 | `CloudSyncClient`, `SyncWorker` |
| 3 | No new classes — `main.py` wires the live versions together |
| 5 | `PoseDetector` (and any other markerless detector) |

### Pi-Side Class Spec — Draft (24 classes)
*Field/method-level detail with a one-line purpose statement per class. Subject to change once implementation starts.*

**Data objects (no behavior)**
- **`Frame`** — Purpose: one image and when it was taken. Fields: `image`, `captured_at`.
- **`FootKeypoints`** — Purpose: one foot's heel and toe points in image pixels — what every detector returns. Fields: `foot_side`, `heel_px {x, y}`, `toe_px {x, y}`, `confidence`, `source` (marker/pose).
- **`FootMeasurement`** — Purpose: one foot's heel, toe, and angle in board millimeters — what everything after calibration uses. Fields: `foot_side`, `heel_mm {x, y}`, `toe_mm {x, y}`, `angle_deg`, `confidence`, `source`.
- **`AttemptTrigger`** — Purpose: hands over one detected attempt: the last-contact frame with both feet in pixels and in millimeters. Fields: `frame`, `keypoints {left, right}`, `measurements {left, right}`.
- **`SessionContext`** — Purpose: who is diving and which dive. Fields: `session_id`, `diver_id`, `dive_type`.
- **`AttemptRecord`** — Purpose: one completed attempt, ready to store or upload. Fields: `attempt_id`, `captured_at`, `session_id`, `diver_id`, `dive_type` (null when unassigned), `left_foot`, `right_foot` (`FootMeasurement`), `thumbnail_path`. Methods: `to_dict()`.

**Interfaces**
- **`FrameSource`** — Purpose: one way to get frames, recorded or live, so the same pipeline runs on both. Methods: `start()`, `read() → Optional[Frame]` (nothing when a clip ends), `stop()`.
- **`FootDetector`** — Purpose: one contract for every detection method. Methods: `detect(frame) → list[FootKeypoints]` — an empty list when no foot is found, never an error, so implementations are interchangeable.
- **`SessionProvider`** — Purpose: one way to learn the current session, from a local file or the cloud. Methods: `current_session() → Optional[SessionContext]` — returns immediately, never waits on the network.

**Implementations**
- **`CameraCapture`** *(FrameSource; Phases 0, 1, 3)* — Purpose: the only class that talks to the camera, through Picamera2. Fields: `resolution` (1440×1080), `frame_rate`, `exposure_us` (short, to freeze motion).
- **`VideoFileSource`** *(FrameSource; Phase 1)* — Purpose: plays back a recorded clip as frames with their original timestamps. Fields: `path`.
- **`MarkerDetector`** *(FootDetector; primary)* — Purpose: finds the four markers and reports each foot's heel and toe points. Fields: `dictionary`, `marker_ids` (`{left: {heel: 10, toe: 11}, right: {heel: 12, toe: 13}}`), `detector_params` (sub-pixel corner refinement on). Each marker's center (mean of its four refined corners) becomes the heel or toe point; a foot is reported only when both of its markers are found.
  - **`dictionary`:** the ArUco marker dictionary (`cv2.aruco`) — a predefined family of binary square patterns, e.g. `DICT_4X4_50` (4×4-bit grid, 50 unique IDs), not a Python dict. More bits give more IDs and better false-positive rejection but need more pixels per marker; fewer bits let a small marker decode reliably. **Use `DICT_4X4_50`:** markers on skin should be small, and the project needs ~10 IDs (4 foot markers + up to 6 board calibration markers).
- **`PoseDetector`** *(FootDetector; Phase 5)* — Purpose: BlazePose heel/foot_index landmarks as a markerless cross-check. Fields: `model`, `confidence_threshold`.
- **`FixedSession`** *(SessionProvider; Phase 1)* — Purpose: the session read from a session folder, for processing that folder's clips. Fields: `session`.
- **`CloudSyncClient`** *(SessionProvider; Phase 2+)* — Purpose: the only class that talks to Supabase. Fields: `supabase_url`, `supabase_key` (restricted, from `.env`), `cached_session`. Methods: `is_online()`, `upload_thumbnail(path) → url`, `upload(record, thumbnail_url) → bool`, `refresh_session()` (network call, used by `SyncWorker`), `current_session()` (returns the cache, no network).

**Behavior classes**
- **`CalibrationManager`** — Purpose: maps image pixels to board millimeters (lens correction + homography) — pure math, no file I/O. Fields: `camera_matrix`, `dist_coeffs`, `homography_matrix`. Methods: `calibrate_lens(checkerboard_images)`, `calibrate_board(points_px, points_mm)`, `pixel_to_world(x_px, y_px) → (x_mm, y_mm)`, `to_world(keypoints) → FootMeasurement` (converts heel and toe, then computes `angle_deg`), `to_data()`, `from_data(data)` *(classmethod)*.
- **`CalibrationStore`** — Purpose: saves and loads calibration data on disk. Fields: `file_path`. Methods: `save(calibration_data)`, `load()`.
- **`AttemptEventDetector`** — Purpose: decides when one attempt happened and keeps the last frame with both feet planted. Fields: `zone_mm` (takeoff zone on the board), `presence_frames`, `absence_frames`, `state` (idle/tracking), `last_contact`. Methods: `process(frame, keypoints, measurements) → Optional[AttemptTrigger]`.
- **`ThumbnailGenerator`** — Purpose: draws a reviewable image per attempt so the coach can sanity-check a reading. Fields: `overlay_style`, `output_dir`. Methods: `generate(trigger) → thumbnail_path`.
- **`LocalBuffer`** — Purpose: keeps attempt records on local disk (SQLite) so nothing is lost before upload. Fields: `db_path`. Methods: `save(record)`, `get_session(session_id) → list[AttemptRecord]`, `get_unsynced(limit)`, `mark_synced(attempt_id)`.
- **`ClipRecorder`** *(Phase 1)* — Purpose: during practice, saves a short clip of each attempt for analysis afterward. Fields: `camera`, `motion_zone_px`, `motion_threshold`, `pre_seconds`, `post_seconds`, `output_dir`. Methods: `run()`, `stop()`. Keeps the last `pre_seconds` of raw frames in memory; when motion appears in the zone, keeps recording until `post_seconds` after it stops, then writes the clip at high quality. (A pixel zone is fine here: it only decides what to record, not what's measured.)
- **`ReportWriter`** *(Phase 1)* — Purpose: turns a session's attempts into files the coach can read without the app. Fields: `buffer`, `output_dir`. Methods: `write_session_report(session_id)` → a CSV and a one-page HTML summary (foot positions and angles per attempt).
- **`SyncWorker`** *(Phase 2+)* — Purpose: moves results to the cloud without ever slowing capture. Fields: `buffer`, `cloud` (`CloudSyncClient`), `interval_s`. Methods: `sync_once()` (if online: refresh the session cache, upload pending attempts with thumbnails, mark them synced), `start()` (repeats `sync_once` on a background thread), `stop()`.

**Orchestrator**
- **`CaptureService`** — Purpose: runs the measurement pipeline over any frame source — a recorded clip or the live camera — and stores each attempt locally. Fields: `frame_source`, `detector`, `calibration`, `event_detector`, `thumbnail_generator`, `buffer`, `session_provider` (all passed in). Methods: `run()` (until the source ends or `stop()` is called), `stop()`.

### Backend data model (Supabase, Phase 2+)

| Entity | Fields |
|---|---|
| **Athlete** | `diver_id, name, foot_length_mm, foot_width_mm, heel_toe_marker_distance_mm, marker_placement_photo_url, created_at` |
| **Session** | `session_id, diver_id, status (active/ended — used from Phase 3), current_dive_type, started_at, ended_at, location, coach_notes` |
| **CoachTarget** | `target_id, diver_id, dive_type, reference_point (toe/heel), left_target {x_mm, y_mm, angle_deg}, right_target {x_mm, y_mm, angle_deg}, tolerance_radius_mm, tolerance_angle_deg, active_from` |
| **Attempt** | mirror of the Pi's `AttemptRecord`, with `thumbnail_url` instead of a local path; `session_id` null means unassigned |

**DRY note:** define `AttemptRecord` (Pi) and `Attempt` (backend) from one shared schema so they don't drift apart.

### Dashboard (Streamlit, Phase 2–3)
- **History:** attempts by session, date, and dive type, with thumbnails.
- **Consistency:** each foot's position against the coach's target zone, angle deviation per foot, and trends; deviations computed at display time, measured at the target's reference point (toe for forward-facing takeoffs, heel for backward).
- **Targets:** per diver, per dive type, per foot — position, angle, tolerances, reference point.
- **Sessions (Phase 3):** Start/End session, choose diver and dive type, assign unassigned attempts.

### Mobile app (React Native, Phase 4)

| Class | Functionality |
|---|---|
| **SessionRepository** | `startSession(diverId, diveType)`, `setDiveType(sessionId, diveType)`, `endSession(sessionId)`, `getActive()` |
| **AttemptRepository** | `fetchHistory(diverId, filters)`, `subscribeLive(sessionId, onUpdate)`, `assign(attemptIds, sessionId)` |
| **AthleteProfileRepository** | `getProfile(diverId)`, `updateProfile(profile)` |
| **CoachTargetRepository** | `getTarget(diverId, diveType)`, `setTarget(target)` |
| **DeviationCalculator** | `computeDeviation(attempt, target) → per foot {deviation_mm, deviation_angle_deg, within_tolerance}`, at display time |
| **SessionControlViewModel** | Start/end a session, pick diver and dive type, assign unassigned attempts |
| **ConsistencyDashboardViewModel** | Position and angle charts; mean deviation, % within tolerance, std dev |
| **LiveSessionViewModel** | Real-time feed during practice |
| **CoachTargetEditorViewModel** | Edit target position, angle, tolerances, and reference point per dive type |
| **AthleteProfileViewModel** | Edit foot dimensions, heel-to-toe marker distance, and the marker-placement photo |

## Open / Not Yet Decided
- **Phase 0 answers:** marker durability on wet skin, glare under pool lighting, the mount's viewing angle, clip recording and encoding on the Pi 5.
- Attempt-detection thresholds (`presence_frames`, `absence_frames`, motion trigger) — tune on Phase 0–1 footage.
- Dive type is entered per session for now; automatic detection is deferred.
- On arrival: camera cable length, PoE HAT + Active Cooler fit, camera housing dimensions.
- Budget for Phase 3 (follow-on request).
- One shared schema for `AttemptRecord` and `Attempt`.
