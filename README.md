# CCTV Monitoring with YOLOv8 and Polygon-Based Detection

**Applied computer vision / automation prototype · Python, OpenCV, Selenium, Ultralytics YOLOv8**

This repository contains a research/prototype notebook for analyzing CCTV imagery when the camera feed is available through a **web-based viewer** instead of a conventional stream API. The workflow uses Selenium to capture rendered browser screenshots, YOLOv8 to detect people, and OpenCV polygon masks to decide whether a detection lies in a designated image region. It also includes basic centroid tracking, screenshot archiving, connection checks, and a prototype notification API integration.

> **Status:** Historical implementation / proof of concept, **not a standalone surveillance product**, security-certified intrusion detector, or verified production deployment. The notebook is provided as an example of engineering around a constrained video interface. Its original environment-specific configuration must be replaced before running it elsewhere.

## Motivation

Standard computer-vision pipelines often expect a direct RTSP/video file input. Here, the source workflow relied on a CCTV interface shown in a web browser. Rather than integrating directly with the camera backend, the prototype used **browser screenshot acquisition** as an input to object detection.

This approach avoids assuming that direct access to the underlying video stream is available, but it introduces costs and limitations: browser rendering overhead, screenshot latency, window scaling, UI changes, and reduced control over frame timing.

## System flow

```text
Web-based CCTV viewer
        |
        v
Selenium / Chrome screenshot
        |
        v
Decode image with OpenCV
        |
        +--> Load or manually define four-point polygon regions
        |
        v
YOLOv8 person detection (class 0)
        |
        v
Bounding-box centroids + polygon mask membership
        |
        +--> Visual annotations and per-frame counters
        |
        +--> CentroidTracker.update(...) [experimental track IDs]
        |
        +--> Conditional alarm/connection-check HTTP integration
        |
        v
Saved/processed screenshots and on-screen display
```

The pipeline works on **screenshots of the displayed CCTV page**, rather than directly decoding the original stream.

## Main features visible in the code

### Browser-based frame acquisition

A Selenium `webdriver.Chrome()` instance opens a configured web-viewer URL, waits for rendering, and captures screenshots with `get_screenshot_as_png()`. OpenCV decodes PNG bytes into NumPy image arrays.

**Implications:** The camera viewer must already be reachable and authorized; Selenium/Chrome and an appropriate browser environment must be installed. Actual sample rate depends on browser and inference overhead.

### Person detection

The prototype initializes `YOLO('yolov8x.pt')` and calls `model.predict(..., classes=[0])`, requesting the **person** class. For each detected box, it calculates a center point and draws a bounding box.

The code does not show training or fine-tuning of a custom YOLOv8 model. Model weights are an external dependency and are **not stored here**.

### Manually defined detection regions

The user can click four image points for each polygon in an OpenCV window. Polygon coordinates are saved to, or loaded from, `polygon_coordinates.csv`, using `polygon_id`, `point_id`, `x`, and `y` fields. The script creates an image mask with `cv2.fillPoly`.

It classifies a detected person as inside the monitored area when the **bounding-box centroid** falls in the mask. This is a deliberate simplification: a person partially overlapping the polygon may be counted outside when the centroid is outside it.

### Centroid tracking and counts

A `CentroidTracker` class uses centroid-distance assignment (`scipy.spatial.distance.cdist`) and a disappearance counter to maintain tentative track IDs. However, the on-screen `In Area` and `Outside Area` metrics are **per-frame detection counts**. The code should **not** be described as a robust line-crossing tracker or unique-person entry counter; the presence of a detection inside the polygon is sufficient for its event count. Occlusion and identity switches are not comprehensively handled.

### Alerts and connectivity checks

The prototype includes functions for camera reachability checks and posting an alert with a screenshot to a configured HTTP endpoint. The code can trigger an alert when in-area detections occur and uses a time interval for connection checks.

This is an integration prototype and should not be interpreted as a proven service-level agreement, guaranteed alert delivery system, or hardened incident-response workflow.

### Image retention and display

The script saves raw and annotated screenshots to local directories, removes older files from selected folders (keeping the latest five under its cleanup rule), and displays annotated frames in an OpenCV window. This retention policy is **local and partial**, not an overall privacy/data-retention assurance.

## Repository files

| File | Description |
| --- | --- |
| [`yolov8_cctv_ai_polygon_revision.ipynb`](yolov8_cctv_ai_polygon_revision.ipynb) | Single source notebook containing the implementation: Chrome screenshot capture, YOLOv8 detection, polygon setup, centroid tracking, alert calls, and display loop |
| [`README.md`](README.md) | Architecture, behavior, environment, limitations and ethical-use notes |
| [`.gitignore`](.gitignore) | Basic exclusions for local artifacts |

Despite the `.ipynb` format, the main uploaded notebook is essentially a **single long Python code cell**, not a tutorial with pre-executed examples.

## Software dependencies and setup

The source imports **OpenCV, NumPy, requests, pythonping, Selenium, Ultralytics, SciPy, and standard-library modules**. It also requires Chrome/compatible WebDriver and a suitable YOLOv8 model weight file.

Suggested starting environment for an **authorized, isolated test**:

```bash
python -m venv .venv
# Activate the environment
python -m pip install numpy opencv-python scipy requests pythonping selenium ultralytics notebook
jupyter notebook
```

These are indicative dependencies; exact tested versions are not recorded. You must supply your own approved test footage/viewer, configure permitted network endpoints and local paths, and ensure the GUI can display OpenCV windows. The original implementation may assume `__file__`, which is not normally defined in interactive notebooks; that path logic must be adapted before executing it in Jupyter.

**Never run the historical network calls against real equipment without authorization.** A generic sample or replay environment is preferable for academic review.

## Important technical limitations

- **Screenshot throughput:** The source declares a target 15 fps delay, but actual throughput is affected by rendering, inference, and browser overhead; no measured real-time fps result is provided.
- **Detection ≠ identity:** No strong re-identification, duplicate suppression across frames, or consistent unique-person counts.
- **Region presence ≠ crossing:** The notification condition is based on centroid presence within a polygon, not a confirmed direction-of-travel crossing.
- **Resolution sensitivity:** Saved polygons are pixel coordinates; browser resize, zoom, or UI layout changes can invalidate them.
- **Limited robustness:** Network failures, viewer layout changes, model false positives, and concurrent alert requests require additional testing and error handling.
- **No benchmark:** No labeled test dataset, precision/recall, latency distribution, false-alarm rate, or validated SLA evidence is included.
- **Sensitive configuration:** Historical source includes network-/facility-specific constants and HTTP connection details. They must be reviewed, removed or parameterized, and any exposed endpoints or credentials assessed before sharing or deploying the code.

## Responsible use and future work

Only use the system on camera footage you are authorized to process, subject to organizational privacy, security and retention policies. For a publishable demonstration, substitute a local video/mock browser viewer and **synthetic or properly licensed** imagery, place endpoint configuration in environment variables, add tests for polygon membership, measure latency and false alarms, and separate detection from the alarm-delivery service.

## Project contribution and attribution

This project showcases practical integration of computer vision with a constrained browser-viewer interface, image-region logic, and automation. **YOLOv8, OpenCV, Selenium, and SciPy are third-party libraries and not original models developed in this repository.** The repository should only be shared to the extent permitted by the original project owner and applicable policies.

---
*Project documentation describes what is visible in the current notebook; it does not claim features, results, or deployment validation not demonstrated in the source.*