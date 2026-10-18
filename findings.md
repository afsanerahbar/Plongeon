# Diving Foot Placement Tracker — Findings

## Project Concept
Foot placement on the board during takeoff matters for consistency and coaching feedback. Goal: a camera system that measures both feet's placement and angle on every attempt, and a way for the coach to track history and consistency against a recommended target.

## Sensing Approach — Evolution

1. **Foot-worn sensors (pressure insoles / IMUs)** — ruled out. They measure pressure/motion but have no way to know *where* the foot is relative to the board.
2. **Board-mounted pressure mat / touch sensor (FSR array)** — ruled out. Lower accuracy, physical wear on the board, and prone to noise from water/texture.
3. **Industrial ultrasonic distance sensor (SICK UC20 / UM18-217, IO-Link)** — considered, then dropped. Reasonable accuracy and good noise-averaging, but the IO-Link interface master is built for Windows/industrial PC software stacks — Raspberry Pi (ARM Linux) driver/SDK support is uncertain, adding real integration risk.
4. **Single-axis laser displacement sensor (Micro-Epsilon optoNCDT, Keyence IL series)** — viable only if forward/back distance from the board tip were enough. Micron-level accuracy over analog or RS-485/Modbus RTU (well-supported on Pi). Limitation: single axis — no side-to-side centering, no angle.
5. **Overhead camera + ArUco fiducial markers (chosen)** — full foot placement is a 2D problem (forward/back **and** side-to-side) plus an angle. A calibrated camera suits this better than a 3D depth camera (stereo/ToF depth error is typically 2–5mm). Achievable precision is set by camera resolution and geometry — see Accuracy Budget.
6. **Phone camera at 240fps (considered 2026-10-01, not chosen)** — cheaper and no extra hardware, but rolling shutter, not a dedicated device, and it drops most of the embedded-systems work the independent study is built around.

## Related Research: DiveNet (DFKI, 2023)
Paper: *DiveNet: Dive Action Localization and Physical Pose Parameter Extraction for High Performance Training* (Murthy, Taetz, Lekhra, Stricker — IEEE Access 2023).
- Single static camera (60Hz) + a mapping between the board plane and the image, recomputed **for every dive** — the same idea this plan now uses for the moving board (see Accuracy Budget, issue 2).
- Reported accuracy: COM error ~6px, dive peak height sensitivity ~20cm, mean joint angle error ~10°, pose regression ~70% PCK — good for research-grade form analysis, too coarse for a ±1° foot angle.
- **No open-source code or usable dataset found.** A GitHub search led to an unrelated repo (`aim-uofa/Poseur`) — a bad search association, verified and discarded. The dataset host (`divenet.kl.dfki.de`) now redirects to an unrelated DFKI project. The full paper PDF is blocked from automated fetch.
- **Comparing DiveNet's numbers to our task:** the COM and height numbers come from tracking the whole dive and fitting a projectile curve — harder than our task. The keypoint numbers are per-frame localization — the same task category as the optional markerless work (Phase 5).

## Advisor Review (2026-10-18)
A review by the project's professor advisor found three geometry problems that the earlier plan missed, plus problems with experiment design, practical setup, and scope. They're folded into the sections below; this is the summary.

| # | Issue | Where it's addressed |
|---|---|---|
| 1 | Markers sit above the board, so a flat board-to-image mapping puts them in the wrong place (up to ~6.6° of angle error) | Accuracy Budget, Calibration, Class Spec |
| 2 | The board moves tens of centimeters during a dive; one calibration at the start can't follow it | Accuracy Budget, Calibration, Class Spec |
| 3 | The camera height needed for the resolution conflicts with the diver's clearance | Accuracy Budget, Mounting, Phase 0 |
| 4 | The bench test only tested the easy case | Accuracy Budget (bench test) |
| 5 | ±1° between sessions depends on gluing markers within ~2mm | Requirements |
| 6 | The hurdle would trigger two attempts; which moment to measure is undecided | Requirements, Class Spec |
| 7 | Storage: raw clips would fill the 64GB card | Capture |
| 8 | Network login and clock at the pool | Network & Clock |
| 9 | Phases 0–4 are too much for one independent study | Project Breakdown |
| 10 | Too many interfaces too early | Software Architecture |

Smaller corrections: the lens field of view is 45° horizontal (not 45–55°); the optional 90° lens is likely the wrong direction; OpenCV links should be 4.x; define an angle convention for backward dives; plan for condensation in a sealed box (Phase 3).

## Requirements (updated 2026-10-18)

