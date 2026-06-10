# PDE Physical Chain - Software Design Document

**Version:** 2.0  
**Date:** June 10, 2026  
**Author:** System Architecture Team  
**Status:** Reflects actual C++17 implementation

---

## 1. Overview

The **PDE Physical Chain** provides a unified C++17 library for controlling the complete SLM and camera hardware chain:
- **1-2 SLMs** (Meadowlark 1024x1024 PCIe) - Configurable for dev (1 SLM) or production (2 SLMs)
- **1-2 Cameras** (Allied Vision hr25MCX) via CXP-12 frame grabber - Configurable for dev/production
- **1 Frame Grabber** (Coaxlink Quad CXP-12 Value, Euresys eGrabber SDK 26.02)
- **YAML Configuration** - Runtime device selection by serial number

### Key Features
- **C++17 implementation**: RAII, move semantics, smart pointers
- **FSM-based control**: State machine for acquisition lifecycle
- **Zero-copy DMA**: Frame ownership transfer via RAII
- **Worker thread architecture**: Serializes SDK access (SDKs not thread-safe)
- **MQTT monitoring**: Continuous temperature telemetry
- **Python bindings**: pybind11 wrapper for Python integration
- **Flexible configuration**: Supports 1 SLM/camera (dev) or 2 SLM/camera (production)

---

## 2. Architecture Overview

### 2.1 System Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Application (C++ or Python)                       │
│  - Generates SLM patterns                                            │
│  - Controls Frame lifetime (ownership transfer)                      │
│  - Processes captured images                                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
           ┌───────────────────┴────────────────────┐
           │      PDEPhysicalChain Class (Main)      │
           │  - FSM state machine                     │
           │  - Worker thread for SDK access          │
           │  - MQTT monitoring thread                │
           │  - Coordinates SLM + Camera              │
           └────┬──────────────┬────────────┬────────┘
                │              │            │
    ┌───────────┴──┐    ┌──────┴─────┐   ┌─┴────────────┐
    │SLMController │    │  Camera(s) │   │    Frame     │
    │ (Singleton)  │    │ (1 or 2)   │   │ (Zero-copy)  │
    └──────┬───────┘    └──────┬─────┘   └──────────────┘
           │                   │
    ┌──────┴──────────┐ ┌─────┴────────────────────┐
    │ Meadowlark SDK  │ │  Euresys eGrabber SDK    │
    │ (Blink SDK)     │ │  (GenICam/GenTL)         │
    │ - 1-2 SLMs      │ │  - Frame grabber control │
    │ - PCIe          │ │  - DMA buffer management │
    └─────────────────┘ └──────────────────────────┘
```

### 2.2 Key Design Patterns

**Singleton Pattern (SLMController)**
- Required by Meadowlark SDK architecture
- Manages 1-2 SLMs internally (slm_t and/or slm_r)
- Device discovery by serial number

**Multi-Instance Pattern (Camera)**
- One Camera instance per physical device
- Each wraps its own eGrabber interface
- Move-only class (no copying)

**RAII Ownership Transfer (Frame)**
- Frame object holds unique_ptr to DMA buffer
- Ownership transfers from library to application
- Buffer automatically released back to pool when Frame destroyed

**Worker Thread Pattern**
- SDKs are not thread-safe
- Worker thread serializes all SDK operations
- Command queue for thread-safe communication

---

## 3. Class Design

### 3.1 PDEPhysicalChain (Main Orchestrator)

**Responsibilities:**
- FSM state management
- SLM and camera coordination
- Worker thread management
- MQTT temperature monitoring

**Finite State Machine:**

```
┌──────────────┐
│ Uninitialized│
└──────┬───────┘
       │ Constructor (loads YAML config)
       ↓
  ┌────────┐
  │  Idle  │
  └────┬───┘
       │
       │ acquisitionStart(N)
       │ (allocates N DMA buffers)
       ↓
  ┌────────┐              
  │ Ready  │◄─────────────────────────┐
  └────┬───┘                          │
       │                              │
       │ setSLMCaptureCamera()        │
       │                              │
       ↓                              │
  ┌──────────────┐                    │
  │First_SLM_write│                   │
  └──────┬────────┘                   │
         │                            │
         ↓                            │
  ┌──────────────┐                    │
  │Second_SLM_write│                  │
  └──────┬─────────┘                  │
         │                            │
         ↓                            │
   ┌─────────┐                        │
   │ Capture │────────────────────────┘
   └─────────┘   (returns Frame objects,
       │          transitions back to Ready)
       │
       │ acquisitionStop()
       │ (deallocates buffers)
       ↓
  ┌────────┐
  │  Idle  │
  └────────┘
