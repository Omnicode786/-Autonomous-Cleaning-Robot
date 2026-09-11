# Autonomous Cleaning Robot

Low-cost university project: an autonomous floor-cleaning robot using **ROS 2**, a **Raspberry Pi Pico**, **wheel encoders + IMU**, an **iPhone LiDAR/ARKit camera**, and inexpensive cleaning hardware.

> Target software: **Ubuntu 24.04 + ROS 2 Jazzy + Gazebo Harmonic + RTAB-Map + Nav2**.

---

## 1. What We Already Have

| Part | Status | Use |
|---|---|---|
| 2x DC geared motors (no encoders) | ✅ Have | Differential drive |
| Ultrasonic sensor(s) | ✅ Have | Close-range obstacle detection |
| MPU6050 | ✅ Have | IMU / angular motion |
| 5-channel IR array | ✅ Have | Floor/cliff experiments after calibration |
| Battery cells | ✅ Have | Robot power |
| iPhone with LiDAR | ✅ Have | RGB + depth + ARKit pose |
| Laptop/PC | ✅ Use existing | ROS 2, SLAM, Nav2, simulation |

The iPhone LiDAR is used **as part of the complete iPhone through ARKit**. We are not removing the LiDAR module.

---

## 2. What We Need to Buy

Approximate low-cost Pakistan prices checked in **September 2026**. Prices will move, so treat them as a budget range.

| Required part | Est. price (PKR) | Why we need it |
|---|---:|---|
| Raspberry Pi Pico RP2040 | **1,200–1,500** | Real-time motor/sensor controller |
| 2-wheel encoder kit / compatible optical encoders | **600–800** | Wheel speed, PID and odometry |
| TB6612FNG dual motor driver | **560–1,000** | Drives both wheel motors efficiently |
| 3S BMS | **300–450** | Battery protection/balancing* |
| 12.6 V 3S charger | **500–650** | Charge the battery pack* |
| LM2596 buck converter | **250–350** | Regulated logic/5 V power |
| 12 V mini vacuum donor | **1,000–1,500** | Blower + filter/dust hardware |
| Logic-level MOSFET/load driver | **300–600** | Pico-controlled vacuum/brush switching |
| Chassis + caster + mounts | **1,000–1,500** | Robot mechanical base |
| Brush + small DC motor | **500–800** | Move debris toward suction |
| Fuse, switch, wire, connectors, level shifting, phone mount, etc. | **700–1,200** | Assembly and electrical safety |

**Expected remaining budget: ~PKR 7,000–10,500.** A sensible target is around **PKR 8,000–9,000**.

\*Assumes we build a **3S lithium-ion pack** from the cells we already have. Battery wiring, BMS and charger must match the actual cells and load current.

### Purchase references

