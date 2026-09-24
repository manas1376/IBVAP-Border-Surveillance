# IBVAP — Intelligent Border Video Analytics Platform

### Smart India Hackathon 2026 · Problem Statement 26187

**Ministry of Home Affairs / Sashastra Seema Bal**

IBVAP is a software-based video analytics platform that I developed as a solution for **Smart India Hackathon 2026 Problem Statement 26187**.

The main idea is simple: instead of requiring expensive proprietary smart cameras or separate hardware for every AI capability, IBVAP takes video from **existing CCTV cameras, webcams, RTSP streams, mobile cameras, YouTube streams, or recorded footage** and processes it through a unified analytics pipeline.

The system combines **AI-based object detection, multi-object tracking, face recognition, ANPR, spatial rules, evidence capture, alerting, and tamper-evident event logging** into one operator-facing platform.

> **Core idea:** Convert existing video infrastructure into an intelligent surveillance system through software.

---

## What IBVAP Can Do

### AI-Based Detection & Tracking

The main analytics pipeline uses:

* **YOLO11** for object detection
* **ByteTrack** for multi-object tracking
* **OpenCV** for video processing
* **CLAHE** for improving visibility in low-light scenes

The system currently tracks:

* Person
* Bicycle
* Car
* Motorcycle
* Bus
* Train
* Truck
* Boat

Tracking gives each detected object a persistent identity across frames, allowing the platform to reason about **movement and behaviour**, rather than treating every frame independently.

---

## Rule-Based Border Surveillance

AI detection alone only tells us what is visible in a frame.

IBVAP adds a decision layer on top of those detections.

### Supported rules

| Rule            | Purpose                                           |
| --------------- | ------------------------------------------------- |
| Tripwire        | Detect directional crossing of a virtual boundary |
| Restricted Zone | Detect entry and continued presence               |
| Loiter Zone     | Detect objects remaining in an area for too long  |
| Direction Rule  | Detect movement against an allowed direction      |
| Night Movement  | Detect movement when the scene is visually dark   |

For example:

```text
Camera
   ↓
YOLO11
   ↓
Person detected
   ↓
ByteTrack
   ↓
Person #17 tracked
   ↓
Rule Engine
   ↓
Person crosses restricted fence
   ↓
Security Event
```

This separation between **AI detection** and **decision logic** is an important part of the architecture.

---

# Face Recognition

IBVAP includes an optional face-recognition pipeline using:

* **SCRFD** — face detection
* **ArcFace** — face embedding generation
* **Cosine similarity** — watchlist matching

The process is approximately:

```text
Video Frame
    ↓
SCRFD
    ↓
Face detected
    ↓
ArcFace
    ↓
Face embedding
    ↓
Compare with watchlist embeddings
    ↓
Similarity threshold
    ↓
Possible watchlist match
```

Watchlist embeddings are stored in the database and loaded when the system starts.

The platform also maintains a short-lived identity cache so the same tracked face does not need to be processed repeatedly.

---

# Automatic Number Plate Recognition

The ANPR pipeline combines vehicle detection, image processing and OCR.

### Processing flow

```text
Vehicle detected
      ↓
Plate candidate localisation
      ↓
Crop at source resolution
      ↓
Image preprocessing
      ↓
EasyOCR
      ↓
Indian plate-format validation
      ↓
Multi-frame consensus
      ↓
Confidence scoring
      ↓
Verified / uncertain plate
```

### Additional processing

The implementation includes:

* Multiple OCR preprocessing variants
* Multi-frame voting
* Indian registration-format validation
* State-code validation
* Confidence scoring
* Moving-vehicle deblurring
* Priority for vehicles approaching the frame edge
* Suppression of low-confidence evidentiary reads

This is useful because a single OCR result from a moving vehicle should not automatically be treated as reliable evidence.

---

# Video Sources

IBVAP is not tied to a particular camera manufacturer.

It can work with:

| Source         | Example                     |
| -------------- | --------------------------- |
| USB webcam     | `0`, `1`, `2`               |
| Laptop camera  | Built-in webcam             |
| Mobile camera  | DroidCam / similar          |
| RTSP camera    | Network CCTV                |
| HTTP/MJPEG     | Network video source        |
| YouTube stream | Public video source         |
| Recorded video | MP4 / supported video files |

This makes the prototype easier to demonstrate without requiring specialised surveillance hardware.

---

# System Architecture

