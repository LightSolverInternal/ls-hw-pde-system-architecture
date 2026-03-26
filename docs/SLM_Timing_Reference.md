# SLM Timing Reference - Meadowlark 1024x1024

This document provides timing diagrams and operational details for the Meadowlark 1024x1024 SLM PCIe interface.

Source: PCIe User Manual (1024x1024) + analysis

---

## Quick Reference: Optimal Pattern for 1000 fps

**Two approaches available:** Software Timing (automatic) or External Hardware Trigger

### Option A: Software Timing (No External Hardware Required)

**SIMPLEST - Automatic buffer flipping**

```python
# Setup
SetWaitForTrigger(board=1, waitForTrigger=False)  # Disable external triggers

# Main loop - achieves ~1ms per frame (1000 fps)
for img in image_sequence:
    Write_image(img)         # DMA ~1ms, automatic flip when ready
    success = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
    # Blocks until hardware ready (or timeout)
    if not success:
        print("Timeout or error!")
        break
    
# Memory bank flip happens automatically inside Write_image()
# ImageWriteComplete() likely blocks ~0.696ms (or returns immediately if already ready)
```

### Option B: External Hardware Trigger

**MORE CONTROL - Requires external trigger hardware (Arduino/FPGA/DAQ)**

```python
# Setup
SetWaitForTrigger(board=1, waitForTrigger=True)  # Enable external triggers

# Main loop
for img in image_sequence:
    Write_image(img)         # DMA ~1ms, returns when DMA complete
                             # SLM now waiting for external trigger...
    
    # [External hardware sends TTL falling edge to SLM SMA input]
    
    success = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
    # BLOCKS waiting for:
    # 1. External trigger arrives (TTL falling edge)
    # 2. Memory bank flips
    # 3. Image displays (~0.696ms)
    # Returns 1 on success, 0 on timeout
    
    if not success:
        print("Trigger timeout!")
        break
    
# External hardware must send TTL pulses to control frame rate
# For 1000 fps: external hardware sends triggers at 1ms intervals (1000 Hz)
# Trigger is physical TTL signal from external hardware, not software function
```

**Key difference from software mode:**
- Software mode: `ImageWriteComplete()` waits for display complete (fast, ~0.7ms or immediate)
- External trigger mode: `ImageWriteComplete()` waits for external trigger + display complete
- Blocking call with timeout protection

### Key Insights (Both Options):
- `Write_image()` takes ~1ms (DMA time)
- Previous image display takes 0.696ms (completes during DMA)
- `ImageWriteComplete()` likely **blocks** until ready (or timeout) based on "stop waiting" language in datasheet
- Achieves 1ms per frame with full safety verification
- **DMA time (1ms) > pixel load time (0.696ms)** → pixel loading always complete when DMA finishes

**Note on ImageWriteComplete():** The datasheet says "stop **waiting**" which suggests blocking behavior. However, the 0/1 return value could also support a polling pattern. Code examples show blocking with timeout check (most likely behavior). If unsure, test the SDK or use a `while not done:` polling loop.

---

## Memory Architecture

### Dual-Port Memory Banks

**Purpose:** Enable continuous image streaming without frame drops

```
┌─────────────────────────────────────────┐
│        PCIe Controller Memory           │
│                                         │
│  ┌──────────┐         ┌──────────┐    │
│  │ Buffer A │         │ Buffer B │    │
│  │ 1024x1024│         │ 1024x1024│    │
│  └────┬─────┘         └────┬─────┘    │
│       │                    │           │
│       └─────── Swap ───────┘           │
│                 ↓                      │
│         To SLM Display                 │
└─────────────────────────────────────────┘

Operation:
- One buffer displays to SLM
- Other buffer receives DMA
- Buffers swap roles after each flip
```

---

## DMA Flow (PC/GPU → SLM)

### Complete Path:

```
1. Application Memory (PC RAM or GPU memory)
              ↓
   Write_image() initiates DMA
              ↓
2. PCIe Controller Buffer (A or B)
              ↓
   Memory bank flip (buffer swap)
              ↓
3. Active buffer → SLM pixels (0.696ms load time)
```

**Key Point:** DMA goes to PCIe controller memory, NOT directly to SLM pixels.

---

## API Calls and Timing

### Basic Sequence (Software Timing, No External Trigger)

#### Conservative Pattern (with ImageWriteComplete):

