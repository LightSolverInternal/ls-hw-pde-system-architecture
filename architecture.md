# System Architecture

## 1. Introduction

This document describes the system architecture for the laser-based computational POC system, as specified in the System Requirements document.

The architecture follows a distributed, microservices-based approach with clear separation between:
- **Data Path** – image generation, upload to SLM, and capture from camera
- **Control Path** – temperature control, timing/triggering, and operational state management

The system is designed for flexibility, modularity, and experimental iteration, consistent with its POC nature.

---

## 2. Architecture Overview

### 2.1 Architecture Principles

The system architecture is guided by the following principles:

1. **Distributed Control** – subsystems operate autonomously using publish-subscribe communication
2. **Stateless APIs** – core APIs do not maintain operational state; state management handled at higher layer
3. **Hybrid Orchestration** – microservices for each subsystem + central orchestrator for high-level workflows
4. **Separation of Concerns** – clear boundaries between data path and control path
5. **Configurability** – behavior driven by configuration files (setpoints, error policies, modes)

### 2.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     High-Level Application Layer                 │
│                    (Workflows, UI, Notebooks)                    │
└────────────────────────────────┬────────────────────────────────┘
                                 │
┌────────────────────────────────┴────────────────────────────────┐
│                        Orchestrator (Python)                     │
│                    (Runs on PC with GPU)                         │
│  ┌─────────────────────┐        ┌─────────────────────────┐    │
│  │    Data Path API    │        │   Control Path API      │    │
│  └─────────────────────┘        └─────────────────────────┘    │
└────────────────────────────────┬────────────────────────────────┘
                                 │
              ┌──────────────────┴──────────────────┐
              │         MQTT Broker                 │
              └──────────────────┬──────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │                            │                            │
┌───┴────────┐ ┌────────────┐   │   ┌────────────────────┐   │   ┌──────────────────┐
│SLM1 Service│ │SLM2 Service│   │   │  Camera Service    │   │   │  Laser Driver    │
│(Meadowlark)│ │(Meadowlark)│   │   │ (2x Cameras via    │   │   │  Service (MQTT)  │
│            │ │            │   │   │  Frame Grabber)    │   │   │                  │
└────────────┘ └────────────┘   │   └────────────────────┘   │   └──────────────────┘
                                 │

┌─────────────────┐   ┌──────────────────────┐   ┌────────────────────┐
│ TEC Controllers │   │  PID Controllers     │   │  Temp Sensors      │
│ (DC3145A/MQTT)  │   │  (MQTT)              │   │  (MQTT)            │
└─────────────────┘   └──────────────────────┘   └────────────────────┘

┌─────────────────┐   ┌──────────────────────┐
│  Picomotor      │   │  Timing/Trigger HW   │
│  Controller     │   │  (1kHz Pulse Gen +   │
│                 │   │   0.7ms Delay +      │
│                 │   │   BNC Splitter)      │
└─────────────────┘   └──────────────────────┘
```

---

## 3. System Decomposition

### 3.1 Data Path Subsystem

**Purpose:**  
Manages the flow of image data to the SLMs and from the cameras.

**System Configuration:**
- **2 SLMs** (Meadowlark 1024x1024 PCIe) – display independent phase patterns
- **2 Cameras** (Allied Vision Prosilica GT 5120NIR) – capture SLM images simultaneously
- **1 Frame Grabber** (Coaxlink Quad CXP-12) – interfaces both cameras
- **1 kHz Pulse Generator** – triggers both SLMs
- **Programmable Delay Circuit** (0.7ms) – delays SLM2 output to camera triggers
- **BNC T-Connector** – splits delayed trigger to both cameras

**Responsibilities:**
- Upload independent phase patterns to both SLMs
- Capture synchronized images from both cameras
- Coordinate timing: SLM updates → pixel settling → camera capture
- Ensure both SLM images are settled before camera capture

**Key Components:**
- **SLM1 Service** – wraps Meadowlark PCIe driver for SLM #1
- **SLM2 Service** – wraps Meadowlark PCIe driver for SLM #2
- **Camera Service** – wraps frame grabber (eGrabber SDK), manages both cameras
- **Display Servers** – send images to SLM1 and SLM2 (existing components, adapted)
- **Capture Server** – receives images from both cameras (existing component, adapted)

**API (Stateless):**
- `slm1.upload_image(image, wait_for_trigger=True)`
- `slm2.upload_image(image, wait_for_trigger=True)`
- `camera_service.capture_dual(timeout_ms)`
- `set_output_pulse(slm_id, enabled)`

**Trigger Architecture:**

```
1 kHz Pulse Generator (1ms period)
       │
       ├──────────────────────┬──────────────────────┐
       │                      │                      │
       ↓                      ↓                      │
   SLM1 Trigger          SLM2 Trigger               │
   (no output)           (output enabled)           │
       │                      │                      │
   [Flip + Settle]       [Flip + Settle]            │
   (0.696ms min)         (0.696ms min)              │
                              │                      │
                         Output Pulse                │
                          (200µs)                    │
                              │                      │
                              ↓                      │
                      Delay Circuit                  │
                        (0.7ms)                      │
                              │                      │
                              ↓                      │
                      BNC T-Connector                │
                         ↙        ↘                  │
                   Camera1      Camera2              │
                   [Capture]    [Capture]            │
                                                      │
       Next pulse (t+1ms) ←───────────────────────────┘