```text
                  VIDEO SOURCES
                      │
       ┌──────────────┼──────────────┐
       │              │              │
      RTSP          Webcam        Video File
       │              │              │
       └──────────────┼──────────────┘
                      ↓
              Video Capture Layer
                      ↓
             Latest Frame Buffer
                      ↓
              Analytics Pipeline
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
     YOLO11       ByteTrack      Scene Analysis
        │             │             │
        └─────────────┼─────────────┘
                      ↓
                 Rule Engine
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
   Face Recognition             ANPR
  SCRFD + ArcFace             EasyOCR
          │                       │
          └───────────┬───────────┘
                      ↓
                Event Manager
                      │
       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
    Database        Evidence       Alerts
       │              │              │
     SQLite       Snapshot/Clip   Webhook/SMS
       │              │
       └──────────────┼──────────────┘
                      ↓
               FastAPI Backend
                      ↓
                 Dashboard
```

---

# Engineering Behind the Pipeline

One of the design decisions in IBVAP is that every camera does not continuously build an unlimited queue of frames.

Each camera uses a **single-slot frame buffer**:

```text
Capture Thread
      ↓
 latest frame
      ↓
Analytics Thread
```

If inference becomes slower than the incoming video, older frames are discarded instead of allowing memory usage to grow indefinitely.

The system therefore prioritises:

> **Fresh frames over processing every historical frame.**

### Thread model

| Component             | Responsibility                          |
| --------------------- | --------------------------------------- |
| Capture thread        | Reads and decodes camera frames         |
| Analytics thread      | Detection, tracking, rules and overlays |
| Face enrichment stage | Face detection/recognition              |
| ANPR enrichment stage | Plate recognition                       |
| Escalation worker     | Alarm/SMS/webhook dispatch              |
| FastAPI/ASGI          | HTTP, WebSocket and streaming           |

Optional face and ANPR stages are duty-limited so that adding more cameras does not allow enrichment workloads to grow without control.

---

# Technology Stack

| Layer                | Technology          | Purpose                        |
| -------------------- | ------------------- | ------------------------------ |
| Language             | Python 3.11+        | Main application               |
| Object Detection     | YOLO11              | Detect objects                 |
| Tracking             | ByteTrack           | Track objects across frames    |
| Deep Learning        | PyTorch             | Model execution                |
| Face Detection       | SCRFD               | Detect faces                   |
| Face Recognition     | ArcFace             | Generate face embeddings       |
| OCR                  | EasyOCR             | Read number plates             |
| Computer Vision      | OpenCV              | Video/image processing         |
| Numerical Processing | NumPy               | Image and numerical operations |
| Backend              | FastAPI             | REST/WebSocket API             |
| Server               | Uvicorn             | Run FastAPI                    |
| Database             | SQLite              | Application/event storage      |
| ORM                  | SQLAlchemy          | Database interaction           |
| Validation           | Pydantic            | API data validation            |
| PDF                  | ReportLab           | Event reports                  |
| Frontend             | HTML/CSS/JavaScript | Operator dashboard             |
| Testing              | pytest              | Automated tests                |
| Containerisation     | Docker              | Deployment                     |

The frontend intentionally does not depend on a large JavaScript framework or external CDN. This keeps the dashboard lightweight and suitable for an isolated/local network environment.

---

# Evidence & Event Integrity

A surveillance system should not only generate an alert; it should also preserve the evidence associated with that alert.

When a significant event occurs, IBVAP can generate:

* Snapshot
* Video clip
* Event metadata
* ANPR plate crop
* Timestamp
* Track information
* Integrity information

### Evidence flow

```text
Security Event
      ↓
Snapshot + Video Clip
      ↓
Event Record
      ↓
SHA-256 Hash
      ↓
Previous Event Hash
      ↓
Hash Chain
      ↓
Merkle Checkpoint
```

The hash-chain mechanism makes unauthorised modification easier to detect because changing or removing an earlier record affects subsequent hashes.

The platform also provides integrity verification and checkpoint operations.

---

# Alerting & Escalation

Events are assigned a severity level.

| Severity | Example                                          |
| -------- | ------------------------------------------------ |
| CRITICAL | Inbound fence crossing, watchlist match          |
| HIGH     | Restricted-zone entry, loitering, night movement |
| MEDIUM   | Outbound crossing, subsystem degradation         |
| LOW      | Zone exit, plate read                            |
| INFO     | Routine detections                               |

