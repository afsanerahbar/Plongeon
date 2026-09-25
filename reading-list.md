# Reading List — Step-by-Step Prep Order

For Charles (no prior Raspberry Pi / programming experience) working toward building the diving foot-placement tracker described in `findings.md`. Work through these roughly in order — later steps assume earlier ones are comfortable, not just skimmed.

Links are plain text (not markdown-linked) since they need to be copy/pasted, not clicked. A fallback search term is given for each in case a URL has moved.

---

## Step 0 — Absolute Beginner (start here if you've never used a Pi before)

1. **The Official Raspberry Pi Beginner's Guide (free PDF)** — unboxing, first boot, basic OS navigation, zero assumed knowledge.
   https://magazines-static.raspberrypi.org/books/full_pdfs/000/000/005/original/Beginners_Guide_v1.pdf
   fallback search: `official raspberry pi beginner's guide free pdf`

2. **Raspberry Pi Foundation — Introduction to Python pathway** — learn Python from scratch (variables, loops, functions), no Pi hardware needed yet.
   https://projects.raspberrypi.org/en/pathways/python-intro
   fallback search: `raspberry pi foundation intro to python pathway`

3. **Raspberry Pi Foundation — Introduction to Physical Computing with Raspberry Pi** — bridges from software to hardware (sensors, camera reacting to the physical world). Closest official material to what this project actually needs.
   https://www.raspberrypi.org/courses/learn-python
   fallback search: `raspberry pi foundation introduction to physical computing course`

---

## Step 1 — Pi Fundamentals & Setup

4. **Headless Pi setup** — configuring SSH/Wi-Fi at flash time via Raspberry Pi Imager, no monitor needed.
   https://raspberrypi-guide.github.io/getting-started/raspberry-pi-headless-setup
   fallback search: `raspberry pi guide headless setup`

5. **Camera software (libcamera / rpicam-hello)** — official docs for testing the camera module before writing any code.
   https://www.raspberrypi.com/documentation/computers/camera_software.html
   fallback search: `raspberry pi official docs camera software libcamera`

6. **Python venv** — isolating project packages from system Python (required on Bookworm).
   https://docs.python.org/3/library/venv.html
   fallback search: `python docs venv`

7. **SQLite3** — for local buffering on the Pi when Wi-Fi drops.
   https://docs.python.org/3/library/sqlite3.html
   fallback search: `python docs sqlite3`

---

## Step 2 — Networking & Reliability

8. **Tailscale** — secure remote access to the Pi without exposing SSH to the open internet.
   https://tailscale.com
   fallback search: `tailscale official site raspberry pi`

   Walkthrough: https://pimylifeup.com/raspberry-pi-tailscale/
   fallback search: `pi my life up installing tailscale raspberry pi`

9. **systemd services** — running the capture script persistently, auto-restart on crash, auto-start on boot.
   Official reference: https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
   fallback search: `freedesktop systemd.service manual`

   Friendlier intro: https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files
   fallback search: `digitalocean understanding systemd units and unit files`

---

## Step 3 — Computer Vision & Pose Detection (core of the project)

*(Reordered 2026-09-25: marker-based tracking is now the primary method — see `findings.md` "Requirement Update" — because it gives position AND orientation together, which BlazePose's landmarks alone can't do reliably to ±1°. BlazePose moves to a secondary/cross-check role.)*

10. **OpenCV ArUco marker detection & pose estimation (primary method — start here)** — detecting markers and reading their position + rotation, the core technique behind the `MarkerDetector` class.
    https://docs.opencv.org/4.x/d5/dae/tutorial_aruco_detection.html
    fallback search: `opencv detection of aruco markers tutorial official docs`

11. **OpenCV homography** — the math that converts a detected pixel coordinate (marker or landmark) into a real-world position on the board.
    Concepts: https://docs.opencv.org/4.13.0/d9/dab/tutorial_homography.html
    fallback search: `opencv basic concepts of homography tutorial`

    Applied example: https://docs.opencv.org/4.13.0/d1/de0/tutorial_py_feature_homography.html
    fallback search: `opencv feature matching homography python tutorial`

12. **MediaPipe Pose Landmarker (secondary/cross-check)** — the 33-keypoint BlazePose model, including where `heel` and `foot_index` landmarks sit. Useful as a markerless position cross-check, not for the ±1° orientation requirement.
    https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker
    fallback search: `mediapipe pose landmarker guide google ai edge`

    Background: https://research.google/blog/on-device-real-time-body-pose-tracking-with-mediapipe-blazepose/
    fallback search: `blazepose on-device real-time body pose tracking google research blog`

    Installing on Raspberry Pi OS Bookworm (required — MediaPipe isn't yet compatible with the newest Pi OS/Trixie/Python 3.13): https://randomnerdtutorials.com/install-mediapipe-raspberry-pi/
    fallback search: `random nerd tutorials install mediapipe raspberry pi`

    Troubleshooting thread: https://forums.raspberrypi.com/viewtopic.php?t=367296
    fallback search: `raspberry pi forums how install mediapipe bookworm`

13. **Classical CV foot segmentation + orientation (Phase 1.5 markerless cross-check)**
    Background subtraction: https://docs.opencv.org/3.4.20/d8/d38/tutorial_bgsegm_bg_subtraction.html
    fallback search: `opencv background subtraction tutorial official docs`

    Contour properties + `fitEllipse` (for position and orientation): https://docs.opencv.org/4.13.0/d1/d32/tutorial_py_contour_properties.html
    fallback search: `opencv contour properties fitEllipse tutorial`

    PCA-based orientation (alternative to fitEllipse): https://docs.opencv.org/3.4/d1/dee/tutorial_introduction_to_pca.html
    fallback search: `opencv introduction to pca orientation tutorial`

    Worked example (extreme points): https://pyimagesearch.com/2016/04/11/finding-extreme-points-in-contours-with-opencv/
    fallback search: `pyimagesearch finding extreme points in contours with opencv`

---

## Step 4 — Software Design (for implementing the class architecture)

14. **Python abstract base classes / interfaces** — needed to implement the `FootDetector` interface from the SOLID-revised class design, so `MarkerDetector`/`PoseDetector`/future detectors are swappable without editing `CaptureService`.
    Official: https://docs.python.org/3/library/abc.html
    fallback search: `python docs abc abstract base classes`

    Friendlier walkthrough (ABCs vs. Protocols): https://realpython.com/python-interface/
    fallback search: `real python implementing interfaces abcs and protocols`

---

## Optional Background Reading
- **DiveNet paper** (Murthy et al., IEEE Access 2023) — the closest published research to this project (static camera + per-dive homography for physical parameter extraction). No usable open-source code or dataset was found (see `findings.md`), so this is conceptual background only, not something to install.
  fallback search: `DiveNet dive action localization physical pose parameter extraction`
