# Diving Foot Placement Tracker — Findings

## Project Concept
Charles is a varsity diver. Foot placement on the board during takeoff matters for consistency and coaching feedback. Goal: sensor hardware that measures foot placement per attempt, syncing to a mobile app that tracks history and shows consistency against the coach's recommended target.

## Sensing Approach — Evolution

1. **Foot-worn sensors (pressure insoles / IMUs)** — ruled out. They measure pressure/motion but have no way to know *where* the foot is relative to the board.
2. **Board-mounted pressure mat / touch sensor (FSR array)** — ruled out. Lower accuracy, physical wear on the board, and prone to noise from water/texture.
3. **Industrial ultrasonic distance sensor (SICK UC20 / UM18-217, IO-Link)** — considered, then dropped. Reasonable accuracy and good noise-averaging (beam cone smooths out board texture/water droplets), but the IO-Link interface master is built for Windows/industrial PC software stacks — Raspberry Pi (ARM Linux) driver/SDK support is uncertain, adding real integration risk.
4. **Single-axis laser displacement sensor (Micro-Epsilon optoNCDT, Keyence IL series)** — viable fallback if only forward/back distance from the board tip matters. Micron-level accuracy, avoids IO-Link risk by using analog or RS-485/Modbus RTU output instead (well-supported on Pi via `pymodbus` or an ADC HAT). Limitation: single axis only — no side-to-side centering data.
5. **Overhead camera + ArUco fiducial marker** — selected because full foot placement is a 2D problem (forward/back **and** side-to-side centering), which a coach actually critiques. A calibrated 2D vision approach outperforms 3D depth cameras here because the board is a flat, known plane — homography calibration + fiducial marker gives sub-mm precision, better than typical stereo/ToF depth accuracy (2–5mm).
6. **Markerless pivot (current direction)** — decided to drop the marker on the diver in favor of a phased approach (see below), keeping only fixed calibration markers on the board.

## Related Research: DiveNet (DFKI, 2023)
Paper: *DiveNet: Dive Action Localization and Physical Pose Parameter Extraction for High Performance Training* (Murthy, Taetz, Lekhra, Stricker — IEEE Access 2023).
- Validates the core strategy: single static camera (60Hz) + homography between the board plane and image, computed **per-dive without manual calibration**, extracts physical parameters (center of mass, joint keypoints, dive height) in metric scale.
- Reported accuracy: COM error ~6px, dive peak height sensitivity ~20cm, mean joint angle error ~10°, pose regression ~70% PCK — i.e. good for research-grade action/form analysis, but coarser than the sub-mm precision our ArUco-marker approach targets.
- **No open-source code or usable dataset found.** GitHub search led to an unrelated repo (`aim-uofa/Poseur`) with no connection to DiveNet — that was a bad/hallucinated search association, verified and discarded. The dataset host (`divenet.kl.dfki.de`) now redirects to an unrelated DFKI project; the dataset appears dead/inaccessible as of 2026-09. Full paper PDF is blocked from automated fetch (403 on DFKI and ResearchGate).
- Takeaway: can't reuse DiveNet's code/data directly, but its per-dive automatic homography idea and its published accuracy numbers are a useful reference point for our own markerless phase.
- **Important nuance on comparing DiveNet's numbers to our task (2026-09-18):** DiveNet's numbers split into two different kinds of task, and only one transfers cleanly to us.
  - *COM trajectory (~6px) and peak height (~20cm sensitivity)* come from tracking across the **entire dive** (approach through flight) and fitting a projectile-motion curve to it — a multi-frame, compounding-error task, harder than ours. These numbers are **not** a fair ceiling for our accuracy.
  - *2D joint keypoints (~70% PCK, ~10° mean joint angle error)* is **per-frame keypoint localization** — the same underlying task category we depend on (finding heel/foot_index in one well-defined frame). This is the more directly relevant number, though we should still expect to do better: we control calibration and camera placement (fixed, close, purpose-mounted) far more tightly than DiveNet's more general capture setup. The Phase 1 validation test (below) will give our actual number rather than borrowing DiveNet's.

## Requirement Update (2026-09-25): Both Feet + Orientation (±1°)