Alerts can be delivered through:

* Dashboard WebSocket
* Webhook
* GPIO alarm
* SMS

Network operations are handled asynchronously so that a slow notification service does not block the video analytics pipeline.

Webhook requests can also use **HMAC-SHA256 signatures** for request authentication.

---

# Dashboard

The dashboard provides an operator-oriented interface rather than exposing only raw API responses.

### Operations view

Provides:

* Multi-camera live view
* FPS
* Object count
* Pipeline latency
* Camera status
* AI overlays
* Event indicators

### Playback

Recorded sources support:

* Pause/resume
* 0.5×–2× playback speed
* Digital zoom
* Pan
* Frame review

For live cameras, pausing the dashboard display does **not** stop the underlying analytics pipeline.

### Rule editor

Operators can draw:

* Virtual fences
* Restricted areas
* Loiter zones
* Direction lines

The coordinates are stored in the same analytics coordinate system used by the rule engine.

### Event log

Operators can:

* Filter events
* Search events
* View snapshots
* View clips
* Inspect event details
* Export filtered records as PDF

---

# Offline Video Analysis

The same analytics pipeline can also be used with uploaded recordings.

```text
Upload Video
     ↓
Analysis Session
     ↓
Video Processing
     ↓
Detection + Tracking
     ↓
Rules + Recognition
     ↓
Events
     ↓
Annotated Output
```

This allows the system to be demonstrated even when a live CCTV camera is unavailable.

---

# REST API & Real-Time Communication

The backend is built with **FastAPI**.

Major API groups include:

### Cameras

```text
GET    /api/cameras
POST   /api/cameras
GET    /api/cameras/{id}
PUT    /api/cameras/{id}
DELETE /api/cameras/{id}
GET    /api/cameras/{id}/snapshot
GET    /stream/{id}
```

### Events

```text
GET /api/alerts
GET /api/alerts/{id}
GET /api/alerts/{id}/snapshot
GET /api/alerts/{id}/clip
GET /api/alerts/export.pdf
```

### Recognition

```text
GET /api/anpr/detections
GET /api/anpr/status
GET /api/faces/detections
GET /api/watchlist
POST /api/watchlist
```

### Real-time alerts

```text
WebSocket
/ws/alerts
```

### Integrity

```text
GET  /api/integrity/verify
GET  /api/integrity/tip
POST /api/integrity/checkpoint
POST /api/integrity/certificate
```

Interactive API documentation is available through FastAPI at:

```text
/docs
```

---

# Project Structure

```text
IBVAP/
│
├── api/
│   ├── main.py
│   └── schemas.py
│
├── core/
│   ├── config.py
│   ├── models.py
│   ├── database.py
│   ├── camera.py
│   ├── video_source.py
│   ├── sources.py
│   ├── analytics.py
│   ├── analysis.py
│   ├── detections.py
│   ├── events.py
│   ├── evidence.py
│   ├── hashchain.py
│   ├── report.py
│   ├── notify.py
│   ├── alarm.py
│   ├── sms.py
│   ├── youtube.py
│   └── timeutil.py
│
├── cv/
│   ├── detector.py
│   ├── rules.py
│   ├── geometry.py
│   ├── face.py
│   ├── anpr.py
│   ├── scene.py
│   └── overlay.py
│
├── dashboard/
│   └── index.html
│
├── static/
│   ├── css/
│   │   └── style.css
│   └── js/
│       └── app.js
│
├── tests/
│
├── manage.py
├── verify_capabilities.py
├── verify_system.py
├── benchmark.py
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

---

# Running the Project

## 1. Clone the repository

```bash
git clone <repository-url>
cd IBVAP
```

## 2. Install dependencies

```bash
pip install -r requirements.txt
```

## 3. Initialise the database

```bash
python manage.py init
```

## 4. Register a camera

For a webcam:

```bash
python manage.py camera-add \
  --name "Gate 1" \
  --url 0 \
  --location "North Post"
