# Reading List — by Project Phase

Read each phase's list before starting that phase (see "Project Breakdown" in `findings.md`). Phase 0 is the longest because it covers the basics everything else builds on.

---

## Before Phase 0 — Absolute Beginner (start here if you've never used a Pi or Python)

1. **The Official Raspberry Pi Beginner's Guide (free PDF)** — unboxing, first boot, basic OS navigation, zero assumed knowledge.
   https://magazines-static.raspberrypi.org/books/full_pdfs/000/000/005/original/Beginners_Guide_v1.pdf
   fallback search: `official raspberry pi beginner's guide free pdf`

2. **Raspberry Pi Foundation — Introduction to Python pathway** — learn Python from scratch (variables, loops, functions).
   https://projects.raspberrypi.org/en/pathways/python-intro
   fallback search: `raspberry pi foundation intro to python pathway`

3. **Raspberry Pi Foundation — Physical Computing with Python** — Python code reacting to devices connected to the Pi.
   https://github.com/raspberrypilearning/physical-computing-with-python
   fallback search: `raspberrypilearning physical computing with python`

---

## Phase 0 — Feasibility

4. **Headless Pi setup** — SSH and Wi-Fi set at flash time with Raspberry Pi Imager, no monitor needed. Choose **Raspberry Pi OS Bookworm**, not the newest Trixie.
   https://raspberrypi-guide.github.io/getting-started/raspberry-pi-headless-setup
   fallback search: `raspberry pi guide headless setup`

5. **Arducam IMX296 setup** — the `config.txt` changes the Pi 5 needs for this camera (`camera_auto_detect=0`, `dtoverlay=imx296`).
   https://docs.arducam.com/Raspberry-Pi-Camera/Native-camera/Global-Shutter/1.58MP-IMX296/
   fallback search: `arducam 1.58MP IMX296 raspberry pi wiki`

6. **Camera software (rpicam-hello / rpicam-vid)** — test the camera and record raw clips from the command line before writing code.
   https://www.raspberrypi.com/documentation/computers/camera_software.html
   fallback search: `raspberry pi official docs camera software libcamera`

7. **Picamera2 (Python camera library)** — capturing and recording frames from Python; becomes `CameraCapture` and `ClipRecorder` in Phase 1.
   Manual: https://datasheets.raspberrypi.com/camera/picamera2-manual.pdf
   fallback search: `picamera2 manual pdf raspberry pi`

   Source and examples: https://github.com/raspberrypi/picamera2
   fallback search: `raspberrypi picamera2 github`

8. **Python venv** — isolating project packages from system Python (required on Bookworm).
   https://docs.python.org/3/library/venv.html
   fallback search: `python docs venv`

9. **JupyterLab** — the notebook used on a laptop to analyze Phase 0 recordings.
   https://jupyter.org/install
   fallback search: `jupyterlab install`

10. **OpenCV ArUco marker detection** — detecting the four foot markers and refining their corners.
    https://docs.opencv.org/4.x/d5/dae/tutorial_aruco_detection.html
    fallback search: `opencv detection of aruco markers tutorial official docs`

11. **OpenCV camera (lens) calibration** — removing lens distortion with a printed checkerboard.
    https://docs.opencv.org/3.4/dc/dbb/tutorial_py_calibration.html
    fallback search: `opencv camera calibration python tutorial`

12. **OpenCV homography** — converting a corrected pixel coordinate into a position on the board in millimeters.
    Concepts: https://docs.opencv.org/4.13.0/d9/dab/tutorial_homography.html
    fallback search: `opencv basic concepts of homography tutorial`

    Applied example: https://docs.opencv.org/4.13.0/d1/de0/tutorial_py_feature_homography.html
    fallback search: `opencv feature matching homography python tutorial`

---

## Phase 1 — Offline pipeline

13. **Python abstract base classes / interfaces** — needed for the three interfaces (`FrameSource`, `FootDetector`, `SessionProvider`).
    Official: https://docs.python.org/3/library/abc.html
    fallback search: `python docs abc abstract base classes`

    Friendlier walkthrough (ABCs vs. Protocols): https://realpython.com/python-interface/
    fallback search: `real python implementing interfaces abcs and protocols`