- **Both feet must be tracked**, not just one — a relatively easy addition, since BlazePose already reports landmarks for both feet in one frame, and marker-based tracking just means two markers (one per foot) instead of one.
- **Foot orientation (toe-pointing angle) matters, to within 1°** — a much harder requirement than position alone, and it changes which method should be primary.
  - **Why 1° is hard:** orientation isn't measured directly — it's derived from two points (e.g., heel and toe), by computing the angle between them. For a ~270mm-long foot, resolving 1° of rotation means detecting the toe point shifting sideways relative to the heel by only ~5mm — and reliably detecting a 1° change needs the underlying point measurements accurate to roughly 1mm. Two independently-estimated points each carrying their own noise compound when subtracted to get an angle.
  - **This rules out markerless BlazePose landmarks as the primary method for orientation** — a general-purpose human pose model isn't reliable to ~1mm per point in a steep, oblique camera angle, so the derived angle would be noisy well beyond ±1°. Fine for a rough position starting point, not for this angular tolerance.
  - **⚠️ Correction: divers are barefoot, not shod.** Earlier phases in this doc describe markers "on the shoe/heel" — competitive diving is performed barefoot, so any physical marker needs to go directly on bare skin (small waterproof adhesive tags at a fixed anatomical point, e.g., base of the big toe and the heel) — the same technique used for skin-mounted motion-capture markers in sports biomechanics. Feasible, just a different application method than a shoe sticker.

### Two approaches that can actually hit 1° — both worth pursuing
1. **ArUco marker per foot (2 markers, one per foot, on bare skin)** — elevated from "Phase 2 fallback" to the **primary recommended method**, specifically because of the orientation requirement. A detected ArUco marker's 4 corners give **position AND rotation together** in one reading, with sub-degree accuracy achievable via standard corner-based pose estimation (well-established in robotics/AR tracking) — a fundamentally better fit for an angle requirement than deriving an angle from two separately-estimated landmark points.
2. **Classical CV: fit an ellipse/principal axis to each foot's silhouette** (via background subtraction + `cv2.fitEllipse` or PCA on the segmented blob) — averages over many contributing edge pixels rather than two noisy point estimates, which can give stable orientation. Worth running as a markerless cross-check alongside the marker approach, not as a replacement for it.

## Athlete Profile (added 2026-09-25)
Lightweight per-athlete profile, needed to support the marker-based approach and data quality generally — not a strict requirement for the core measurement to function, but genuinely useful:
- **Foot length and width** — enables a personal offset correction (detected marker/landmark position → true contact edge) and outlier rejection (a wildly different implied foot size flags a bad detection).
- **Reference photo of marker placement** — keeps skin-marker application consistent session to session, so measured inconsistency reflects the diver's actual foot placement, not drift in where the marker was glued that day.
- **Scope:** just length, width, and a placement reference photo — no full 3D scan or shape model needed.
- **Data model note:** the per-attempt record schema (see Data Pipeline section below) now includes a `diver_id` field — the athlete profile is a natural extension of that, and sets up clean support for multiple athletes later without a redesign.

## Recommended Solution: Overhead Global Shutter Camera, Phased Marker Strategy

*(Note: given the orientation requirement above, treat the marker-based approach — described here as "Phase 2" — as the primary method, not a last-resort fallback. Phase 1/1.5 markerless approaches remain useful for position-only validation and cross-checking.)*

**Phase 1 (start here) — markerless:**
- **Camera:** Raspberry Pi Global Shutter Camera — global shutter is essential to avoid motion blur/distortion from the fast-moving foot at takeoff (rolling shutter cameras would smear the reading).
- **Foot detection:** No marker on the diver. Need **heel and toe** positions, not just ankle — MoveNet's COCO-17 keypoints only include ankle, but **MediaPipe BlazePose's 33-point landmark set includes explicit `heel` and `foot_index` (near-big-toe) landmarks** for both feet. This means deliberately flashing **Raspberry Pi OS Bookworm (Python 3.11)**, not the newest Trixie, specifically to get a working MediaPipe install (see compatibility note above).
- **Calibration:** Fixed ArUco/checkerboard reference markers placed permanently **on the board only** (e.g. small painted dots or tape marks at known points), used to compute the homography (pixel → real-world mm). Calibrate under a static test load (someone standing on the board), since the board deflects several inches under the diver's weight during the exact moment being measured.
- **Validation step before trusting this phase:** place a foot (or marked test object) at several known board positions, compare BlazePose's detected heel/toe positions to ground truth, and check the error against the coach's actual tolerance. BlazePose is trained on general human pose imagery, not this specific steep/oblique board-camera angle, so its accuracy here must be measured, not assumed.