```

**Header (pde_physical_chain.h):**

```cpp
#include <memory>
#include <string>
#include <vector>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include "slm.h"
#include "camera.h"
#include "frame.h"
#include <yaml-cpp/yaml.h>
#include <mosquitto.h>

enum class State {
    Uninitialized,
    Idle,
    Ready,
    First_SLM_write,
    Second_SLM_write,
    Capture
};

struct CaptureResult {
    bool success;
    std::optional<Frame> camera1_frame;  // Only if camera1 configured
    std::optional<Frame> camera2_frame;  // Only if camera2 configured
    std::string error_message;
};

class PDEPhysicalChain {
public:
    // Constructor: Loads YAML config, initializes devices
    explicit PDEPhysicalChain(const std::string& config_path);
    
    // Destructor: Stops worker threads, releases resources
    ~PDEPhysicalChain();
    
    // Move-only (no copying)
    PDEPhysicalChain(const PDEPhysicalChain&) = delete;
    PDEPhysicalChain& operator=(const PDEPhysicalChain&) = delete;
    PDEPhysicalChain(PDEPhysicalChain&&) = default;
    PDEPhysicalChain& operator=(PDEPhysicalChain&&) = default;
    
    // Acquisition lifecycle
    void acquisitionStart(int num_buffers);  // Idle → Ready
    void acquisitionStop();                  // Ready → Idle
    
    // Main capture function
    // Transitions: Ready → First_SLM_write → Second_SLM_write → Capture → Ready
    CaptureResult setSLMCaptureCamera(
        const uint8_t* slm_t_data,   // nullptr if single-SLM config
        const uint8_t* slm_r_data    // nullptr if single-SLM config
    );
    
    // Runtime parameter control
    void setExposure(int camera_id, double exposure_us);
    void setGain(int camera_id, double gain_db);
    void setTriggerDelay(int camera_id, double delay_us);
    
    // Temperature monitoring
    double getSLMTemperature(int slm_id) const;
    double getCameraTemperature(int camera_id) const;
    
    // State query
    State getState() const { return state_; }
    
private:
    // Configuration
    YAML::Node config_;
    bool has_slm_t_;
    bool has_slm_r_;
    bool has_camera1_;
    bool has_camera2_;
    
    // FSM state
    State state_;
    std::mutex state_mutex_;
    
    // Hardware components
    std::unique_ptr<SLMController> slm_controller_;  // Singleton
    std::unique_ptr<Camera> camera1_;
    std::unique_ptr<Camera> camera2_;
    
    // Worker thread (SDK not thread-safe)
    std::thread worker_thread_;
    std::queue<std::function<void()>> command_queue_;
    std::mutex queue_mutex_;
    std::condition_variable queue_cv_;
    std::atomic<bool> worker_running_;
    
    void workerThreadLoop();
    template<typename Func>
    auto executeOnWorker(Func&& func) -> decltype(func());
    
    // MQTT monitoring thread
    std::thread mqtt_thread_;
    std::atomic<bool> mqtt_running_;
    mosquitto* mqtt_client_;
    std::string mqtt_broker_;
    int mqtt_port_;
    
    void mqttThreadLoop();
    void publishTemperatures();
    
    // Internal methods
    void loadConfiguration(const std::string& config_path);
    void initializeDevices();
    void validateStateTransition(State from, State to);
};
```

---

### 3.2 SLMController (Singleton)

**Why Singleton:** Meadowlark Blink SDK architecture requires single controller managing all SLMs.

**Header (slm.h):**

```cpp
#include <memory>
#include <string>
#include <Blink_SDK.h>  // Meadowlark SDK

class SLMController {
public:
    // Get singleton instance
    static SLMController& getInstance();
    
