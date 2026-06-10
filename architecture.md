# System Architecture

## 1. Introduction

This document describes the system architecture for the laser-based computational POC system, as specified in the System Requirements document.

The architecture uses a **unified C++17 library** (PDEPhysicalChain) for the data path with clear separation between:
- **Data Path** – SLM and camera control via unified C++ library
- **Control Path** – temperature control, laser control, and operational state management via MQTT

The system is designed for flexibility, modularity, and experimental iteration, consistent with its POC nature.

**Key Architectural Change from Original Design:**
The original microservices-based design evolved into a unified C++17 library due to SDK constraints (thread-safety requirements) and performance considerations (zero-copy DMA buffer management).

---

## 2. Architecture Overview

### 2.1 Architecture Principles

The system architecture is guided by the following principles:

1. **Unified Data Path** – Single C++ library coordinates SLM and camera hardware
2. **Worker Thread Pattern** – Serializes SDK access (SDKs not thread-safe)
3. **Zero-Copy DMA** – RAII-based buffer ownership transfer for performance
4. **YAML Configuration** – Runtime device selection and parameter tuning
5. **Separation of Concerns** – Data path (C++ library) vs Control path (MQTT)
6. **Configurability** – Supports 1-2 SLMs and 1-2 cameras for dev/production modes

### 2.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  High-Level Application Layer                    │
│              (Workflows, UI, Jupyter Notebooks)                  │
│                    Python or C++ Applications                    │
└────────────────────────────────┬────────────────────────────────┘
                                 │
┌────────────────────────────────┴────────────────────────────────┐
│                  Application Orchestrator (Python)               │
│                      (Runs on PC with GPU)                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │           PDEPhysicalChain (C++17 Library)                │  │
│  │         - FSM-based state management                      │  │
│  │         - Worker thread for SDK serialization             │  │
│  │         - MQTT monitoring thread                          │  │
│  │         - Python bindings via pybind11                    │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────┬────────────────────────────────┘
                                 │
              ┌──────────────────┴──────────────────┐
              │         MQTT Broker                 │
              │    (Temperature telemetry,          │
              │     laser control)                  │
              └──────────────────┬──────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │                            │                            │
┌───┴───────────┐  ┌─────────────┴──────┐  ┌─────────────────┴───┐
│  Laser Driver │  │ TEC Controllers    │  │  Temperature        │
│  Service      │  │ (DC3145A/MQTT)     │  │  Sensors (MQTT)     │
│  (MQTT)       │  │                    │  │                     │
└───────────────┘  └────────────────────┘  └─────────────────────┘
```

**PDEPhysicalChain Library Internal Architecture:**

```
┌────────────────────────────────────────────────────────────────┐
│                    PDEPhysicalChain Class                       │
│  ┌─────────────┐  ┌────────────────┐  ┌────────────────────┐ │
│  │  Main API   │  │  Worker Thread │  │  MQTT Thread       │ │
│  │  (Public)   │  │  (SDK Access)  │  │  (Monitoring)      │ │
│  └──────┬──────┘  └────────┬───────┘  └─────────┬──────────┘ │
│         │                  │                     │             │
│         │ Command Queue    │                     │             │
│         └─────────────────►│                     │             │
│                            │                     │             │
│         ┌──────────────────┴─────────────────────┘             │
│         │                                                       │
│    ┌────┴──────┐    ┌──────────┐    ┌──────────┐             │
│    │SLMControl │    │ Camera 1 │    │ Camera 2 │             │
│    │(Singleton)│    │(Instance)│    │(Instance)│             │
│    └─────┬─────┘    └─────┬────┘    └─────┬────┘             │
└──────────┼────────────────┼───────────────┼──────────────────┘
           │                │               │
    ┌──────┴──────────┐ ┌───┴───────────────┴───────────┐
    │ Meadowlark SDK  │ │   Euresys eGrabber SDK        │
    │  (Blink SDK)    │ │   (Frame Grabber Control)     │
    │  - 1-2 SLMs     │ │   - DMA Buffer Management     │
    │  - PCIe         │ │   - Zero-copy Frame Objects   │
    └──────┬──────────┘ └──────────────┬────────────────┘
           │                           │
    ┌──────┴──────────┐         ┌──────┴─────────────────┐
    │   SLM Hardware  │         │   Coaxlink Quad CXP-12 │
    │  1-2 x 1024x1024│         │    Frame Grabber       │
    │  Meadowlark PCIe│         │   (DMA Pool: N buffers)│
    └─────────────────┘         └──────────┬─────────────┘
                                           │
                                ┌──────────┴──────────┐
                                │  1-2 x hr25MCX      │
                                │  Allied Vision      │
                                │  (CoaXPress CXP-12) │
                                └─────────────────────┘
