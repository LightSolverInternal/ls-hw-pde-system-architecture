# Preliminary Design Review (PDR)
## Laser-Based Computational POC System

---

## Slide 1: Title & Overview

**Preliminary Design Review**  
**Laser-Based Computational POC System**

**Purpose:**
- Demonstration of alternative compute technology using controlled laser manipulation
- End-to-end execution of mathematical problems
- Validation of physical feasibility, stability, and controllability

**Scope:** POC system - not optimized for cost, size, or manufacturability

---

## Slide 2: System Requirements Summary

**Operational Responsibilities:**
- OR-1: Temperature control – gain crystal
- OR-2: Temperature control – SLM devices
- OR-3: Timing and trigger coordination
- OR-4: Diode current control (NEW)

**Key Functional Requirements:**
- FR-1: Controlled laser operation
- FR-2: High degree of automation
- FR-3: Assisted calibration with sensor feedback
- FR-4: Mode-dependent operation
- FR-5: Deterministic and repeatable behavior

**Lifecycle Stages:**
- Stage 1: Build and assembly (manual, assisted)
- Stage 2: Calibration (semi-automated)
- Stage 3: Normal operation (autonomous)

---

## Slide 3: Architecture Principles

**Key Design Principles:**
- **Distributed Control** – autonomous subsystems with pub-sub communication
- **Stateless APIs** – state management at higher layer only
- **Hybrid Orchestration** – microservices + central orchestrator
- **Separation of Concerns** – clear data path vs. control path
- **Configurability** – behavior driven by config files

**Architecture Pattern:**
- Microservices for each hardware subsystem
- MQTT message bus for inter-service communication
- Python-based orchestrator on main PC
- Stateless APIs exposed to application layer

---

## Slide 4: High-Level Architecture Diagram

```
Application Layer (UI, Jupyter, Scripts)
            ↓
    Orchestrator (Python on PC)
    ├── Data Path API
    └── Control Path API
            ↓
      MQTT Broker
            ↓
    ┌───────┼───────┬───────────┐
    ↓       ↓       ↓           ↓
  SLM   Camera   Laser    Temperature
Service Service  Driver   Controllers
```

**Communication:** MQTT pub-sub pattern  
**Deployment:** PC (orchestrator, SLM, camera) + ESP32/RPI nodes (control)

---

## Slide 5: Data Path Architecture

**Purpose:** Manage image flow to SLM and from camera

**Components:**
- SLM Service (Meadowlark 1024x1024, PCIe)
- Camera Service (Allied Vision Prosilica GT 5120NIR)
- Display Server (image upload)
- Capture Server (image download)

**Key APIs:**
- `upload_sequence(images, mode)`
- `capture_sequence(num_frames, mode)`
- `set_trigger_mode(mode)`

**Trigger Modes (Configurable):**
1. SLM triggers Camera
2. Camera triggers SLM
3. External trigger controller

---

## Slide 6: SLM Timing & Synchronization

**Meadowlark SLM (1024x1024) Timing:**
- External trigger input: TTL falling edge
- Output pulse: up to 0.696 ms after trigger
- Image update: 0.696 ms after output pulse
- Total latency: ~1.4 ms per frame

**Synchronization Strategy:**
- Mode-dependent triggering (configurable)
- SMA connectors for trigger I/O
- Optional "flip immediate" mode for reduced latency

**Integration:**
- Orchestrator configures trigger mode per operational workflow
- Deterministic timing ensures repeatability

---

## Slide 7: Control Path Architecture

**Purpose:** Temperature stability, laser control, safety

**Temperature Control (SLM):**
- Sensor → `SLM1/temperature` (MQTT)
- PID Controller → `SLM1/pwm` (MQTT)
- DC3145A TEC Driver → drives TEC (MQTT via ESP32)
- Setpoint: config file + runtime MQTT control

**Laser Diode Control:**
- UNILDD-A-CW-27-60 laser driver
- ESP32 + MQTT interface
- Control: enable/disable + current setpoint (DAC)
- Operation: primarily CW mode
- Periodic validation (every few minutes)

**Cooling:**
- HRR030-WF-20-TU chiller (autonomous, water cooling)

---

## Slide 8: Hardware Components

**Optical:**
- SLM: Meadowlark 1024x1024 (PCIe)
- Camera: Allied Vision Prosilica GT 5120NIR + frame grabber
- Laser Diode: DILAS MMF-IS11 (IR pump)
- Gain Crystal: TBD

