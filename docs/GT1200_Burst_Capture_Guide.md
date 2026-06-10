# Allied Vision GT1200 Prosilica - Burst Capture System Guide

## System Overview
Capture 10 images in rapid burst mode every 1 minute using Allied Vision GT1200 Prosilica camera.

## Hardware Setup

### Connections
1. **Camera**: Allied Vision GT1200 Prosilica (GigE Vision compliant)
2. **Network**: Connect camera to PoE Ethernet switch
3. **PC**: Connect Windows machine to same switch
4. **Power**: Camera powered via PoE (802.3af/at)

### Network Configuration
- Set camera to static IP or use DHCP with reservation
- Ensure jumbo frames enabled (MTU 9000) for best performance
- Dedicated GigE NIC recommended (not shared with internet traffic)
- Windows firewall: Allow GigE Vision protocol (UDP ports)

## Software Requirements

### SDK Installation
1. **Vimba X SDK** (Allied Vision): Download from Allied Vision website
2. **Vimba X Viewer**: For testing and configuration
3. Install GigE network drivers included with Vimba X
4. **Python Package**: `pip install vmbpy`

## Burst Configuration

### Camera Settings (via Vimba X API or Viewer)
```
Acquisition Mode: MultiFrame
Frame Count: 10
Trigger Source: Software (or FixedRate)
Exposure Time: <adjust based on lighting, e.g., 1000-10000 µs>
Acquisition Frame Rate: Maximum (depends on exposure, typically 100+ fps)
Pixel Format: Mono8 or Mono12 (based on requirements)
```

### Key Parameters
- **Buffer allocation**: Allocate 10+ frame buffers in application
- **Bandwidth**: Ensure sufficient (GT1200 @ 1920×1200 ≈ 230 MB/s for Mono8)
- **Storage**: ~2.3 MB per image (Mono8), ~23 MB per burst

## Implementation Approach

### Option 1: Software Trigger (Simple)
```python
# Pseudocode
while True:
    camera.startCapture()
    for i in range(10):
        camera.softwareTrigger()
        image = camera.getFrame()
        saveImage(image, timestamp)
    camera.stopCapture()
    sleep(60 seconds)
```

### Option 2: Scheduled Task (Robust)
- Use Windows Task Scheduler to run capture script every minute
- Script performs burst capture and exits
- More resilient to failures

### Option 3: Free-Running with External Timer
- Set camera to continuous mode
- Use timer/threading to queue 10 captures every 60 seconds
- Best for precise timing requirements

## Code Framework (Python with vmbpy)

```python
from vmbpy import *
import time
from datetime import datetime

def capture_burst():
    with VmbSystem.get_instance() as vmb:
        cameras = vmb.get_all_cameras()
        with cameras[0] as cam:
            
            # Configure burst
            cam.AcquisitionMode.set('MultiFrame')
            cam.AcquisitionFrameCount.set(10)
            cam.TriggerSource.set('Software')
            
            # Start acquisition
            cam.start_streaming(handler=frame_handler, buffer_count=10)
            
            # Trigger 10 frames
            for i in range(10):
                cam.TriggerSoftware.run()
                time.sleep(0.01)  # Small delay between triggers
            
            cam.stop_streaming()

def frame_handler(cam: Camera, stream: Stream, frame: Frame):
    if frame.get_status() == FrameStatus.Complete:
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        img = frame.as_numpy_ndarray()
        # Save image (e.g., using opencv or PIL)
        # cv2.imwrite(f'burst_{timestamp}_{frame.get_id():02d}.png', img)
    cam.queue_frame(frame)

# Main loop
while True:
    capture_burst()
    time.sleep(60)
```

## Key Considerations

### Performance
- **Burst rate**: GT1200 can sustain ~100 fps depending on exposure
- **10 images**: Takes ~100-200 ms at high frame rate
- **Network bandwidth**: Verify switch supports GigE speeds

### Reliability
- Handle camera disconnection errors
- Implement logging for monitoring
- Test thermal stability for continuous operation

### Storage
- ~138 MB/hour (10 images × 60 bursts × 2.3 MB)
- Implement rotation policy or regular cleanup
- Consider on-the-fly compression (e.g., JPEG, PNG)

### Timing Accuracy
- Software timing: ±100ms accuracy
- For precise timing: Use hardware trigger from external timer
- Monitor CPU load to prevent jitter

## Testing Checklist
- [ ] Camera detected by Vimba X Viewer
- [ ] Jumbo frames working (test with large packet size)
- [ ] Burst capture works manually
- [ ] Timing loop maintains 1-minute interval
- [ ] Disk space management implemented
- [ ] Error handling tested (disconnect/reconnect)

## References
- Vimba X SDK Documentation (https://www.alliedvision.com/en/products/vimba-sdk)
- GigE Vision Standard (EMVA)
- Camera datasheets in `/docs/datasheets/ProslilicaGT/`
