# DroneAR_Game

A low-latency **Augmented Reality (AR) game for FPV drones**.

The goal is to create a system where virtual objects — such as rings, checkpoints, gates, and obstacles — are overlaid onto the FPV video and remain fixed in the physical 3D environment so you make a game out of it.

The drone's position and orientation need to be estimated in real time so that the virtual objects can be correctly projected into the FPV camera view.

---

## Project Status

> **This project is currently in the brainstorming, research, and system-design phase. No custom hardware has been built yet.**

The purpose of this project at its current stage is to explore the architecture required to build a low-latency FPV AR system, identify the main technical challenges, investigate possible hardware and software solutions, and evaluate the feasibility of different approaches.

At the moment the project is as much about **solving engineering problems and assessing the difficulties involved** as it is about developing the final game.

The development will start with software prototypes and measurements before moving toward custom hardware.

---

# The Main Problem

An FPV AR system has two major requirements:

1. **Very low video latency**
2. **Accurate real-time 3D pose estimation**

The system needs to determine where the drone is and where it is pointing while processing the FPV video fast enough that the virtual objects do not visibly lag behind the real world.

Even relatively small amounts of latency can be problematic for FPV flight.

---

# Approach 1 — HDMI Capture

The first prototype uses an HDZero digital FPV system. The receiver's HDMI output is captured by a USB capture card and processed by a computer.

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
     ├── OpenCV
     ├── Pose Estimation
     └── AR Rendering
```

This approach is relatively simple and is useful for the initial software proof of concept.

However, the video has already been decoded by the HDZero receiver before it reaches the HDMI output. The system then takes that video and introduces another processing chain:

```text
Decoded Video
      │
      ▼
    HDMI
      │
      ▼
HDMI Capture
      │
      ▼
 USB Transfer
      │
      ▼
Software Buffer
      │
      ▼
Computer Vision
      │
      ▼
   Rendering
```

The problem is therefore not necessarily the HDMI interface itself. The concern is the additional **HDMI → capture card → USB → software** pipeline and the buffering that can occur at each stage.
.

The exact latency of this pipeline needs to be measured experimentally, but if the total latency becomes sufficiently large, this approach will not be suitable for real-time drone flight. 

---

# Approach 2 — Access the Decoded Video Inside the VRX

The long-term goal is to investigate whether the decoded video can be accessed **inside the HDZero receiver, before it is converted to HDMI**.

A simplified view of the receiver is:

```text
RF
│
▼
RF Receiver
│
▼
Demodulation
│
▼
Error Correction / Packet Reconstruction
│
▼
Video Decoding
│
▼
┌─────────────────────────┐
│     DECODED VIDEO       │
│                         │
│  ← Potential access    │
│     point               │
└────────────┬────────────┘
             │
             ▼
        HDMI Output
```

The idea is to investigate whether the decoded digital video stream can be tapped or duplicated at this point and sent directly to a dedicated processing system.

The potential architecture would therefore be:

```text
Drone Camera
     │
     ▼
 HDZero VTX
     │
     │ 5.8 GHz
     ▼
 HDZero VRX
     │
     │
     ├──────────────► HDMI
     │
     │ Decoded Video
     ▼
Custom Processing Hardware
     │
     ├── Image Processing
     ├── ArUco / AprilTag Detection
     ├── Visual Tracking
     ├── IMU Fusion
     └── 6-DoF Pose Estimation
              │
              ▼
        AR / Game System
```

This could remove the unnecessary HDMI and USB capture stages from the computer-vision pipeline.

---

# Hardware Challenge

At this point the project becomes both a **software and hardware engineering problem**.

The receiver needs to be investigated to determine:

* What hardware performs the RF demodulation?
* Where does the video become a decoded pixel stream?
* What video format is used internally?
* Can the internal video stream be accessed without disrupting the normal HDMI output?
* Can an FPGA or SoC process the stream with sufficiently low latency?
* How much buffering is introduced by the receiver and processing hardware?

The goal is not to assume that the internal architecture works a certain way, but to **reverse-engineer and verify the actual video pipeline**.

---

# Software / Computer Vision

Once access to the video is established, the next problem is determining the drone's position and orientation.

The initial software prototype will investigate:

```text
FPV Video
    │
    ▼
Camera Calibration
    │
    ▼
ArUco / AprilTag Detection
    │
    ▼
Camera Pose Estimation
    │
    ▼
Drone 6-DoF Pose
    │
    ▼
3D World
    │
    ▼
Virtual Gates / Rings
```

The first goal is to make a virtual object remain fixed at a known 3D position while the camera moves around it.




Pose Estimation During Temporary Marker Loss

A second major challenge is that the drone cannot rely on continuously detecting an ArUco marker.

During flight, the marker may temporarily disappear because of:

Motion blur
Occlusion
The drone rotating away from the marker
The marker leaving the camera's field of view
Video noise or poor lighting
Rapid drone movement

If the AR system simply stops rendering whenever the marker is not detected, the virtual geometry would suddenly disappear and reappear. This would make the system unusable for a fast-moving FPV drone.

Instead, the system needs to estimate the drone's pose during short periods where visual tracking is unavailable.

For example:

ArUco detected
      │
      ▼
Measure drone pose
      │
      ▼
Estimate velocity
      │
      ▼
ArUco temporarily lost
      │
      ▼
Predict new position
      │
      ▼
Continue rendering AR
      │
      ▼
ArUco detected again
      │
      ▼
Correct accumulated error

A simple first approach would be to estimate the drone's velocity from consecutive position measurements



Once all these works reliably, the system can be extended toward:

* Multiple gates
* Checkpoints
* 3D race courses
* Scoring


## Current Focus

The current focus is **not yet building the final hardware**.

The goal is to answer the fundamental engineering questions first:

> **Can the drone's 3D position be estimated accurately enough?**

> **Can the FPV video be processed with sufficiently low latency?**

> **Can the HDZero receiver's decoded video be accessed before the HDMI output?**

> **Can the complete system be implemented without introducing unacceptable latency?**

Only after these questions are answered will the custom hardware design begin.