```
┌─────────────────────────────────────────────────────────────┐
│                        Timeline                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ t=0ms      Write_image(image1)                             │
│            ↓ DMA starts (PC/GPU → PCIe buffer)             │
│            ↓                                                │
│ t=~1ms     Write_image() RETURNS                           │
│            DMA complete                                     │
│            Memory bank FLIP (automatic)                     │
│            Output pulse (200µs LOW)                         │
│            Image loading to SLM pixels starts               │
│            ↓                                                │
│ t=~1ms     ImageWriteComplete() called                     │
│            ↓ Waits for SLM ready...                        │
│            ↓                                                │
│ t=~1.7ms   ImageWriteComplete() RETURNS TRUE               │
│            Hardware ready for next image                    │
│            ↓                                                │
│ t=~1.7ms   Write_image(image2)                             │
│            ↓ DMA to other buffer                           │
│            ...cycle repeats                                 │
└─────────────────────────────────────────────────────────────┘

Frame rate with ImageWriteComplete(): ~590 fps (1.7ms period)
```

#### Maximum Throughput Pattern (parallel operation):

```
┌─────────────────────────────────────────────────────────────┐
│         Parallel DMA + Pixel Loading Timeline               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ t=0ms      Write_image(img1) - DMA to Buffer A             │
│            ↓                                                │
│ t=1ms      Write_image(img1) RETURNS                       │
│            Flip: Buffer A displays, Buffer B writable       │
│            ┌─────────────────┬─────────────────┐           │
│            │   Buffer A      │    Buffer B     │           │
│            │  displaying     │    writable     │           │
│            │  (0.696ms)      │                 │           │
│            └─────────────────┴─────────────────┘           │
│            ↓                                                │
│ t=1ms      Write_image(img2) - DMA to Buffer B             │
│            ↓ (parallel to Buffer A displaying!)            │
│            ↓                                                │
│ t=1.696ms  Buffer A done displaying                        │
│            (Buffer B DMA still in progress until t=2ms)     │
│            ↓                                                │
│ t=2ms      Write_image(img2) RETURNS                       │
│            Flip: Buffer B displays, Buffer A writable       │
│            ┌─────────────────┬─────────────────┐           │
│            │   Buffer B      │    Buffer A     │           │
│            │  displaying     │    writable     │           │
│            │  (0.696ms)      │                 │           │
│            └─────────────────┴─────────────────┘           │
│            ↓                                                │
│ t=2ms      Write_image(img3) - DMA to Buffer A             │
│            ↓ (parallel to Buffer B displaying!)            │
│            ...cycle repeats every 1ms                       │
└─────────────────────────────────────────────────────────────┘

Frame rate without ImageWriteComplete(): ~1000 fps (1ms period)

Key insight: DMA (1ms) > pixel load (0.696ms)
→ Pixel loading completes BEFORE next DMA finishes
→ Bottleneck is DMA time, not pixel settling time
```

---

## Trigger Modes

The SLM supports two trigger modes, configured via API:

### Mode 1: Software Timing (Automatic Flipping)

**Setup:**
```python
SetWaitForTrigger(board=1, waitForTrigger=False)
```

**Behavior:**
- Memory bank flip happens **automatically** inside Write_image()
- No external hardware required
- Simplest operation mode

**Optimal Pattern (1000 fps):**
```python
SetWaitForTrigger(board=1, waitForTrigger=False)

for img in image_sequence:
    Write_image(img)         # DMA ~1ms, automatic flip
    ImageWriteComplete()     # Returns immediately (previous done)
    
# Achieves ~1ms per frame = 1000 fps
```

### Mode 2: External Hardware Trigger

**Setup:**
```python
SetWaitForTrigger(board=1, waitForTrigger=True)
```

**Behavior:**
- SLM waits for **external TTL pulse** on SMA input connector
- `Write_image()` completes DMA, but buffer flip DOES NOT happen yet
- `ImageWriteComplete()` **BLOCKS** waiting for:
  1. External trigger to arrive (falling edge)
  2. Memory bank flip
  3. Image display to complete (~0.696ms)
- Trigger must be **falling edge** (high → low transition)
- Trigger comes from external hardware (Arduino, FPGA, DAQ, function generator)
- More control over exact timing

**Physical Connection:**
```
External Trigger Source (Arduino/FPGA/DAQ)
         |
         | TTL Signal (falling edge)
         |
         ↓
    SLM SMA Input Connector ("Input A")
```

**Trigger Signal Requirements:**
- **Type:** TTL (0-5V or 0-3.3V)
- **Edge:** Falling edge (high → low)
- **Connector:** SMA input at bottom of PCIe controller

**Example External Hardware:**
- Arduino/ESP32 GPIO
- NI DAQ digital output
- Function generator (TTL mode)
- FPGA timing controller
- Pulse generator