**Phase 1.5 (markerless refinement, if BlazePose's foot landmarks aren't precise enough) — two alternative approaches:**
- **Option A — Classical CV foot segmentation:** since the camera view and board background are fixed/known, background-subtract the static board texture to isolate the foot silhouette, then geometrically compute the true toe-tip and heel-back extreme points along the board's length axis. Purpose-built for this one controlled scene, likely more precise than a general-purpose pose model's landmarks. No training data needed.
- **Option B — Custom-trained keypoint model (added 2026-09-18):** fine-tune a small keypoint-detection model specifically on our own board/camera setup, rather than relying on BlazePose's general-purpose human-pose training data. Same underlying logic as Option A (purpose-built beats general-purpose for a narrow, controlled task), but via supervised fine-tuning instead of hand-engineered CV heuristics.
  - Requires collecting labeled training data: photos/frames of a foot at various known board positions, each hand-annotated with the true heel/toe pixel location. A few hundred labeled images (fine-tuning from a pretrained backbone, not training from scratch) is a realistic scope.
  - **Training and inference happen on different hardware.** Fine-tuning benefits from a GPU (a laptop or cloud GPU) — not the Pi. Only the final, small exported model runs (inference-only) on the Pi.
  - This is really a **vision/keypoint model**, not an "LLM" in the generative-text sense — worth keeping the terminology straight since it determines which tooling applies (e.g. TensorFlow/PyTorch fine-tuning workflows, not language-model fine-tuning).
  - Legitimate independent-study extension in its own right (data collection, labeling, fine-tuning, deployment is a real curriculum arc), not just an accuracy trick — worth documenting in the funding narrative if Charles wants to take this on.

**Phase 2 (now the primary method, given the ±1° orientation requirement):**
- Two ArUco markers, one per foot, applied directly to bare skin (waterproof adhesive, fixed anatomical placement per the athlete profile above) — not shoe stickers, since diving is barefoot.
- Gives position **and** orientation together per foot, sub-pixel/sub-mm position and sub-degree rotation via OpenCV's ArUco corner detection.
- Camera, mount, and board calibration markers are unchanged from the original plan — this is a change to the foot-detection step, not the camera/mounting design.

- **Output:** per foot, real-world (X, Y) position in millimeters **and** orientation in degrees, both directly comparable to the coach's target position/angle + tolerance.

### Mounting
- Mount on the board's **handrails**, not the board plank itself — handrails are bolted to the fixed stand, not the flexing plank, so they don't move with the diver's bounce.
- Use a rigid boom/clamp arm extending from the handrail near the **fulcrum (fixed) end**, angled to look down at the **front third of the board** (the takeoff zone).
- Keep the camera angle as **steep/overhead as possible** — a shallow oblique angle turns the board's vertical deflection under load into horizontal position error (parallax); a steep downward angle minimizes this.
- Camera and Pi both need **weatherproof enclosures** (splash/humidity/chlorine protection).
- Avoid running mains power near the pool deck without a GFCI circuit and an electrician; PoE or battery power is the safer DIY path.

## Equipment Lists