- **Both feet** are tracked on every attempt.
- **Position** of each foot on the board: forward/back and side-to-side, within a tolerance in millimeters **to be set by the coach**.
- **Orientation** (toe-pointing angle) of each foot, to within **±1°** — scope to be confirmed with the coach (see below).
- **Divers are barefoot**, so anything attached to the foot goes directly on skin.

**Why ±1° is hard:** orientation is derived from two points (heel and toe) by computing the angle of the line between them. For a ~270mm foot, 1° of rotation moves the toe sideways relative to the heel by only ~5mm, so each point must be accurate to roughly 1mm. Noise in two independently-estimated points compounds when they're subtracted. A general-purpose pose model like BlazePose can't do that, so markers are the primary method.

**Is ±1° the right requirement? (issue 5)** If the heel marker is placed 5mm to one side of where it was last session, the heel-to-toe line rotates ~1.1° even with a perfect camera. Comparing angles **between sessions** to ±1° therefore needs markers placed within ~2mm every time, which a reference photo won't achieve. Comparing **within a session** (one marker placement) avoids that problem and is much easier. Ask the coach what decision the angle number drives.

**Which moment is measured? (issue 6)** A running takeoff has a hurdle: the diver lands on the board, presses it down, and leaves at last contact. Candidates are the hurdle landing, the deepest press, or last contact. Last contact is when the heels are highest, which makes issue 1 worse. Ask the coach which moment matters.

**Questions for the coach (before Phase 0):**
1. What decision does the angle number drive — consistency within a practice, or comparison across weeks?
2. What position tolerance matters, in millimeters?
3. Which moment matters: hurdle landing, deepest press, or last contact?
4. Where on the board do the feet actually land, for forward and backward takeoffs? (This sets the size of the area the camera must cover — see issue 3.)

**Angle convention:** 0° points along the board's long axis toward the tip, angles increase counterclockwise as seen from above, range −180° to 180°. Backward takeoffs face the fulcrum, so their angles sit near ±180°; deviations are computed with wraparound so 179° and −179° are 2° apart, not 358°.

## Accuracy Budget (updated 2026-10-18)

### Camera numbers
- Arducam IMX296: 1440×1080 pixels, 3.45µm pixels, 6mm lens, field of view **45° horizontal × 33° vertical** (55° diagonal) per Arducam's spec.
- Millimeters per pixel = distance × 3.45µm ÷ 6mm ≈ **0.58mm per pixel per meter of distance**.
- To cover 1.6m of board across the 45° width, the camera must be ~1.9m away, giving ~1.1mm per pixel. A smaller area to cover means a closer camera or a narrower lens (see issue 3).

### 1. Markers sit above the board's surface
- The board-to-image mapping (a homography) is only correct for points **on** the board's surface. Markers on top of the foot sit 3–8cm above it; at takeoff divers rise onto their toes, so the heel is higher still; on backward takeoffs the heels hang off the board.
- A point above the surface lands in the wrong place: the camera's line of sight through it continues down to a different spot on the board. **Example:** heel marker 8cm up, toe marker 3cm up, camera 30° from vertical. The heel shifts ~46mm and the toe ~17mm, a difference of ~29mm. If that difference is sideways to the foot, the angle error is ~6.6°. Even with the camera straight down, markers near the frame's edge shift ~25mm.
- **Fix:** use the camera's full 3D position relative to the board (`cv2.solvePnP`), and place each marker at its real height instead of on the surface. Marker heights above the sole are measured once per athlete (Athlete Profile).
- **Catch:** a fixed height only holds while the foot is flat. When the heel rises, its height changes. Phase 0 tests two ways to handle this: placing both markers at the same height on the foot (their shifts then mostly cancel in the angle), and estimating each marker's height from its own size and shape in the image. The moment chosen in Requirements (issue 6) decides how much this matters.

### 2. The board moves
- Board tip deflection during a dive is tens of centimeters, and the board tilts as it bends. With the camera ~1.9m above the board, a 30cm drop changes the scale by ~14%; a foot 0.6m from the center of the frame then reads ~80mm off. Calibrating once at the start can't correct this.
- **Fix:** board reference markers visible in **every frame**, and the board's position and tilt recomputed for every frame from them (DiveNet does the same per dive). This also means the temporary mount in Phases 0–2 doesn't need a fresh board calibration each session, as long as the board markers are in view.
- **Where the board markers go** is a Phase 0 question. The board's side edges are vertical, so markers there aren't on the top surface and only work with the 3D approach from issue 1 and a camera angle that sees them. Markers on the top surface near the edges, outside where feet land, need the pool's approval and must not change the surface's grip.