```

---

## 3. System Decomposition

### 3.1 Data Path - PDEPhysicalChain Library

**Purpose:**  
Unified C++17 library providing synchronized control of SLMs and cameras for the complete data acquisition pipeline.

**Implementation:**  
- **Language:** C++17 (RAII, move semantics, smart pointers)
- **Build System:** CMake 3.15+
- **Library:** Shared library (`libpde_physical_chain.so`)
- **Python Bindings:** pybind11 wrapper
- **Configuration:** YAML-based runtime config

**Hardware Configuration:**
- **1-2 SLMs** (Meadowlark 1024x1024 PCIe) – configured via YAML
- **1-2 Cameras** (Allied Vision hr25MCX) – configured via YAML  
- **1 Frame Grabber** (Coaxlink Quad CXP-12 Value) – interfaces cameras
- **No External Trigger Hardware** – camera has built-in configurable delay

**Flexible Deployment Modes:**
```yaml
# Development Mode (single device testing)
devices:
  slms:
    transmission:
      serial: "SLM-12345"
  cameras:
    camera1:
      serial: "CAM-11111"

# Production Mode (dual devices)
devices:
  slms:
    transmission:
      serial: "SLM-12345"
    reflection:
      serial: "SLM-67890"
  cameras:
    camera1:
      serial: "CAM-11111"
    camera2:
      serial: "CAM-22222"
```

**Key Responsibilities:**
- FSM-based acquisition lifecycle management
- SLM image upload with settling verification
- Zero-copy camera capture via DMA
- Frame ownership transfer to application
- MQTT temperature monitoring
- Worker thread serialization (SDK thread-safety)

**Core Classes:**

1. **PDEPhysicalChain** – Main orchestrator
   - FSM state management (Idle → Ready → Capture → Ready)
   - Worker thread for SDK serialization
   - MQTT monitoring thread
   - Configuration loading (YAML)

2. **SLMController** – Singleton pattern
   - Required by Meadowlark SDK architecture
   - Manages 1-2 SLMs by serial number
   - PCIe DMA transfer (~1ms per SLM)
   - Settling verification via `ImageWriteComplete()`

3. **Camera** – Multi-instance pattern
   - One instance per physical camera
   - Built-in trigger delay (replaces external function generator)
   - Runtime parameter control (exposure, gain, delay)
   - Zero-copy capture via `grabFrameZeroCopy()`

4. **Frame** – RAII ownership transfer
   - Wraps DMA buffer with `unique_ptr<ScopedBuffer>`
   - Automatic buffer release on destruction
   - Zero-copy data access for application

**Trigger Architecture (Hardware-Based):**

```
Application calls setSLMCaptureCamera()
           │
           ↓
    ┌──────────────┐
    │  SLM Write 1 │ PCIe DMA (~1ms) + Settling (~1ms)
    └──────┬───────┘
           │ ImageWriteComplete() blocks until stable
           │ **CRITICAL:** Prevents race condition
           ↓
    ┌──────────────┐
    │  SLM Write 2 │ PCIe DMA (~1ms) + Settling (~1ms)
    └──────┬───────┘
           │
           ↓ Hardware trigger pulse from SLM
    ┌──────────────┐
    │    Camera    │ Built-in delay ensures SLM settled
    │   Trigger    │ (configured via setTriggerDelay())
    └──────┬───────┘
           │
           ↓ DMA to buffer pool
    ┌──────────────┐
    │ Frame Object │ Zero-copy ownership transfer
    │  (RAII)      │ to application
    └──────────────┘
```

**No External Function Generator:**
- Original design used external 1kHz pulse generator + delay circuit
- **Eliminated:** Camera has built-in programmable trigger delay
- Trigger pulse comes directly from SLM hardware output
- Simpler, fewer components, more reliable

**API (C++):**

```cpp
// Initialize from YAML config
PDEPhysicalChain chain("config/pde_config.yaml");

// Start acquisition (allocate N DMA buffers)
chain.acquisitionStart(10);  // State: Idle → Ready

// Capture loop (single frame per call)
for (int i = 0; i < 1000; i++) {
    // Prepare SLM patterns (CPU memory)
    uint8_t* slm_t_data = generate_pattern_t(i);
    uint8_t* slm_r_data = generate_pattern_r(i);
    
    // Capture single frame (blocks ~3-5ms + exposure)
    // State: Ready → First_SLM_write → Second_SLM_write → Capture → Ready
    CaptureResult result = chain.setSLMCaptureCamera(slm_t_data, slm_r_data);
    
    if (result.success) {
        // Zero-copy access to DMA buffers
        process(result.camera1_frame->getData());
        process(result.camera2_frame->getData());
    }
    // Frames auto-release when result goes out of scope
}

// Stop acquisition (deallocate buffers)
chain.acquisitionStop();  // State: Ready → Idle
```

**API (Python via pybind11):**

```python
import pde_physical_chain as pde
import numpy as np