### Solution A — Overhead Camera, Phased Marker Strategy (Recommended)
| Item | Purpose | Approx. Price | Phase |
|---|---|---|---|
| Raspberry Pi 5 (8GB) | Runs pose model/OpenCV processing + app sync | $200 (was $80 at launch &mdash; risen due to a 2026 memory/LPDDR4 shortage; verify current price before ordering) | 1 & 2 |
| Raspberry Pi Global Shutter Camera module (bare sensor, CS mount) | Motion-blur-free capture | $50 (stock varies by retailer — check PiShop, SparkFun, CanaKit, Pimoroni, Micro Center if one is out of stock; Arducam OV9782/AR0234 *color* global-shutter modules are CSI-compatible alternatives — avoid Arducam's mono-only sensors like OV9281/OV7251/OV2311, since pose models expect RGB input) | 1 & 2 |
| CS-mount lens | **Not included with the camera module** — required separately, or buy CanaKit's bundled kit (module + telescopic + wide-angle lens) instead of the bare module | $20–40 (or bundled) | 1 & 2 |
| CSI camera cable (1–2m) | Reaches boom-mounted camera | $10–15 | 1 & 2 |
| Articulating boom/clamp arm | Mounts camera to handrail at steep angle | $30–60 | 1 & 2 |
| Weatherproof camera housing w/ clear window | Splash/chlorine protection | $20–40 | 1 & 2 |
| Weatherproof Pi enclosure (vented, cable glands) | Protects Pi electronics | $25–40 | 1 & 2 |
| MicroSD card (32–64GB) | Pi OS + storage | $10–15 | 1 & 2 |
| Pi power supply (5V/USB-C) | Power | $10–15 | 1 & 2 |
| Laminated ArUco/checkerboard calibration markers (for board, fixed) | Homography calibration | $5–10 | 1 & 2 |
| MediaPipe Pose or MoveNet (software, no cost) | Markerless foot/ankle keypoint detection | $0 | 1 |
| Laminated ArUco marker stickers (for shoe) | Precise tracking target, added only if Phase 1 accuracy is insufficient | $5–10 | 2 (fallback) |
| PoE HAT + PoE injector *(optional)* | Safer power delivery | $25–30 | 1 & 2 |
| Heatsink/fan case *(optional)* | Cooling under continuous processing | $10–15 | 1 & 2 |

## FINAL Purchase Plan (confirmed 2026-09-14)

**PiShop order — $227.85**
| Item | Price |
|---|---|
| Official microSD 64GB w/ Raspberry Pi OS 64-bit | $29.95 |
| Raspberry Pi 5/8GB + 27W USB-C PSU + Active Cooler bundle | $197.90 |

⚠️ *Do NOT buy the "6mm Wide Angle Lens for Raspberry Pi HQ Camera CS" ($34.00) — wrong mount (CS, not M12), redundant with the Arducam camera's bundled lens, and belongs to the rolling-shutter HQ Camera product line we ruled out.*

**Amazon order — $182.06**
| Item | Price |
|---|---|
| PoE HAT for Raspberry Pi 5/CM5 (802.3af/at) | $22.07 |
| Sixfab Outdoor IP65 Project Enclosure | $75.00 |
| Arducam IMX296 Color Global Shutter Camera, M12 lens + Pi 5 cable included | $84.99 |

**Confirmed total: $409.91** (well under the $600 funding request)

⚠️ **Buy the Arducam camera first** — was down to "Only 1 left in stock" when last checked; this camera line has been out of stock at 5 of 6 other retailers checked.

**After the OS card arrives:** reflash it with Raspberry Pi OS **Bookworm** (not the pre-loaded default, likely Trixie) via Raspberry Pi Imager — required for MediaPipe compatibility (see reading list / prerequisites above).

**Still outstanding (not yet purchased):**
- Weatherproof camera housing — source once camera+lens physical dimensions are known
- Articulating boom/clamp arm — measure handrail diameter first
- Arducam 90° Wide Angle M12 Lens ($17.99, uctronics.com/arducam.com) — **hold off**; only buy if testing shows the bundled ~45–55° FOV lens doesn't fit the whole takeoff zone in frame at actual mounting distance
- ArUco/checkerboard calibration markers — DIY printed, free

## Purchase Plan (Two Orders) — superseded by FINAL plan above, kept for reference

### Order 1 — PiShop.us (official Raspberry Pi reseller)
| Item | Link |
|---|---|
| Raspberry Pi 5 (8GB) | https://www.pishop.us/product/raspberry-pi-5-8gb/ |
| Global Shutter Camera module | ⚠️ **Out of stock at PiShop as of 2026-09-13** — see stock note below |
| 6mm Wide Angle CS-mount lens | https://www.pishop.us/product/6mm-wide-angle-lens-for-raspberry-pi-hq-camera-cs/ |
| Camera Cable for Raspberry Pi 5 (22-pin adapter) | https://www.pishop.us/product/camera-cable-for-raspberry-pi-5/ |
| Official 27W USB-C Power Supply | https://www.pishop.us/product/raspberry-pi-27w-usb-c-power-supply-black-us/ |
| Official Active Cooler | https://www.pishop.us/product/raspberry-pi-active-cooler/ |

### Order 2 — Amazon (mounting/enclosure/PoE gap items)
| Item | Link |
|---|---|
| Sixfab IP65 Outdoor Enclosure (weatherproof Pi housing) | https://www.amazon.com/Outdoor-Enclosure-Raspberry-Development-Boards/dp/B09TRZ5BTB |
| PoE HAT for Pi 5 (5V/5A, 802.3af/at, confirmed Pi 5 compatible) | https://www.amazon.com/Raspberry-Ethernet-at-Compliant-Standard-Compatible/dp/B0D7SDGXKL |
| MicroSD card (32–64GB, A2-rated) | not yet verified — search `sandisk extreme microsd 64gb a2` |
| Weatherproof camera housing | not yet verified — search `weatherproof mini camera housing cs lens` (measure camera+lens dimensions once they arrive) |
| Articulating boom/clamp arm | not yet verified — search `articulating clamp arm mount camera` (measure handrail diameter first) |

### ⚠️ Global Shutter Camera stock pattern — worth tracking as a real risk
Out of stock at **three separate retailers checked so far**: Adafruit (CS Lens Mount variant), CanaKit (bundle kit), and now PiShop (bare module). This looks like a genuine supply constraint on this specific product, not a one-off.
- **Retailers not yet checked for live stock:** The Pi Hut (https://thepihut.com/products/raspberry-pi-global-shutter-camera), SparkFun (https://www.sparkfun.com/raspberry-pi-global-shutter-camera.html), Pimoroni (https://shop.pimoroni.com/en-us/products/raspberry-pi-global-shutter-camera), Micro Center (https://www.microcenter.com/product/663805/raspberry-pi-global-shutter-camera).
- **If all remain out of stock:** fall back to the Arducam **color** global-shutter alternative (OV9782 or AR0234) already documented above — avoid Arducam's mono-only sensors (OV9281/OV7251/OV2311) since the pose model needs RGB input.

### 🚫 Confirmed stock (as of 2026-09-13) — genuine shortage, not a retailer fluke
| Retailer | Status |
|---|---|
| Adafruit (PID 5702) | Out of stock |
| CanaKit (bundle kit) | Sold out |
| PiShop.us | Out of stock |
| The Pi Hut | Out of stock (£48) |
| Pimoroni | Out of stock (£42.50) |
| **SparkFun** | **Backorder** ($63.75) — only live orderable option; ships when restocked, no ETA confirmed |
| Micro Center | Unchecked (site blocked automated access — check manually) |

**5 of 6 checked retailers are out of stock.** Given this pattern, treat an Arducam color global-shutter module as the real primary plan, not just a documented fallback. Avoid Arducam's mono-only sensors (OV9281/OV7251/OV2311, and the innomaker/TOP1 mono listings on Amazon) since the pose model needs RGB input.

### Amazon alternatives (checked 2026-09-13, not caught in the shortage above)
| Option | Link | Notes |
|---|---|---|
| **Arducam IMX296 Color GS Camera (top pick)** | https://www.amazon.com/Arducam-Raspberry-Equipped-15-22pin-Flexible/dp/B0C3VGMTRH | Same sensor as the official RPi module; **includes M12 lens and the Pi 5 FPC cable** — solves two compatibility gaps in one purchase |
| Arducam AR0234, 2.3MP color, wide-angle | https://www.amazon.com/Arducam-Global-Shutter-Raspberry-Pivariety/dp/B09984RJZP | Higher res; uses Arducam's "Pivariety" driver — needs `libcamera` (fine on our Bookworm setup), not the legacy camera stack |
| Arducam OV9782, 1MP color | https://www.amazon.com/Arducam-Shutter-Distortion-Without-Microphones/dp/B0CLXZ29F9 | Lower res; **this listing is USB/UVC, not CSI** — different wiring path than planned |

### Solution B — Intel RealSense Depth Camera (fallback, no markers, less precise)
| Item | Purpose | Approx. Price |
|---|---|---|
| Raspberry Pi 5 (8GB) | Runs `librealsense`/`pyrealsense2` + app sync | $200 (see pricing note above) |
| Intel RealSense D435 (or D455) | Stereo depth camera | $200–350 |
| Articulating boom/clamp arm | Mounts camera to handrail | $30–60 |
| Weatherproof housing w/ IR-transparent window | Must not block IR sensing | $30–50 |
| Weatherproof Pi enclosure | Protects Pi electronics | $25–40 |
| USB 3.0 cable, short run (or active/repeater) | Stable USB3 bandwidth | $15–25 |
| Powered USB hub *(optional)* | Reliable camera power | $15–20 |
| MicroSD card + Pi power supply | Pi OS/storage + power | $20–30 |

### Solution C — Single-Axis Laser Displacement Sensor (forward/back only)
| Item | Purpose | Approx. Price |
|---|---|---|
| Laser triangulation sensor (Micro-Epsilon optoNCDT / Keyence IL series), analog or RS-485/Modbus output | Micron-level forward/back distance | $800–1,500 |
| Mounting bracket at board's fixed base/fulcrum end | Stable anchor point | $20–40 |
| ADS1115 I2C ADC HAT *(analog)* or USB-to-RS485 adapter *(Modbus)* | Interfaces sensor to Pi | $15–25 |
| Raspberry Pi 5 (8GB) | Runs read loop + app sync | $200 (see pricing note above) |
| M12 sensor cable | Connects sensor to interface | $25–50 |
| DC power supply (12–24V, sensor-specific) | Powers the sensor | $20–30 |
| Weatherproof Pi enclosure | Protects Pi electronics | $25–40 |
| MicroSD card + Pi power supply | Pi OS/storage + power | $20–30 |

## Architecture Decision
- No laptop poolside — a **Raspberry Pi bridges** the sensor to the phone app (sensor/camera → Pi → phone), rather than sensor → phone directly or sensor → PC.
- Target platform: **mobile app** (iOS/Android).

## Data Pipeline & App Architecture

### On-Pi: Capture → Detect → Package
- Crop camera feed to the calibrated takeoff-zone ROI; watch for the foot keypoint (Phase 1: pose model) or marker (Phase 2) to appear then disappear.
- Log position from the **last valid contact frame** (moment before liftoff) as that attempt's reading — no manual "start" trigger needed.
- Per-attempt record (updated 2026-09-25 for both feet + orientation): `attempt_id, timestamp, session_id, diver_id, dive_type, left_foot {x_mm, y_mm, angle_deg}, right_foot {x_mm, y_mm, angle_deg}, detection_confidence, coach_target {x_mm, y_mm, angle_deg, tolerance_radius_mm, tolerance_angle_deg} (per foot), deviation_mm, deviation_angle_deg, thumbnail (frame w/ keypoint/marker overlay), short_clip_ref (optional)`.
- Thumbnail/clip included so the coach can visually sanity-check a reading, not just trust a number.

### Sync: Pi → App
- **Cloud sync recommended** (Supabase) over local-network-only — enables review of history/consistency from anywhere, not just poolside. Pi buffers locally and syncs when Wi-Fi is available.

### Mobile App
- Live/session view (attempts appear near real-time during practice).
- History view (filterable by session/date/dive type).
- Consistency dashboard: 2D scatter of foot positions vs. coach's target + tolerance zone, plus deviation-over-time trend line.
- Coach controls: set/adjust target position + tolerance per dive type, annotate attempts.
- Single-user (Charles) to start — no multi-athlete/role complexity yet.

### Stack
- **App:** React Native or Flutter (cross-platform).
- **Backend:** Supabase (Postgres) (DB, real-time sync, thumbnail/clip storage).
- **Pi:** Python (MediaPipe/OpenCV native), run as a `systemd` service for auto-restart/boot persistence.

## Raspberry Pi: Prerequisites to Learn First
1. Headless Pi OS setup (Raspberry Pi Imager, enable SSH before first boot).
2. Basic Linux CLI (Pi OS is Debian-based) — `apt`, file permissions.
3. Camera setup via the `libcamera` stack — test with `rpicam-hello` before writing code.
4. Python `venv` + `pip` for isolated package installs.
5. **MediaPipe is confirmed NOT compatible with current Raspberry Pi OS (Trixie / Python 3.13)** as of 2026-09 — default to **TensorFlow Lite + MoveNet** (`tflite-runtime`) instead, or deliberately install the older Bookworm OS (Python 3.11/3.12) if MediaPipe is preferred.
6. OpenCV homography/perspective transform (`cv2.findHomography`, `cv2.perspectiveTransform`) — this is the pixel→real-world-mm math.
7. `systemd` service files (auto-start on boot, auto-restart on crash) + `journalctl` for remote log reading.
8. Pi Wi-Fi config (`raspi-config`/`nmcli`) + Python `requests` for talking to the cloud backend.
9. SQLite basics, for local buffering when Wi-Fi drops.
10. Pi security hygiene: change default password, keep OS updated, use Tailscale (not open SSH) for remote access.

## Compute Responsibility Split: Pi vs. Cloud vs. Phone
Principle: **Pi senses, Cloud stores, Phone computes/presents.**

- **Raspberry Pi:** camera capture (ROI-cropped), pose model inference (foot keypoint), liftoff-detection logic, homography transform (pixel → real-world mm), thumbnail generation, local SQLite buffering + retry-upload.
- **Cloud (Supabase):** passive data store only — raw attempt records, coach target/tolerance settings, thumbnail/clip file storage, real-time sync to the app. No image processing or math happens here.
- **Phone App:** fetches raw position + target data; computes deviation-from-target **at display time** (not baked in at capture), so retroactive target changes recalculate historical charts correctly; owns all dashboards/visualization and the coach's target-editing UI.
- Rationale: the Pi shouldn't need to know about coaching targets — it only reports foot position. Keeps sensing code simple/testable in isolation and lets coaching logic evolve without touching the Pi.

## Software Architecture: Classes & Components (revised 2026-09-25 for SOLID)

An initial class sketch was reviewed against SOLID principles; this is the corrected version. Key fixes from the review: extracted thumbnail generation out of the data record (SRP), introduced a `FootDetector` interface so detection methods are swappable without touching the orchestrator (OCP/DIP), and made the orchestrator receive its dependencies via constructor injection rather than instantiating them internally (DIP, testability).

### Pi-side software (Python)

**`FootDetector` (abstract interface)** — `detect(frame) → list[FootMeasurement]`. All detection methods implement this so `CaptureService` depends on the abstraction, not concrete classes; adding a new detection method (e.g. the Phase 1.5 custom-trained model) means adding a new implementing class, not editing the orchestrator. Implementations must agree on how they signal "no foot detected" (empty list, not an exception) so they're truly substitutable (Liskov).

| Class | Fields | Functionality |
|---|---|---|
| **MarkerDetector** `implements FootDetector` | `dictionary`, `marker_ids_by_foot` (e.g. `{left: 12, right: 13}`) | `detect(frame) → list[FootMeasurement]` via ArUco corner detection |
| **PoseDetector** `implements FootDetector` | `model`, `confidence_threshold` | `detect(frame) → list[FootMeasurement]` via BlazePose heel/foot_index landmarks |
| **FootMeasurement** (value object) | `foot_side`, `position_mm {x,y}`, `angle_deg`, `confidence`, `source` | Plain data container; no behavior |
| **CalibrationManager** (pure geometry — split from persistence) | `homography_matrix`, `board_reference_points_px`, `board_reference_points_mm` | `calibrate(reference_points)`, `pixel_to_world(x_px, y_px) → (x_mm, y_mm)` |
| **CalibrationStore** (persistence — split out) | `file_path` | `save(calibration_data)`, `load() → calibration_data` |
| **CameraCapture** | `resolution`, `frame_rate`, `roi_bounds` | `start()`, `stop()`, `get_frame()`, `crop_to_roi(frame)` |
| **AttemptEventDetector** | `presence_threshold_frames`, `state` (idle/tracking) | `process_frame(frame) → Optional[AttemptTrigger]` — watches ROI for foot presence→absence, returns the last valid contact frame |
| **ThumbnailGenerator** (extracted from AttemptRecord — SRP fix) | `overlay_style` | `generate(frame, measurements) → thumbnail_path` — draws detected keypoints/markers on the frame |
| **AttemptRecord** (pure data — no behavior) | `attempt_id, timestamp, session_id, diver_id, dive_type, left_foot, right_foot, thumbnail_path, clip_path` | `to_dict()` only |
| **LocalBuffer** (SQLite) | `db_path` | `save(record)`, `get_unsynced()`, `mark_synced(attempt_id)` |
| **CloudSyncClient** | `api_url`, `api_key` | `upload(record) → bool`, `upload_thumbnail(path) → url`, `is_online()` |
| **CaptureService** (orchestrator, constructor-injected) | `detector: FootDetector`, `calibration: CalibrationManager`, `event_detector: AttemptEventDetector`, `thumbnail_generator: ThumbnailGenerator`, `buffer: LocalBuffer`, `sync_client: CloudSyncClient` | `run()` — main loop, the `systemd` service entrypoint. Dependencies are injected, not instantiated internally, so swapping `MarkerDetector` ↔ `PoseDetector` ↔ a future custom-model detector requires zero changes to this class |

### Backend data model (Supabase)

| Entity | Fields |
|---|---|
| **Athlete** | `diver_id, name, foot_length_mm, foot_width_mm, marker_placement_photo_url, created_at` |
| **Session** | `session_id, diver_id, date, location, coach_notes` |
| **CoachTarget** | `target_id, diver_id, dive_type, left_target {x_mm,y_mm,angle_deg}, right_target {x_mm,y_mm,angle_deg}, tolerance_radius_mm, tolerance_angle_deg, active_from` |
| **Attempt** | server-side mirror of `AttemptRecord` above |

**DRY note:** `AttemptRecord` (Pi-side) and `Attempt` (backend) are independently-defined mirrors of each other — legitimate, since the Pi-side has local file paths where the backend has URLs, but worth defining as a shared schema/contract deliberately rather than letting the two drift out of sync by accident over time.

### Mobile app (client-side)

| Class | Functionality |
|---|---|
| **AttemptRepository** | `fetchHistory(diverId, filters)`, `subscribeLive(sessionId, onUpdate)` |
| **AthleteProfileRepository** | `getProfile(diverId)`, `updateProfile(profile)` |
| **CoachTargetRepository** | `getTarget(diverId, diveType)`, `setTarget(target)` |
| **DeviationCalculator** | `computeDeviation(attempt, target) → {deviation_mm, deviation_angle_deg, within_tolerance}` — computed **at display time**, not stored, so retroactive target edits recalculate history correctly |
| **ConsistencyDashboardViewModel** | `refresh()`, `filterByDiveType()` — feeds scatter/trend-chart views, computes mean deviation / % within tolerance / std dev |
| **LiveSessionViewModel** | Real-time feed during practice, subscribes to new synced attempts |
| **CoachTargetEditorViewModel** | Create/edit target position + angle + tolerance per dive type |
| **AthleteProfileViewModel** | View/edit foot dimensions and marker-placement reference photo |

## Reading List
- [The Raspberry Pi Guide — headless setup](https://raspberrypi-guide.github.io/getting-started/raspberry-pi-headless-setup)
- [Raspberry Pi official docs — Camera software (libcamera/rpicam-hello)](https://www.raspberrypi.com/documentation/computers/camera_software.html)
- [MediaPipe Alternatives on Trixie (Cytron)](https://www.cytron.io/tutorial/mediapipe-alternative-in-trixie) — why MediaPipe breaks on current Pi OS
- [PINTO0309/TensorflowLite-bin (GitHub)](https://github.com/PINTO0309/TensorflowLite-bin) — prebuilt `tflite-runtime` wheels for Pi
- [OpenCV — Basic concepts of homography](https://docs.opencv.org/4.13.0/d9/dab/tutorial_homography.html)
- [OpenCV — Feature Matching + Homography (Python)](https://docs.opencv.org/4.13.0/d1/de0/tutorial_py_feature_homography.html)
- [freedesktop.org — systemd.service manual](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
- [DigitalOcean — Understanding systemd Units and Unit Files](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files)
- [docs.python.org — venv](https://docs.python.org/3/library/venv.html)
- [docs.python.org — sqlite3](https://docs.python.org/3/library/sqlite3.html)
- [Tailscale](https://tailscale.com) (official install)
- [Pi My Life Up — Installing Tailscale on Raspberry Pi](https://pimylifeup.com/raspberry-pi-tailscale/)

## Open / Not Yet Decided
- Exact liftoff-detection heuristic (frame-count threshold, confidence threshold) — needs tuning against real footage.
- Dive-type tagging: automatic (from pose/motion pattern) vs. manual coach/athlete tag per session.
- Multi-foot tracking: some dive types may need both feet (e.g. back/reverse dives on the edge) rather than a single point — revisit once Phase 1 accuracy testing is underway.
