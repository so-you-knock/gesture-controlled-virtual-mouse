# 🖱️ Gesture-Controlled Virtual Mouse

**Control your OS cursor with hand gestures via webcam — no API keys, no accounts, just a camera.**

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![OpenCV](https://img.shields.io/badge/OpenCV-5C3EE8?style=flat&logo=opencv&logoColor=white)
![MediaPipe](https://img.shields.io/badge/MediaPipe-0097A7?style=flat&logo=google&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green.svg)

---

## What it does

| Gesture | Action |
|---|---|
| Index finger up, others curled | Move cursor |
| Thumb + index pinch | Left click |
| Thumb + middle pinch | Right click |
| Index + middle up together ("peace sign") | Scroll |
| Hold the left-click pinch and move | Drag |

## The story behind this build

Every "virtual mouse with OpenCV" tutorial online uses `mediapipe.solutions.hands` — the **legacy** MediaPipe API. The current `mediapipe` package has fully removed it; only the newer **Tasks API** (`HandLandmarker`) ships now. This project hit that `AttributeError`, dug into what the installed package actually contains, and migrated to the current API — a genuine debugging story, not a copy-pasted tutorial that happens to still work.

## Architecture

```
Webcam (cv2.VideoCapture)
        │
        ▼
MediaPipe HandLandmarker (Tasks API, VIDEO running mode)
        │  21 (x, y) landmarks per detected hand
        ▼
gestures.py — pure functions, zero camera/mouse dependency:
   - finger_is_up()       per-finger relative comparison (tip vs PIP joint)
   - classify_gesture()   move / left_click / right_click / scroll / drag
   - map_to_screen()      shrunk active-region → full screen mapping
   - EMASmoother          exponential moving average, damps jitter
        │
        ▼
pyautogui — actual OS mouse control
```

**Why the split:** you can't unit-test "did the webcam see a hand" without real hardware — but gesture classification and coordinate math are pure functions. Pulling them into their own module means **10 unit tests run with no camera attached at all** (`test_gestures.py`), and all 10 pass.

## Key engineering decisions

- **Coordinate mapping, not 1:1 pixel mapping** — the active region is shrunk by a margin so a comfortable hand movement near center-frame covers the whole screen, instead of forcing your hand to the camera's literal edge to hit screen corners.
- **EMA smoothing** on cursor position — raw fingertip coordinates jitter frame-to-frame even when your hand is perfectly still; exponential moving average damps that without lagging behind real movement the way a plain average would.
- **Click debouncing / hold-to-drag** — a state machine (`prev_gesture`, `cooldown_counter`) ensures a single pinch fires exactly one click instead of ten, and a *held* pinch escalates cleanly into a drag.
- **Graceful degradation** — if `pyautogui` can't attach to a display (e.g. a headless machine), the app still runs and shows hand tracking + gesture labels, it just skips the OS mouse calls instead of crashing.

## Verified live

Tested with a real webcam: correctly tracked 21 hand landmarks, classified gestures accurately in real time at ~23 FPS, and moved the actual OS cursor.

## Tech stack

Python · OpenCV · MediaPipe (Tasks API) · NumPy · PyAutoGUI

## Project structure

```
gestures.py          Pure gesture-classification and coordinate-mapping logic
test_gestures.py     10 unit tests — no camera required
virtual_mouse.py      Webcam capture, hand tracking, OS mouse control
requirements.txt
```

## Run it

```bash
pip install -r requirements.txt
curl -o hand_landmarker.task -L https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task
python3 test_gestures.py    # optional — 10/10 tests, no camera needed
python3 virtual_mouse.py    # the actual app — needs a webcam
```

## License

MIT