# Initialize
chain = pde.PDEPhysicalChain("config/pde_config.yaml")
chain.acquisition_start(10)

# Capture loop
for i in range(1000):
    slm_t = np.random.randint(0, 256, (1024, 1024), dtype=np.uint8)
    slm_r = np.random.randint(0, 256, (1024, 1024), dtype=np.uint8)
    
    result = chain.set_slm_capture_camera(slm_t, slm_r)
    
    if result.success:
        # Zero-copy numpy view of DMA buffer
        img1 = np.array(result.camera1_frame, copy=False)
        img2 = np.array(result.camera2_frame, copy=False)
        process(img1, img2)

chain.acquisition_stop()
```

**Timing Characteristics:**
- SLM DMA: ~1ms per SLM (PCIe bandwidth)
- SLM Settling: ~1ms per SLM (pixel response time)
- **ImageWriteComplete()** after first SLM: Critical for dual-SLM race prevention
- Camera capture: Depends on exposure + readout
- **Total per cycle:** ~3-5ms + exposure time

**Performance:**
- Theoretical max: ~200-250 Hz (excluding exposure/processing)
- Actual: Limited by exposure time and application processing
- Zero-copy architecture minimizes latency
- DMA buffer pool prevents allocation overhead

**Current Capture Mode:**
- **Single-frame capture:** One frame per `setSLMCaptureCamera()` call
- Each call returns `CaptureResult` with one frame per camera

**Future Enhancement - Burst Mode:**
- Single SDK command can capture multiple consecutive frames
- Would return `vector<Frame>` per camera (C++) or `list[Frame]` (Python)
- Easy to implement - minimal SDK changes required
- Useful for high-speed sequences without SLM updates
- See [Future Considerations](#131-planned-enhancements) for details

**See Also:**
- [Detailed Design: PDE_Physical_Chain_Design.md](docs/PDE_Physical_Chain_Design.md)
- [Sequence Diagrams: Dual SLM](docs/setSLMCaptureCamera_with_dual_SLM.puml)
- [Sequence Diagrams: Single SLM](docs/setSLMCaptureCamera_with_single_SLM.puml)
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



#### 3.1.1 Memory Management Architecture

**Current Implementation: CPU Memory Only**

The PDEPhysicalChain library currently uses **CPU memory (RAM)** exclusively for all data transfers and processing:

```
┌─────────────────────────────────────────────────────────┐
│                      CPU Memory (RAM)                    │
│                                                           │
│  ┌──────────────────────┐    ┌────────────────────┐    │
│  │  Application         │    │  DMA Buffer Pool   │    │
│  │  - SLM patterns      │    │  (Frame Grabber)   │    │
│  │  - Processing        │    │  - N buffers       │    │
│  └──────┬───────────────┘    └─────────▲──────────┘    │
│         │                               │                │
│         │ Copy to SDK                   │ Zero-copy      │
│         │                               │ (RAII Frame)   │
│         ↓                               │                │
│  ┌──────────────────────┐              │                │
│  │  Meadowlark SDK      │              │                │
│  │  (SLM Buffers)       │              │                │
│  └──────┬───────────────┘              │                │
└─────────┼────────────────────────────────┼───────────────┘
          │                                │
          │ PCIe DMA                       │ PCIe DMA
          ↓                                ↑
   ┌──────────────┐                ┌───────────────┐
   │ SLM Hardware │                │ Frame Grabber │
   │ (Meadowlark) │                │ (Coaxlink)    │
   └──────────────┘                └───────────────┘
```

**Memory Allocation:**
- **SLM Patterns:** Application allocates (malloc/new), copies to SDK
- **Camera Buffers:** Managed by Euresys eGrabber SDK DMA pool
- **Frame Objects:** Zero-copy RAII wrappers around DMA buffers
- **Processing:** Application must copy from DMA buffer if needed beyond Frame lifetime

**Zero-Copy Camera Path:**
The Frame class provides zero-copy access to DMA buffers:
- `Frame::getData()` returns pointer to DMA buffer (no copy)
- Application reads directly from DMA memory
- When Frame destructor runs, buffer returns to pool
- **Critical:** Do not use data pointer after Frame is destroyed

**Limitations (SDK Constraints):**
- **No cudaMallocHost support:** Meadowlark SDK bindings reject GPU pointers
- **No pinned memory:** Cannot use `cudaMallocHost()` with current SDK
- **No GPU Direct:** Frame grabber SDK does not support direct GPU transfer yet
- **SLM copy overhead:** Must copy pattern from application memory to SDK buffer

**Future Enhancement: GPU Memory Support**

GPU memory support is a planned enhancement pending SDK updates:

```
┌─────────────────────────────────────────────────────────┐
│                   NVIDIA GPU Memory                      │
│                                                           │
│  ┌──────────────────────┐    ┌────────────────────┐    │
│  │  Phase Patterns      │    │  Captured Images   │    │
│  │  Generation (CUDA)   │    │  (from Cameras)    │    │
│  │  - CUDA kernels      │    │  - cudaMallocHost  │    │
│  └──────┬───────────────┘    └─────────▲──────────┘    │
│         │                               │                │
│         │ GPU-GPU Processing            │                │
│         │ (no CPU copy)                 │                │
│         └──────────►────────────────────┘                │
│                                                           │
└─────────┼────────────────────────────────┼───────────────┘
          │ PCIe DMA (pinned)              │ PCIe DMA (direct)
          ↓                                ↑
   ┌──────────────┐                ┌───────────────┐
   │ SLM Hardware │                │ Frame Grabber │
   └──────────────┘                └───────────────┘
