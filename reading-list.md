# Reading List — by Project Phase

Read each phase's list before starting that phase (see "Project Breakdown" in `findings.md`). The independent study's deliverable is Phases 0–1, with Phase 2 as a stretch goal; later phases are future work.

**OpenCV version note:** use OpenCV **4.7 or later**, and read the **4.x** docs. The ArUco code changed in 4.7 (detection now goes through `cv2.aruco.ArucoDetector`, and the old per-marker pose function was removed in favor of `cv2.solvePnP`). Older tutorials and Stack Overflow answers use the removed functions and fail with confusing errors.

---

## Before Phase 0 — Absolute Beginner

1. **The Official Raspberry Pi Beginner's Guide (free PDF)** — unboxing, first boot, basic OS navigation, zero assumed knowledge.
   https://magazines-static.raspberrypi.org/books/full_pdfs/000/000/005/original/Beginners_Guide_v1.pdf
   fallback search: `official raspberry pi beginner's guide free pdf`

2. **Raspberry Pi Foundation — Introduction to Python pathway** — learn Python from scratch (variables, loops, functions).
   https://projects.raspberrypi.org/en/pathways/python-intro
   fallback search: `raspberry pi foundation intro to python pathway`

3. **The Linux command line for beginners (Ubuntu)** — written for people with no command-line experience. The Pi is used entirely from a terminal, so this comes first.
   https://ubuntu.com/tutorials/command-line-for-beginners
   fallback search: `ubuntu tutorial linux command line for beginners`

4. **Git — Pro Git book, chapters 1–2** — saving work in small commits from day one. The full book is free; the first two chapters are enough to start.
   https://git-scm.com/book/en/v2
   fallback search: `pro git book free online`

---

## Phase 0 — Feasibility

5. **Headless Pi setup** — SSH and Wi-Fi set at flash time with Raspberry Pi Imager, no monitor needed. Choose **Raspberry Pi OS Bookworm**, not the newest Trixie.
   https://raspberrypi-guide.github.io/getting-started/raspberry-pi-headless-setup
   fallback search: `raspberry pi guide headless setup`

6. **Arducam IMX296 setup** — the `config.txt` changes the Pi 5 needs for this camera (`camera_auto_detect=0`, `dtoverlay=imx296`).
   https://docs.arducam.com/Raspberry-Pi-Camera/Native-camera/Global-Shutter/1.58MP-IMX296/
   fallback search: `arducam 1.58MP IMX296 raspberry pi wiki`

7. **Camera software (rpicam-hello / rpicam-vid)** — test the camera and record raw clips from the command line before writing code.
   https://www.raspberrypi.com/documentation/computers/camera_software.html
   fallback search: `raspberry pi official docs camera software libcamera`

8. **Picamera2 (Python camera library)** — capturing and recording frames from Python; becomes `CameraCapture` and `ClipRecorder` in Phase 1.
   Manual: https://datasheets.raspberrypi.com/camera/picamera2-manual.pdf
   fallback search: `picamera2 manual pdf raspberry pi`

   Source and examples: https://github.com/raspberrypi/picamera2
   fallback search: `raspberrypi picamera2 github`

9. **Python venv** — isolating project packages from system Python (required on Bookworm).
   https://docs.python.org/3/library/venv.html
   fallback search: `python docs venv`

10. **JupyterLab** — the notebook used on a laptop to analyze Phase 0 recordings.
    https://jupyter.org/install
    fallback search: `jupyterlab install`

11. **OpenCV ArUco marker detection** — detecting the foot and board markers and refining their corners.
    https://docs.opencv.org/4.x/d5/dae/tutorial_aruco_detection.html
    fallback search: `opencv 4.x detection of aruco markers tutorial`

12. **OpenCV camera (lens) calibration** — removing lens distortion with a printed checkerboard.
    https://docs.opencv.org/4.x/dc/dbb/tutorial_py_calibration.html
    fallback search: `opencv 4.x camera calibration python tutorial`

