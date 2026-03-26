# Open Issues and Questions

This document tracks open technical questions, ambiguities in vendor documentation, and items requiring clarification or testing.

---

## Hardware & Vendor Documentation

### Issue #1: ImageWriteComplete() Blocking Behavior - Meadowlark SLM

**Status:** Open - Awaiting vendor clarification  
**Priority:** High  
**Component:** Meadowlark 1024x1024 SLM PCIe SDK  
**Date Raised:** March 25, 2026

**Question:**
Does `ImageWriteComplete()` function block (wait) until the hardware is ready, or does it return immediately with a status code that requires polling?

**Context:**
The PCIe User Manual (1024x1024) datasheet contains language that suggests blocking behavior:
- Section 4.3.7 states: "the software will **stop waiting** for a trigger when the timeout condition is met"
- The phrase "stop waiting" implies the function blocks/waits

However:
- The function returns `int` (1 for success, 0 for failure/timeout)
- This return pattern is also consistent with a status check/poll function

**Impact on Implementation:**
- **If blocking:** Use simple pattern:
  ```python
  success = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
  if not success:
      handle_timeout()
  ```

- **If polling required:** Use loop pattern:
  ```python
  done = False
  while not done:
      done = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
  ```

**Questions for Meadowlark:**
1. Does `ImageWriteComplete()` block until hardware is ready (or timeout)?
2. Or does it return immediately with 0/1 status requiring polling?
3. In external trigger mode, when does the function return relative to:
   - Trigger arrival?
   - Memory bank flip?
   - Pixel display completion?
4. What is the typical return time in software timing mode vs. external trigger mode?

**Related Documentation:**
- [docs/SLM_Timing_Reference.md](docs/SLM_Timing_Reference.md)
- PCIe User Manual (1024x1024), Section 4.3.7

**Action Items:**
- [ ] Contact Meadowlark technical support
- [ ] Test actual SDK behavior empirically
- [ ] Update timing reference document with confirmed behavior
- [ ] Update architecture document if flow changes

---

### Issue #2: Pixel Voltage Transition Behavior - Meadowlark SLM

**Status:** Open - Awaiting vendor clarification  
**Priority:** Medium  
**Component:** Meadowlark 1024x1024 SLM Hardware  
**Date Raised:** March 26, 2026

**Question:**
During image updates, do pixel voltages transition directly from the old value to the new value, or do they reset to 0V (or another intermediate state) before settling to the target voltage?

**Context:**
When updating a pixel value from one image to the next (e.g., from value A in IMG1 to value B in IMG2):
- The datasheet documents the 0.696ms loading time
- The double-buffering mechanism is explained
- However, the physical voltage transition behavior at the pixel level is not documented

**Possible Behaviors:**
1. **Direct transition:** VA → VB (most likely for analog pixel addressing)
2. **Reset transition:** VA → 0V → VB (may occur if buffers are cleared)
3. **Other intermediate state:** VA → Vx → VB

**Impact on System:**
- Important for synchronization with measurement equipment
- May affect optical performance during transitions
- Could influence minimum settling time requirements

**Question for Meadowlark:**
During image updates, do pixel voltages transition directly from the old value to the new value, or do they reset to 0V (or another intermediate state) before settling to the target voltage?

**Related Documentation:**
- [docs/SLM_Timing_Reference.md](docs/SLM_Timing_Reference.md) - Section on pixel loading
- PCIe User Manual (1024x1024) - Section 1.2 (Phase Modulation)

**Action Items:**
- [ ] Contact Meadowlark technical support
- [ ] Update documentation with confirmed behavior
- [ ] Assess impact on synchronization requirements

---

### Issue #3: Multiple Trigger Handling Behavior - Meadowlark SLM

**Status:** Open - Awaiting vendor clarification  
**Priority:** Medium  
**Component:** Meadowlark 1024x1024 SLM External Trigger Mode  
**Date Raised:** March 26, 2026

**Question:**
If multiple external triggers arrive between `Write_image()` returning and `ImageWriteComplete()` returning, what happens to the additional triggers?

**Context:**
When using external trigger mode with a continuous trigger source:
```python
Write_image(img)        # DMA completes (~1ms)
# Hardware now waiting for trigger
# Trigger 1 arrives     → Causes flip + 0.696ms loading
# Trigger 2 arrives     → ??? (during loading)
# Trigger 3 arrives     → ??? (during loading)
ImageWriteComplete()    # Returns after loading complete
```

**Possible Behaviors:**
1. **Only first trigger processed**: Subsequent triggers ignored while hardware busy (most likely)
2. **Triggers queued**: Hardware buffers multiple triggers (unlikely given no mention in spec)
3. **Triggers cause errors**: Multiple triggers generate error conditions
4. **Last trigger wins**: Hardware latches most recent trigger edge

**What the Spec Says:**
- "Triggers received before a DMA is started but after ImageWriteComplete has returned will be ignored"
- "An output pulse is generated to acknowledge receipt of **the trigger**" (singular)
- No mention of trigger queuing, buffering, or multiple trigger handling