```

**Required for GPU Support:**
1. **Meadowlark SDK:** Accept cudaMallocHost pointers in write_image()
2. **Euresys SDK:** Enable CUDA Direct GPU Transfer mode
3. **Application:** Generate patterns on GPU via CUDA kernels

**Benefits (when available):**
- Eliminate CPU←→GPU copies (~5-20ms per transfer)
- Enable GPU-accelerated pattern generation
- Support real-time GPU feedback loops
- Reduce CPU load, improve sustained throughput

**Status:**
- See [OPEN_ISSUES.md](OPEN_ISSUES.md) for GPU memory investigation status
- Vendor engagement required for SDK updates
- Current CPU-only implementation meets POC requirements

---

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
Supports alignment and calibration procedures for optical components.

**Key Components:**
- **Picomotor Controller 8742** – controls piezo motors for optical alignment
- **Image Processing Feedback** – processes camera images to guide calibration
- **PDEPhysicalChain Library** – provides SLM and camera access for calibration workflows

**Operation:**
- Used during Stage 1 (Build) and Stage 2 (Calibration)
- May support partial automation of calibration workflows
- External to PDEPhysicalChain library (uses library as building block)

**Calibration Applications:**
- SLM wavefront correction (flatness calibration)
- Optical axis alignment
- Focus optimization
- Power calibration

---

## 4. Application Layer Architecture

### 4.1 High-Level Application Workflows

**Architecture Pattern:**

```
┌─────────────────────────────────────────────────────┐
│         Jupyter Notebooks / Python Scripts           │
│           (User Experiments & Workflows)             │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────┴───────────────────────────────┐
│         Application Orchestrator (Python)            │
│  - Workflow sequencing                               │
│  - MQTT control path integration                     │
│  - Error handling and logging                        │
│  - High-level operational modes                      │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │                           │
┌───────┴──────────┐     ┌──────────┴────────┐
│ PDEPhysicalChain │     │  MQTT Broker      │
│   (C++/Python)   │     │  (Control Path)   │
│  - Data path     │     │  - Temperature    │
│  - FSM control   │     │  - Laser control  │
└──────────────────┘     └───────────────────┘
```

**Deployment:**
- **PDEPhysicalChain Library:** Runs on PC with frame grabber and SLM
- **MQTT Broker:** Runs on PC or separate server
- **Application Orchestrator:** Python layer coordinating workflows
- **Jupyter Notebooks:** User experimentation environment

**Application Examples:**

1. **Calibration Workflow:**
   ```python
   # Initialize library
   chain = PDEPhysicalChain("config.yaml")
   chain.acquisition_start(10)
   
   # Run flatness calibration
   slm_correction = calibrate_slm_flatness(chain)
   
   # Apply correction to all patterns
   for pattern in experiment_patterns:
       corrected = apply_correction(pattern, slm_correction)
       result = chain.set_slm_capture_camera(corrected, None)
       analyze(result.camera1_frame)
   
   chain.acquisition_stop()
   ```

2. **Feedback Loop Experiment:**
   ```python
   chain = PDEPhysicalChain("config.yaml")
   chain.acquisition_start(50)
   
   # Iterative optimization
   pattern = initial_guess()
   for iteration in range(1000):
       result = chain.set_slm_capture_camera(pattern, None)
       
       if result.success:
           metric = compute_metric(result.camera1_frame)
           pattern = update_pattern(pattern, metric)  # Gradient descent
       else:
           handle_error(result.error_msg)
           break
   
   chain.acquisition_stop()
   ```

3. **Long-Duration Data Collection:**
   ```python
   chain = PDEPhysicalChain("config.yaml")
   chain.acquisition_start(20)
   
   # Collect dataset
   dataset = []
   for i in range(10000):
       pattern_t, pattern_r = generate_patterns(i)
       result = chain.set_slm_capture_camera(pattern_t, pattern_r)
       
       if result.success:
           # Save to disk
           save_to_hdf5(result.camera1_frame, f"frame_{i}_cam1.h5")
           save_to_hdf5(result.camera2_frame, f"frame_{i}_cam2.h5")
       
       # Monitor temperature via MQTT (async)
       check_temperature_alarm()
   
   chain.acquisition_stop()
   ```

### 4.2 Orchestrator Responsibilities

The **Application Orchestrator** is a Python layer (not part of PDEPhysicalChain library) providing:

**Workflow Management:**
- Startup sequence (temperature stabilization, laser warmup, acquisition start)
- Calibration workflows (SLM flatness, alignment, focus)
- Operational modes (scanning, optimization, data collection)
- Shutdown sequence (safe laser disable, acquisition stop)

**Control Path Integration:**
- MQTT client for temperature monitoring
- MQTT client for laser control
- Alarm handling and safety responses
- Status aggregation (SLM temp, laser state, buffer pool status)

**Error Handling:**
- Timeout handling (DMA, settling, capture)
- Buffer exhaustion recovery (see OPEN_ISSUES.md Test #1)
- SDK error translation and logging
- Graceful degradation strategies

**Configuration Management:**
- YAML config file generation
- Device discovery and enumeration
- Parameter validation before library initialization

**User Interface:**
- REST API for external applications (optional)
- Jupyter notebook integration patterns
- Logging and telemetry export

**NOT Responsibilities of Orchestrator:**
- Low-level SLM/camera control (handled by PDEPhysicalChain)
- FSM state management (handled by PDEPhysicalChain)
- DMA buffer management (handled by PDEPhysicalChain)
- Worker thread management (handled by PDEPhysicalChain)

---

## 5. Threading Architecture

### 5.1 Worker Thread Pattern

**Motivation:**  
Both Meadowlark Blink SDK and Euresys eGrabber SDK are **NOT thread-safe**. All SDK calls must be serialized.

**Implementation:**

```
┌─────────────────────────────────────────────────────┐
│              PDEPhysicalChain Class                  │
│                                                       │
│  ┌────────────────┐         ┌──────────────────┐   │
│  │  Public API    │         │  Worker Thread    │   │
│  │  (User Thread) │         │  (SDK Executor)   │   │
│  └───────┬────────┘         └────────┬─────────┘   │
│          │                           │              │
│          │  Command                  │              │
│          │  Queue ────────────────►  │              │
│          │  (thread-safe)            │              │
│          │                           │              │
│          │  Result                   │              │
│          │  (blocking) ◄─────────────│              │
│          │                           │              │
│          │                           ↓              │
│          │                  ┌────────────────────┐ │
│          │                  │  SLMController     │ │
│          │                  │  Camera 1          │ │
│          │                  │  Camera 2          │ │
│          │                  │  (SDK calls)       │ │
│          │                  └────────────────────┘ │
└──────────┼─────────────────────────────────────────┘
           │
           ↓
    User gets result
    (blocking call returns)