**Flow with External Trigger:**
```python
SetWaitForTrigger(board=1, waitForTrigger=True)

for img in image_sequence:
    Write_image(img)         # DMA completes (~1ms), SLM waits for trigger
                             # Buffer flip does NOT happen yet!
    
    # [External hardware sends TTL falling edge to SLM SMA input]
    
    success = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
    # BLOCKS waiting for: trigger arrival → flip → display (~0.696ms)
    # Returns 1 when ready, 0 on timeout
    
    if not success:
        print("Trigger timeout!")
        break
    
# For 1000 fps: external hardware must send triggers at 1ms intervals
# ImageWriteComplete() blocks until trigger arrives and display completes
```

---

## External Trigger Timing Diagram

### With External Hardware Trigger

```
External Trigger IN (falling edge from external source):
         ↓
         |
    ┌────┴─────────────────────────────────────────────────┐
    │                                                       │
    │  Up to 0.696ms delay                                 │
    │        ↓                                              │
    │  Output Pulse (SLM acknowledgment):                  │
    │      ___           ___                                │
    │  ___| LOW |_______| (normally HIGH, 200µs pulse)     │
    │        ↓                                              │
    │  Memory bank flip                                     │
    │        ↓                                              │
    │  Image loads to SLM pixels                           │
    │        ↓                                              │
    │  ~0.696ms later                                       │
    │        ↓                                              │
    │  ImageWriteComplete() returns TRUE                   │
    │                                                       │
    └───────────────────────────────────────────────────────┘
```

### External Trigger Sequence (Basic):

```python
SetWaitForTrigger(board=1, waitForTrigger=True)

# Prepare image
Write_image(image1)          # DMA to buffer, waits for trigger

# [External hardware sends TTL falling edge pulse to SLM]

# SLM responds:
# - Output pulse generated (up to 0.696ms after trigger)
# - Memory bank flips
# - Image loads to SLM

# Wait until SLM ready for next image
ImageWriteComplete()         # Blocks until ready (~0.696ms)

# Now safe to repeat for next image
```

**Note:** There is NO `send_trigger()` function in the API. The trigger is a physical TTL signal from external hardware.

---

## Output Pulse Details

**Signal Characteristics:**
- **Type:** TTL
- **Normal State:** HIGH
- **Pulse:** LOW for 200 microseconds (µs)
- **Connector:** SMA on PCIe controller

**Generated When:**
- Software timing: when memory bank flip occurs
- External trigger: acknowledges trigger receipt (up to 0.696ms after)

**Use Cases:**
1. Trigger camera (SLM → Camera sync)
2. Synchronization with external equipment
3. Debug/monitoring on oscilloscope

**Enable in code:**
```python
SetOutputPulse(board=1, outputPulse=True)
```

---

## Continuous Streaming Example

### Double-Buffering in Action:

#### Conservative Approach (Safe, 590 fps):
```
Image 1:
  Write_image(img1)      → DMA to Buffer A
  [flip]                 → Buffer A displays, Buffer B writable
  ImageWriteComplete()   → Wait for SLM ready (0.696ms)

Image 2:
  Write_image(img2)      → DMA to Buffer B (while A displays)
  [flip]                 → Buffer B displays, Buffer A writable
  ImageWriteComplete()   → Wait for SLM ready (0.696ms)

Image 3:
  Write_image(img3)      → DMA to Buffer A (while B displays)
  [flip]                 → Buffer A displays, Buffer B writable
  ImageWriteComplete()   → Wait for SLM ready (0.696ms)

...continues alternating every 1.7ms
```

#### Maximum Throughput Approach (1000 fps):
```
Image 1:
  Write_image(img1)      → DMA to Buffer A (~1ms)
  [flip]                 → Buffer A displays (0.696ms), Buffer B writable
  
Image 2:
  Write_image(img2)      → DMA to Buffer B (~1ms)
                         → Overlaps with Buffer A displaying!
  [flip]                 → Buffer B displays (0.696ms), Buffer A writable
  
Image 3:
  Write_image(img3)      → DMA to Buffer A (~1ms)  
                         → Overlaps with Buffer B displaying!
  [flip]                 → Buffer A displays, Buffer B writable

...continues alternating every 1ms
```

**Benefit:** While one buffer displays on SLM (0.696ms), you're doing DMA to the other buffer (1ms). Since DMA takes longer, pixel loading is always complete when you need to flip.

---

## Critical Timing Values (1024x1024)

