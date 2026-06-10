# PDE System – Workstation PC Requirements

**Version:** 2.0  
**Date:** April 28, 2026  
**Purpose:** Unified PC specification for complete PDE imaging system

---

## System Components Summary

This workstation will support:
- **2× Meadowlark PCIe SLMs** (1024×1024, Transmission + Reflection)
- **2× Allied Vision HR25MCX cameras** (CXP-12) via Coaxlink frame grabber
- **3× Allied Vision Prosilica GT1200 cameras** (GigE) via PoE switch
- **GPU-accelerated processing** for real-time imaging

---

## 1. Form Factor

**Requirement:**
- Desktop / workstation tower (ATX or mATX)
- Full-size tower preferred for thermal management and expansion

**Explicitly Not Allowed:**
- Laptops
- Mini-PCs or NUCs
- Rack-mount servers (unless explicitly approved for thermal design)

---

## 2. Operating System

**Required:**
- Ubuntu 20.04 LTS (64-bit)
- HWE kernel 5.15.x
- **Secure Boot: DISABLED**

**Compatibility Notes:**
- Meadowlark SDK requires Ubuntu 20.04 with kernel 5.15.x
- Newer Ubuntu versions (22.04+) or kernel 6.x are **NOT supported**

**Explicitly Not Allowed:**
- Virtual machines (VMware, VirtualBox, KVM)
- Docker containers
- WSL (Windows Subsystem for Linux)

---

## 3. CPU

**Minimum:**
- x86-64 Intel or AMD processor
- 4 physical cores (8 threads)

**Recommended:**
- Intel i7-12700 / AMD Ryzen 7 5800X or better
- 8+ physical cores for parallel camera acquisition + GPU processing
- Support for AVX2 instructions (for image processing)

---

## 4. Memory (RAM)

**Minimum:** 16 GB DDR4

**Recommended:** 32 GB DDR4 or DDR5
- Supports buffering for 5 cameras + GPU transfers
- Typical usage: ~12 GB (OS + applications + frame buffers)

---

## 5. Storage

**Minimum:**
- 500 GB NVMe SSD (OS + applications + data)
- Separate data drive recommended for image storage

**Recommended:**
- 1 TB NVMe SSD (OS + applications)
- Additional 2+ TB SATA SSD or HDD for image archives

---

## 6. GPU (Mandatory)

**Required:**
- Dedicated NVIDIA GPU with CUDA support
- OpenCL 1.2 or higher (required by Meadowlark SDK)
- Minimum 6 GB VRAM
- Installed in physical PCIe x16 slot

**Recommended GPUs:**
- NVIDIA RTX 3060 (12 GB) or RTX 4060 (8 GB)
- NVIDIA RTX 3070 / RTX 4070
- NVIDIA T1000 / T400 (for professional workstations)

**Explicitly Not Allowed:**
- Intel integrated graphics
- AMD GPUs
- Passthrough or virtualized GPUs

---

## 7. PCIe Slot Requirements

**Critical:** This system requires **4 PCIe devices**. Verify motherboard has sufficient slots.

| Device | Slot Requirement | Notes |
|--------|------------------|-------|
| **NVIDIA GPU** | PCIe x16 (Gen 3/4) | Full bandwidth required |
| **Meadowlark SLM Controller** | PCIe x8 or x16 (Gen 3) | 1 slot per SLM<br>**2 slots total for 2 SLMs** |
| **Coaxlink Quad CXP-12** | PCIe x8 (Gen 3) | Frame grabber for HR25MCX cameras |

**Minimum Slot Configuration:**
- 1× PCIe x16 (GPU)
- 2× PCIe x8 or x16 (2 SLMs)
- 1× PCIe x8 (Coaxlink frame grabber)

**Total: 4 PCIe slots (1× x16 + 3× x8 minimum)**

**Important:**
- All slots must be directly on motherboard (no PCIe bridges, riser cards, or adapters)
- Slots must be physically accessible (check case and cooling clearances)
- Verify PCIe lane allocation in BIOS (some motherboards share lanes)

---

## 8. Network Interface Card (NIC)

### 8.1 For Prosilica GT1200 Cameras (Required)

**Specification:**
- Dual or Quad-port Gigabit Ethernet NIC with PoE+ support
- **OR** Standard GigE NIC + external PoE switch (8+ ports)