```

**Command Queue:**
- Thread-safe queue (std::queue + std::mutex + std::condition_variable)
- User thread pushes command, blocks on condition variable
- Worker thread pops command, executes, signals completion
- **Tag variable:** Each command includes a unique tag for identification

**Current Implementation (Single Command In-Flight):**
- Only **one command at a time** - fully blocking API
- User thread blocks until command completes before issuing next command
- Simplifies synchronization and error handling
- Sufficient for current POC requirements

**Future Enhancement (Multiple Commands In-Flight):**
- Tag variable added to support future non-blocking API
- Would allow multiple commands queued/executing simultaneously
- User could check command completion by tag
- Requires more complex state tracking and error handling
- Not implemented yet - deferred to future enhancement

**API Blocking Behavior:**
- `setSLMCaptureCamera()` blocks until capture completes (~3-5ms + exposure)
- `acquisitionStart()` blocks until buffer pool allocated
- `acquisitionStop()` blocks until buffers deallocated
- All SDK calls happen on worker thread only
- No concurrent commands - each call blocks until completion

**Benefits:**
- Thread-safety guarantee for SDK calls
- Clean separation: user thread vs SDK thread
- Predictable serialization (FIFO command queue)
- Easy to extend (add new commands)
- Tag infrastructure ready for future non-blocking API

### 5.2 MQTT Monitoring Thread

**Purpose:**  
Continuously publish SLM temperature telemetry to MQTT broker for external monitoring.

**Implementation:**
- Separate thread (`std::thread`) running `mqttMonitoringLoop()`
- Periodically reads SLM temperature (if available from SDK)
- Publishes to MQTT topic: `pde/slm/temperature`
- Uses libmosquitto C++ wrapper

**Frequency:**
- 1 Hz typical (configurable via YAML)
- Non-blocking (does not impact data path performance)

**Status:**
- Runs continuously while PDEPhysicalChain object exists
- Stops gracefully on destruction

---

## 6. Error Handling and Safety

---

**Error Categories:**

1. **Hardware Errors:**
   - SLM communication timeout (PCIe DMA failure)
   - Camera frame loss or corruption
   - Frame grabber DMA buffer exhaustion
   - Temperature sensor disconnect

2. **Timing Errors:**
   - ImageWriteComplete() timeout (SLM settling failure)
   - Camera capture timeout (exposure or readout issue)
   - MQTT publish failure (monitoring thread)

3. **Configuration Errors:**
   - Invalid YAML config (missing devices, bad parameters)
   - Device enumeration failure (serial number mismatch)
   - Buffer pool allocation failure

**Error Handling Strategy:**

**Library Level (PDEPhysicalChain):**
- Return error codes and messages via `CaptureResult` struct
- Log errors to stderr and/or file
- Do NOT throw exceptions (C API compatibility)
- Do NOT auto-retry (application decides)

**Application Level (Orchestrator):**
- Implement retry policies (configurable)
- Safety responses:
  - Disable laser on critical errors
  - Notify via MQTT alerts
  - Log to persistent storage
- Graceful degradation (e.g., continue with single camera if one fails)

**Safety Mechanisms:**

1. **Temperature Monitoring:**
   - MQTT thread continuously publishes SLM temperature
   - Application monitors for out-of-range values
   - Automatic laser disable if temperature alarm

2. **Laser Interlock:**
   - Application-level logic (not in library)
   - Requires temperature stability before laser enable
   - Automatic disable on system errors

3. **Buffer Pool Management:**
   - Library prevents over-subscription (returns error if no buffers)
   - Application must handle buffer exhaustion gracefully
   - See OPEN_ISSUES.md Test #1 for exhaustion behavior

4. **Timeout Protection:**
   - All blocking calls have configurable timeouts
   - Worker thread prevents indefinite hangs
   - Application can abort operations

---

## 7. Hardware Components

### 7.1 Optical Components

| Component | Model | Quantity | Function | Interface |
|-----------|-------|----------|----------|-----------|  
| SLM | Meadowlark 1024x1024 PCIe | 1-2 | Phase modulation | PCIe via Blink SDK |
| Camera | Allied Vision hr25MCX | 1-2 | Image capture | CoaXPress CXP-12 |
| Frame Grabber | Coaxlink Quad CXP-12 Value | 1 | Camera interface, DMA buffers | PCIe 3.0 x8 |
| Laser Diode | DILAS MMF-IS11 | 1 | Pump source (IR) | Laser driver (analog) |
| Gain Crystal | TBD | 1 | Laser emission | N/A |

**Key Hardware Changes from Original Design:**
- **Removed:** External 1 kHz pulse generator
- **Removed:** Programmable delay circuit (0.7ms)
- **Removed:** BNC T-connector and cabling
- **Reason:** Camera has built-in configurable trigger delay

**Frame Grabber Capabilities:**
- 4x CoaXPress CXP-12 connections (6700 MB/s sustained bandwidth per PCIe spec)
- 20 digital I/O lines (differential, TTL, isolated)
- DMA buffer pool managed by eGrabber SDK
- Hardware trigger support (not used - camera has built-in delay)

### 7.2 Control Hardware

| Component | Model | Function | Interface |
|-----------|-------|----------|-----------|
| TEC Driver (SLM) | DC3145A | Temperature control | MQTT (ESP32/RPI) |
| Laser Driver | UNILDD-A-CW-27-60 | Diode current control | MQTT (ESP32), DAC |
| Chiller | HRR030-WF-20-TU | Water cooling | Autonomous |
| Picomotor Controller | 8742 | Optical alignment | USB/Ethernet (vendor API) |

### 7.3 Compute Hardware

| Component | Purpose |
|-----------|---------|
| PC (Linux) | PDEPhysicalChain library, application, MQTT broker |
| NVIDIA GPU | Future: GPU-accelerated pattern generation (not currently used) |
| Raspberry Pi / ESP32 | MQTT-based control nodes (temperature, laser driver) |

**PC Requirements:**
- PCIe 3.0 slots for SLMs and frame grabber
- Linux OS (tested on Ubuntu 20.04+)
- RAM: 16 GB+ (for DMA buffer pools and processing)
- See [PC_Requirements.md](PC_Requirements.md) for details

---

## 8. Software Components

### 8.1 Core Library

| Component | Language | Purpose |
|-----------|----------|---------|
| PDEPhysicalChain | C++17 | Data path orchestration (FSM, worker threads) |
| SLMController | C++17 | Singleton SLM control (Meadowlark Blink SDK wrapper) |
| Camera | C++17 | Multi-instance camera control (eGrabber SDK wrapper) |
| Frame | C++17 | RAII zero-copy DMA buffer wrapper |
| Python bindings | pybind11 | Python API for library |

### 8.2 External Services (Control Path)

| Service | Language | Deployment | Purpose |
|---------|----------|------------|---------|
| MQTT Broker | Mosquitto | PC/Server | Message bus for control path |
| Laser Driver Service | C++/MicroPython | ESP32 | Laser current control via MQTT |
| TEC Controller Service | C++/MicroPython | ESP32 | TEC PWM control via MQTT |
| Temperature Sensor Service | C++/MicroPython | ESP32 | Temperature publishing via MQTT |
| PID Controller Service | Python | RPI/PC | PID loop via MQTT |

### 8.3 Dependencies

**C++ Dependencies:**
- Meadowlark Blink SDK (SLM control)
- Euresys eGrabber SDK 26.02 (frame grabber)
- yaml-cpp (configuration parsing)
- libmosquitto (MQTT client for monitoring)
- pybind11 (Python bindings)

**Build Tools:**
- CMake 3.15+
- C++17 compiler (GCC 7+, Clang 5+)

**Python Dependencies:**
- numpy (array interface)
- pyyaml (config management in orchestrator)
- paho-mqtt (MQTT client in orchestrator)

---

## 9. Communication & Integration

### 9.1 MQTT Topics

**Temperature Control:**
- `pde/slm/temperature` – published by PDEPhysicalChain monitoring thread
- `control/slm1/setpoint` – published by orchestrator or user
- `control/slm1/pwm` – published by PID controller
- `actuators/slm1/tec/pwm` – subscribed by DC3145A TEC driver

**Laser Control:**
- `laser/setpoint` – desired current (A)
- `laser/enable` – on/off command
- `laser/status` – current state and measured current

**System Status:**
- `pde/status` – operational state (Ready, Capturing, Idle, Error)
- `pde/errors` – error messages and alerts
- `pde/metrics` – frame rate, buffer pool utilization

### 9.2 Library API (C++)

**Initialization:**
```cpp
PDEPhysicalChain(const std::string& config_yaml_path);
~PDEPhysicalChain();
```

**Acquisition Control:**
```cpp
void acquisitionStart(int num_buffers);  // Idle → Ready
void acquisitionStop();                   // Ready → Idle
```

**Data Acquisition:**
```cpp
CaptureResult setSLMCaptureCamera(
    const uint8_t* slm_transmission_data,
    const uint8_t* slm_reflection_data  // nullptr if single SLM
);
```

**Configuration:**
```cpp
void setSLMParameter(const std::string& param, double value);
void setCameraParameter(const std::string& camera_name, 
                        const std::string& param, double value);