| Parameter | Value | Notes |
|-----------|-------|-------|
| DMA time | ~1ms | PC/GPU memory → PCIe buffer |
| Memory bank flip | Instant | Buffer pointer swap |
| SLM pixel load time | 0.696ms | Buffer → SLM pixels (overlaps with next DMA) |
| Output pulse width | 200µs | LOW pulse duration |
| Trigger acknowledge delay | Up to 0.696ms | External trigger → output pulse |
| Min frame interval (safe) | ~1.7ms | With ImageWriteComplete() |
| Min frame interval (max) | ~1ms | Without ImageWriteComplete() (parallel) |
| Max frame rate (safe) | ~590 fps | Conservative (1.7ms period) |
| Max frame rate (theoretical) | ~1000 fps | DMA-limited (1ms period, parallel operation) |

---

## Common Mistakes to Avoid

### ❌ Calling Write_image() Without Understanding Timing
```python
# Bad: No waiting at all, no trigger mode set
Write_image(img1)
Write_image(img2)  # Might work, might not - depends on timing
```

### ⚠️ Overly Conservative (works but slow)
```python
SetWaitForTrigger(board=1, waitForTrigger=False)

Write_image(img1)
ImageWriteComplete()  # Blocks unnecessarily if called too early
Write_image(img2)
ImageWriteComplete()  # Blocks again
# Works but only ~590 fps
```

### ❌ Using `send_trigger()` - This Function Doesn't Exist!
```python
# THIS IS WRONG - send_trigger() is not in the Meadowlark API
Write_image(img)
send_trigger()  # ERROR: No such function!
```

**Correct approaches:**
- **Software timing:** Flip happens automatically (Mode A)
- **External trigger:** Use physical hardware to send TTL signal (Mode B)

### ✅ OPTIMAL Pattern A: Software Timing (1000 fps + safe)
```python
SetWaitForTrigger(board=1, waitForTrigger=False)

for img in images:
    Write_image(img)
    success = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
    if not success:
        break
# Achieves 1ms per frame with full safety, automatic flipping
```

### ✅ OPTIMAL Pattern B: External Hardware Trigger (1000 fps + safe)
```python
SetWaitForTrigger(board=1, waitForTrigger=True)

for img in images:
    Write_image(img)
    # [External hardware sends trigger via TTL to SMA connector]
    
    success = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
    # Blocks waiting for trigger + display complete
    if not success:
        break  # Timeout
    
# Frame rate controlled by external trigger timing
# External timing control with full safety
```

### ⚡ Maximum Throughput (no safety checks, advanced)
```python
SetWaitForTrigger(board=1, waitForTrigger=False)

# If you understand the timing and DMA > pixel load time:
Write_image(img1)     # Returns after ~1ms DMA
Write_image(img2)     # Safe because pixel load (0.696ms) done
Write_image(img3)     # Each call ~1ms apart = 1000 fps

# Risk: If DMA occasionally takes >1.696ms, you might drop frames
# No ImageWriteComplete() checking
```

### ❌ Sending Trigger Before DMA Complete
```python
Write_image(img1)
send_trigger()  # ERROR: DMA might not be complete yet
```

### ✅ Correct Pattern
```python
Write_image(img1)       # Returns when DMA complete
send_trigger()          # Now safe to trigger
ImageWriteComplete()    # Wait for SLM ready
```

---

## Memory Source: PC vs GPU

### GPU Memory (Recommended)
```python
import cupy as cp

# Generate on GPU
image_gpu = generate_phase_pattern_cuda()  

# Pass to SLM (if SDK supports GPU pointers)
Write_image(board=1, image=image_gpu.data.ptr)
```

**Advantages:**
- No CPU-GPU transfer overhead
- Both GPU and SLM on PCIe bus
- Sustain high frame rates

### PC System Memory
```python
import numpy as np

# Generate on CPU
image_cpu = np.zeros((1024, 1024), dtype=np.uint8)

# Pass to SLM
Write_image(board=1, image=image_cpu.ctypes.data)
```

**Use when:**
- Simple/low-rate operation
- Images from disk
- Debugging

**Note:** Check Meadowlark SDK documentation for GPU memory support.

---

## Trigger Mode Summary

### Trigger Input Modes (SetWaitForTrigger)

**Mode 1: Software Timing (Automatic)**
```python
SetWaitForTrigger(board=1, waitForTrigger=False)
```
- SLM controls its own timing
- Memory flip automatic after DMA
- No external hardware required
- Simplest operation mode

