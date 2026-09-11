# Autonomous Cleaning Robot

Low-cost university project for a **phone-free autonomous cleaning robot** using ROS 2, a Raspberry Pi Pico, wheel encoders, MPU6050, ultrasonic/IR sensors and a static room map created once with an iPhone LiDAR.

> **Target stack:** Ubuntu 24.04 + ROS 2 Jazzy + Gazebo Harmonic + Nav2. The iPhone is used only during room mapping and is removed during normal cleaning.

---

## Core Idea

We will use two operating modes.

### 1. One-Time Mapping Mode

```text
Handheld iPhone LiDAR / ARKit
          |
          v
3D room scan / RGB-D / pose
          |
          v
Laptop + ROS 2 / RTAB-Map or point-cloud processing
          |
          v
2D occupancy map: room.yaml + room.pgm
```

### 2. Normal Cleaning Mode — No iPhone / No LiDAR

```text
Static room map
      |
      v
Laptop: ROS 2 + Nav2 + coverage planner
      |
      +---- wheel encoders + MPU6050 -> odometry / EKF
      +---- ultrasonic -> local obstacle avoidance
      +---- IR array -> floor safety / known home marker
      |
      v
USB serial
      |
Raspberry Pi Pico
      |
      +---- wheel PID -> drive motors
      +---- vacuum / brush control
```

### Is this viable?

**Yes, for a small controlled university-demo environment.** A Nav2 map can be created separately and later loaded from disk. The difficulty is not the static map — it is **localization after the LiDAR is removed**.

Without a runtime LiDAR/camera, AMCL cannot give us continuous global correction. Our baseline therefore uses:

- accurate wheel encoders;
- MPU6050 gyro + encoder fusion;
- a fixed, physically marked **home/start pose**;
- odometry reset/alignment at the beginning of every run;
- ultrasonic sensors for collision avoidance, not primary localization;
- the IR array as a cheap home/floor landmark if calibration allows;
- short cleaning zones and periodic re-zeroing if drift becomes significant.

This is realistic for a university prototype, but it is **not commercial robot-vacuum-grade localization**. If dead-reckoning drift is too large, the documented fallback is a cheap camera + fiducial markers rather than buying an expensive LiDAR.

---

## Hardware We Already Have

| Part | Status | Use |
|---|---|---|
| 2x DC geared motors, no encoders | ✅ Have | Differential drive |
| Ultrasonic sensor(s) | ✅ Have | Close obstacle detection |
| MPU6050 | ✅ Have | Gyroscope / IMU |
| 5-channel IR array | ✅ Have | Floor/cliff experiments and home marker |
| Battery cells | ✅ Have | Main power |
| iPhone with LiDAR | ✅ Have | **One-time handheld room mapping only** |
| Laptop / PC | ✅ Have | ROS 2, Nav2, mapping tools and planning |

## Hardware We Need to Buy

Approximate low-cost Pakistan budget ranges, checked September 2026.

| Required part | Est. PKR | Purpose |
|---|---:|---|
| Raspberry Pi Pico RP2040 | **1,200–1,500** | Low-level controller |
| 2 compatible wheel encoders | **600–800** | Wheel feedback + odometry |
| TB6612FNG dual motor driver | **550–1,000** | Drive both motors* |
| 3S BMS | **300–450** | Battery protection** |
| 12.6 V 3S charger | **500–650** | Battery charging** |
| LM2596 buck converter | **250–350** | Regulated logic power |
| Vacuum/blower donor or motor | **1,000–3,000** | Suction system |
| Logic-level MOSFET/load driver | **300–600** | Vacuum/brush switching |
| Chassis + caster + mounts | **1,000–1,500** | Mechanical base |
| Brush + small DC motor | **500–800** | Cleaning brush |
| Fuse, switch, wire, connectors, level shifting, hardware | **700–1,200** | Safe assembly |

**Expected additional budget: roughly PKR 7,000–12,000**, depending mainly on the suction hardware and chassis.

\*Check the **stall current** of our motors before buying the driver. TB6612FNG is suitable only if the motors are within its current capability.

\**Only if our existing cells are suitable for a 3S pack. The BMS, charger and wiring must match the exact cells and load.

Before buying encoders, verify that they can physically mount to the existing motor/wheel shafts. If not, replacing only the two drive motors with encoder-equipped geared motors may be cheaper and more reliable.

---

# Learning + Build Pathway

Detailed stage guides are in [`pathway/`](./pathway/README.md). Complete them in order.