```

For an RTSP source, use the camera's RTSP URL instead.

## 5. Start IBVAP

```bash
python manage.py run
```

Then open:

```text
http://localhost:8000/dashboard
```

API documentation:

```text
http://localhost:8000/docs
```

Health check:

```text
http://localhost:8000/health
```

---

# GPU Support

IBVAP can run on CPU, but NVIDIA GPU acceleration can significantly improve real-time workloads.

For CUDA-enabled PyTorch, install the appropriate CUDA build before installing the remaining dependencies.

The detector configuration can use different YOLO11 model sizes depending on available hardware:

| Model   | Use case                                             |
| ------- | ---------------------------------------------------- |
| YOLO11n | Lightweight / CPU-oriented deployment                |
| YOLO11s | Balanced default configuration                       |
| YOLO11m | Higher compute budget / improved small-object recall |

---

# Testing

The project includes automated tests covering:

* API behaviour
* Camera lifecycle
* Rule engine
* Alert policy
* ANPR
* Face recognition cache
* Playback
* Occupancy
* Evidence
* PDF reports
* Hash-chain integrity
* Dashboard initialisation
* Resource cleanup
* Regression cases

Run the test suite with:

```bash
python -m pytest tests/ -q
```

Verbose mode:

```bash
python -m pytest tests/ -v
```

The project also contains an end-to-end verification utility:

```bash
python verify_system.py
```

and a capability audit:

```bash
python manage.py verify
```

---

# Performance

On an NVIDIA RTX 4060 Laptop GPU using a 640 × 384 analytics resolution, the project was benchmarked with the following results:

### Single camera

| Metric                            |  Result |
| --------------------------------- | ------: |
| Median inference latency          | 18.7 ms |
| Median end-to-end latency         | 24.7 ms |
| P95 end-to-end latency            | 58.8 ms |
| Median frame age                  |   17 ms |
| Median publish-to-browser latency |  3.5 ms |

### Multi-camera

Testing with eight concurrent cameras produced:

* 8/8 cameras online
* Approximately 57–96 aggregate FPS
* Median `/health` latency around 6 ms
* Median camera API latency around 6.5 ms

Actual performance will vary depending on GPU/CPU, video resolution, number of cameras and enabled recognition stages.

---

# Why This Approach?

Traditional intelligent surveillance deployments can require specialised cameras, dedicated recognition appliances and separate systems for different analytics tasks.

IBVAP takes a different software-first approach:

```text
Existing Camera
      ↓
IBVAP
      ↓
Detection
Tracking
Rules
Face Recognition
ANPR
Evidence
Alerts
      ↓
Operator Dashboard
```

The goal is not to replace every existing surveillance component, but to demonstrate how **existing video infrastructure can be upgraded with software-based intelligence**.

---

# My Contribution

For this SIH implementation, my work focuses on building and integrating the complete application layer around the computer-vision models.

Key areas include:

* Designing the end-to-end video analytics pipeline
* Integrating YOLO11 and ByteTrack
* Implementing spatial and behavioural rules
* Integrating face recognition using SCRFD and ArcFace
* Building the ANPR processing and consensus layer
* Developing the FastAPI backend
* Building the operator dashboard
* Implementing camera/source management
* Implementing event and evidence handling
* Adding hash-chain based integrity verification
* Integrating WebSocket-based live events
* Adding alarm/SMS/webhook escalation
* Supporting offline video analysis
* Testing and performance benchmarking

The underlying AI models such as YOLO11, SCRFD, ArcFace and EasyOCR are existing technologies. The project contribution is the **integration, orchestration and application-specific intelligence built around them**.

# Author Section

## 👨‍💻 Developer

**Manas Sharma**
B.Tech CSE | UIT RGPV Shivpuri
Computer Vision • AI Systems • Web Development • Cybersecurity

I worked on the development and integration of this project, including the AI pipeline, backend APIs, surveillance rules, evidence handling, dashboard integration, and overall system workflow.

🔗 **GitHub:** [Manas Sharma](https://github.com/manas1376)
🔗 **LinkedIn:** [Manas Sharma](https://www.linkedin.com/in/manas-sharma-0629a732/)


# Future Improvements

Possible next steps include:

* Multi-GPU deployment
* Edge deployment on lower-power hardware
* Improved distant-face recognition
* More robust plate localisation
* Additional Indian plate formats
* Camera health prediction
* Cross-camera object correlation
* More advanced behavioural analytics
* Role-based operator access
* Distributed deployment for larger camera networks

---

# Acknowledgements

IBVAP relies on several open-source technologies, including:

* Ultralytics YOLO
* ByteTrack
* PyTorch
* InsightFace
* EasyOCR
* OpenCV
* FastAPI
* SQLAlchemy
* ReportLab

Their respective licenses and attribution requirements should be preserved alongside this project.

---

## License

MIT — see [`LICENSE`](LICENSE).