**Mode 2: External Trigger Input**
```python
SetWaitForTrigger(board=1, waitForTrigger=True)
```
- External device sends TTL trigger to SLM SMA input
- SLM waits for trigger before flipping buffers
- SLM generates output pulse to acknowledge
- Frame rate controlled by external trigger source
- Use when: Need precise timing control from external source

### Output Pulse Usage (Both Modes)

**Mode 3: SLM Output Pulse → Camera/External Device**
```python
SetOutputPulse(board=1, outputPulse=True)
```
- SLM output pulse can trigger external devices (e.g., camera)
- Works with both software timing and external trigger modes
- Pulse generated when memory bank flips
- Ensures camera captures after image settles
- Most common for SLM → Camera synchronization

**Typical Configurations:**

| Configuration | Input Trigger | Output Pulse | Use Case |
|---------------|---------------|--------------|----------|
| Standalone | Software (auto) | Disabled | Simple operation, no sync needed |
| SLM → Camera | Software (auto) | Enabled | SLM controls timing, triggers camera |
| External → SLM | External hardware | Optional | External controller sets timing |
| Full Sync | External hardware | Enabled | External controller, SLM triggers camera |

---

## Understanding the Parallel Operation

### Why 1000 fps is Theoretically Possible

The **dual-buffer architecture** enables parallel operation:

```
While Buffer A displays (0.696ms):
  → Buffer B receives DMA (1ms)
  
Since DMA time (1ms) > pixel load time (0.696ms):
  → Pixel loading always finishes before next DMA completes
  → Other buffer is always ready when Write_image() returns
  → Bottleneck is DMA throughput, not pixel settling
```

**Mathematical proof:**
- Start Write_image() at t=0
- DMA completes at t=1ms, flip happens
- Buffer starts displaying (0.696ms)
- At t=1ms, start next Write_image() to other buffer
- Previous buffer finishes displaying at t=1.696ms
- Next DMA completes at t=2ms (before next flip needed)
- ✓ No conflict, cycle repeats every 1ms

**When this breaks down:**
- If DMA takes >1.696ms occasionally → frame drop
- If you call ImageWriteComplete() → adds 0.696ms wait
- If image generation is slow → reduces effective rate

### Practical Recommendations

### ❌ Conservative Pattern (590 fps):
```python
# Safe but slower - waits even when not needed
SetWaitForTrigger(board=1, waitForTrigger=False)

while True:
    Write_image(next_image)
    ImageWriteComplete()  # Blocks 0.696ms even if already done (if called too early)
    # ~1.7ms per iteration = ~590 fps
```

### ⚠️ Risky Pattern (1000 fps, unsafe):
```python
# Fast but might drop frames if timing varies
SetWaitForTrigger(board=1, waitForTrigger=False)

while True:
    Write_image(next_image)  # Assumes timing, no safety check
    # ~1ms per iteration = ~1000 fps
    # Risk: no verification previous image done
```

### ✅ OPTIMAL PATTERN A: Software Timing (1000 fps, safe, simple):
```python
# Best for: Simple setup, no external hardware needed
# Automatic buffer flipping

SetWaitForTrigger(board=1, waitForTrigger=False)

for img in image_sequence:
    Write_image(img)           # DMA ~1ms, automatic flip
    success = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
    # Blocks until ready (likely returns quickly/immediately if timing right)
    if not success:
        break  # Timeout
    
# Result: ~1ms per iteration = ~1000 fps with safety checks
# No external hardware required
```

### ✅ OPTIMAL PATTERN B: External Trigger (1000 fps, safe, more control):
```python
# Best for: Precise timing control, synchronization with other equipment
# Requires external trigger hardware (Arduino/FPGA/DAQ)

SetWaitForTrigger(board=1, waitForTrigger=True)

# Main loop
for img in image_sequence:
    Write_image(img)           # DMA ~1ms
    
    # [External hardware sends TTL falling edge]
    
    success = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
    # Blocks until trigger + display complete
    if not success:
        break  # Timeout
    
# Result: Frame rate controlled by external trigger timing
# External hardware controls exact timing
```

**Why both optimal patterns work:**
- Pattern A (Software): `ImageWriteComplete()` returns immediately if timing is right (DMA overlap)
- Pattern B (External): Poll `ImageWriteComplete()` until trigger arrives and display completes
- Both achieve high frame rates with safety checking
- Pattern A: Simpler (automatic)  
- Pattern B: More control (external timing)

