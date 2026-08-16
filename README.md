# DroneAR_Game
A drone Augmented Reality (AR) Game that you can play on your fpv drone setup.


The problem that the project needs to overcame.

## Low-Latency Video Pipeline

The first prototype uses the HDZero receiver's HDMI output and a USB capture card to bring the FPV video into the computer:

```text
Drone Camera
     │
     ▼
 HDZero VTX
     │
     │ 5.8 GHz digital link
     ▼
 HDZero VRX
     │
     │ HDMI
     ▼
 USB Capture Card
     │
     │ USB video stream
     ▼
 Computer
     │
     ▼
 OpenCV / Pose Estimation
     │
     ▼
 AR Rendering
```

This approach is simple and useful for the initial proof of concept, but it introduces additional processing stages. The HDZero receiver has already decoded the wireless video and converted it into an HDMI-compatible video signal. The capture card then has to **receive and decode the HDMI signal again**, convert it into a format suitable for USB, transfer it to the computer, and the software must then buffer and process the frames.

The problem is therefore not necessarily the HDMI conversion itself. The concern is the additional **HDMI → capture card → USB → software video pipeline**, where buffering and frame transfers can introduce latency.

For an FPV application, even a few additional frames of delay are significant. At 60 FPS, for example:

* 1 frame ≈ **16.7 ms**
* 2 frames ≈ **33.3 ms**
* 3 frames ≈ **50 ms**

Therefore, the initial pipeline will be measured experimentally rather than assuming a specific latency.

### Final Approach

The long-term goal is to access the video **inside the HDZero receiver, after the RF signal has been decoded but before the HDMI output stage**.

```text
Drone Camera
     │
     ▼
 HDZero VTX
     │
     │ 5.8 GHz digital link
     ▼
 HDZero VRX
     │
     │
     ├──────────────► HDMI ──────► Goggles/Display
     │
     │ decoded digital video
     ▼
 Custom Processing Board
     │
     ├── Image Processing
     ├── ArUco / AprilTag Detection
     ├── Visual Odometry
     ├── IMU Fusion
     └── 6-DoF Pose Estimation
              │
              ▼
             HDMI
              |
              |
              ▼
         Goggles/Display
        AR / Game System
```

Instead of converting the received video to HDMI and then capturing and converting it back into a digital stream, the processing board would access the **decoded video stream directly**.

This should eliminate the unnecessary HDMI/capture-card stage and, more importantly, gives much greater control over buffering and latency.

The first objective is therefore to reverse-engineer the HDZero receiver's video path and determine the format and interface of the decoded video before it reaches the HDMI transmitter. If accessible, this signal could potentially be duplicated or tapped and fed directly into an FPGA/SoC-based processing system.

The project will initially use the conventional HDMI + capture-card pipeline to develop and validate the computer-vision system. Once the tracking and AR system works, the receiver-side video path will be investigated as the low-latency hardware solution.