**Impact on System:**
- Critical for continuous trigger sources (e.g., 1kHz pulse generator)
- Affects robustness of timing architecture
- May need trigger gating/synchronization logic

**Questions for Meadowlark:**
1. If multiple external triggers arrive between `Write_image()` returning and `ImageWriteComplete()` returning:
   - Does only the **first trigger** cause a memory bank flip?
   - Are subsequent triggers ignored during the flip/loading period (0.696ms)?
2. Regarding output pulses:
   - Is only **one output pulse** generated (acknowledging the first trigger)?
   - Or does the hardware generate an output pulse for each received trigger?
3. Is there any trigger queuing or buffering mechanism in the hardware?
4. What is the recommended practice: should external trigger sources be gated to provide exactly one trigger per `Write_image()` call?

**Related Documentation:**
- [docs/SLM_Timing_Reference.md](docs/SLM_Timing_Reference.md) - External trigger mode
- PCIe User Manual (1024x1024), Section 3.6 (Triggering), Section 4.3.7 (ImageWriteComplete)

**Action Items:**
- [ ] Contact Meadowlark technical support
- [ ] Test behavior with oscilloscope/logic analyzer
- [ ] Update timing reference with confirmed behavior
- [ ] Design trigger gating logic if needed

---

## System Architecture

### Issue #4: Trigger Mode Selection Logic

**Status:** Open - Design decision needed  
**Priority:** Medium  
**Date Raised:** March 25, 2026

**Question:**
When should the system use software timing vs. external hardware triggers?

**Options:**
1. **Software Timing Mode:**
   - Pros: Simpler, no external hardware
   - Cons: Less precise synchronization with camera/external equipment

2. **External Trigger Mode:**
   - Pros: Precise timing control, better sync with camera
   - Cons: Requires trigger hardware (Arduino/FPGA/DAQ etc.)

**Open Sub-Questions:**
- Should mode be configurable per operational stage (calibration vs. normal operation)?
- What trigger source should be used (Arduino, FPGA, dedicated timing controller)?
- How to coordinate external trigger timing to achieve 1000 fps target?

**Related Documentation:**
- [architecture.md](architecture.md) - Section 3.1 Data Path Subsystem
- [docs/SLM_Timing_Reference.md](docs/SLM_Timing_Reference.md)

**Action Items:**
- [ ] Define trigger mode requirements per operational stage
- [ ] Select/specify external trigger hardware if needed
- [ ] Design trigger timing coordination mechanism

---

## Data Management

### Issue #4: Image Storage Format and Strategy

**Status:** Open - Requirements clarification needed  
**Priority:** Medium  
**Date Raised:** March 24, 2026

**Question:**
What format should be used for storing captured images, and what is the retention policy?

**Context from Requirements:**
- Feedback images (control loop): temporary, can be discarded
- Result images (computation output): persistent storage + cloud upload

**Open Questions:**
1. Storage format: TIFF, HDF5, NumPy `.npy`, or raw?
2. Metadata schema: what to include with each image?
3. Cloud storage structure and retention policy
4. Local disk retention before cloud upload
5. Compression strategy (if any)

**Related Documentation:**
- [requirements.md](requirements.md) - Section 8
- [architecture.md](architecture.md) - Section 7 Data Management

**Action Items:**
- [ ] Define metadata schema
- [ ] Select storage format
- [ ] Define cloud upload workflow
- [ ] Specify retention policies

---

## Performance & Optimization

### Issue #5: GPU Memory Support in Meadowlark SDK

**Status:** Open - Vendor clarification needed  
**Priority:** Medium  
**Component:** Meadowlark PCIe SDK  
**Date Raised:** March 25, 2026

**Question:**
Does the Meadowlark Python SDK support passing GPU memory pointers (CuPy/CUDA) directly to `Write_image()`, or must images be in CPU memory?

**Impact:**
- **If GPU supported:** Avoid GPU→CPU copy overhead, maintain high throughput
- **If CPU only:** Need to copy images from GPU to CPU before SLM upload

**Test Case:**
```python
import cupy as cp

# Generate image on GPU
image_gpu = cp.zeros((1024, 1024), dtype=cp.uint8)

# Can we pass GPU pointer directly?
Write_image(board=1, image=image_gpu.data.ptr)  # Works?

# Or must copy to CPU first?
image_cpu = image_gpu.get()  # GPU→CPU copy
Write_image(board=1, image=image_cpu.ctypes.data)
```

**Questions for Meadowlark:**
1. Does SDK support CUDA device pointers?
2. Does SDK support CuPy array objects?
3. Performance recommendation for GPU-based workflows?

**Action Items:**
- [ ] Test GPU pointer support
- [ ] Contact Meadowlark if unclear
- [ ] Document recommended pattern in timing reference

---

## Integration & Testing