### 3. Camera height vs. the diver's clearance
- Covering 1.6m at ~1.1mm per pixel puts the camera ~1.9m above the board. A diver's raised arms and hurdle reach higher than that. The plan's goals pull against each other: a steep viewing angle, high resolution, and staying clear of the diver.
- **Fix:** before buying any mount, fill in a **geometry worksheet** from measurements taken at the pool (board length and width, 1m or 3m board, handrail positions, ceiling height, how high the diver reaches in the hurdle, and where the feet land). It gives camera height, lens, millimeters per pixel, and clearance from the diver.
- The likely answer is a **narrower lens mounted higher**, the opposite of the 90° wide lens listed earlier. **Don't buy the 90° lens.**

### Other accuracy factors
- **The angle is computed in board coordinates**, after lens correction and 3D placement of both markers. An angle measured in raw pixels is skewed by perspective.
- **Lens distortion** is corrected with a one-time checkerboard calibration (`cv2.calibrateCamera`).
- **Two markers per foot** (heel and base of big toe, ~250mm apart): a 0.5px error in a marker's center costs ~0.1° of angle, and how each marker is rotated on the skin doesn't matter. One marker per foot would cost ~0.7° from corner noise alone.
- **Marker placement** limits angle comparisons between sessions (issue 5).

### Bench accuracy test (Phase 0, issue 4)
A flat paper template on a table would probably pass and give false confidence. The test covers:
- **Flat case:** a printed foot template with heel and toe markers, rotated in known steps and moved to known positions.
- **Raised markers:** the template on blocks at measured heights, then a real foot with the heel raised.
- **Real geometry:** the camera at the distance and angle expected at the pool (from the geometry worksheet).
- **Ground truth about 5× better than the requirement:** e.g. a large printed protractor readable to ~0.2°, and positions measured with a ruler to ~1mm.
- **Repeated trials** (20+ per case), reporting mean error, standard deviation, and maximum error.
- **Pass:** maximum angle error under 1° in every case, and position error within the coach's tolerance.

## Recommended Solution

### Markers: two ArUco markers per foot
- **Four markers** from `DICT_4X4_50`: left heel `10`, left toe `11`, right heel `12`, right toe `13`.
- **Placement:** heel and base of the big toe, recorded in the athlete's reference photo. Each marker's height above the sole is measured once (Athlete Profile).
- **Size:** ~20–25mm — enough pixels to detect; the angle comes from the heel–toe line.
- **Material:** waterproof, matte (to avoid glare) printed markers — e.g. waterproof vinyl or temporary-tattoo paper — with a skin-safe adhesive. They must survive a water entry on every attempt and pool chlorine (tested in Phase 0).
- **Board markers:** a few more `DICT_4X4_50` markers fixed to the board and visible in every frame, to track its movement (issue 2). Placement decided in Phase 0.
- **Output per foot:** heel and toe positions on the board in millimeters, and the foot angle in degrees.

### Camera
- **Arducam IMX296 color global-shutter module** with its bundled 6mm M12 lens (45° horizontal FOV) and Pi 5 cable. Global shutter keeps the fast-moving foot sharp.
- **Pi 5 setup on Bookworm:** in `/boot/firmware/config.txt`, set `camera_auto_detect=0` and add `dtoverlay=imx296` under `[all]` (append `,cam0` if using the CAM0 port).
- **Lens choice comes from the geometry worksheet** (issue 3). A narrower M12 lens may be needed; the 90° lens is off the list.