14. **SQLite3** — local storage of attempt records (`LocalBuffer`).
    https://docs.python.org/3/library/sqlite3.html
    fallback search: `python docs sqlite3`

---

## Phase 2 — Cloud + simple dashboard

*Supabase was chosen over Firebase because the data model (Athlete, Session, CoachTarget, Attempt, linked by `diver_id`/`session_id`) is relational — Supabase is hosted Postgres, a natural fit for foreign keys and joins.*

15. **Supabase — Getting Started** — overview and core concepts.
    https://supabase.com/docs/guides/getting-started
    fallback search: `supabase docs getting started`

16. **Python client (`supabase-py`)** — what `CloudSyncClient` uses to upload attempts and read sessions.
    https://github.com/supabase-community/supabase-py/blob/main/README.md
    fallback search: `supabase-py python client github readme`

17. **Supabase Storage Quickstart** — uploading attempt thumbnails.
    https://supabase.com/docs/guides/storage/quickstart
    fallback search: `supabase storage quickstart`

18. **Streamlit** — building the coach's web dashboard in Python.
    https://docs.streamlit.io/get-started
    fallback search: `streamlit get started docs`

---

## Phase 3 — Real-time on the Pi

19. **systemd services** — running `main.py` on boot and restarting it after a crash.
    Official reference: https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
    fallback search: `freedesktop systemd.service manual`

    Friendlier intro: https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files
    fallback search: `digitalocean understanding systemd units and unit files`

20. **Tailscale** — secure remote access to the installed Pi without exposing SSH to the internet.
    https://tailscale.com
    fallback search: `tailscale official site raspberry pi`

    Walkthrough: https://pimylifeup.com/raspberry-pi-tailscale/
    fallback search: `pi my life up installing tailscale raspberry pi`

---

## Phase 4 — Mobile app

*React Native and Supabase's JavaScript client (including Realtime for the live view) — start from the Supabase docs hub at https://supabase.com/docs when this phase begins.*

---

## Phase 5 — Optional: markerless

21. **MediaPipe Pose Landmarker** — the 33-keypoint BlazePose model, including `heel` and `foot_index`. A position cross-check, not precise enough for the ±1° angle.
    https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker
    fallback search: `mediapipe pose landmarker guide google ai edge`

    Background: https://research.google/blog/on-device-real-time-body-pose-tracking-with-mediapipe-blazepose/
    fallback search: `blazepose on-device real-time body pose tracking google research blog`

    Installing on Raspberry Pi OS Bookworm (it doesn't install on Trixie): https://randomnerdtutorials.com/install-mediapipe-raspberry-pi/
    fallback search: `random nerd tutorials install mediapipe raspberry pi`

    Troubleshooting thread: https://forums.raspberrypi.com/viewtopic.php?t=367296
    fallback search: `raspberry pi forums how install mediapipe bookworm`

22. **Classical CV foot segmentation + orientation**
    Background subtraction: https://docs.opencv.org/3.4.20/d8/d38/tutorial_bgsegm_bg_subtraction.html
    fallback search: `opencv background subtraction tutorial official docs`

    Contour properties + `fitEllipse`: https://docs.opencv.org/4.13.0/d1/d32/tutorial_py_contour_properties.html
    fallback search: `opencv contour properties fitEllipse tutorial`

    PCA-based orientation: https://docs.opencv.org/3.4/d1/dee/tutorial_introduction_to_pca.html
    fallback search: `opencv introduction to pca orientation tutorial`

    Worked example (extreme points): https://pyimagesearch.com/2016/04/11/finding-extreme-points-in-contours-with-opencv/
    fallback search: `pyimagesearch finding extreme points in contours with opencv`

---

## Optional Background Reading
- **DiveNet paper** (Murthy et al., IEEE Access 2023) — the closest published research (static camera + per-dive homography). No usable open-source code or dataset was found (see `findings.md`), so this is conceptual background only.
  fallback search: `DiveNet dive action localization physical pose parameter extraction`