    // Initialize with 1 or 2 SLMs by serial number
    void initialize(const std::string& slm_t_serial,
                   const std::string& slm_r_serial = "");
    
    // SLM operations
    void writeImage(int slm_id, const uint8_t* data, size_t size);
    void loadLinearLUT(int slm_id);
    double readTemperature(int slm_id);
    
    // Device info
    bool hasTransmissionSLM() const { return slm_t_board_ != -1; }
    bool hasReflectionSLM() const { return slm_r_board_ != -1; }
    
    ~SLMController();
    
private:
    SLMController() = default;  // Private constructor
    SLMController(const SLMController&) = delete;
    SLMController& operator=(const SLMController&) = delete;
    
    int slm_t_board_ = -1;  // Board ID for transmission SLM
    int slm_r_board_ = -1;  // Board ID for reflection SLM
    
    void discoverSLM(const std::string& serial, int& board_id);
};
```

**Key Implementation Details:**
- Discovers SLMs by serial number (not index-based)
- Manages 0-2 SLMs depending on configuration
- All SDK calls go through this single instance
- CPU memory only (SDK limitation)

---

### 3.3 Camera (Multi-Instance)

**Why Multi-Instance:** Each camera gets its own eGrabber interface for independent control.

**Header (camera.h):**

```cpp
#include <memory>
#include <string>
#include <EGrabber.h>  // Euresys SDK
#include "frame.h"

class Camera {
public:
    // Constructor: Opens camera by index (0 or 1)
    explicit Camera(int camera_index, const std::string& pixel_format = "Mono12");
    
    // Move-only (wraps unique hardware resource)
    Camera(const Camera&) = delete;
    Camera& operator=(const Camera&) = delete;
    Camera(Camera&&) = default;
    Camera& operator=(Camera&&) = default;
    
    ~Camera();
    
    // Acquisition lifecycle (allocates/deallocates DMA buffer pool)
    void startAcquisition(int num_buffers);
    void stopAcquisition();
    
    // Zero-copy capture (returns locked DMA buffer via Frame object)
    Frame grabFrameZeroCopy(int timeout_ms = 5000);
    
    // Runtime parameters
    void setExposure(double exposure_us);
    void setGain(double gain_db);
    void setTriggerDelay(double delay_us);  // Built-in camera delay
    
    // Monitoring
    double getTemperature();
    
    // Camera info
    int getWidth() const { return width_; }
    int getHeight() const { return height_; }
    std::string getPixelFormat() const { return pixel_format_; }
    
private:
    std::unique_ptr<Euresys::EGrabber<>> grabber_;
    int width_;
    int height_;
    std::string pixel_format_;
    bool acquisition_started_ = false;
};
```

**Key Implementation Details:**
- One instance per physical camera
- Built-in trigger delay (no external function generator needed)
- Returns Frame objects for zero-copy ownership transfer
- DMA buffer pool allocated during `startAcquisition()`
- Buffer released back to pool when Frame destroyed

---

### 3.4 Frame (Zero-Copy RAII)

**Critical for Performance:** Frame class enables zero-copy DMA buffer management with automatic lifetime control.

**Header (frame.h):**

```cpp
#include <memory>
#include <cstddef>
#include <EGrabber.h>

class Frame {
public:
    // Constructor: Takes ownership of ScopedBuffer
    explicit Frame(std::unique_ptr<Euresys::ScopedBuffer> buffer);
    
    // Move-only (unique ownership of DMA buffer)
    Frame(const Frame&) = delete;
    Frame& operator=(const Frame&) = delete;
    Frame(Frame&&) = default;
    Frame& operator=(Frame&&) = default;
    
    // Destructor: Automatically releases buffer back to pool
    ~Frame();
    
    // Data access
    void* getData() const;
    size_t getSize() const;
    int getWidth() const;
    int getHeight() const;
    uint64_t getTimestamp() const;
    
    // Metadata
    uint64_t getFrameID() const;
    std::string getPixelFormat() const;
    
private:
    std::unique_ptr<Euresys::ScopedBuffer> buffer_;
};
```

**Ownership Transfer Pattern:**

```cpp
// Library allocates and locks buffer from pool
Frame frame = camera.grabFrameZeroCopy();