```

**Status Queries:**
```cpp
std::string getState() const;  // FSM state
double getSLMTemperature() const;
int getAvailableBuffers() const;
```

### 9.3 Library API (Python)

```python
class PDEPhysicalChain:
    def __init__(self, config_yaml_path: str)
    def acquisition_start(self, num_buffers: int) -> None
    def acquisition_stop(self) -> None
    def set_slm_capture_camera(self, 
                               slm_t: np.ndarray,
                               slm_r: Optional[np.ndarray]) -> CaptureResult
    def set_slm_parameter(self, param: str, value: float) -> None
    def set_camera_parameter(self, camera: str, param: str, value: float) -> None
    def get_state(self) -> str
    def get_slm_temperature(self) -> float
    def get_available_buffers(self) -> int

class CaptureResult:
    success: bool
    error_msg: str
    camera1_frame: Optional[Frame]  # Single frame (current)
    camera2_frame: Optional[Frame]  # Single frame (current)
    # Future: camera1_frames: list[Frame]  # Burst mode
    # Future: camera2_frames: list[Frame]  # Burst mode

class Frame:
    def get_data(self) -> np.ndarray  # Zero-copy view
    width: int
    height: int
    timestamp: float
```

---

## 10. Data Management

### 10.1 Configuration Files (YAML)

**Primary Config:**
```yaml
devices:
  slms:
    transmission:
      serial: "SLM-12345"
      lut_file: "/path/to/calibration.lut"
    reflection:  # Optional (omit for single SLM)
      serial: "SLM-67890"
      lut_file: "/path/to/calibration2.lut"
  
  cameras:
    camera1:
      serial: "CAM-11111"
      exposure_us: 10000
      gain_db: 0.0
      trigger_delay_us: 700
    camera2:  # Optional (omit for single camera)
      serial: "CAM-22222"
      exposure_us: 10000
      gain_db: 0.0
      trigger_delay_us: 700