> 🎥 **Video-first route:** [Open the Video-First Learning Syllabus](./pathway/VIDEO-FIRST-LEARNING.md). It contains curated playlists and focused videos for every stage. Use it as the primary learning route, and use the detailed stage files as the actual build/lab manuals.

| Stage | Guide | Finished when... |
|---:|---|---|
| 1 | [Linux, Git & Workspace](./pathway/01-linux-git-workspace.md) | Development machine and repo workflow are ready |
| 2 | [ROS 2 Fundamentals](./pathway/02-ros2-fundamentals.md) | You can build nodes, topics and launch files |
| 3 | [Robot Model, TF & Simulation](./pathway/03-robot-model-tf-simulation.md) | Simulated differential-drive robot works |
| 4 | [Pico, Electronics & Drive Base](./pathway/04-pico-electronics-drive-base.md) | Pico safely controls both motors |
| 5 | [Encoders, PID & Wheel Odometry](./pathway/05-encoders-pid-odometry.md) | Closed-loop wheel control and odometry work |
| 6 | [Sensors & Sensor Fusion](./pathway/06-sensors-and-fusion.md) | IMU/range/IR data is calibrated and fused |
| 7 | [Pico ↔ ROS 2 Bridge](./pathway/07-pico-ros2-bridge.md) | ROS commands the physical base and receives telemetry |
| 8 | [Offline iPhone LiDAR Mapping](./pathway/08-offline-iphone-lidar-mapping.md) | A validated `room.yaml` + `room.pgm` map exists |
| 9 | [Phone-Free Localization](./pathway/09-phone-free-localization.md) | Robot can follow the static map with acceptable drift |
| 10 | [Nav2 Navigation & Obstacle Avoidance](./pathway/10-nav2-navigation.md) | Robot reaches goals while avoiding obstacles |
| 11 | [Coverage Planning & Cleaning](./pathway/11-coverage-and-cleaning.md) | Robot systematically cleans a selected zone |
| 12 | [Integration, Testing & Evaluation](./pathway/12-integration-testing.md) | Full system is repeatable and measured |
| 13 | [Novelty / AI Extensions](./pathway/13-novelty-ai.md) | Optional final-year innovation is layered on safely |

---

## Final ROS Architecture

```text
                         STATIC MAP
                    room.yaml + room.pgm
                             |
                             v
                       Nav2 map_server
                             |
      +----------------------+----------------------+
      |                                             |
      v                                             v
Coverage / Waypoints                           Nav2 Planner
                                                     |
                                                     v
                                                  cmd_vel
                                                     |
                                                     v
                                             cleanbot_bridge
                                                     |
                                                  USB serial
                                                     |
                                                     v
+----------------------------------------------------------------+
| Raspberry Pi Pico                                             |
| encoder counters -> wheel PID -> motor driver -> wheel motors |
| ultrasonic / IR / MPU6050 -> telemetry                        |
| watchdog + vacuum/brush control                               |
+----------------------------------------------------------------+
             |                           |
             v                           v
       encoder odometry             range / IMU topics
             |                           |
             +----------> EKF <-----------+
                            |
                            v
                     odom -> base_link

map -> odom is initialized from the known physical home pose.
```

---

## Recommended Repository Layout

```text
.
├── README.md
├── pathway/                 # learning/build guides
├── firmware/                # Pico firmware
├── src/                     # ROS 2 packages
│   ├── cleanbot_description/
│   ├── cleanbot_bridge/
│   ├── cleanbot_bringup/
│   ├── cleanbot_navigation/
│   └── cleanbot_coverage/
├── config/                  # EKF, Nav2, robot parameters
├── maps/                    # room.yaml + room.pgm
├── simulation/              # Gazebo worlds/config
├── hardware/                # wiring, dimensions, BOM
└── experiments/             # calibration + test results
```

---

## Final Project Target

```text
ONE-TIME:
iPhone LiDAR scan -> validated static room map -> remove iPhone

EVERY CLEANING RUN:
known home pose
      +
encoders + MPU6050 -> fused dead-reckoning
      +
ultrasonic / IR -> local safety
      +
static map + Nav2 -> autonomous navigation
      +
coverage planner -> systematic cleaning
      +
Pico -> wheel PID + vacuum + brush
      =
PHONE-FREE AUTONOMOUS CLEANING ROBOT
```

After the base system is reliable, the strongest novelty options are **coverage analytics**, a **safe high-level cleaning agent**, and optionally a **cheap camera for dirt-aware adaptive cleaning**. See Stage 13.