// Ownership transfers to application
// Buffer is locked and unavailable to pool

process_image(frame.getData(), frame.getSize());

// When frame goes out of scope, destructor releases buffer
// Buffer automatically returns to pool
// Ready for next capture
```

**Why This Matters:**
- **Zero-copy**: Application gets direct pointer to DMA buffer
- **Automatic cleanup**: No manual buffer management
- **Pool exhaustion control**: Application controls buffer lifetime
- **Thread-safe**: RAII ensures proper cleanup even with exceptions

---

## 4. YAML Configuration

**Example: config/pde_config.yaml**

```yaml
devices:
  slms:
    # Set serial to "" or omit to disable
    transmission:
      serial: "SLM-12345-T"
      width: 1024
      height: 1024
    reflection:
      serial: "SLM-67890-R"
      width: 1024
      height: 1024
  
  cameras:
    # Set serial to "" or omit to disable
    camera1:
      index: 0
      serial: "CAM-11111"
      exposure_us: 1000.0
      gain_db: 0.0
      trigger_delay_us: 50.0  # Built-in camera delay
      pixel_format: "Mono12"
    camera2:
      index: 1
      serial: "CAM-22222"
      exposure_us: 1000.0
      gain_db: 0.0
      trigger_delay_us: 50.0
      pixel_format: "Mono12"

mqtt:
  broker: "localhost"
  port: 1883
  topic_prefix: "pde/physical_chain"
  publish_interval_ms: 1000

acquisition:
  default_buffer_count: 10
  timeout_ms: 5000
```

**Single-SLM/Camera Development Configuration:**

```yaml
devices:
  slms:
    transmission:
      serial: "SLM-12345-T"
      width: 1024
      height: 1024
    # reflection: omitted - single SLM mode
  
  cameras:
    camera1:
      index: 0
      serial: "CAM-11111"
      exposure_us: 1000.0
      gain_db: 0.0
      trigger_delay_us: 50.0
      pixel_format: "Mono12"
    # camera2: omitted - single camera mode

mqtt:
  broker: "localhost"
  port: 1883
  topic_prefix: "pde/dev"
  publish_interval_ms: 1000
```

---

## 5. Timing and Sequence

**Detailed Sequence Diagrams:**
- [**Dual SLM/Camera Configuration**](setSLMCaptureCamera_with_dual_SLM.puml) - Production mode (2 SLMs + 2 cameras)
- [**Single SLM/Camera Configuration**](setSLMCaptureCamera_with_single_SLM.puml) - Development mode (1 SLM + 1 camera)

**Key Timing Characteristics:**

| Phase | Duration | Notes |
|-------|----------|-------|
| SLM DMA | ~1ms per SLM | PCIe bandwidth limited |
| SLM Pixel Settling | ~1ms per SLM | Physics-limited |
| `ImageWriteComplete()` | Blocks until stable | **Critical for dual-SLM** |
| Hardware Trigger | Immediate | From SLM output pulse |
| Camera Built-in Delay | Configurable | Set via `setTriggerDelay()` |
| Camera Capture | Varies | Depends on exposure |
| **Total (dual-SLM)** | **~3-5ms + exposure** | Plus processing time |

**Critical Implementation Details:**

1. **ImageWriteComplete() After First SLM:**
   ```cpp
   slm_controller_->writeImage(SLM_T, data);  // DMA ~1ms
   slm_controller_->ImageWriteComplete(SLM_T); // Wait ~1ms
   // First SLM now STABLE - prevents race condition
   slm_controller_->writeImage(SLM_R, data);  // DMA ~1ms
   // Second SLM settling happens while waiting for trigger
   ```
   
   **Why Critical:** Without waiting for SLM_T to stabilize before writing SLM_R, there's a race condition where SLM_T might still be settling when cameras trigger.

2. **Hardware Trigger (Not Software):**
   - Trigger pulse comes **FROM SLM hardware** (output pulse)
   - Camera has **built-in configurable delay**
   - No external function generator needed
   - Ensures both SLMs are settled before capture

3. **Zero-Copy Buffer Flow:**
   ```
   DMA Pool → grabFrameZeroCopy() → Frame object → Application → ~Frame() → DMA Pool
   ```
   Buffer lifecycle controlled by Frame object lifetime (RAII).

---

## 6. Memory Management

**Current Implementation: CPU Memory Only**

The implementation currently uses **CPU RAM only** (malloc/new). Both Meadowlark and Euresys SDKs do not currently support GPU memory or pinned host memory (cudaMallocHost).

**Memory Requirements:**

| Component | Memory Type | Size (per device) | Notes |
|-----------|-------------|-------------------|-------|
| SLM input | CPU RAM (required) | 1024×1024 = 1 MB | Meadowlark SDK limitation |
| Camera output | CPU RAM (DMA) | Variable (resolution dependent) | hr25MCX: up to 5120×5120×2 bytes |

**Buffer Pool:**
- Allocated during `acquisitionStart(N)`
- N buffers in DMA pool
- Each `grabFrameZeroCopy()` locks one buffer
- Buffer released when Frame destroyed
- **Critical**: If application holds N frames without releasing, pool is exhausted

**Future GPU Support:**
- Would require SDK updates from vendors
- Direct GPU DMA would eliminate CPU→GPU copy
- Currently requires explicit CPU→GPU memcpy after capture

---

## 7. Worker Thread Architecture

**Why Needed:** Both Meadowlark Blink SDK and Euresys eGrabber SDK are **not thread-safe**.

**Implementation:**

```cpp
// Main thread
void PDEPhysicalChain::setSLMCaptureCamera(...) {
    // This call blocks but doesn't directly access SDK
    auto result = executeOnWorker([&]() {
        // Worker thread executes this lambda
        slm_controller_->writeImage(...);
        camera1_->grab FrameZeroCopy(...);
        return result;
    });
    return result;
}