frame_grabber:
  buffer_pool_size: 10
  timeout_ms: 5000

mqtt:
  broker: "localhost"
  port: 1883
  temperature_publish_rate_hz: 1.0
```

### 10.2 Calibration Data

- **SLM LUT Files:** Binary calibration lookup tables (vendor format)
- **Flatness Correction:** Application-specific phase corrections
- **Stored:** Filesystem (referenced in YAML config)

### 10.3 Captured Data

**Image Format:**
- Raw sensor data (Mono8, Mono12, etc. depending on camera config)
- Metadata: timestamp, frame number, exposure settings
- Application saves to disk (HDF5, TIFF, NPY, etc.)

**Telemetry:**
- MQTT messages forwarded to cloud or local logging
- Metrics: frame rate, errors, temperature history
- Stored: Time-series database (InfluxDB) or flat files

---

## 11. Observability

### 11.1 Logging

**Library Logging:**
- stderr output (errors and warnings)
- Optional file logging (configurable via YAML)
- Log levels: ERROR, WARNING, INFO, DEBUG

**Orchestrator Logging:**
- Python logging framework
- Application-specific log files
- Centralized logging (optional: syslog, cloud)

### 11.2 Monitoring

**Real-Time Metrics (MQTT):**
- SLM temperature (1 Hz)
- Frame rate (calculated by application)
- Buffer pool utilization
- Error counts

**Dashboards:**
- Grafana + InfluxDB (time-series visualization)
- Custom Jupyter notebook dashboards

### 11.3 Debugging

**Tools:**
- Verbose logging mode (DEBUG level)
- SDK diagnostics (Meadowlark and Euresys tools)
- PlantUML sequence diagrams for understanding flow
- Unit tests and integration tests

**Common Issues:**
- ImageWriteComplete() timeout → SLM hardware or PCIe issue
- Capture timeout → Camera exposure too long or frame grabber DMA issue
- Buffer exhaustion → Application not releasing Frame objects fast enough

---

## 12. Operational Stages

### 12.1 Stage 1 – Build and Assembly

- Manual operation
- Application provides observability
- Picomotor controller used for alignment
- Limited automation

### 12.2 Stage 2 – Calibration

- Semi-automated workflows via Python scripts
- SLM LUT calibration (vendor tools + custom scripts)
- Image-based feedback loops (e.g., optimize alignment using captured images)
- PID tuning for temperature control

### 12.3 Stage 3 – Normal Operation

- Fully autonomous operation
- Application executes pre-defined workflows (scanning, optimization, data collection)
- Minimal manual intervention
- Deterministic and repeatable behavior

---

## 13. Future Considerations

### 13.1 Planned Enhancements

1. **GPU Memory Support:**
   - Requires SDK vendor updates (Meadowlark and Euresys)
   - Enable CUDA Direct GPU Transfer for zero-copy GPU workflow
   - See OPEN_ISSUES.md for investigation status

2. **Burst Capture Mode:**
   - Current: Single frame per capture command
   - Future: Burst mode capturing multiple consecutive frames
   - Implementation: Single SDK command to camera (easy to add)
   - Return type: `vector<Frame>` (C++) or `list[Frame]` (Python) per camera
   - Use case: High-speed sequences without SLM pattern updates
   - Benefit: Reduced per-frame overhead, better for rapid acquisition

3. **Non-Blocking API / Multiple Commands In-Flight:**
   - Current: Blocking API with one command at a time
   - Future: Non-blocking API allowing multiple queued/executing commands
   - Tag variable infrastructure already in place
   - Requires async completion mechanism (callbacks or polling by tag)
   - Enables higher throughput for independent operations

4. **Multi-Threading Performance:**
   - Parallel SLM writes (if SDK updated to support thread-safety)
   - GPU-accelerated pattern generation (CUDA kernels)

5. **Advanced Trigger Modes:**
   - External trigger input (if needed for synchronization with other equipment)
   - Frame grabber I/O toolbox for complex triggering (currently not used)

### 13.2 Out of Scope (POC)

- Production-grade packaging and deployment
- Web UI or REST API (optional for orchestrator)
- Advanced image processing pipelines (application-specific)
- Data provenance and experiment tracking (application-specific)
- High availability and failover

---

## 14. References

- [System Requirements](requirements.md)
- [PC Requirements](PC_Requirements.md)
- [Detailed Software Design](docs/PDE_Physical_Chain_Design.md)
- [Sequence Diagrams: Dual SLM](docs/setSLMCaptureCamera_with_dual_SLM.puml)
- [Sequence Diagrams: Single SLM](docs/setSLMCaptureCamera_with_single_SLM.puml)
- [Open Issues and Test Requirements](OPEN_ISSUES.md)
- [Design Decisions](decisions.md)
- [Architecture Review: SDR](reviews/SDR.md)
- [Architecture Review: PDR](reviews/PDR.md)
- [Architecture Review: CDR](reviews/CDR.md)
- Meadowlark Blink SDK Documentation
- Euresys eGrabber SDK 26.02 Documentation
- Coaxlink Quad CXP-12 User Manual (docs/datasheets/)
- Allied Vision hr25MCX Datasheet