13. **Camera pose with solvePnP** — finding where the board is relative to the camera in each frame, so markers can be placed at their real height above it (findings.md, issues 1–2). This is the core geometry of the project.
    Official reference (methods and what they need): https://docs.opencv.org/4.x/d5/d1f/calib3d_solvePnP.html
    fallback search: `opencv 4.x perspective-n-point pnp pose computation`

    Source of that page, if the docs site is slow: https://github.com/opencv/opencv/blob/4.x/modules/calib3d/doc/solvePnP.markdown

    Worked example (head pose, same technique, updated for recent OpenCV): https://learnopencv.com/head-pose-estimation-using-opencv-and-dlib/
    fallback search: `learnopencv head pose estimation solvepnp`

14. **OpenCV homography** — mapping image points onto a flat surface. Read it to understand why a flat mapping misplaces markers that sit above the board (findings.md, issue 1).
    Concepts: https://docs.opencv.org/4.x/d9/dab/tutorial_homography.html
    fallback search: `opencv 4.x basic concepts of homography tutorial`

    Applied example: https://docs.opencv.org/4.x/d1/de0/tutorial_py_feature_homography.html
    fallback search: `opencv 4.x feature matching homography python tutorial`

---

## Phase 1 — Offline pipeline (deliverable)

15. **Python abstract base classes** — needed for the one interface in Phase 1, `FrameSource`.
    Official: https://docs.python.org/3/library/abc.html
    fallback search: `python docs abc abstract base classes`

    Friendlier walkthrough (ABCs vs. Protocols): https://realpython.com/python-interface/
    fallback search: `real python implementing interfaces abcs and protocols`

16. **Python dataclasses** — the data objects (`Frame`, `FootKeypoints`, `BoardPose`, …) are dataclasses, a few lines each.
    https://docs.python.org/3/library/dataclasses.html
    fallback search: `python docs dataclasses`

17. **SQLite3** — local storage of attempt records (`LocalBuffer`).
    https://docs.python.org/3/library/sqlite3.html
    fallback search: `python docs sqlite3`

---

## Phase 2 — Cloud + simple dashboard (stretch goal)

*Supabase was chosen over Firebase because the data model (Athlete, Session, CoachTarget, Attempt, linked by `diver_id`/`session_id`) is relational — Supabase is hosted Postgres.*

18. **Supabase — Getting Started** — overview and core concepts.
    https://supabase.com/docs/guides/getting-started
    fallback search: `supabase docs getting started`

19. **Python client (`supabase-py`)** — what `CloudSyncClient` uses to upload attempts.
    https://github.com/supabase-community/supabase-py/blob/main/README.md
    fallback search: `supabase-py python client github readme`

20. **Supabase Storage Quickstart** — uploading attempt thumbnails.
    https://supabase.com/docs/guides/storage/quickstart
    fallback search: `supabase storage quickstart`

21. **Streamlit** — building the coach's web dashboard in Python.
    https://docs.streamlit.io/get-started
    fallback search: `streamlit get started docs`

---

## Future work (Phases 3–5)

**Phase 3 — Real-time on the Pi**
- **systemd services** — running `main.py` on boot and restarting it after a crash.
  https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html (official); friendlier intro: https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files
- **Tailscale** — secure remote access to the installed Pi.
  https://tailscale.com; walkthrough: https://pimylifeup.com/raspberry-pi-tailscale/

**Phase 4 — Mobile app:** React Native and Supabase's JavaScript client; start from https://supabase.com/docs.

**Phase 5 — Markerless (optional)**
- **MediaPipe Pose Landmarker** (BlazePose, including `heel` and `foot_index`): https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker. Installs on Bookworm, not Trixie: https://randomnerdtutorials.com/install-mediapipe-raspberry-pi/
- **Classical CV:**
  - Background subtraction: https://docs.opencv.org/4.x/d1/dc5/tutorial_background_subtraction.html
  - Contour properties + `fitEllipse`: https://docs.opencv.org/4.x/d1/d32/tutorial_py_contour_properties.html
  - PCA-based orientation: https://docs.opencv.org/4.x/d1/dee/tutorial_introduction_to_pca.html

---

## Optional Background Reading
- **DiveNet paper** (Murthy et al., IEEE Access 2023) — the closest published research (static camera, board mapping recomputed per dive). No usable open-source code or dataset was found (see `findings.md`), so this is conceptual background only.
  fallback search: `DiveNet dive action localization physical pose parameter extraction`