**Control Hardware:**
- TEC Driver: DC3145A (MQTT via ESP32/RPI)
- Laser Driver: UNILDD-A-CW-27-60 (MQTT via ESP32)
- Chiller: HRR030-WF-20-TU (autonomous)
- Picomotor Controller: 8742 (calibration/alignment)

**Compute:**
- PC with NVIDIA GPU (SLM, camera, orchestrator)
- Raspberry Pi / ESP32 nodes (distributed control)

---

## Slide 9: Software Components

**Core Services (Python on PC):**
- Orchestrator
- SLM Service (wraps Meadowlark SDK)
- Camera Service (wraps Allied Vision SDK)
- Display Server & Capture Server

**Distributed Services (ESP32/RPI):**
- Laser Driver Service (MQTT)
- TEC Controller Service (MQTT)
- Temperature Sensor Service (MQTT)
- PID Controller Service (MQTT)

**Middleware:**
- MQTT Broker (local)
- MQTT Bridge (metrics to AWS)

**Configuration:**
- YAML/JSON config files (setpoints, error policies, modes)

---

## Slide 10: Communication & MQTT Topics

**Temperature Control:**
- `sensors/slm1/temperature`
- `control/slm1/setpoint`
- `control/slm1/pwm`
- `actuators/slm1/tec/pwm`

**Laser Control:**
- `laser/setpoint` (current in A)
- `laser/enable` (on/off)
- `laser/status` (state, measured current)

**System:**
- `system/state`
- `system/errors`

**Pattern:** Publish-subscribe, loosely coupled

---

## Slide 11: Data Management

**Image Storage:**

**Feedback Images (Control Loop):**
- Temporary storage (memory/local disk)
- Discarded after use
- Used for calibration, alignment, closed-loop control

**Result Images (Computation Output):**
- Persistent local storage
- Uploaded to cloud (AWS)
- Accompanied by metadata (timestamp, config, run ID)

**Formats:** TBD (TIFF, HDF5, NumPy)

**Logs & Metrics:**
- Logs: local files (JSON lines, structured)
- Metrics: MQTT → AWS via bridge

---

## Slide 12: Error Handling & Safety

**Error Policies (Configurable per Error Type):**

| Error Type | Threshold | Response |
|------------|-----------|----------|
| Critical overtemp | SLM > 50°C | Laser shutdown, alert |
| Warning overtemp | SLM > 45°C | Alert, log, continue |
| Trigger timeout | No trigger 5s | Retry or abort |
| Comm loss | MQTT disconnect | Retry, alert |

**Safety Mechanisms:**
- Laser enable/disable via orchestrator
- Automatic disable on critical errors
- Hardware temperature cutoff (if available)
- Graceful shutdown sequence (laser first, then TEC ramp-down)

---

## Slide 13: Observability

**Monitoring:**
- Temperature sensors publish continuously
- Laser driver publishes status periodically
- Orchestrator aggregates system health

**Logging:**
- All services log locally (structured format)
- Timestamp, service, level, message, context

**Metrics:**
- Forwarded to AWS via MQTT bridge
- Frame rates, temperatures, uptime, error counts
- Supports post-experiment analysis

---

## Slide 14: Operational Stages

**Stage 1 – Build and Assembly:**
- Manual operation by trained engineers
- Orchestrator provides observability
- Picomotor controller for alignment
- Limited automation

**Stage 2 – Calibration:**
- Semi-automated workflows
- SLM LUT calibration (vendor tools + custom scripts)
- Image-based feedback loops
- PID tuning for temperature control

**Stage 3 – Normal Operation:**
- Fully autonomous
- Pre-defined workflows executed by orchestrator
- Minimal manual intervention
- Deterministic and repeatable

---

## Slide 15: Development & Integration Plan

**Phase 1: Infrastructure Setup**
- MQTT broker setup
- Basic orchestrator skeleton
- MQTT topic structure and testing

**Phase 2: Control Path**
- Temperature control loop (SLM)
- Laser driver integration
- Error handling and safety testing

**Phase 3: Data Path**
- SLM service (Meadowlark SDK integration)
- Camera service (Allied Vision SDK integration)
- Trigger mode implementation and testing

**Phase 4: Integration & Testing**
- End-to-end workflows
- Calibration procedures
- Performance validation

---

## Slide 16: Risks & Mitigation

**Risk: SLM-Camera Synchronization Complexity**
- Multiple trigger modes increase testing complexity
- **Mitigation:** Modular design, test each mode independently

**Risk: Temperature Stability**
- Critical for repeatable laser operation
- **Mitigation:** Validated PID tuning, hardware-level cutoff