**Timing breakdown (both patterns):**
```
t=0ms:    Write_image(img2) starts DMA
          [Previous img1 still displaying from t=-1ms]
          
t=1ms:    Write_image(img2) returns (DMA done)
          [img1 finished displaying at t=0.3ms - already done!]
          
t=1ms:    ImageWriteComplete() checks → returns immediately (TRUE)

Pattern A: Automatic flip happens, img2 starts displaying
Pattern B: [External trigger arrives, flip happens, img2 starts displaying]

t=1ms:    Write_image(img3) starts DMA (parallel to img2 display)

Cycle repeats every 1ms = 1000 fps ✓
```

---

## Advanced: Flip Immediate Mode (1024x1024 only)

**Normal Mode:**
```
Trigger → up to 0.696ms → Output pulse → Image loads
```

**Flip Immediate Mode:**
```
Trigger → Output pulse immediately → Image loads
```

**Use case:** Minimize latency between trigger and response

**Trade-off:** May interrupt ongoing pixel refresh

---

## Complete API Reference

### Key Functions from Meadowlark SDK

**SetWaitForTrigger()**
```python
int SetWaitForTrigger(int board, bool waitForTrigger)
```
- **board:** SLM board number (typically 1)
- **waitForTrigger:** 
  - `False` = Software timing (automatic flip)
  - `True` = Wait for external hardware trigger
- **Returns:** 1 for success, 0 for failure
- **When enabled:** External TTL trigger (falling edge) must be supplied to SMA input

**Write_image()**
```python
int Write_image(int board, unsigned char* image, unsigned int trigger_timeout_ms)
```
- **board:** SLM board number (typically 1)
- **image:** Pointer to 1D array of pixel data (1024×1024 = 1,048,576 bytes)
- **trigger_timeout_ms:** Timeout for trigger (if external triggers enabled)
- **Returns:** When DMA complete (NOT when hardware ready for next image)
- **Image data:** Each pixel is 1 byte (0-255 gray levels, applies LUT)

**ImageWriteComplete()**
```python
int ImageWriteComplete(int board, unsigned int trigger_timeout_ms)
```
- **board:** SLM board number (typically 1)
- **trigger_timeout_ms:** Timeout in milliseconds (e.g., 5000 for 5 seconds)
- **Returns:** 
  - **1** = Success - Hardware ready for next Write_image()
  - **0** = Timeout or failure
- **Behavior:** Likely **BLOCKS (waits)** until ready or timeout
  - Software timing mode: Waits until previous display complete (~0.696ms)
  - External trigger mode: Waits for trigger arrival + display complete
  - If timeout: stops waiting and returns 0
- **Usage (most likely):**
```python
# Blocking call - waits until ready or timeout
success = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
if not success:
    print("Timeout or error!")
```
- **Alternative usage (if non-blocking):**
```python
# Polling pattern (if function is non-blocking)
done = False
while not done:
    done = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
```
- **Note:** Datasheet says "stop **waiting**" suggesting blocking behavior, but returns 0/1 like a check function. Actual behavior may need testing to confirm.

**SetOutputPulse()**
```python
int SetOutputPulse(int board, bool outputPulse)
```
- **board:** SLM board number (typically 1)
- **outputPulse:** Enable (True) or disable (False) output pulse
- **Pulse:** 200µs LOW pulse on SMA output when memory bank flips
- **Use:** Trigger camera or other equipment synchronized to SLM

**SetFlipImmediate()** (1024x1024 only)
```python
int SetFlipImmediate(int board, bool flipImmediate)
```
- **board:** SLM board number (typically 1)
- **flipImmediate:** Enable (True) or disable (False)
- **Function:** Interrupt image refresh for faster trigger response
- **Caution:** May affect DC balancing, use carefully

### No `send_trigger()` Function

**Important:** There is NO software function to send triggers. Triggers come from:
- **Software timing mode:** Automatic (internal to Write_image())
- **External trigger mode:** Physical TTL signal from external hardware to SMA connector

---

## Dual-SLM Synchronized Operation

**Use Case:** System with 2 SLMs displaying independent patterns, 2 cameras capturing both simultaneously.

### Architecture Overview

**Components:**
- **2 SLMs** (Meadowlark 1024x1024 PCIe) - independent phase patterns
- **2 Cameras** - synchronized capture of both SLM images
- **1 kHz Pulse Generator** - triggers both SLMs (continuous pulses)
- **Programmable Delay Circuit** (0.7ms) - delays SLM2 output
- **BNC T-Connector** - splits delayed trigger to both cameras

**Configuration Strategy:**
- **SLM1:** External trigger mode, output pulse **DISABLED**
- **SLM2:** External trigger mode, output pulse **ENABLED**
- Both SLMs receive same 1kHz pulse train
- Only SLM2 generates output pulse (for cameras)