### Capture: video, not single pictures
- The camera records ~60 frames per second. The foot is planted for only a fraction of a second, and there's no way to fire a single photo at exactly that moment; at 60fps the system gets a few dozen planted frames to choose from.
- **Offline mode (Phases 1–2):** the Pi saves a short clip of each attempt during practice and analyzes the clips afterward.
- **Live mode (Phase 3+):** frames are analyzed as they arrive and discarded; each attempt keeps one still image with measurements drawn on.
- **Storage (issue 7):** one raw color frame is ~4.7MB, so a 5-second clip at 60fps is ~1.4GB and 40 dives ~56GB — more than the 64GB card, and slow to write. Instead:
  - **grayscale** (ArUco doesn't need color): one-third the size;
  - **crop to the takeoff zone** and keep clips short (~3 seconds);
  - save frames as **high-quality JPEG** rather than raw — a few GB per practice, while keeping marker corners sharp. Heavy video compression blurs corners and costs accuracy.
  - A USB SSD is the fallback if Phase 0 shows that isn't enough.
- **The Pi 5 has no hardware video encoder.** The Pi keeps the last few seconds of frames in memory (a few hundred MB in grayscale) and writes the clip after the attempt. Phase 0 confirms the approach.

### Calibration
- **Lens (one-time):** ~15–20 photos of a printed checkerboard at different angles → `camera_matrix` and `dist_coeffs`.
- **Board layout (one-time):** measure where each board marker sits on the board, in millimeters.
- **Board position (every frame):** the board markers in each frame give the board's position and tilt relative to the camera (`cv2.solvePnP`). This replaces the earlier "calibrate once under a static load" step.

### Markerless options (Phase 5, optional)
- **BlazePose:** MediaPipe's 33-point model includes `heel` and `foot_index` — position cross-check only. MediaPipe installs on Raspberry Pi OS **Bookworm** (Python 3.11), not Trixie (Python 3.13); re-checked 2026-10-18 and still true, so the Pi stays on Bookworm. MoveNet is not an option: its keypoints stop at the ankle.
- **Classical CV:** background subtraction + `cv2.fitEllipse` / PCA on each foot's silhouette.
- **Custom-trained keypoint model:** fine-tuned on clips collected in Phases 0–3 (a vision model, not an LLM); training on a GPU, only the exported model runs on the Pi.

## Project Breakdown (risk-first, offline-first — adopted 2026-10-01, scope set 2026-10-18)

Answer the riskiest questions first, deliver something the coach can use early, and add live results and an app only once the measurement is proven.

**Scope for the independent study (issue 9):** the deliverable is **Phases 0–1**, with **Phase 2 as a stretch goal** and **Phases 3–5 as future work**. Learning Python, OpenCV, SQL, a web dashboard, and React Native from zero in one study is too much; a working, well-measured Phase 1 is a strong result on its own.

| Phase | What gets built | Exit test |
|---|---|---|
| **0. Feasibility** (2–3 weeks) | Coach questions answered. **Pool site visit:** tape measure, photos, geometry worksheet. Pi + camera + printed markers on a temporary mount; record clips at the bench and in one pool session; analyze in a Jupyter notebook. Test the 3D marker-height fix and board-marker placement. | Bench test passes in every case (max angle error under 1°, position within the coach's tolerance). Markers survive 10+ water entries. Markers detected in ≥90% of planted frames in pool lighting. The geometry worksheet shows a camera position that covers the landing area at ≤1.2mm per pixel and stays clear of the diver's reach. |
| **1. Offline pipeline** (deliverable) | The Pi records a short clip per attempt during practice; afterward the clips run through the pipeline → CSV, thumbnails, and a one-page session report. | The coach gets per-attempt numbers after each practice; repeated readings of the same planted foot agree within 0.5°. |
| **2. Cloud + simple dashboard** (stretch) | Supabase tables; results upload after processing; a Streamlit dashboard shows history and consistency; the coach edits targets. | The coach views trends from anywhere and sets targets. |
| **3. Real-time on the Pi** (future work) | Live pipeline, uploads during practice, permanent mount with PoE, `systemd` service, Start/End session. | Results within seconds of each dive; runs a whole practice unattended. |
| **4. Mobile app** (future work) | React Native app. | The full product. |
| **5. Markerless** (future work, optional) | BlazePose, classical CV, or a custom model trained on collected clips. | Accuracy close to the markers, without markers. |

## Athlete Profile
Lightweight per-athlete profile (a local file in Phase 1, Supabase from Phase 2):
- **Foot length and width.**
- **Height of each marker above the sole** (heel and toe), measured with the foot flat — needed to place markers correctly in 3D (issue 1).
- **Heel-to-toe marker distance**, measured once. It should match every reading, so a different measured distance flags a probable bad detection.
- **Reference photo of marker placement** (both markers on both feet), for consistent placement across sessions.

## Mounting & Power

### Phases 0–2: temporary setup
- **Mount:** a temporary clamp on the handrail, or a tall stand beside the board, set up and removed each practice — **with pool and coach approval**, and clear of the diver's walk-out, hurdle, and arm swing. Its position comes from the geometry worksheet (issue 3).
- **Power:** a USB-C power bank (5V/3A, Power Delivery). The Pi 5 runs fine at 3A; it only limits current to USB accessories, which this setup doesn't use. No wiring near the water.
- **The Pi sits next to the camera** in its enclosure: the bundled camera cable is short (check its length on arrival), and camera-cable runs beyond ~1m become unreliable.

### Phase 3 (future work): permanent installation
- **Mount** position from the geometry worksheet; the handrails are bolted to the fixed stand and don't move with the board, but may not allow a high enough or steep enough view (issue 3).
- **Power: PoE.** An **802.3at (PoE+)** injector plugs into a GFCI outlet away from the pool edge; one outdoor-rated Ethernet cable carries power to the PoE HAT, so only low-voltage DC reaches the deck. Plain 802.3af can't power a Pi 5. Don't connect USB-C power while on PoE. Check that the PoE HAT fits alongside the Active Cooler and inside the Sixfab enclosure.
- **Condensation:** a sealed box in a warm, humid pool hall collects moisture inside; use a breathable vent membrane and check for fogging on the camera window.
- The 27W USB-C supply is for bench development indoors.

## Network & Clock (issue 8)
- **Network:** school Wi-Fi often needs a login page or an enterprise login (like eduroam) that a headless Pi can't easily get past. Phase 1 needs no network (clips are processed on the Pi or copied off afterward). From Phase 2, plan on a phone hotspot or a wired Ethernet port, with IT's permission.
- **Clock:** without a network, the Pi can't set its clock, so `captured_at` timestamps could be wrong. The Pi 5 has a built-in real-time clock that keeps time with a small battery (~$5); set the time once while online.

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
| Arducam IMX296 Color Global Shutter Camera, M12 lens + Pi 5 cable included | $94.99 |

**Total ordered: $419.91**

The advisor review reports Arducam's own store price for the camera as **$64.99** (not yet confirmed here) — worth checking if a replacement is ever needed.

⚠️ *Do NOT buy the "6mm Wide Angle Lens for Raspberry Pi HQ Camera CS" ($34.00) — wrong mount (CS, not M12). Skip any cart containing the Raspberry Pi HQ Camera (PID 4561): it's a rolling-shutter camera. Don't buy the 90° M12 lens (issue 3).*

**After the microSD arrives:** reflash it with Raspberry Pi OS **Bookworm** via Raspberry Pi Imager — the pre-loaded image is likely Trixie.

### Phase 0 (buy now)
| Item | Est. price | Note |
|---|---|---|
| Foot-marker material + skin-safe adhesive | $10–20 | waterproof, matte |
| USB-C power bank, 5V/3A Power Delivery | $25–40 | borrow one if possible |
| Pi 5 real-time-clock battery | ~$5 | keeps timestamps correct offline |
| Temporary mount (clamp or tall stand) | $0–30 | borrow if possible; position from the geometry worksheet |
| Checkerboard, foot and board markers, protractor template | free | print |

### After Phase 0 passes
| Item | Est. price | Note |
|---|---|---|
| Articulating boom/clamp arm | $30–60 | sized from the geometry worksheet |
| Weatherproof camera housing | $20–40 | sized from the real camera + lens |
| Replacement M12 lens, if the worksheet calls for a narrower one | ~$15–40 | chosen from the worksheet |

### Phase 3 (future work)
| Item | Est. price | Note |
|---|---|---|
| PoE+ injector (802.3at, ≥30W) | $25–35 | search `802.3at poe+ injector` |
| Outdoor-rated Ethernet cable (Cat6) | $20–40 | measure the run to the GFCI outlet first |

**Budget:** Phases 0–2 come to about **$510–655**, depending on the lens and how much can be borrowed, against the $600 request. Phase 3 adds **$45–75** as future work.

### Camera sourcing history
- The official Raspberry Pi Global Shutter Camera was out of stock at 5 of 6 retailers on 2026-09-13; the Arducam IMX296 uses the same sensor and was in stock.
- Other Arducam options considered: AR0234 (2.3MP color, "Pivariety" driver) and OV9782 (1MP, but that listing is USB, not CSI). Monochrome sensors were avoided because the Phase 5 BlazePose option needs color; recording in grayscale is still fine for markers.
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
2. Detect all markers: the four foot markers and the board markers.
3. Compute the board's position and tilt for this frame from the board markers.
4. Correct for lens distortion, place each foot marker in 3D at its known height, express heel and toe in board millimeters, and compute each foot's angle.
5. Track the attempt in the takeoff zone (defined in board millimeters) and take the reading at the moment the coach chose (issue 6). The hurdle must not count as a separate attempt.
6. Draw a thumbnail and store the attempt record locally.

### Offline mode (Phases 1–2)
- **Before practice:** `record.py --diver <id> --dive <type>` creates a session folder with the session details; run it again if the dive type changes (or fix dive types in the report afterward).
- **During practice:** the Pi records a short clip of each attempt into the session folder.
- **After practice:** `process_clips.py` runs every clip through the pipeline and writes a CSV, thumbnails, and a one-page session report; in Phase 2 it also uploads results.
- **Clips are deleted after processing by default**; they're kept only with athlete consent, to build a dataset for Phase 5.

### Live mode (Phase 3+, future work)
- `main.py` runs the same pipeline on the live camera. The coach starts a session in the dashboard or app, which marks a Session row active in Supabase; a background worker on the Pi reads it every few seconds, and each attempt is tagged with the cached session. With no active session, attempts are saved as unassigned.

### Per-attempt record
`attempt_id, captured_at, session_id, diver_id, dive_type` (null = unassigned), `left_foot {heel_x_mm, heel_y_mm, toe_x_mm, toe_y_mm, angle_deg, confidence}`, `right_foot {…}`, thumbnail. The record holds **no coach targets or deviations** — the Pi only reports where the feet were.

## Compute Responsibility Split
Principle: **Pi senses, Cloud stores, Dashboard/App computes and presents.**

- **Raspberry Pi:** recording, marker detection, board tracking, 3D placement of markers, foot angles, attempt detection, thumbnails, local storage; uploads from Phase 2.
- **Cloud (Supabase, Phase 2+):** passive store — attempts, sessions, athletes, coach targets, thumbnails.
- **Report (Phase 1), Dashboard (Phase 2), App (future):** compute deviation from target **at display time**, so changing a target recalculates history correctly; deviations use the angle convention's wraparound.

## Stack
- **Pi:** Python on Raspberry Pi OS **Bookworm** — OpenCV 4.x (4.7 or later: the ArUco API changed in 4.7, and older tutorials fail with confusing errors), Picamera2, SQLite; `supabase-py` from Phase 2.
- **Phase 0 analysis:** Jupyter notebook on a laptop.
- **Version control:** Git, with small commits from day one.
- **Backend (Phase 2):** Supabase (Postgres).
- **Dashboard (Phase 2):** Streamlit, in Python.
- **App (future work):** React Native (TypeScript).

## Prerequisites to Learn, by Phase
- **Phase 0:** Linux command line; Git basics; headless Pi setup (flash **Bookworm**, SSH enabled); camera setup (`config.txt`, `rpicam-hello`, Picamera2 recording); Python `venv`; Jupyter; OpenCV 4.x ArUco detection, lens calibration, and camera pose (`solvePnP`).
- **Phase 1:** Python classes and dataclasses, one abstract base class (`FrameSource`); SQLite.
- **Phase 2:** Supabase and `supabase-py`; Streamlit.
- **Future work:** `systemd`, Tailscale (Phase 3); React Native (Phase 4); MediaPipe, classical CV (Phase 5).
- See `reading-list.md` for links, ordered by phase.

## Security & Privacy
- **Supabase key on the Pi (Phase 2+):** a restricted key with row-level security that only allows inserting attempts, uploading thumbnails, and reading the active session — never the service-role key. It lives in a `.env` file listed in `.gitignore`; never commit it (this repo is on GitHub).
- **Consent:** get athlete consent for recording and storing images, and follow the school's policy for athletes under 18. Clips are deleted after processing unless consent covers keeping them.

## Software Architecture (Pi side, revised 2026-10-18)

**Visual design doc:** phases, components, class diagram, call sequences, and the Phase 1 build order — `pi-side-design.html` in this repo, published at https://claude.ai/artifact/WgcJtuM2NdzgV3FbFQs9vv

**Design principles:**
- **Add an interface only when a second implementation exists (issue 10).** `FrameSource` qualifies in Phase 1: frames come from recorded clips and from the live camera. Detection and sessions each have one implementation until much later, so they start as plain classes and become interfaces when the second implementation arrives (`SessionProvider` in Phase 3, `FootDetector` in Phase 5). In Python that change is small.
- **3D geometry, per frame:** the board's position is recomputed from board markers in every frame, and foot markers are placed at their real height (issues 1–2).
- **Detectors return pixel coordinates**; `CalibrationManager` turns them into board millimeters and angles.
- `CaptureService` has no network code; uploads (Phase 2) run in `SyncWorker` on its own thread.
- Dependencies are passed in through constructors, so each class can be tested alone. Data objects are plain Python dataclasses, a few lines each.

### Entry scripts
| Script | Phase | What it does |
|---|---|---|
| `calibrate.py` | 0–1 | Lens calibration from checkerboard photos; records the board-marker layout. Saved through `CalibrationStore`. |
| `record.py` | 1 | Creates a session folder (diver, dive type), then runs `ClipRecorder` during practice. |
| `process_clips.py` | 1 | After practice: runs each clip through `CaptureService` (with `VideoFileSource` + `FixedSession`), then `ReportWriter`; in Phase 2 also `SyncWorker.sync_once()`. |
| `main.py` | 3 (future) | Live: `CaptureService` with `CameraCapture`, plus `SyncWorker`. |

### Classes by phase
| Phase | Classes |
|---|---|
| 0 | None yet — notebook prototypes. What passes becomes the Phase 1 classes. |
| 1 | Data: `Frame`, `MarkerDetections`, `FootKeypoints`, `BoardPose`, `FootMeasurement`, `AttemptTrigger`, `SessionContext`, `AttemptRecord`. Interface: `FrameSource`. Behavior: `CameraCapture`, `VideoFileSource`, `ClipRecorder`, `MarkerDetector`, `CalibrationManager`, `CalibrationStore`, `AttemptEventDetector`, `ThumbnailGenerator`, `FixedSession`, `LocalBuffer`, `ReportWriter`, `CaptureService`. |
| 2 (stretch) | `CloudSyncClient`, `SyncWorker` |
| 3 (future) | `SessionProvider` interface, implemented by `FixedSession` and `CloudSyncClient` |
| 5 (future) | `FootDetector` interface, implemented by `MarkerDetector` and `PoseDetector` |

### Pi-Side Class Spec — Draft (Phase 1, plus Phase 2)
*Field/method-level detail with a one-line purpose per class. Subject to change after Phase 0.*

**Data objects (dataclasses, no behavior)**
- **`Frame`** — one image and when it was taken. Fields: `image`, `captured_at`.
- **`MarkerDetections`** — everything the detector found in one frame. Fields: `feet` (list of `FootKeypoints`), `board_markers` (marker ID → four corners in pixels).
- **`FootKeypoints`** — one foot's heel and toe points in pixels. Fields: `foot_side`, `heel_px`, `toe_px`, `confidence`.
- **`BoardPose`** — where the board is relative to the camera in this frame. Fields: `rotation`, `translation`, `reprojection_error`.
- **`FootMeasurement`** — one foot in board coordinates. Fields: `foot_side`, `heel_mm`, `toe_mm`, `angle_deg` (see Angle convention), `confidence`.
- **`AttemptTrigger`** — one detected attempt at the chosen moment. Fields: `frame`, `keypoints {left, right}`, `measurements {left, right}`.
- **`SessionContext`** — who is diving and which dive. Fields: `session_id`, `diver_id`, `dive_type`, `marker_heights_mm {heel, toe}` (from the athlete profile).
- **`AttemptRecord`** — one completed attempt, ready to store. Fields: `attempt_id`, `captured_at`, `session_id`, `diver_id`, `dive_type`, `left_foot`, `right_foot`, `thumbnail_path`. Methods: `to_dict()`.

**Interface**
- **`FrameSource`** — one way to get frames, recorded or live. Methods: `start()`, `read() → Frame or None` (None when a clip ends), `stop()`.

**Behavior classes**
- **`CameraCapture`** *(FrameSource)* — the only class that talks to the camera, through Picamera2. Fields: `resolution`, `frame_rate`, `exposure_us` (short, to freeze motion), `grayscale`.
- **`VideoFileSource`** *(FrameSource)* — plays back a recorded clip as frames with their original timestamps. Fields: `path`.
- **`ClipRecorder`** — during practice, saves a short clip of each attempt. Fields: `camera`, `motion_zone_px`, `motion_threshold`, `pre_seconds`, `post_seconds`, `output_dir`. Methods: `run()`, `stop()`. Keeps the last `pre_seconds` of grayscale frames in memory; when motion appears in the zone, keeps recording until `post_seconds` after it stops, then writes the frames as high-quality JPEGs.
- **`MarkerDetector`** — finds the foot markers and the board markers. Fields: `dictionary` (`DICT_4X4_50`), `foot_marker_ids` (`{left: {heel: 10, toe: 11}, right: {heel: 12, toe: 13}}`), `board_marker_ids`, `detector_params` (sub-pixel corner refinement on). Methods: `detect(frame) → MarkerDetections`. Each foot marker's center (mean of its four refined corners) becomes the heel or toe point; a foot is reported only when both of its markers are found.
  - **`dictionary`:** the ArUco marker family (`cv2.aruco`), not a Python dict. `DICT_4X4_50` is a 4×4-bit grid with 50 IDs: small markers decode reliably, and the project needs ~10 IDs.
- **`CalibrationManager`** — turns pixels into board coordinates, in 3D. Pure math, no file I/O. Fields: `camera_matrix`, `dist_coeffs`, `board_layout_mm` (where each board marker sits on the board). Methods: `calibrate_lens(checkerboard_images)`; `board_pose(board_markers) → BoardPose` (`cv2.solvePnP`, every frame); `to_world(keypoints, board_pose, marker_heights_mm) → FootMeasurement` (corrects lens distortion, traces each point's line of sight to the plane at that marker's height above the board, expresses heel and toe in board millimeters, then computes the angle); `to_data()`, `from_data(data)`.
- **`CalibrationStore`** — saves and loads calibration data on disk. Fields: `file_path`. Methods: `save(data)`, `load()`.
- **`AttemptEventDetector`** — decides when one attempt happened and picks the frame to measure. Fields: `zone_mm`, `presence_frames`, `absence_frames` (longer than the hurdle's flight, ~0.6s ≈ 36 frames at 60fps, so the hurdle isn't a second attempt), `measure_at` (hurdle landing, deepest press, or last contact — from the coach's answer), `state`. Methods: `process(frame, keypoints, measurements) → AttemptTrigger or None`.
- **`ThumbnailGenerator`** — draws a reviewable image per attempt. Fields: `overlay_style`, `output_dir`. Methods: `generate(trigger) → thumbnail_path`.
- **`FixedSession`** — the session read from a session folder. Fields: `session`. Methods: `current_session() → SessionContext`.
- **`LocalBuffer`** — keeps attempt records on local disk (SQLite). Fields: `db_path`. Methods: `save(record)`, `get_session(session_id)`, `get_unsynced(limit)`, `mark_synced(attempt_id)`.
- **`ReportWriter`** — turns a session's attempts into files the coach can read. Fields: `buffer`, `output_dir`. Methods: `write_session_report(session_id)` → a CSV and a one-page HTML summary.
- **`CaptureService`** — runs the pipeline over a frame source and stores each attempt. Fields: `frame_source`, `detector` (`MarkerDetector`), `calibration`, `event_detector`, `thumbnail_generator`, `buffer`, `session` (`FixedSession`). Methods: `run()`, `stop()`.

**Phase 2 (stretch)**
- **`CloudSyncClient`** — the only class that talks to Supabase. Fields: `supabase_url`, `supabase_key` (restricted, from `.env`). Methods: `is_online()`, `upload_thumbnail(path) → url`, `upload(record, thumbnail_url) → bool`.
- **`SyncWorker`** — moves results to the cloud without slowing capture. Fields: `buffer`, `cloud`, `interval_s`. Methods: `sync_once()`, `start()`, `stop()`.

### Backend data model (Supabase, Phase 2)

| Entity | Fields |
|---|---|
| **Athlete** | `diver_id, name, foot_length_mm, foot_width_mm, heel_marker_height_mm, toe_marker_height_mm, heel_toe_marker_distance_mm, marker_placement_photo_url, created_at` |
| **Session** | `session_id, diver_id, status (active/ended — used from Phase 3), current_dive_type, started_at, ended_at, location, coach_notes` |
| **CoachTarget** | `target_id, diver_id, dive_type, reference_point (toe/heel), left_target {x_mm, y_mm, angle_deg}, right_target {x_mm, y_mm, angle_deg}, tolerance_radius_mm, tolerance_angle_deg, active_from` |
| **Attempt** | mirror of the Pi's `AttemptRecord`, with `thumbnail_url` instead of a local path; `session_id` null means unassigned |

Define `AttemptRecord` (Pi) and `Attempt` (backend) from one shared schema so they don't drift apart.

### Dashboard (Streamlit, Phase 2)
- **History:** attempts by session, date, and dive type, with thumbnails.
- **Consistency:** each foot's position against the coach's target zone and its angle deviation, with trends; deviations computed at display time, measured at the target's reference point (toe for forward-facing takeoffs, heel for backward).
- **Targets:** per diver, per dive type, per foot — position, angle, tolerances, reference point.

### Mobile app (future work)
React Native, after Phase 3: live view, session control, target editor, history — see the 2026-10-01 version of this file in git history for the class sketch.

## Open / Not Yet Decided
- **Coach answers:** what the angle drives (within vs. between sessions), position tolerance in mm, which moment to measure, where the feet land.
- **Phase 0 answers:** marker durability on wet skin and glare; handling a rising heel (equal-height markers vs. per-marker height estimate); board-marker placement and approval; camera position and lens from the geometry worksheet; clip storage and write speed on the Pi 5.
- Attempt-detection thresholds — tune on Phase 0–1 footage.
- On arrival: camera cable length.