```

**Timing Guarantees (Worst Case):**
- SLM1 triggered at t=0ms, settled by t=0.696ms
- SLM2 triggered at t=1ms (next pulse), settled by t=1.696ms
- Cameras triggered at t=1.7ms (1ms + 0.7ms delay)
- At camera trigger time:
  - SLM1 image: settled for 1.7ms ✅ (extra margin)
  - SLM2 image: settled for 0.7ms ✅ (just over 0.696ms minimum)
  - Both cameras capture simultaneously ✅

**GPU Memory Workflow:**

The system leverages GPU memory throughout the data path to maximize performance:

```
┌────────────────────────────────────────────────────────┐
│                    NVIDIA GPU Memory                   │
│                                                        │
│  ┌──────────────────────┐    ┌────────────────────┐  │
│  │  Phase Patterns      │    │  Captured Images   │  │
│  │  Generation (CUDA)   │    │  (from Cameras)    │  │
│  │  - SLM1 pattern      │    │  - Camera1 image   │  │
│  │  - SLM2 pattern      │    │  - Camera2 image   │  │
│  └───────┬──────────────┘    └─────────▲──────────┘  │
│          │                              │             │
│          │ GPU-GPU Processing           │             │
│          │ (no CPU copy)                │             │
│          └──────────►───────────────────┘             │
│                                                        │
└──────────┼──────────┬───────────────────┼─────┬───────┘
           │          │                   │     │
           │ DMA      │ DMA               │ DMA │ DMA  
           ↓          ↓                   ↑     ↑
    ┌──────┴───┐  ┌──┴───────┐    ┌──────┴─────┴─────┐
    │ SLM1 PCIe│  │ SLM2 PCIe│    │  Frame Grabber   │
    │(Meadowlk)│  │(Meadowlk)│    │   (Coaxlink)     │
    └──────┬───┘  └──┬───────┘    └─────┬────┬───────┘
           │         │                   │    │
           ↓         ↓                   ↑    ↑
    ┌──────┴───┐  ┌──┴───────┐    ┌─────┴──┐ │
    │   SLM1   │  │   SLM2   │    │Camera1 │ │
    │ Display  │  │ Display  │    │        │ │
    │1024x1024 │  │1024x1024 │    │        │ │
    └──────────┘  └──────────┘    └────────┘ │
                                   ┌──────────┴┐
                                   │  Camera2  │
                                   │           │
                                   │           │
                                   └───────────┘
```

**Benefits:**
- **Zero-copy architecture** – images generated and captured directly in GPU memory
- **Low latency** – no CPU memory transfers in critical path
- **High throughput** – GPU-GPU processing for feedback loops
- **Reduced CPU load** – CPU free for orchestration and control tasks

**Hardware Support:**
- **SLM:** Meadowlark SDK supports GPU memory pointers (verification needed - see OPEN_ISSUES.md #4)
- **Frame Grabber:** Coaxlink Quad CXP-12 supports NVIDIA CUDA Direct GPU Transfer
- **GPU:** NVIDIA GPU with CUDA capability (required by SLM)

**Implementation - Dual-SLM Operation:**
```python
import cupy as cp  # CUDA arrays in GPU memory

# One-time setup
slm1.load_lut()
slm1.set_wait_for_trigger(True)   # External trigger mode
slm1.set_output_pulse(False)      # Silent flip (no output pulse)