### Timing Diagram - Worst Case

```
Time    SLM1              SLM2              Cameras
────────────────────────────────────────────────────────
t=0ms   Pulse A arrives   (waiting)         (waiting)
        ├─ Bank flip
        ├─ Settling...
        
t=0.696ms Image settled   (waiting)         (waiting)

t=1ms   (still settled)   Pulse B arrives   (waiting)
                          ├─ Bank flip
                          ├─ Settling...
                          └─ Output pulse
                                 ↓
                            Delay Circuit (0.7ms)
                          
t=1.696ms                 Image settled      (waiting)

t=1.7ms Both images settled ← ← ← ← Trigger arrives!
        (1.7ms margin)    (0.7ms margin)    ├─ Cam1 capture
        ✅ SAFE           ✅ SAFE            └─ Cam2 capture

t=2ms   Pulse C arrives   (waiting)         (capturing...)
        ├─ New flip...
```

**Key Insight:** SLM1 gets triggered up to 1ms earlier than SLM2, providing extra settling margin.

### Implementation Code

```python
# ========================================
# ONE-TIME SETUP
# ========================================

# SLM1 setup
slm1.load_lut()
slm1.set_wait_for_trigger(board=1, wait=True)   # External trigger mode
slm1.set_output_pulse(board=1, enabled=False)   # NO output pulse

# SLM2 setup
slm2.load_lut()
slm2.set_wait_for_trigger(board=2, wait=True)   # External trigger mode
slm2.set_output_pulse(board=2, enabled=True)    # Generates pulse for cameras

# Hardware connections:
# - 1kHz generator → SLM1 trigger input (SMA)
# - 1kHz generator → SLM2 trigger input (SMA) 
# - SLM2 output → 0.7ms delay circuit → BNC T → Camera1 + Camera2

# ========================================
# MAIN OPERATIONAL LOOP
# ========================================

for iteration in range(num_frames):
    # Generate patterns (e.g., on GPU)
    pattern1 = generate_phase_pattern_slm1()  # 1024x1024 array
    pattern2 = generate_phase_pattern_slm2()  # 1024x1024 array
    
    # Upload to SLMs (DMA: ~1ms each)
    Write_image(board=1, image=pattern1)
    Write_image(board=2, image=pattern2)
    
    # Block until both DMAs complete and images loaded
    success1 = ImageWriteComplete(board=1, trigger_timeout_ms=5000)
    success2 = ImageWriteComplete(board=2, trigger_timeout_ms=5000)
    
    if not (success1 and success2):
        print(f"SLM timeout! SLM1: {success1}, SLM2: {success2}")
        break
    
    # At this point:
    # - Both SLMs have new images loaded into inactive buffer
    # - Both waiting for next 1kHz pulse(s)
    # - Worst case: SLM1 triggered on next pulse (t=0), SLM2 on pulse after (t=1ms)
    # - Cameras triggered at t=1.7ms (both SLMs settled)
    
    # Capture from both cameras (blocking call)
    img1, img2 = camera_service.capture_dual(timeout_ms=100)
    
    # Process feedback (e.g., on GPU if using CUDA Direct Transfer)
    feedback = process_dual_images(img1, img2)
    
    # Use feedback to compute next iteration patterns
    # ...
```

### Timing Analysis

**DMA Upload Time:**
- `Write_image()` returns when DMA complete (~1ms per SLM)
- Can execute sequentially or in parallel (if SDK supports threading)

**Worst-Case Trigger Alignment:**
- SLM1 receives pulse at t=0ms
- SLM2 receives next pulse at t=1ms
- Maximum skew: 1ms (one pulse period)

**Settling Time Margins:**
- **SLM1:** Minimum 1.7ms settling (1ms skew + 0.7ms delay)
- **SLM2:** Minimum 0.7ms settling (delay circuit duration)
- **Required:** 0.696ms for 1024x1024 SLM
- **Margin:** SLM1 = 1.004ms extra, SLM2 = 0.004ms extra ✅

**Camera Trigger Timing:**
- SLM2 output pulse: 200µs width
- Delay circuit: 0.7ms
- Total delay from SLM2 flip: ~0.7ms
- Both cameras triggered simultaneously via BNC T-connector

### Best-Case vs Worst-Case

**Best Case:**
Both SLMs triggered by same 1kHz pulse → settle in parallel → cameras triggered 0.7ms later
```
t=0ms:   SLM1 + SLM2 both receive pulse (synchronized)
t=0.7ms: Cameras triggered (both SLMs settled)
```