// Worker thread loop
void PDEPhysicalChain::workerThreadLoop() {
    while (worker_running_) {
        std::function<void()> command;
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            queue_cv_.wait(lock, [&]{ return !command_queue_.empty() || !worker_running_; });
            if (!worker_running_) break;
            command = std::move(command_queue_.front());
            command_queue_.pop();
        }
        command();  // Execute on worker thread
    }
}
```

**Thread Safety Model:**
- Application can call APIs from any thread
- All SDK access serialized through worker thread
- Command queue provides thread-safe communication
- MQTT monitoring runs on separate thread (suspended during capture)

---

## 8. Example Usage

### 8.1 C++ Example

```cpp
#include "pde_physical_chain.h"
#include <iostream>

int main() {
    try {
        // Load configuration
        PDEPhysicalChain chain("config/pde_config.yaml");
        
        // Start acquisition (allocates 10 DMA buffers)
        chain.acquisitionStart(10);
        
        // Allocate SLM patterns (CPU memory)
        std::vector<uint8_t> slm_t_data(1024 * 1024);
        std::vector<uint8_t> slm_r_data(1024 * 1024);
        
        // Capture loop
        for (int i = 0; i < 100; i++) {
            // Generate patterns
            generatePattern(slm_t_data.data(), i);
            generatePattern(slm_r_data.data(), i);
            
            // Capture (blocks until complete)
            CaptureResult result = chain.setSLMCaptureCamera(
                slm_t_data.data(),
                slm_r_data.data()
            );
            
            if (result.success) {
                // Process frames (zero-copy access)
                if (result.camera1_frame) {
                    processFrame(*result.camera1_frame);
                }
                if (result.camera2_frame) {
                    processFrame(*result.camera2_frame);
                }
                // Frames auto-release when result goes out of scope
            } else {
                std::cerr << "Capture failed: " << result.error_message << std::endl;
            }
        }
        
        // Stop acquisition (deallocates buffers)
        chain.acquisitionStop();
        
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
        return 1;
    }
    
    return 0;
}
```

### 8.2 Python Example (via pybind11)

```python
import pde_physical_chain as pde
import numpy as np