**Risk: Distributed System Coordination**
- MQTT message loss or latency
- **Mitigation:** QoS settings, retry logic, monitoring

**Risk: Integration with Vendor SDKs**
- Meadowlark and Allied Vision SDK learning curve
- **Mitigation:** Early prototyping, vendor support engagement

**Risk: Data Volume**
- High-resolution camera images at high frame rates
- **Mitigation:** GPU acceleration, selective storage, cloud upload

---

## Slide 17: Open Questions & TBDs

**Data Path:**
- Preferred API style (sync vs async vs transaction-based)
- Image storage format (TIFF, HDF5, NumPy)
- Trigger mode selection logic

**Control Path:**
- Gain crystal temperature control interface (vendor-specific)
- Chiller status monitoring (if available)

**Integration:**
- REST API design for orchestrator
- User interface (CLI, web UI, Jupyter integration)
- Data provenance and experiment tracking

**Performance:**
- Target frame rates for different operational modes
- Latency budgets for trigger-to-capture sequences

---

## Slide 18: Next Steps

**Immediate:**
- Finalize component procurement (if not complete)
- Set up development environment (PC, MQTT broker)
- Initialize orchestrator repository structure

**Short Term:**
- Implement basic MQTT communication infrastructure
- Integrate SLM SDK (Meadowlark) and test image upload
- Implement temperature control loop (SLM TEC)

**Medium Term:**
- Integrate camera SDK and test capture
- Implement trigger modes and synchronization testing
- Laser driver integration and safety testing

**Long Term:**
- End-to-end workflow testing
- Calibration procedure automation
- Performance optimization and validation

---

## Slide 19: Success Criteria

**PDR Exit Criteria:**
- Architecture documented and reviewed ✓
- Requirements traceability established
- Key technical risks identified
- Integration plan defined

**System-Level Success Criteria:**
- Controlled laser operation (startup, steady-state, shutdown)
- Temperature stability within defined tolerances
- Deterministic SLM-camera synchronization
- Automated calibration workflows (assisted)
- Repeatable computation results

---

## Slide 20: Summary

**Architecture Highlights:**
- Distributed microservices with MQTT communication
- Hybrid orchestration (stateless APIs + high-level coordination)
- Clear separation: data path vs control path
- Configurable trigger modes and error policies
- Safety-first design (laser control, temperature monitoring)

**Key Components:**
- Meadowlark SLM 1024x1024 (PCIe)
- Allied Vision camera + frame grabber
- DILAS laser diode + UNILDD driver
- DC3145A TEC controllers (MQTT)
- Python orchestrator on GPU-equipped PC

**Status:** Ready to proceed to implementation phase

---

## Backup Slides

### Backup: Picomotor Controller Integration

**Component:** 8742 Picomotor Controller

**Purpose:** Optical alignment (calibration stage)

**Interface:** USB/Ethernet (vendor API)

**Usage:**
- Fine adjustment of optical elements during build
- Automated or semi-automated calibration routines
- Integration with image-processing feedback loops

**Status:** Purchased, integration plan TBD

---

### Backup: MQTT QoS Strategy

**QoS Levels:**
- **QoS 0** (At most once): Non-critical telemetry
- **QoS 1** (At least once): Commands, setpoints
- **QoS 2** (Exactly once): Safety-critical (laser enable/disable)

**Retained Messages:**
- Setpoints (temperature, laser current)
- System state

**Last Will Testament:**
- Services publish offline status on disconnect

---

### Backup: Configuration File Structure

**Example YAML:**
```yaml
temperature_control:
  slm1:
    setpoint: 25.0  # Celsius
    pid:
      kp: 1.0
      ki: 0.1
      kd: 0.01
    thresholds:
      warning: 45.0
      critical: 50.0

laser:
  current_setpoint: 10.0  # Amps
  enable: false
  mode: CW

trigger_mode: external_controller

errors:
  critical_overtemp:
    action: shutdown_laser
  warning_overtemp:
    action: alert
```

---

### Backup: Technology Stack Summary

**Languages:**
- Python: Orchestrator, SLM/camera services, high-level control
- C++/MicroPython: ESP32 firmware (laser driver, TEC, sensors)

**Frameworks:**
- MQTT: Eclipse Mosquitto broker
- SDKs: Meadowlark Blink (SLM), Allied Vision Vimba (camera)

**Infrastructure:**
- OS: Linux (PC), FreeRTOS/MicroPython (ESP32)
- Cloud: AWS (metrics, data storage)
- Version Control: Git

**Development Tools:**
- VS Code, Jupyter notebooks
- Datasheets stored in repo (`docs/datasheets/`)
