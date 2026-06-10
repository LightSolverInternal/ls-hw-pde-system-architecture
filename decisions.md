# Design Decisions

## Purpose

This document records key architectural and implementation decisions made during the development of the PDE Physical Chain system. Each decision includes rationale, alternatives considered, and trade-offs.

---

## Table of Contents

1. [Programming Language: C++17](#1-programming-language-c17)
2. [Monolithic Library vs Microservices](#2-monolithic-library-vs-microservices)
3. [Singleton SLMController Pattern](#3-singleton-slmcontroller-pattern)
4. [Multi-Instance Camera Pattern](#4-multi-instance-camera-pattern)
5. [Worker Thread for SDK Serialization](#5-worker-thread-for-sdk-serialization)
6. [Zero-Copy Frame Class (RAII)](#6-zero-copy-frame-class-raii)
7. [YAML Configuration](#7-yaml-configuration)
8. [CPU Memory Only (No GPU Memory)](#8-cpu-memory-only-no-gpu-memory)
9. [Python Bindings via pybind11](#9-python-bindings-via-pybind11)
10. [Hardware Trigger from SLM (No External Generator)](#10-hardware-trigger-from-slm-no-external-generator)
11. [Finite State Machine (FSM)](#11-finite-state-machine-fsm)
12. [MQTT for Monitoring](#12-mqtt-for-monitoring)
13. [Shared Library Packaging](#13-shared-library-packaging)
14. [Single-Frame Capture (Burst Mode Deferred)](#14-single-frame-capture-burst-mode-deferred)

---

## 1. Programming Language: C++17

### Decision
Implement the PDEPhysicalChain library in **C++17**.

### Rationale
1. **RAII Patterns:** Automatic resource management (DMA buffers, SDK handles) critical for reliability
2. **Move Semantics:** Efficient ownership transfer of Frame objects without copying
3. **Smart Pointers:** Automatic lifetime management (`unique_ptr` for Frame buffers)
4. **Performance:** Zero-copy data path requires low-level memory control
5. **SDK Requirements:** Meadowlark and Euresys SDKs are C/C++ libraries

### Alternatives Considered
- **Python:** Too slow for low-latency control, no RAII, difficult to manage DMA buffers
- **C:** No RAII, no move semantics, harder to maintain and extend
- **Rust:** Learning curve too steep for team, SDK bindings more complex

### Trade-Offs
- **Pro:** Better performance, safer resource management, cleaner API
- **Con:** Longer development time vs Python, requires compilation

### Status
**Implemented.** C++17 selected for core library with Python bindings for user applications.

---

## 2. Monolithic Library vs Microservices

### Decision
Use a **unified C++17 library** (PDEPhysicalChain) instead of distributed microservices.

### Rationale
1. **SDK Thread-Safety Constraint:** Meadowlark and Euresys SDKs are NOT thread-safe
2. **Serialization Required:** All SDK calls must be serialized on a single thread
3. **Timing Criticality:** Microsecond-level coordination between SLMs and cameras
4. **Simplified Deployment:** Single library instead of multiple services
5. **Reduced IPC Overhead:** No MQTT/network latency in critical path

### Alternatives Considered
- **Microservices Architecture:**
  - Separate SLM1 Service, SLM2 Service, Camera Service
  - MQTT for communication
  - **Rejected:** SDK thread-safety makes this impractical, timing too loose

### Trade-Offs
- **Pro:** Simpler architecture, better performance, enforced serialization
- **Con:** Less distributed, cannot scale horizontally (not needed for POC)

### Status
**Implemented.** Original microservices design replaced with unified library.

---

## 3. Singleton SLMController Pattern

### Decision
Implement **SLMController as a singleton** managing all SLM devices by serial number.

### Rationale
1. **Meadowlark SDK Requirement:** SDK architecture assumes single global controller
2. **Resource Sharing:** All SLMs share PCIe resources and SDK state
3. **Initialization Order:** SDK must be initialized once before any SLM access
4. **Thread-Safety:** Single controller ensures serialized access via worker thread

### Alternatives Considered
- **Multi-Instance SLM Class:**
  - One object per SLM device
  - **Rejected:** Conflicts with Meadowlark SDK internal architecture
  - Would require complex coordination of SDK initialization

### Trade-Offs
- **Pro:** Matches SDK design, simple initialization, clear ownership
- **Con:** Global state (singleton pattern), slightly less flexible

### Status
**Implemented.** SLMController is a singleton, accessed via `SLMController::getInstance()`.

---

## 4. Multi-Instance Camera Pattern

### Decision
Implement **Camera as a multi-instance class** (one per physical camera).

### Rationale
1. **Clean API:** Each camera object represents one physical device
2. **Independent Control:** Cameras have independent settings (exposure, gain, delay)
3. **eGrabber SDK Design:** SDK supports multiple GenTL cameras per frame grabber
4. **RAII Lifecycle:** Camera object lifetime tied to acquisition state

### Alternatives Considered
- **Singleton CameraController:**
  - Similar to SLMController pattern
  - **Rejected:** Not required by eGrabber SDK architecture
  - Would make API more awkward (all operations require camera ID)

### Trade-Offs
- **Pro:** Cleaner API, matches SDK design, better encapsulation
- **Con:** Slightly more complex initialization (enumerate devices)

### Status
**Implemented.** One Camera instance per physical camera, managed by PDEPhysicalChain.

---

## 5. Worker Thread for SDK Serialization

### Decision
Use a **dedicated worker thread** executing SDK calls from a command queue. **Current implementation supports only one command in-flight at a time** (blocking API).

### Rationale
1. **SDK Thread-Safety:** Neither Meadowlark nor Euresys SDKs are thread-safe
2. **Forced Serialization:** Queue ensures FIFO execution order
3. **Blocking API:** User thread blocks on condition variable until worker completes
4. **Predictable Execution:** No race conditions, deterministic order
5. **Single Command Simplicity:** One command at a time simplifies synchronization and error handling

### Alternatives Considered
- **Multi-Threading with Locks:**
  - Mutex protection around SDK calls
  - **Rejected:** Still risk SDK internal state corruption
- **Async API with Callbacks:**
  - Non-blocking API with completion callbacks
  - **Rejected:** More complex for users, harder to debug
- **Multiple Commands In-Flight:**
  - Allow multiple queued commands executing/completing asynchronously
  - **Deferred:** Tag variable added for future support, but not implemented yet

### Trade-Offs
- **Pro:** Guaranteed thread-safety, simple mental model, debuggable
- **Con:** Slight overhead (queue + context switch), user thread blocks

### Implementation Details
- **Tag Variable:** Each command includes a unique tag for identification
  - **Current use:** Not actively used (single command blocks until completion)
  - **Future use:** Will enable tracking multiple in-flight commands
  - **Benefit:** Infrastructure ready for future non-blocking API without breaking changes

### Status
**Implemented.** Worker thread runs continuously, executes commands from queue. Single command in-flight (fully blocking). Tag infrastructure present for future enhancement.

---

## 6. Zero-Copy Frame Class (RAII)

### Decision
Implement **Frame class with RAII** wrapping DMA buffers as `unique_ptr<ScopedBuffer>`.

### Rationale
1. **Buffer Lifecycle:** DMA buffer must return to pool when no longer needed
2. **RAII Guarantee:** Frame destructor automatically releases buffer
3. **Zero-Copy:** Application reads directly from DMA memory (no copy)
4. **Ownership Transfer:** Move semantics transfer ownership to application
5. **Prevent Leaks:** Impossible to forget to release buffer

### Alternatives Considered
- **Manual Buffer Management:**
  - Application calls `release_buffer(buffer_id)`
  - **Rejected:** Easy to forget, leads to buffer pool exhaustion
- **Reference Counting:**
  - `shared_ptr` for buffers
  - **Rejected:** Unnecessary overhead, ownership should be unique

### Trade-Offs
- **Pro:** Safe, automatic, zero-copy, prevents resource leaks
- **Con:** Application must not use data pointer after Frame destroyed

### Status
**Implemented.** Frame objects own DMA buffers via `unique_ptr`, automatic release on destruction.

---

## 7. YAML Configuration

### Decision
Use **YAML configuration files** for runtime device and parameter configuration.

### Rationale
1. **Flexibility:** Switch between 1-2 SLMs and 1-2 cameras without recompilation
2. **Human-Readable:** Easy to edit and understand
3. **Version Control:** Config files tracked in Git
4. **Development vs Production:** Different configs for different modes
5. **Parameter Tuning:** Exposure, gain, delays adjustable without code changes

### Alternatives Considered
- **Hardcoded Configuration:**
  - Device IDs and parameters in C++ code
  - **Rejected:** Requires recompilation for every change
- **Command-Line Arguments:**
  - Pass parameters as CLI args
  - **Rejected:** Too many parameters, difficult to manage
- **JSON:**
  - Similar to YAML
  - **Rejected:** Less human-readable (more verbose)

### Trade-Offs
- **Pro:** Flexible, readable, version-controlled, easy to change
- **Con:** Requires yaml-cpp dependency, runtime parsing overhead (minimal)

### Status
**Implemented.** YAML config specifies SLM/camera serial numbers and all parameters.

---

## 8. CPU Memory Only (No GPU Memory)

### Decision
**Current implementation uses CPU memory (RAM)** exclusively. GPU memory is a future enhancement.

### Rationale
1. **SDK Limitation:** Meadowlark SDK bindings reject `cudaMallocHost()` pointers
2. **Frame Grabber SDK:** eGrabber 26.02 does not expose CUDA Direct GPU Transfer API
3. **POC Scope:** CPU-only meets performance requirements for POC
4. **Vendor Engagement Required:** GPU support requires SDK updates from vendors

### Alternatives Considered
- **GPU Memory (cudaMallocHost):**
  - Zero-copy SLM patterns and camera captures in GPU memory
  - **Rejected (for now):** SDK limitations prevent implementation
- **Manual GPU↔CPU Copies:**
  - Use cudaMemcpy for transfers
  - **Accepted:** Application can do this if needed (flexibility)

### Trade-Offs
- **Pro:** Works with current SDK versions, simpler implementation
- **Con:** Cannot eliminate GPU↔CPU copies if application uses GPU

### Future Path
- Engage with Meadowlark to add `cudaMallocHost()` support
- Engage with Euresys to expose CUDA Direct GPU Transfer API
- See [OPEN_ISSUES.md](OPEN_ISSUES.md) for investigation status

### Status
**Implemented** (CPU-only). GPU memory support deferred to future enhancement.

---

## 9. Python Bindings via pybind11

### Decision
Use **pybind11** to create Python bindings for PDEPhysicalChain C++ library.

### Rationale
1. **User Productivity:** Python is primary language for experiments and workflows
2. **Numpy Integration:** Zero-copy numpy views of Frame buffers
3. **Jupyter Notebooks:** Interactive experimentation and visualization
4. **Minimal Overhead:** pybind11 is lightweight, efficient bindings
5. **C++ Ecosystem:** Standard choice for C++/Python interop

### Alternatives Considered
- **SWIG:**
  - Older binding generator
  - **Rejected:** More verbose, less modern, harder to maintain
- **ctypes:**
  - Python standard library for C FFI
  - **Rejected:** Requires C API, no automatic type conversions
- **Cython:**
  - Python superset compiling to C
  - **Rejected:** Requires writing Cython code, steeper learning curve

### Trade-Offs
- **Pro:** Excellent Python/C++ interop, automatic type conversions, numpy integration
- **Con:** Adds pybind11 dependency, slight compilation complexity

### Status
**Implemented.** Python module `pde_physical_chain` wraps C++ library via pybind11.

---

## 10. Hardware Trigger from SLM (No External Generator)

### Decision
Use **hardware trigger pulse directly from SLM** with camera's built-in programmable delay. **Eliminate external function generator, delay circuit, and BNC splitter.**

### Rationale
1. **Camera Built-In Delay:** Allied Vision hr25MCX has configurable trigger delay (e.g., 700 µs)
2. **Simpler System:** Fewer components (no external generator, no delay circuit, no cables)
3. **More Reliable:** Fewer failure points, less wiring complexity
4. **Sufficient Timing:** Camera delay ensures SLM settled before capture
5. **Cost:** Eliminates need for external hardware (~$200-500 savings)

### Alternatives Considered
- **External 1 kHz Pulse Generator + Delay Circuit:**
  - Original design with function generator, programmable delay, BNC T-connector
  - **Rejected:** Unnecessary complexity, camera has built-in capability
- **Frame Grabber I/O Toolbox:**
  - Use frame grabber digital I/O for timing
  - **Rejected:** Simpler to use camera's built-in delay

### Trade-Offs
- **Pro:** Simpler, cheaper, more reliable, fewer cables
- **Con:** Delay is per-camera (not global), but this is acceptable

### Status
**Implemented.** External trigger hardware removed, camera built-in delay used.

---

## 11. Finite State Machine (FSM)

### Decision
Implement **FSM with states: Uninitialized → Idle → Ready ⟲ (First_SLM_write → Second_SLM_write → Capture → Ready)**.

### Rationale
1. **Clear Lifecycle:** Explicit states for acquisition lifecycle (start/stop)
2. **Error Prevention:** API calls only valid in specific states
3. **Debuggability:** State visible to application for diagnostics
4. **Predictable Behavior:** Deterministic state transitions

### Alternatives Considered
- **Stateless API:**
  - No explicit state machine
  - **Rejected:** Hard to prevent misuse (e.g., capture before acquisition start)
- **Implicit State:**
  - Internal state not exposed to application
  - **Rejected:** Harder to debug, less visibility

### Trade-Offs
- **Pro:** Clear contract, prevents misuse, easy to debug
- **Con:** Slightly more complex API (must call acquisitionStart before capture)

### Status
**Implemented.** FSM enforces valid API call sequences, state queryable via `getState()`.

---

## 12. MQTT for Monitoring

### Decision
Use **MQTT for temperature telemetry** published from PDEPhysicalChain library.

### Rationale
1. **Separation of Concerns:** Temperature monitoring is control path, not data path
2. **Asynchronous:** Does not block data path operations
3. **External Visibility:** Application or external services can monitor via MQTT
4. **Existing Infrastructure:** MQTT broker already used for laser/TEC control
5. **libmosquitto:** Lightweight C++ library for MQTT client

### Alternatives Considered
- **Application Polling:**
  - Application calls `getSLMTemperature()` periodically
  - **Rejected:** Requires application to manage polling, less flexible
- **Callback API:**
  - Library calls application-provided callback with temperature
  - **Rejected:** More coupling, harder to integrate with external systems

### Trade-Offs
- **Pro:** Decoupled, flexible, integrates with existing MQTT infrastructure
- **Con:** Requires MQTT broker running, adds libmosquitto dependency

### Status
**Implemented.** MQTT monitoring thread publishes `pde/slm/temperature` at 1 Hz.

---

## 13. Shared Library Packaging

### Decision
Build PDEPhysicalChain as a **versioned shared library** (`libpde_physical_chain.so.1.0.0`).

### Rationale
1. **Modularity:** Library can be used by multiple applications
2. **Versioning:** Semantic versioning (MAJOR.MINOR.PATCH) for API compatibility
3. **Dynamic Linking:** Application can update library without recompilation (if ABI compatible)
4. **Python Bindings:** pybind11 module links against shared library
5. **Standard Practice:** Matches Unix/Linux conventions

### Alternatives Considered
- **Static Library:**
  - `.a` file linked at compile time
  - **Rejected:** Larger binaries, must recompile app for library updates
- **Header-Only Library:**
  - All code in header files
  - **Rejected:** Increases compile time, exposes implementation

### Trade-Offs
- **Pro:** Flexible, standard, supports versioning and dynamic linking
- **Con:** Requires proper installation, library path management (LD_LIBRARY_PATH)

### Status
**Implemented.** CMake builds `libpde_physical_chain.so.1.0.0` with symlinks.

---

## 14. Single-Frame Capture (Burst Mode Deferred)

### Decision
Implement **single-frame capture** mode only. Each `setSLMCaptureCamera()` call returns one frame per camera. **Burst mode deferred to future enhancement.**

### Rationale
1. **Simplicity:** Single-frame model easier to implement and debug for POC
2. **Sufficient for POC:** Current experiments don't require burst capture
3. **API Clarity:** Clear one-to-one mapping: one call → one result
4. **RAII Simplicity:** Single Frame object lifetime easier to manage
5. **Easy to Add Later:** Burst mode requires minimal SDK changes (single camera command)

### Current Implementation
- `CaptureResult` contains:
  - `camera1_frame: Optional<Frame>` - single frame
  - `camera2_frame: Optional<Frame>` - single frame
- Each capture operation returns exactly one frame per camera
- Application loops to capture multiple frames with different SLM patterns

### Future Enhancement: Burst Mode
- Single camera command captures N consecutive frames
- Return type: `vector<Frame>` (C++) or `list[Frame]` (Python) per camera
- **Easy to implement:** Camera SDK already supports burst acquisition
- **Use case:** High-speed sequences without SLM pattern changes
- **Benefit:** Reduced per-frame overhead, better for rapid acquisition

### Alternatives Considered
- **Burst Mode Initially:**
  - Implement burst capture from the start
  - **Deferred:** Added complexity not needed for POC
  - Can be added later without breaking existing API
- **Configurable Mode:**
  - Support both single and burst via parameter
  - **Deferred:** Keep API simple for now

### Trade-Offs
- **Pro:** Simpler implementation, clearer API, meets current needs
- **Con:** Must call function N times for N frames (slight overhead)

### Status
**Implemented** (single-frame only). Burst mode documented as future enhancement, easy to add with single SDK command.

---

## Summary of Key Decisions

| Decision | Selected Approach | Key Reason |
|----------|------------------|------------|
| Language | C++17 | RAII, move semantics, performance |
| Architecture | Monolithic library | SDK thread-safety constraint |
| SLM Control | Singleton pattern | Meadowlark SDK requirement |
| Camera Control | Multi-instance | Clean API, matches SDK |
| Threading | Worker thread + queue | SDK serialization |
| Memory | RAII Frame class | Automatic buffer release |
| Configuration | YAML | Flexibility, human-readable |
| GPU Memory | CPU-only (current) | SDK limitations |
| Python Bindings | pybind11 | Best C++/Python interop |
| Trigger | Camera built-in delay | Simpler than external hardware |
| State Management | FSM | Clear lifecycle, prevent misuse |
| Monitoring | MQTT | Decoupled, existing infrastructure |
| Packaging | Shared library | Modularity, versioning |
| Capture Mode | Single-frame (current) | Simplicity; burst easy to add later |

---

## Change History

| Date | Author | Change |
|------|--------|--------|
| 2026-06-10 | Avital | Initial version - documented implementation decisions |

---

## References

- [Architecture Document](architecture.md)
- [Detailed Design](docs/PDE_Physical_Chain_Design.md)
- [Open Issues](OPEN_ISSUES.md)
- Meadowlark Blink SDK Documentation
- Euresys eGrabber SDK 26.02 Documentation