**Option A: Integrated PoE NIC (Preferred)**
- Example: Intel I350-T4 with PoE injector
- 4× RJ45 Gigabit Ethernet ports
- Supports 802.3af/at PoE via external injector or PoE switch
- Jumbo frames (MTU 9000) support

**Option B: Standard Multi-Port NIC + PoE Switch (Recommended)**
- NIC: Intel I350-T4 or similar (4-port GigE)
- External PoE switch: 8-port GigE with PoE+ (802.3at)
  - Minimum 30W per port for camera power
  - Managed switch preferred (VLAN, QoS support)
  - Examples: Netgear GS308P, TP-Link TL-SG1005P

**Requirements:**
- 3 ports for GT1200 cameras (can expand to more)
- 1 port for management/internet (optional)
- Jumbo Frames enabled (MTU 9000) for optimal throughput
- Dedicated subnet for camera traffic (e.g., 192.168.10.0/24)

**Total Bandwidth:**
- 3× GT1200 @ 1920×1200, Mono8, 100 fps ≈ 690 MB/s aggregate
- Well within GigE capacity with proper configuration

### 8.2 General Network

**Recommended:**
- Onboard Gigabit Ethernet for general network access (separate from cameras)
- If using integrated NIC for cameras, second NIC for internet access recommended

---

## 9. Power Supply

**Minimum:** 650W 80+ Bronze

**Recommended:** 750W 80+ Gold or Platinum
- Accounts for: GPU (250W), CPU (125W), 2 SLMs (50W), frame grabber (20W), cameras via PoE (external), peripherals
- Modular PSU preferred for cable management
- **Must have 6-pin or 8-pin PCIe power connectors** for Coaxlink auxiliary power

---

## 10. BIOS / Firmware Settings

**Required Configuration:**

| Setting | Value | Reason |
|---------|-------|--------|
| **Secure Boot** | **OFF** | Meadowlark drivers require unsigned kernel modules |
| **IOMMU** (VT-d / AMD-Vi) | **ON** | Required for PCIe device isolation |
| **PCIe ASPM** (Power Saving) | **OFF** | Prevents frame grabber timing issues |
| **Above 4G Decoding** | **ON** | If available, helps with PCIe resource allocation |
| **Resizable BAR** | Optional | Can improve GPU performance |

---

## 11. Cooling & Thermal Management

**Recommended:**
- CPU cooler rated for 150W+ TDP
- Case with at least 3× 120mm fans (2 intake, 1 exhaust)
- Adequate clearance for GPU (check length: 270-310mm)
- Positive air pressure to reduce dust

---

## 12. Frame Grabber – Coaxlink Quad CXP-12

**Model:** Euresys Coaxlink Quad CXP-12 Value

**Specifications:**
- 4× CoaXPress CXP-12 connections (using 2 for HR25MCX cameras)
- PCIe 3.0 x8 interface (6,700 MB/s sustained bandwidth)
- 20× digital I/O lines for triggering
- Power over CoaXPress (PoCXP): 17W per channel @ 24V
- Micro-BNC (HD-BNC) connectors

**Requirements:**
- PCIe 3.0 x8 slot (or x16 running at x8)
- 6-pin PCIe auxiliary power cable from PSU
- CoaXPress cables (included with cameras or purchased separately)

**Software:**
- Euresys eGrabber SDK (Linux supported)
- GenICam / GenTL compliant

---

## 13. Cables & Connectors

**Coaxpress (for HR25MCX):**
- 2× CXP-12 coaxial cables (75Ω, Micro-BNC)
- Length: ≤40m @ CXP-12 speed

**GigE (for GT1200):**
- 3× Cat6 or Cat6a Ethernet cables
- Shielded recommended for EMI environments
- Length: ≤100m per camera

---

## 14. Software & Drivers

**Operating System:**
- Ubuntu 20.04.6 LTS with HWE kernel 5.15.x

**Required SDKs:**
- Meadowlark Blink SDK (v5.x for Ubuntu 20.04)
- Allied Vision Vimba X SDK (latest, for GT1200)
- Euresys eGrabber SDK 26.02 (for Coaxlink frame grabber and HR25MCX cameras)

**GPU Drivers:**
- NVIDIA proprietary driver (v525+ or latest production)
- CUDA Toolkit 11.8 or 12.x (for future GPU memory support)