- [Raspberry Pi Pico — Digilog](https://digilog.pk/products/raspberry-pi-pico-rp2040-cheap-price-in-pakistan)
- [HC-020K encoder kit example — Daraz](https://www.daraz.pk/products/1set-hc-020k-double-speed-measuring-sensor-module-with-photoelectric-encoders-kit-top-for-arduino-i372652215.html)
- [TB6612FNG — Digilog](https://lite.digilog.pk/products/tb6612fng-motor-driver-module-for-arduino-in-pakistan)
- [3S BMS example — Daraz](https://www.daraz.pk/products/3s-20a-li-ion-lithium-battery-18650-charger-bms-protection-board-126v-i510367353.html)
- [12.6 V 3S charger — Digilog](https://lite.digilog.pk/products/12-6v-1a-3s-3-cell-battery-balance-charger-lithium-18650-3-cell-charger)
- [LM2596 listings — Daraz](https://www.daraz.pk/tag/lm2596-dc-to-dc-buck-converter/)
- [12 V mini vacuum example — Daraz](https://www.daraz.pk/products/portable-mini-car-vacuum-cleaner-dc-12v-i488714266.html)
- [2WD chassis example — College Road](https://colgroad.com/product/2wd-acrylic-robot-car-chassis/)

### Before buying the encoders / motor driver

1. Confirm an encoder disc can mechanically attach to each existing motor/wheel shaft. If not, use another compatible magnetic/optical encoder or replace only the drive motors with encoder-equipped versions.
2. Measure/check each motor's **stall current**. TB6612FNG is good for small geared motors, but use a higher-current driver if the motors exceed its rating.

---

## 3. System Overview

```text
        iPhone (ARKit + LiDAR)
         RGB + Depth + Pose
                 |
              Wi-Fi
                 v
+----------------------------------+
| Laptop: Ubuntu 24.04 + ROS 2     |
|                                  |
| iPhone bridge                    |
| encoder/IMU odometry fusion      |
| RTAB-Map SLAM                    |
| Nav2 navigation                  |
| coverage planner                 |
| cleaning controller              |
+----------------+-----------------+
                 |
              USB serial
                 v
+----------------------------------+
| Raspberry Pi Pico                |
|                                  |
| wheel encoder reading            |
| wheel speed PID                  |
| motor PWM/direction              |
| MPU6050 + ultrasonic + IR        |
| vacuum/brush switching           |
+---------+------------------------+
          |
   motors + vacuum + brush
```

For the first working version, use a simple **USB serial bridge** between ROS 2 and the Pico. **micro-ROS can be added later** if we want ROS-native communication on the microcontroller.

---

# 4. Learning Roadmap

The goal is not to master every topic. Learn enough to complete the checkpoint, then move forward.

## Stage 1 — Linux + Git Basics

**Understand:** terminal navigation, `apt`, permissions, processes, USB serial, Git clone/commit/push/branch.

- 📖 [Ubuntu command-line tutorial](https://ubuntu.com/tutorials/command-line-for-beginners)
- 📖 [GitHub Skills: Introduction to GitHub](https://github.com/skills/introduction-to-github)
- 🎥 [LearnLinuxTV Linux Crash Course playlist](https://www.youtube.com/playlist?list=PLT98CRl2KxKHKd_tH3ssq0HPrThx2hESW)

**Checkpoint:** comfortably install packages, use the terminal and push code to this repo.

---

## Stage 2 — ROS 2 Fundamentals

**Understand:** nodes, topics, messages, publishers/subscribers, services, actions, parameters, packages, workspaces, `colcon`, launch files and rosbag2.

- 📖 [Official ROS 2 Jazzy tutorials](https://docs.ros.org/en/jazzy/Tutorials.html)
- 🎥 [Articulated Robotics — ROS 2 Fundamentals playlist](https://www.youtube.com/playlist?list=PLunhqkrRNRhYYCaSTVP-qJnyUPkTxJnBt)

**Checkpoint:** create a ROS 2 package with a publisher, subscriber and launch file.

---

## Stage 3 — Mobile Robot, TF, URDF & Gazebo

**Understand:** differential drive, `cmd_vel`, URDF/Xacro, TF frames, RViz2 and simulation.

Core frame tree:

```text
map -> odom -> base_link -> sensors
```

- 🎥 [Articulated Robotics — Building a Mobile Robot playlist](https://www.youtube.com/playlist?list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT)
- 📖 [ROS 2 Jazzy tutorials](https://docs.ros.org/en/jazzy/Tutorials.html)
- 📖 [Gazebo Harmonic + ROS 2 integration](https://gazebosim.org/docs/harmonic/ros2_integration/)
- 📖 [Gazebo / ROS installation pairing](https://gazebosim.org/docs/harmonic/ros_installation/)

The Articulated Robotics playlist is excellent for concepts, but some episodes use older ROS/Gazebo versions. Use the **Jazzy/Harmonic documentation for current commands**.

**Checkpoint:** simulated robot drives in Gazebo from `/cmd_vel` and displays correctly in RViz.

---

## Stage 4 — Raspberry Pi Pico + Electronics

**Understand:** GPIO, PWM, I2C, UART/USB serial, timers/interrupts and safe motor switching.

- 📖 [Official Raspberry Pi Pico C/C++ SDK](https://www.raspberrypi.com/documentation/microcontrollers/c_sdk.html)
- 📖 [Pico SDK API documentation](https://www.raspberrypi.com/documentation/pico-sdk/)
- 🎥 [Raspberry Pi Pico beginner course](https://www.youtube.com/watch?v=Ic4ExTusoTw)

**Checkpoint:** Pico independently drives both motors and reads the MPU6050, ultrasonic sensor and IR array.

---

## Stage 5 — Encoders, PID & Wheel Odometry

**Understand:** encoder counts, RPM, quadrature if available, wheel velocity, PID speed control and differential-drive odometry.

- 🎥 [Brian Douglas — Understanding PID Control playlist](https://www.youtube.com/playlist?list=PLn8PRpmsu08pQBgjxYFXSsODEF3Jqmm-y)
- 📖 [ROS `diff_drive_controller` documentation](https://control.ros.org/jazzy/doc/ros2_controllers/diff_drive_controller/doc/userdoc.html)

**Checkpoint:** command a wheel speed and have the Pico maintain it using encoder feedback; publish usable wheel odometry.

---

## Stage 6 — Pico ↔ ROS 2 Communication

Start simple: ROS 2 node on the laptop ↔ USB serial ↔ Pico.

Typical data:

```text
ROS -> Pico:  target left/right speed, vacuum on/off
Pico -> ROS:  encoder counts, IMU, ultrasonic, IR, battery
```

Later option:

- 💻 [micro-ROS Raspberry Pi Pico SDK repository](https://github.com/micro-ROS/micro_ros_raspberrypi_pico_sdk)

**Checkpoint:** `/cmd_vel` eventually controls the physical wheels and sensor data appears as ROS topics.

---

## Stage 7 — Odometry + IMU Fusion

**Understand:** encoder odometry drift, IMU bias, coordinate frames and EKF sensor fusion.

- 📖 [robot_localization documentation](https://docs.ros.org/en/jazzy/p/robot_localization/)

**Checkpoint:** stable `odom -> base_link` transform using wheel odometry + MPU6050 data.

---

## Stage 8 — iPhone LiDAR + ARKit

**Understand:** ARKit world tracking, RGB frames, scene depth, camera intrinsics, pose and point clouds.

- 📖 [Apple ARKit Scene Depth](https://developer.apple.com/documentation/arkit/arconfiguration/framesemantics-swift.struct/scenedepth)
- 📖 [Apple: Displaying a Point Cloud Using Scene Depth](https://developer.apple.com/documentation/arkit/displaying-a-point-cloud-using-scene-depth)
- 💻 [iPhone LiDAR → ROS 2 reference project](https://github.com/MatthewKazan/iPhone-lidar-slam-playground)

**Important:** building a custom ARKit iPhone app normally requires **Xcode on macOS**. If necessary, borrow/use a Mac only for building/deploying the phone app; the robot's ROS computer remains the laptop.

**Checkpoint:** ROS receives synchronized RGB/depth and a correctly transformed iPhone camera pose/point cloud.

---

## Stage 9 — RTAB-Map SLAM

**Understand:** RGB-D SLAM, odometry input, TF, loop closure, map creation and saving maps.

- 💻 [RTAB-Map ROS 2 repository](https://github.com/introlab/rtabmap_ros/tree/ros2)
- 📖 [RTAB-Map tutorials/wiki](https://github.com/introlab/rtabmap/wiki/Tutorials)

**Checkpoint:** manually drive the robot around a room and produce a repeatable map.

---

## Stage 10 — Nav2 Autonomous Navigation

**Understand:** global/local costmaps, planner, controller, behavior tree, localization and obstacle layers.

- 📖 [Nav2 Jazzy Getting Started](https://docs.nav2.org/jazzy/getting_started/)
- 📖 [Nav2 Quickstart](https://docs.nav2.org/jazzy/getting_started/quickstart/quickstart.html)
- 🎥 [Articulated Robotics — Nav2 overview](https://www.youtube.com/watch?v=jkoGkAd0GYk)

**Checkpoint:** click a goal in RViz and the real robot reaches it while avoiding obstacles.

---

## Stage 11 — Cleaning Coverage

Normal Nav2 moves **A → B**. A vacuum must cover an **entire area** systematically.

**Understand:** room boundary, coverage path, lane spacing, already-cleaned cells and obstacle replanning.

- 💻 [OpenNav Coverage repository](https://github.com/open-navigation/opennav_coverage)
- 📖 [Nav2 Coverage Server documentation](https://docs.nav2.org/rolling/configuration_and_development/configuration_guide/others/configuring_coverage_server/)

**Checkpoint:** robot performs lane-by-lane room coverage instead of random wandering.

---

## Stage 12 — Vacuum, Brush & Safety

**Understand:** MOSFET/load switching, motor electrical noise, BMS, buck conversion, fusing, battery monitoring, emergency stop and communication watchdogs.

The Pico should stop the drive motors if ROS commands disappear for a short timeout.

**Checkpoint:** autonomous navigation + coverage + actual suction/brush operation work together safely.

---

# 5. Recommended Build Order

1. **Simulation:** URDF + TF + Gazebo + `/cmd_vel`.
2. **Drive base:** Pico + motor driver + encoders + wheel PID.
3. **Sensors:** MPU6050 + ultrasonic + IR.
4. **ROS bridge:** physical robot controllable from ROS.
5. **Odometry:** encoders + IMU fusion.
6. **iPhone:** RGB-D + ARKit data into ROS.
7. **SLAM:** create/save maps with RTAB-Map.
8. **Navigation:** Nav2 point-to-point autonomy.
9. **Cleaning:** vacuum + brush + coverage planner.
10. **Novelty:** only after the base robot is reliable.

---

# 6. Novelty / Final-Year Upgrade Ideas

### Best option: Dirt-Aware Adaptive Cleaning
Use the iPhone RGB camera with a lightweight vision model to detect visibly dirty regions. Store them as a **dirt heatmap** and automatically slow down or perform extra passes there.

### Semantic Cleaning
Recognize simple zones/surfaces such as carpet, tile, desk area or doorway and change speed, suction or number of passes.

### Safe Supervisor Agent
Add a secondary high-level agent that can interpret commands such as:

> “Clean the room, but spend extra time near the desks.”

The agent may choose only predefined ROS actions such as `CLEAN_ZONE`, `RETRY_ZONE`, `PAUSE` and `RETURN_HOME`. **It must never directly control motor PWM or bypass Nav2/safety logic.**

### Cleaning Analytics
Generate a final map showing coverage %, cleaning time, battery use and dirt hotspots.

---

## Final Target

```text
Encoders + IMU -> reliable odometry
          +
iPhone RGB-D -> RTAB-Map -> map
          +
Nav2 -> autonomous movement
          +
Coverage planner -> systematic room cleaning
          +
Pico -> motors + sensors + vacuum/brush
          =
Autonomous Cleaning Robot
```
