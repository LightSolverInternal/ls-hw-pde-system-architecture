# ls-hw-pde-system-architecture

This repository is the single source of truth for the PDE system design and architecture documentation.

**Scope:**
- System requirements and specifications
- Architecture and design decisions
- Design reviews (SDR / PDR / CDR)
- HW / FW / SW decomposition and interfaces
- Component datasheets and reference documentation

**Note:** This repository contains documentation only. The actual C++17 implementation is in the [ls-hw-pde-physical-chain](../ls-hw-pde-physical-chain/) repository.

---

## Key Documents

### Core Documentation
- **[System Architecture](architecture.md)** - High-level system design, unified C++17 library architecture
- **[Design Decisions](decisions.md)** - Key architectural decisions and rationale
- **[Requirements](requirements.md)** - System requirements and specifications
- **[PC Requirements](PC_Requirements.md)** - Workstation hardware and software specifications

### Detailed Design
- **[PDE Physical Chain Design](docs/PDE_Physical_Chain_Design.md)** - Detailed C++17 library design, FSM, threading, API
- **[Sequence Diagrams (Dual SLM)](docs/setSLMCaptureCamera_with_dual_SLM.puml)** - PlantUML diagram for production mode (2 SLMs + 2 cameras)
- **[Sequence Diagrams (Single SLM)](docs/setSLMCaptureCamera_with_single_SLM.puml)** - PlantUML diagram for development mode (1 SLM + 1 camera)
- **[SLM Timing Reference](docs/SLM_Timing_Reference.md)** - Meadowlark SLM timing characteristics

### Project Management
- **[Open Issues](OPEN_ISSUES.md)** - Open technical questions and test requirements
- **[Risks](risks.md)** - Risk register and mitigation strategies

### Design Reviews
- **[SDR (System Design Review)](reviews/SDR.md)**
- **[PDR (Preliminary Design Review)](reviews/PDR.md)**
- **[CDR (Critical Design Review)](reviews/CDR.md)**

---

## Architecture Overview

The PDE Physical Chain system uses a **unified C++17 library** architecture:
- **PDEPhysicalChain Library:** FSM-based orchestration of SLMs and cameras
- **Singleton SLMController:** Manages 1-2 Meadowlark SLMs via PCIe
- **Multi-Instance Camera:** One per Allied Vision hr25MCX camera (CoaXPress)
- **Worker Thread:** Serializes SDK access (Meadowlark and Euresys SDKs not thread-safe)
- **Zero-Copy Frame:** RAII-based DMA buffer ownership transfer
- **YAML Configuration:** Runtime device selection and parameter tuning
- **Python Bindings:** pybind11 wrapper for user applications

**Key Change from Original Design:**  
The microservices-based architecture evolved into a unified C++17 library due to SDK thread-safety constraints and performance requirements.

---

## Implementation Repository

**Actual Code:** [ls-hw-pde-physical-chain](../ls-hw-pde-physical-chain/)
- C++17 library source code
- Python bindings via pybind11
- CMake build system
- Configuration examples
- Unit tests and integration tests

---

## Documentation Updates

This documentation is actively maintained to reflect the actual implementation. Major updates:
- **June 10, 2026:** Rewrote architecture.md and PDE_Physical_Chain_Design.md for C++17 implementation
- **June 10, 2026:** Created decisions.md documenting key design decisions
- **June 10, 2026:** Added PlantUML sequence diagrams (dual and single SLM modes)
- **June 10, 2026:** Removed external trigger hardware from design (camera has built-in delay)

---