**Build Tools (for PDEPhysicalChain library):**
- CMake 3.15 or newer
- GCC 7+ or Clang 5+ (C++17 support required)

**C++ Dependencies:**
- yaml-cpp (YAML configuration parsing)
- libmosquitto (MQTT client for temperature monitoring)
- pybind11 (Python bindings)

**Python Environment:**
- Python 3.8 - 3.10
- Packages: `vmbpy` (Allied Vision), NumPy, OpenCV, paho-mqtt, PyYAML

---

## 15. Validation Checklist

Before deployment, verify:

### Hardware
- [ ] 4 PCIe slots populated (GPU + 2 SLMs + Coaxlink)
- [ ] All devices detected in `lspci`
- [ ] GPU shows full x16 Gen3/4 link speed
- [ ] Coaxlink shows x8 Gen3 link speed
- [ ] PoE switch powers all 3 GT1200 cameras
- [ ] Jumbo frames enabled on camera NIC (`ip link show`)

### BIOS
- [ ] Secure Boot: OFF
- [ ] IOMMU: ON
- [ ] PCIe ASPM: OFF

### Software
- [ ] Meadowlark SDK installed, OpenCL test passed
- [ ] Vimba X Viewer detects all 3 GT1200 cameras
- [ ] eGrabber detects Coaxlink and connected HR25MCX cameras
- [ ] NVIDIA driver loaded (`nvidia-smi` works)

### Network
- [ ] Camera subnet configured (e.g., 192.168.10.0/24)
- [ ] Static IPs assigned to cameras or DHCP reservations set
- [ ] Ping test to all 3 GT1200 cameras succeeds
- [ ] Bandwidth test: `iperf3` between cameras and PC

---

## 16. Recommended Workstation Examples

**Pre-built Options:**
1. **Dell Precision 5820 Tower**
   - Xeon W or Core i9
   - 4× PCIe slots (1× x16, 3× x8)
   - Supports Ubuntu 20.04

2. **HP Z4 G4 Workstation**
   - Xeon W-2200 series
   - Multiple PCIe configurations
   - Good thermal design

3. **Lenovo ThinkStation P520**
   - Xeon W-2200 series
   - 7× PCIe slots
   - Ubuntu certified

**Custom Build:**
- Motherboard: ASUS Pro WS X570-ACE or Supermicro X12 series
- Ensure 4+ PCIe x8 slots with full lane allocation

---

## 17. Budget Estimate (USD, 2026)

| Component | Estimated Cost |
|-----------|----------------|
| Workstation base (CPU, RAM, storage, PSU, case) | $1,500 - $2,500 |
| NVIDIA RTX 3060/4060 | $400 - $600 |
| Coaxlink Quad CXP-12 | $2,500 - $3,000 |
| GigE PoE switch (8-port) | $150 - $300 |
| Multi-port NIC (optional) | $100 - $200 |
| Cables & accessories | $200 |
| **Total (excluding cameras & SLMs)** | **$4,850 - $6,800** |

---

## 18. Important Notes

1. **PCIe Lane Limitations:** Most consumer motherboards share PCIe lanes. Verify that installing 4 devices doesn't disable slots or reduce bandwidth.

2. **Thermal Considerations:** With 2 SLMs, GPU, and frame grabber, ensure adequate cooling. Monitor temperatures during operation.

3. **Network Isolation:** Keep camera traffic on dedicated subnet. Do not route internet traffic through camera NIC.

4. **Kernel Module Signing:** Meadowlark and Euresys drivers may require disabling Secure Boot and manual module signing.

5. **Real-Time Performance:** For deterministic timing, consider:
   - Disabling CPU frequency scaling (set to performance governor)
   - Isolating CPU cores for real-time tasks
   - Using PREEMPT_RT kernel (advanced)

---

## 19. Support & Documentation

**Meadowlark:**
- Technical support: support@meadowlark.com
- PCIe SLM User Manual

**Euresys:**
- eGrabber documentation: https://documentation.euresys.com/
- Support portal: https://www.euresys.com/support

**Allied Vision:**
- Vimba X documentation: https://www.alliedvision.com/vimba-x
- Application notes for GigE performance tuning

---

**Document Prepared By:** System Architecture Team  
**Review Status:** Draft v2.0  
**Next Review:** Before procurement