**Worst Case:**
SLMs triggered by consecutive pulses → SLM1 gets extra settling time
```
t=0ms:   SLM1 receives pulse
t=1ms:   SLM2 receives pulse + generates output
t=1.7ms: Cameras triggered
```

Both cases are **SAFE** - worst case actually provides more margin for SLM1!

### Error Handling

```python
# Timeout detection
if not success1:
    log_error("SLM1 DMA timeout or trigger timeout")
    # Check hardware: 1kHz generator running? Trigger cable connected?
    
if not success2:
    log_error("SLM2 DMA timeout or trigger timeout")
    # Check hardware: trigger cable, output pulse enabled?

# Camera capture timeout
try:
    img1, img2 = camera_service.capture_dual(timeout_ms=100)
except TimeoutError:
    log_error("Camera capture timeout")
    # Check: delay circuit working? BNC T-connector? Camera triggers enabled?
```

### Design Advantages

1. ✅ **No precise synchronization required** - 1kHz generator doesn't need to trigger both SLMs simultaneously
2. ✅ **Simple hardware** - standard 1kHz pulse generator (function generator, Arduino, etc.)
3. ✅ **Extra settling margin** - SLM1 always has ≥1.7ms (worst case)
4. ✅ **Guaranteed safe capture** - worst-case timing still exceeds 0.696ms requirement
5. ✅ **Single trigger reference** - SLM2 output provides synchronized camera trigger
6. ✅ **Testable** - can verify timing with oscilloscope (probe SLM outputs + camera triggers)

### Hardware Requirements

| Component | Specification | Notes |
|-----------|---------------|-------|
| Pulse Generator | 1kHz TTL, dual outputs | Must drive both SLM trigger inputs |
| Delay Circuit | 0.7ms programmable delay | Input: SLM2 output (200µs), Output: TTL to cameras |
| BNC T-Connector | 50Ω matched | Splits delayed signal to both cameras |
| Cables | SMA (SLMs), BNC (cameras) | Proper impedance matching |

**Delay Circuit Options:**
- Arduino/ESP32 with microsecond timer
- Programmable delay generator (e.g., SRS DG645)
- Precision 555 timer circuit
- FPGA timing controller

### Validation Testing

**Oscilloscope Verification:**
1. **Channel 1:** 1kHz pulse generator output
2. **Channel 2:** SLM1 trigger input
3. **Channel 3:** SLM2 output pulse
4. **Channel 4:** Camera trigger (after delay)

**Expected Timing:**
- Ch2 (SLM1) pulses at 1kHz (1ms period)
- Ch3 (SLM2) pulses 200µs wide, aligned with Ch2 or 1ms offset
- Ch4 (Camera) pulses 0.7ms after Ch3 rising edge

**Frame Rate Calculation:**
```
Cycle time = max(DMA1, DMA2) + worst_case_trigger_delay + camera_capture
           = 1ms + 1.7ms + camera_capture_time
           
If camera_capture = 0.3ms:
Cycle time = 3ms → ~333 fps

If camera_capture = 0.1ms:
Cycle time = 2.8ms → ~357 fps
```

Frame rate limited by **camera capture time**, not SLM settling.

---

## References

- Meadowlark PCIe User Manual (1024x1024), Section 3.6 (Triggering)
- Meadowlark PCIe User Manual (1024x1024), Section 4.3.6-4.3.7 (API)
- Figure 11: Timing diagram with external triggers

---

## Document Revision

**Created:** March 24, 2026  
**Last Updated:** March 25, 2026  
**Author:** System documentation  
**Status:** Reference document for POC system design

**Revision History:**
- **March 24, 2026:** Initial creation with timing analysis
- **March 25, 2026:** Added both software timing and external trigger modes, clarified no `send_trigger()` function exists, added complete API reference
- **March 25, 2026:** Updated `ImageWriteComplete()` behavior - datasheet says "stop **waiting**" implying blocking behavior, updated all code examples to use blocking pattern with timeout check as primary approach
- **March 25, 2026:** Added comprehensive "Dual-SLM Synchronized Operation" section covering 2-SLM + 2-camera architecture with 1kHz pulse generator triggering, timing analysis, implementation code, and validation procedures

**Important Note on ImageWriteComplete():**
The datasheet language is somewhat ambiguous:
- Says "stop **waiting** for a trigger" → suggests **blocking** until ready or timeout
- Returns 0 or 1 → could suggest a **check/poll** function
- Most likely: **BLOCKS** until ready (returns 1) or timeout (returns 0)
- If unsure: test the actual SDK behavior, or use a polling loop pattern for safety