slm2.load_lut()
slm2.set_wait_for_trigger(True)   # External trigger mode
slm2.set_output_pulse(True)       # Generates output pulse for cameras

# Main loop
for iteration in range(num_frames):
    # Generate phase patterns on GPU
    pattern1 = generate_phase_cuda_slm1()  # CuPy array
    pattern2 = generate_phase_cuda_slm2()  # CuPy array
    
    # Upload to both SLMs (DMA: GPU → SLM, ~1ms each)
    slm1.write_image(pattern1)
    slm2.write_image(pattern2)
    
    # Block until both DMAs complete
    success1 = slm1.ImageWriteComplete(timeout_ms=5000)
    success2 = slm2.ImageWriteComplete(timeout_ms=5000)
    
    if not (success1 and success2):
        handle_timeout_error()
        break
    
    # Both SLMs now waiting for 1kHz pulse generator triggers
    # Worst case: SLM1 triggered at t=0, SLM2 at t=1ms
    # Cameras triggered at t=1.7ms (after 0.7ms delay from SLM2 output)
    
    # Capture from both cameras simultaneously (DMA: Camera → GPU)
    img1, img2 = camera_service.capture_dual_to_gpu(timeout_ms=100)
    
    # Process on GPU (no CPU copies) - feedback loop
    processed = process_dual_feedback_cuda(img1, img2)
    
    # Use results for next iteration (stays on GPU)
    # ... feedback computation for next patterns ...

