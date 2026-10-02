# Reading List — Step-by-Step Prep Order

---

## Step 0 — Absolute Beginner (start here if you've never used a Pi before)

1. **The Official Raspberry Pi Beginner's Guide (free PDF)** — unboxing, first boot, basic OS navigation, zero assumed knowledge.
   https://magazines-static.raspberrypi.org/books/full_pdfs/000/000/005/original/Beginners_Guide_v1.pdf
   fallback search: `official raspberry pi beginner's guide free pdf`

2. **Raspberry Pi Foundation — Introduction to Python pathway** — learn Python from scratch (variables, loops, functions), no Pi hardware needed yet.
   https://projects.raspberrypi.org/en/pathways/python-intro
   fallback search: `raspberry pi foundation intro to python pathway`

3. **Raspberry Pi Foundation — Physical Computing with Python** — bridges from software to hardware: Python code reacting to sensors and devices connected to the Pi.
   https://github.com/raspberrypilearning/physical-computing-with-python
   fallback search: `raspberrypilearning physical computing with python`

---

## Step 1 — Pi Fundamentals & Setup

4. **Headless Pi setup** — configuring SSH/Wi-Fi at flash time via Raspberry Pi Imager, no monitor needed. Choose **Raspberry Pi OS Bookworm**, not the newest Trixie (MediaPipe doesn't install on Trixie).
   https://raspberrypi-guide.github.io/getting-started/raspberry-pi-headless-setup
   fallback search: `raspberry pi guide headless setup`

5. **Camera software (libcamera / rpicam-hello)** — official docs for testing the camera from the command line before writing any code.
   https://www.raspberrypi.com/documentation/computers/camera_software.html
   fallback search: `raspberry pi official docs camera software libcamera`

6. **Arducam IMX296 setup** — the `config.txt` changes the Pi 5 needs for this camera (`camera_auto_detect=0`, `dtoverlay=imx296`).
   https://docs.arducam.com/Raspberry-Pi-Camera/Native-camera/Global-Shutter/1.58MP-IMX296/
   fallback search: `arducam 1.58MP IMX296 raspberry pi wiki`

7. **Picamera2 (Python camera library)** — what the `CameraCapture` class actually uses to grab frames from Python.
   Manual: https://datasheets.raspberrypi.com/camera/picamera2-manual.pdf
   fallback search: `picamera2 manual pdf raspberry pi`

   Source and examples: https://github.com/raspberrypi/picamera2
   fallback search: `raspberrypi picamera2 github`

8. **Python venv** — isolating project packages from system Python (required on Bookworm).
   https://docs.python.org/3/library/venv.html
   fallback search: `python docs venv`

9. **SQLite3** — for local buffering on the Pi when Wi-Fi drops.
   https://docs.python.org/3/library/sqlite3.html
   fallback search: `python docs sqlite3`

---

## Step 2 — Networking & Reliability

10. **Tailscale** — secure remote access to the Pi without exposing SSH to the open internet.
    https://tailscale.com
    fallback search: `tailscale official site raspberry pi`

    Walkthrough: https://pimylifeup.com/raspberry-pi-tailscale/
    fallback search: `pi my life up installing tailscale raspberry pi`

11. **systemd services** — running the capture script persistently, auto-restart on crash, auto-start on boot.
    Official reference: https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
    fallback search: `freedesktop systemd.service manual`

    Friendlier intro: https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files
    fallback search: `digitalocean understanding systemd units and unit files`

---

## Step 3 — Computer Vision (core of the project)

*Marker-based tracking is the primary method — two ArUco markers per foot give position and a precise heel-to-toe angle (see `findings.md`, Accuracy Budget). BlazePose and classical CV are secondary cross-checks.*

12. **OpenCV ArUco marker detection (primary method — start here)** — detecting markers and refining their corners, the core technique behind the `MarkerDetector` class.
    https://docs.opencv.org/4.x/d5/dae/tutorial_aruco_detection.html
    fallback search: `opencv detection of aruco markers tutorial official docs`

13. **OpenCV camera (lens) calibration** — removing lens distortion with a printed checkerboard, required before the homography is accurate.
    https://docs.opencv.org/3.4/dc/dbb/tutorial_py_calibration.html
    fallback search: `opencv camera calibration python tutorial`

14. **OpenCV homography** — the math that converts a corrected pixel coordinate into a real-world position on the board.
    Concepts: https://docs.opencv.org/4.13.0/d9/dab/tutorial_homography.html
    fallback search: `opencv basic concepts of homography tutorial`

    Applied example: https://docs.opencv.org/4.13.0/d1/de0/tutorial_py_feature_homography.html
    fallback search: `opencv feature matching homography python tutorial`

15. **MediaPipe Pose Landmarker (secondary cross-check)** — the 33-keypoint BlazePose model, including where `heel` and `foot_index` landmarks sit. A markerless position cross-check, not for the ±1° angle.
    https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker
    fallback search: `mediapipe pose landmarker guide google ai edge`

    Background: https://research.google/blog/on-device-real-time-body-pose-tracking-with-mediapipe-blazepose/
    fallback search: `blazepose on-device real-time body pose tracking google research blog`

    Installing on Raspberry Pi OS Bookworm: https://randomnerdtutorials.com/install-mediapipe-raspberry-pi/
    fallback search: `random nerd tutorials install mediapipe raspberry pi`

    Troubleshooting thread: https://forums.raspberrypi.com/viewtopic.php?t=367296
    fallback search: `raspberry pi forums how install mediapipe bookworm`

16. **Classical CV foot segmentation + orientation (markerless cross-check)**
    Background subtraction: https://docs.opencv.org/3.4.20/d8/d38/tutorial_bgsegm_bg_subtraction.html
    fallback search: `opencv background subtraction tutorial official docs`

    Contour properties + `fitEllipse` (position and orientation): https://docs.opencv.org/4.13.0/d1/d32/tutorial_py_contour_properties.html
    fallback search: `opencv contour properties fitEllipse tutorial`

    PCA-based orientation (alternative to fitEllipse): https://docs.opencv.org/3.4/d1/dee/tutorial_introduction_to_pca.html
    fallback search: `opencv introduction to pca orientation tutorial`

    Worked example (extreme points): https://pyimagesearch.com/2016/04/11/finding-extreme-points-in-contours-with-opencv/
    fallback search: `pyimagesearch finding extreme points in contours with opencv`

---

## Step 4 — Software Design (for implementing the class architecture)

17. **Python abstract base classes / interfaces** — needed to implement the `FootDetector` interface, so `MarkerDetector`, `PoseDetector`, and future detectors are swappable without editing `CaptureService`.
    Official: https://docs.python.org/3/library/abc.html
    fallback search: `python docs abc abstract base classes`

    Friendlier walkthrough (ABCs vs. Protocols): https://realpython.com/python-interface/
    fallback search: `real python implementing interfaces abcs and protocols`

---

## Step 5 — Database (Supabase)

*Chosen over Firebase because the data model (Athlete, Session, CoachTarget, Attempt, linked by `diver_id`/`session_id`) is relational — Supabase is hosted Postgres, a natural fit for foreign keys and joins, unlike Firestore's document model.*

18. **Getting Started (official)** — Supabase overview and core concepts.
    https://supabase.com/docs/guides/getting-started
    fallback search: `supabase docs getting started`

19. **Python client (`supabase-py`)** — what the Pi's `CloudSyncClient` uses to upload attempts and read the active session.
    https://github.com/supabase-community/supabase-py/blob/main/README.md
    fallback search: `supabase-py python client github readme`

20. **Storage Quickstart** — for uploading attempt thumbnails.
    https://supabase.com/docs/guides/storage/quickstart
    fallback search: `supabase storage quickstart`

*Realtime (the app's live view) is app-side work — read it when building the React Native app, starting from the docs hub at https://supabase.com/docs. The Pi doesn't use Realtime; it polls the active session.*

---

## Optional Background Reading
- **DiveNet paper** (Murthy et al., IEEE Access 2023) — the closest published research to this project (static camera + per-dive homography for physical parameter extraction). No usable open-source code or dataset was found (see `findings.md`), so this is conceptual background only.
  fallback search: `DiveNet dive action localization physical pose parameter extraction`