### Issue #6: Camera Synchronization Strategy - DUAL-SLM SYSTEM

**Status:** ✅ RESOLVED - Dual-SLM architecture designed  
**Priority:** High  
**Date Raised:** March 24, 2026  
**Updated:** March 25, 2026  
**Resolution Date:** March 25, 2026

**Original Question:**
How should camera triggering be implemented relative to SLM timing for synchronized dual-camera capture?

**IMPLEMENTED SOLUTION:**

**System Configuration:**
- **2 SLMs** (Meadowlark 1024x1024 PCIe) displaying independent phase patterns
- **2 Cameras** (Allied Vision Prosilica GT 5120NIR) capturing both SLM images simultaneously
- **1 Frame Grabber** (Coaxlink Quad CXP-12) interfacing both cameras
- **1 kHz Pulse Generator** triggering both SLMs (continuous 1ms period)
- **Programmable Delay Circuit** (0.7ms) delaying SLM2 output pulse
- **BNC T-Connector** splitting delayed trigger to both cameras

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
- SLM1 triggered at t=0ms → settled by t=0.696ms
- SLM2 triggered at t=1ms → settled by t=1.696ms  
- Cameras triggered at t=1.7ms (1ms + 0.7ms delay)
- **At camera trigger time (t=1.7ms):**
  - SLM1 image: settled for 1.7ms ✅ (extra margin beyond 0.696ms)
  - SLM2 image: settled for 0.7ms ✅ (just above 0.696ms minimum)
  - Both cameras capture simultaneously ✅

**Key Advantages:**
1. ✅ No need for precise simultaneous SLM triggering
2. ✅ Simple hardware (1kHz generator widely available)
3. ✅ SLM1 gets extra settling time (safety margin)
4. ✅ SLM2 output pulse provides camera trigger reference
5. ✅ Both cameras guaranteed to capture settled images
6. ✅ Synchronized dual-camera capture

**Operational Flow:**
```python
# One-time setup
slm1.set_wait_for_trigger(True)
slm1.set_output_pulse(False)   # Silent flip
slm2.set_wait_for_trigger(True)
slm2.set_output_pulse(True)    # Generates camera trigger

# Loop
slm1.write_image(img1)          # DMA ~1ms
slm2.write_image(img2)          # DMA ~1ms
slm1.ImageWriteComplete()       # Block until loaded
slm2.ImageWriteComplete()       # Block until loaded
# Both now waiting for 1kHz pulses
# Cameras will be triggered after SLM2 output + 0.7ms delay
img1, img2 = camera_service.capture_dual()
```

**Remaining Action Items:**
- [ ] Determine Allied Vision camera capture/exposure time (for frame rate calculation)
- [ ] Select/specify programmable delay circuit (0.7ms)
- [ ] Verify BNC T-connector signal integrity for dual cameras
- [ ] Test timing with oscilloscope to verify margins
- [ ] Define error handling for missed captures

**Related Documentation:**
- [architecture.md](architecture.md) - Section 3.1 Data Path (updated with dual-SLM)
- [docs/SLM_Timing_Reference.md](docs/SLM_Timing_Reference.md) - needs dual-SLM section
- [requirements.md](requirements.md) - OR-3 Timing and Trigger Coordination

---

## Future Considerations

### Issue #7: Pre-loaded Sequence Mode

**Status:** Open - Future optimization  
**Priority:** Low  
**Component:** Meadowlark SLM (firmware rev 2.4+)  
**Date Raised:** March 25, 2026

**Question:**
Should we use the `PreLoad_sequence()` and `StartAutoIncrement()` functions for faster operation?

**Context:**
The datasheet mentions (Section 4.3.11) that firmware rev 2.4+ supports:
- Pre-loading up to 752 images to hardware memory
- Auto-increment through sequence on external triggers
- No software intervention per frame

**Potential Benefits:**
- Higher sustained frame rates
- Reduced software overhead
- More deterministic timing

**Trade-offs:**
- Limited to 752 images
- Less flexibility (must pre-load entire sequence)
- Firmware version requirement

**Action Items:**
- [ ] Check firmware version of purchased SLM
- [ ] Evaluate if use case benefits from this mode
- [ ] Test if applicable to operational workflows

---

## Document Maintenance

**How to Use This File:**
1. Add new issues as they are discovered
2. Update status when progress is made
3. Close issues by moving to a "Resolved" section (or separate file)
4. Reference issue numbers in code comments where relevant
5. Link to relevant documentation

**Issue Status Values:**
- **Open:** Needs action or clarification
- **Blocked:** Waiting on external input
- **In Progress:** Actively being worked on
- **Resolved:** Closed (move to resolved section)

**Priority Levels:**
- **High:** Blocks critical functionality or design decisions
- **Medium:** Important but has workarounds
- **Low:** Nice to have, future optimization

---

## Resolved Issues

<!-- Move closed issues here with resolution notes -->

---

**Last Updated:** March 25, 2026