def main():
    # Load configuration
    chain = pde.PDEPhysicalChain("config/pde_config.yaml")
    
    # Start acquisition
    chain.acquisition_start(10)
    
    # Capture loop
    for i in range(100):
        # Generate patterns (numpy arrays)
        slm_t = np.random.randint(0, 256, (1024, 1024), dtype=np.uint8)
        slm_r = np.random.randint(0, 256, (1024, 1024), dtype=np.uint8)
        
        # Capture
        result = chain.set_slm_capture_camera(slm_t, slm_r)
        
        if result.success:
            # Process captured frames
            if result.camera1_frame is not None:
                img1 = np.array(result.camera1_frame, copy=False)  # Zero-copy view
                process_image(img1)
            
            if result.camera2_frame is not None:
                img2 = np.array(result.camera2_frame, copy=False)
                process_image(img2)
        else:
            print(f"Capture failed: {result.error_message}")
    
    # Stop acquisition
    chain.acquisition_stop()

if __name__ == "__main__":
    main()
```

---

## 9. Dependencies

### Software Dependencies
- **Meadowlark Blink SDK** (PCIe SLM driver)
- **Euresys eGrabber SDK** (26.02+) - Frame grabber and camera control
- **yaml-cpp** - Configuration file parsing
- **libmosquitto** - MQTT temperature monitoring
- **pybind11** (optional) - Python bindings
- **C++17 compiler** (gcc 7+, clang 5+, MSVC 2017+)
- **CMake** 3.15+

### Hardware Dependencies
- **Meadowlark 1024x1024 PCIe SLMs** (1-2 devices)
- **Euresys Coaxlink Quad CXP-12 Value** frame grabber
- **Allied Vision hr25MCX** cameras (1-2 devices) via CoaXPress

**Note:** Function generator NOT required (camera has built-in trigger delay)

---

## 10. Thread Safety and Concurrency

**NOT Thread-Safe:** PDEPhysicalChain is designed for single-threaded application use.

- Do not call methods from multiple threads simultaneously
- Internal worker thread handles SDK serialization
- MQTT monitoring thread is internal only

**Safe Usage:**
```cpp
// Single thread calling APIs
chain.setSLMCaptureCamera(...);
chain.setExposure(...);
```

**Unsafe Usage:**
```cpp
// Multiple threads - NOT SUPPORTED
std::thread t1([&]{ chain.setSLMCaptureCamera(...); });
std::thread t2([&]{ chain.setExposure(...); });  // UNDEFINED BEHAVIOR
```

---

## 11. Performance Considerations

### Typical Timing
- **SLM write**: ~1ms DMA + ~1ms settling = ~2ms per SLM
- **Camera capture**: Depends on exposure time + readout
- **Total per cycle**: ~3-5ms + exposure time

### Bottlenecks
1. SLM settling time (physics-limited)
2. PCIe bandwidth for large SLM patterns
3. DMA buffer pool exhaustion if application doesn't release frames

### Optimization Tips
- Use `acquisitionStart()` with sufficient buffer count (10-20 recommended)
- Release Frame objects promptly (don't accumulate)
- Pre-allocate SLM patterns to avoid malloc in loop
- Use MQTT monitoring sparingly (adds overhead)

---

## 12. Future Enhancements

### Planned
- [ ] GPU memory support (requires SDK updates from vendors)
- [ ] Async API variant with callbacks
- [ ] Multi-frame burst capture mode
- [ ] Built-in pattern generation utilities

### Under Consideration
- [ ] Multiple SLM pair support (>2 SLMs)
- [ ] Network-distributed operation
- [ ] ROI (Region of Interest) capture support

---

## 13. References

- [Meadowlark PCIe SLM User Manual](../docs/datasheets/PCIe_User_Manual_1024x1024.txt)
- [Coaxlink Frame Grabber Datasheet](../docs/datasheets/Coaxlink_Quad_CXP-12_Value.txt)
- [System Architecture Document](../architecture.md)
- [Dual SLM Sequence Diagram](../docs/setSLMCaptureCamera_with_dual_SLM.puml)
- [Single SLM Sequence Diagram](../docs/setSLMCaptureCamera_with_single_SLM.puml)
- [Implementation Repository](https://github.com/your-org/ls-hw-pde-physical-chain)

---

**Document Status:** Updated to match C++17 implementation  
**Last Updated:** June 10, 2026