```

**Timing Characteristics:**
- DMA upload time: ~1ms per SLM (parallel execution possible)
- Worst-case settling: SLM1 = 1.7ms, SLM2 = 0.7ms (both > 0.696ms minimum)
- Camera capture: synchronized, triggered after both SLMs settled
- Frame rate: Limited by slowest component (camera capture + processing)

**Performance Impact:**
- Eliminates GPU↔CPU memory copies (~10-50ms overhead for large images)
- Enables sustained 1000 fps operation with processing
- Critical for real-time feedback loops during calibration and operation

### 3.2 Control Path Subsystem

**Purpose:**  
Manages temperature stability, laser operation, and system-level control.

**Responsibilities:**
- Temperature control of SLM and gain crystal
- Laser diode current control and enable/disable
- Error detection and safety responses
- Configuration and setpoint management

**Key Components:**

#### 3.2.1 Temperature Control (SLM)
**Architecture (existing, preserved):**
- **Temperature Sensor** → publishes `SLM1/temperature` (MQTT)
- **PID Controller** → subscribes to temperature, publishes `SLM1/pwm` [-1:1] (MQTT)
- **DC3145A TEC Driver** → subscribes to PWM, drives TEC (MQTT via ESP32/RPI)

**Configuration:**
- Default setpoint in config file
- Runtime control via MQTT

#### 3.2.2 Temperature Control (Gain Crystal / Pump)
**Architecture:**
- Uses manufacturer's system (external, out of scope for orchestrator)
- System queries status via manufacturer API (if available)

#### 3.2.3 Laser Diode Current Control
**Component:** UNILDD-A-CW-27-60 laser driver

**Architecture:**
- **Laser Driver Service** (ESP32 + MQTT)
  - Subscribes to `laser/setpoint` (current level)
  - Subscribes to `laser/enable` (on/off)
  - Publishes `laser/status` (current, state)
  - DAC output controls current

**Operation:**
- Primary mode: CW (continuous wave)
- Setpoint set via MQTT
- Periodic validation (every few minutes)

#### 3.2.4 Cooling System
**Component:** HRR030-WF-20-TU chiller

**Operation:**
- Water cooling for diode and pump
- Autonomous operation (no orchestrator control)
- May include status monitoring if interface available

### 3.3 Calibration Subsystem

**Purpose:**  
Supports alignment and calibration procedures.

**Key Components:**
- **Picomotor Controller 8742** – controls piezo motors for optical alignment
- **Image Processing Feedback** – processes camera images to guide calibration

**Operation:**
- Used during Stage 1 (Build) and Stage 2 (Calibration)
- May support partial automation of calibration workflows

### 3.4 Orchestrator

**Purpose:**  
Coordinates high-level workflows and provides unified API to application layer.

**Deployment:**
- Runs on PC with NVIDIA GPU (same as SLM)
- Implemented in Python

**Responsibilities:**
- Exposes unified Data Path API to users/applications
- Manages operational workflows (startup, calibration, shutdown)
- Aggregates status from all subsystems
- Implements error policies and safety responses
- Does NOT maintain stateful modes (stateless at API level)

**Interfaces:**
- Publishes/subscribes to MQTT topics
- Calls SLM and Camera service APIs
- Exposes REST or Python API for external applications

---

## 4. Hardware Components

### 4.1 Optical Components

| Component | Model | Quantity | Function | Interface |
|-----------|-------|----------|----------|-----------|  
| SLM | Meadowlark XY Phase 1024x1024 | 2 | Phase modulation (independent patterns) | PCIe (software driver) |
| Camera | Allied Vision Prosilica GT 5120NIR | 2 | Image capture (synchronized) | CoaXPress (CXP-12) |
| Frame Grabber | Coaxlink Quad CXP-12 Value | 1 | Dual camera interface, triggering | PCIe 3.0 x8, 20 digital I/O, 4x CXP ports |
| Pulse Generator | 1 kHz TTL | 1 | Triggers both SLMs | TTL output to SLM trigger inputs |
| Delay Circuit | Programmable delay | 1 | 0.7ms delay for camera triggers | Input: SLM2 output, Output: TTL to cameras |
| Laser Diode | DILAS MMF-IS11 | Pump source (IR) | Laser driver (analog) |
| Gain Crystal | TBD | Laser emission | N/A |

**Frame Grabber Capabilities:**
- 4x CoaXPress CXP-12 connections (5,000 MB/s camera bandwidth)
- 20 digital I/O lines (differential, TTL, isolated inputs/outputs)
- Programmable trigger modes (area-scan and line-scan)
- I/O Toolbox with configurable delays and filtering
- Potential for programmable trigger delay (needs investigation)

### 4.2 Control & Conditioning Hardware

| Component | Model | Function | Interface |
|-----------|-------|----------|-----------|
| TEC Driver (SLM) | DC3145A | Temperature control | MQTT (ESP32/RPI) |
| Laser Driver | UNILDD-A-CW-27-60 | Diode current control | MQTT (ESP32), DAC |
| Chiller | HRR030-WF-20-TU | Water cooling | Autonomous |
| Picomotor Controller | 8742 | Optical alignment | USB/Ethernet (vendor API) |

### 4.3 Compute Hardware

| Component | Purpose |
|-----------|---------|
| PC with NVIDIA GPU | SLM driver, camera acquisition, orchestrator |
| Raspberry Pi / ESP32 | MQTT-based control nodes (temperature, laser driver) |

---

## 5. Software Components

### 5.1 Core Services

| Service | Language | Deployment | Purpose |
|---------|----------|------------|---------|
| Orchestrator | Python | PC | High-level coordination, API gateway |
| SLM Service | Python | PC | SLM control (wraps Meadowlark SDK) |
| Camera Service | Python | PC | Camera acquisition (wraps Allied Vision SDK) |
| Display Server | Python | PC | Image upload to SLM (existing) |
| Capture Server | Python | PC | Image download from camera (existing) |
| Laser Driver Service | C++/MicroPython | ESP32 | Laser current control via MQTT |
| TEC Controller Service | C++/MicroPython | ESP32 | TEC PWM control via MQTT |
| Temperature Sensor Service | C++/MicroPython | ESP32 | Temperature publishing via MQTT |
| PID Controller Service | Python | RPI/PC | PID loop via MQTT |

### 5.2 Middleware

| Component | Purpose |
|-----------|---------|
| MQTT Broker | Message bus for all distributed services |
| MQTT Bridge | Forwards metrics to AWS cloud |

### 5.3 Configuration Management

- **Config Files** (YAML/JSON)
  - Setpoints (temperatures, laser current)
  - Error policies (thresholds, responses)
  - Trigger modes
  - Calibration data (LUT files for SLM)

---

## 6. Communication & Integration

### 6.1 MQTT Topics (Examples)

**Temperature Control:**
- `sensors/slm1/temperature` – published by sensor
- `control/slm1/setpoint` – published by config or user
- `control/slm1/pwm` – published by PID controller
- `actuators/slm1/tec/pwm` – subscribed by DC3145A

**Laser Control:**
- `laser/setpoint` – desired current (A)
- `laser/enable` – on/off command
- `laser/status` – current state and measured current

**System Status:**
- `system/state` – operational state (if maintained at higher layer)
- `system/errors` – error messages and alerts

### 6.2 SLM Timing & Triggering

**From Meadowlark Datasheet (1024x1024):**
- External trigger input: TTL, falling edge
- Output pulse generated: up to 0.696 ms after trigger received
- Image update: 0.696 ms after output pulse
- Flip immediate mode: reduces latency

**Synchronization Strategy (configurable per mode):**
1. **SLM → Camera:**  
   SLM output pulse triggers camera exposure
   
2. **Camera → SLM:**  
   Camera readout complete triggers SLM update
   
3. **External Controller:**  
   Dedicated timing hardware (Arduino/FPGA/etc.) generates triggers for both SLM and camera with defined delays

### 6.3 API Design Principles

- APIs are **stateless** where possible
- Operations are **idempotent** where practical
- Errors return structured error codes and messages
- All APIs support **async operation** (future consideration)

---

## 7. Data Management

### 7.1 Image Storage

**Feedback Images (Control Loop):**
- Stored in memory or temporary local disk
- Discarded after use
- Used for calibration, alignment, closed-loop control

**Result Images (Computation Output):**
- Stored persistently on local disk
- Uploaded to cloud storage (AWS or equivalent)
- Accompanied by metadata (timestamp, configuration, run ID)

**Storage Formats:**
- Raw: vendor-specific (camera native format)
- Processed: TIFF, HDF5, or NumPy `.npy` (TBD based on downstream analysis)

### 7.2 Logs and Metrics

**Logs:**
- Stored locally on PC
- Structured format (JSON lines or similar)
- Rotation and retention policy (TBD)

**Metrics:**
- Published to MQTT
- Forwarded to AWS via MQTT bridge
- Includes: temperatures, frame rates, trigger timing, error counts

---

## 8. Error Handling & Safety

### 8.1 Error Policies (Configurable)

Errors are classified by severity and component. Each error type has a configured response:

| Error Type | Example Threshold | Response |
|------------|-------------------|----------|
| Critical overtemperature | SLM > 50°C | Immediate laser shutdown, alert |
| Warning overtemperature | SLM > 45°C | Alert, log, continue operation |
| Trigger timeout | No trigger for 5s | Alert, retry or abort acquisition |
| Communication loss | MQTT disconnect | Retry, alert if persistent |

### 8.2 Safety Mechanisms

- **Laser Safety:**  
  Enable/disable control via orchestrator  
  Automatic disable on critical errors
  
- **Temperature Protection:**  
  Hardware-level cutoff (if supported by TEC driver)  
  Software-level monitoring and alert
  
- **Graceful Shutdown:**  
  Orchestrator supports controlled shutdown sequence  
  Laser disabled first, then TEC ramped down

---

## 9. Observability

### 9.1 Monitoring

- Temperature sensors publish continuously
- Laser driver publishes status periodically
- Orchestrator aggregates and exposes system health

### 9.2 Logging

- All services log to local files
- Structured logs (timestamp, service, level, message, context)

### 9.3 Metrics

- Forwarded to cloud via MQTT bridge
- Includes operational metrics (frame rate, uptime, error counts)
- Supports post-experiment analysis

---

## 10. Operational Stages

### 10.1 Stage 1 – Build and Assembly

- Manual operation
- Orchestrator provides observability
- Picomotor controller used for alignment
- Limited automation

### 10.2 Stage 2 – Calibration

- Semi-automated workflows
- SLM LUT calibration (vendor tools + custom scripts)
- Image-based feedback loops (e.g., optimize alignment)
- PID tuning for temperature control

### 10.3 Stage 3 – Normal Operation

- Fully autonomous operation
- Orchestrator executes pre-defined workflows
- Minimal manual intervention
- Deterministic and repeatable behavior

---

## 11. Future Considerations

- Definition of concrete trigger mode selection logic
- REST API design for orchestrator
- Integration with Jupyter notebooks or web UI
- Advanced image processing pipelines (GPU acceleration)
- Data provenance and experiment tracking
- Production-grade packaging and deployment (out of scope for POC)

---

## 12. References

- System Requirements (requirements.md)
- Meadowlark SLM PCIe User Manual (1024x1024)
- Component datasheets (stored in `docs/datasheets/`)
