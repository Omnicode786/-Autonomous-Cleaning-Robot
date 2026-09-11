# Video-First Learning Syllabus

This is the **video-first companion** to the detailed stage guides in this folder. The stage `.md` files remain the actual build manuals; this page tells us what to watch first and what to watch when a topic is still unclear.

## How to use this page

- **⭐ Main** = watch this first.
- **▶ Playlist** = best choice when you want a full course/series.
- **🧩 Extra** = focused explanation for a specific part of the stage.
- Do **not** watch everything before building. Watch the Main resources, do the stage exercises, then use Extras only where needed.
- Some excellent ROS videos use Humble or an older Gazebo. Learn the **concept** from them, but use the ROS 2 Jazzy / Gazebo Harmonic documentation already linked in each stage for current commands and package names.

---

# Stage 01 — Linux, Git & Workspace

Open: [01-linux-git-workspace.md](./01-linux-git-workspace.md)

### Watch first

1. ⭐ **The Construct — Linux for Robotics Basics (ROS 2 Jazzy Learning Week, Day 1)**  
   https://www.youtube.com/watch?v=ZD1WHWE0504
2. ▶ **LearnLinuxTV — Linux Crash Course playlist**  
   https://www.youtube.com/playlist?list=PLT98CRl2KxKHKd_tH3ssq0HPrThx2hESW
3. ⭐ **freeCodeCamp — Git and GitHub for Beginners: Crash Course**  
   https://www.youtube.com/watch?v=RGOj5yH7evk

### Useful extras

4. 🧩 **GitHub — A Brief Introduction to Git for Beginners**  
   https://www.youtube.com/watch?v=r8jQ9hVA2qs
5. 🧩 **The Construct — Python for Robotics (Jazzy Learning Week, Day 2)**  
   https://www.youtube.com/watch?v=VeFnm7bLhsM

**Move on when:** Ubuntu terminal use, package installation, Git commits/pushes, Python execution and USB-device basics no longer feel unfamiliar.

---

# Stage 02 — ROS 2 Fundamentals

Open: [02-ros2-fundamentals.md](./02-ros2-fundamentals.md)

### Best complete learning routes

1. ⭐ ▶ **Articulated Robotics — ROS 2 Fundamentals playlist**  
   https://www.youtube.com/playlist?list=PLunhqkrRNRhYYCaSTVP-qJnyUPkTxJnBt
2. ▶ **ROS 2 Comprehensive Tutorial playlist**  
   https://www.youtube.com/watch?v=bDmjX1bXVk0&list=PL8MgID9MCju0GMQDTWzYmfiU3wY_Zdjl5
3. ▶ **The Construct — ROS Developers Open Classes playlist**  
   https://www.youtube.com/playlist?list=PLK0b4e05LnzbuxWCdip-2Tf-SIiZle5NA

### Jazzy-specific starter sequence

4. ⭐ **ROS 2 Jazzy Basics & First Program**  
   https://www.youtube.com/watch?v=DnWu6iVcVRU
5. ⭐ **ROS 2 Topics — Jazzy Learning Week**  
   https://www.youtube.com/watch?v=Yn2nbiPNMkE
6. ⭐ **Hands-On ROS 2 Robot Project — Jazzy Learning Week**  
   https://www.youtube.com/watch?v=-roYHqI_Vc4

**Focus on:** nodes, topics, messages, publishers/subscribers, services, actions, parameters, packages, workspaces, `colcon`, launch files and rosbag2.

**Move on when:** you can create your own ROS package and understand what is happening when one node publishes data and another subscribes to it.

---

# Stage 03 — Robot Model, TF & Simulation

Open: [03-robot-model-tf-simulation.md](./03-robot-model-tf-simulation.md)

### Main course

1. ⭐ ▶ **Articulated Robotics — Building a Mobile Robot playlist**  
   https://www.youtube.com/playlist?list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT

This is one of the most useful series for our project because it follows the same general architecture: differential-drive robot, URDF, TF, simulation, control and navigation.

### URDF / robot description

2. ⭐ **Articulated Robotics — URDF Overview**  
   https://www.youtube.com/watch?v=CwdbsvcpOHM
3. 🧩 **Articulated Robotics — Creating a Rough 3D Robot Model**  
   https://www.youtube.com/watch?v=BcjHyhV0kIs

### TF / coordinate frames

4. ⭐ **Articulated Robotics — The Transform System (TF2)**  
   https://www.youtube.com/watch?v=QyvHhY4Y_Y8

### ros2_control and Gazebo

5. 🧩 **Articulated Robotics — ros2_control Overview**  
   https://www.youtube.com/watch?v=4QKsDf1c4hc
6. ⭐ **Articulated Robotics — Upgrading a ROS Robot to the New Gazebo**  
   https://www.youtube.com/watch?v=fH4gkIFZ6W8
7. ⭐ **Aleksandar Haber — ROS 2 Jazzy + Gazebo Harmonic Mobile Robot Tutorial**  
   https://www.youtube.com/watch?v=campHHAPtyg
8. 🧩 **Aleksandar Haber — ROS 2 Jazzy Mobile Robot + LiDAR Simulation**  
   https://www.youtube.com/watch?v=wOa1m8hzrgQ
9. 🧩 **Articulated Robotics — Using a LiDAR in ROS**  
   https://www.youtube.com/watch?v=eJZXRncGaGM

**Move on when:** the simulated cleanbot appears correctly in RViz/Gazebo, has a valid TF tree and drives from `/cmd_vel`.

---

# Stage 04 — Pico, Electronics & Drive Base

Open: [04-pico-electronics-drive-base.md](./04-pico-electronics-drive-base.md)

### Pico fundamentals

1. ⭐ **Core Electronics — Raspberry Pi Pico Course for Beginners**  
   https://www.youtube.com/watch?v=Ic4ExTusoTw
2. ⭐ **DroneBot Workshop — Raspberry Pi Pico: Control the I/O World**  
   https://www.youtube.com/watch?v=Zy64kZEM_bg

### Motors and power

3. 🧩 **Raspberry Pi Pico + TB6612FNG DC Motor Control**  
   https://www.youtube.com/watch?v=j56artvt-Lk
4. ⭐ **Articulated Robotics — Robot Power**  
   https://www.youtube.com/watch?v=Iye4uVLmj8o
5. 🧩 **Articulated Robotics — Robot Power Part 2**  
   https://www.youtube.com/watch?v=_FGYVgAti9M

### MPU6050

6. ⭐ **Raspberry Pi Pico + MPU6050 Tutorial**  
   https://www.youtube.com/watch?v=HezXoT12E40

**Focus on:** GPIO, PWM, I2C, timers/interrupts, safe voltage levels, motor-driver current limits and keeping noisy motors away from logic power.

**Move on when:** the Pico can independently drive both wheel motors and reliably read the sensors.

---

# Stage 05 — Encoders, PID & Wheel Odometry

Open: [05-encoders-pid-odometry.md](./05-encoders-pid-odometry.md)

### PID control

1. ⭐ ▶ **Brian Douglas — Understanding PID Control playlist**  
   https://www.youtube.com/playlist?list=PLn8PRpmsu08pQBgjxYFXSsODEF3Jqmm-y

Do not just copy PID numbers. Understand setpoint, measured velocity, error, P/I/D terms, saturation and why tuning one wheel independently matters.

### Differential-drive math

2. ⭐ **Aleksandar Haber — Differential Drive Forward Kinematics + Simulation**  
   https://www.youtube.com/watch?v=fx6bxPJ6BEs
3. 🧩 **Aleksandar Haber — Position Control of a Differential-Drive Robot**  
   https://www.youtube.com/watch?v=14xipN7Gx-I

### Pico encoders

4. ⭐ **Interfacing Quadrature Encoders with Raspberry Pi Pico**  
   https://www.youtube.com/watch?v=sgnEUxeNxpM
5. 🧩 **Reading a Quadrature Encoder with Raspberry Pi Pico**  
   https://www.youtube.com/watch?v=jklc2Aq9-1E

### ROS-side wheel control

6. 🧩 **Articulated Robotics — ros2_control on a Real Mobile Robot**  
   https://www.youtube.com/watch?v=4VVrTCnxvSw

**Move on when:** each wheel can hold a requested velocity with feedback and the robot can calculate repeatable encoder odometry.

---

# Stage 06 — Sensors & Sensor Fusion

Open: [06-sensors-and-fusion.md](./06-sensors-and-fusion.md)

### Sensor fusion

1. ⭐ **Automatic Addison — Sensor Fusion & Robot Localization Using ROS 2 Jazzy**  
   https://www.youtube.com/watch?v=XOQTF38lmtE
2. ⭐ ▶ **Brian Douglas — Sensor Fusion and Tracking playlist**  
   https://www.youtube.com/playlist?list=PLn8PRpmsu08ryYoBpEKzoMOveSTyS-h4a

### IMU practical work

3. ⭐ **Raspberry Pi Pico + MPU6050 Tutorial**  
   https://www.youtube.com/watch?v=HezXoT12E40

### Frames matter here too

4. 🧩 **Articulated Robotics — TF2 / Coordinate Transforms**  
   https://www.youtube.com/watch?v=QyvHhY4Y_Y8

**Focus on:** IMU bias, gyro drift, timestamps, covariance, sensor frame orientation, filtering and what an EKF can/cannot magically fix.

**Move on when:** encoder odometry and IMU data are correctly oriented, timestamped and fused without obvious jumps or frame errors.

---

# Stage 07 — Pico ↔ ROS 2 Bridge

Open: [07-pico-ros2-bridge.md](./07-pico-ros2-bridge.md)

Our first implementation should remain the **simple USB serial bridge** described in the stage guide. Learn micro-ROS as the next step, not as a requirement for getting the robot moving.

### micro-ROS / Pico resources

1. ⭐ **micro-ROS on Raspberry Pi Pico**  
   https://www.youtube.com/watch?v=2dGCcT9rxso
2. 🧩 **Raspberry Pi Pico + micro-ROS Tutorial**  
   https://www.youtube.com/watch?v=MBKAZ_2P1Sk
3. 🧩 **Articulated Robotics — Real Robot ros2_control Integration**  
   https://www.youtube.com/watch?v=4VVrTCnxvSw

### ROS communication refresher

4. ▶ **Articulated Robotics — ROS 2 Fundamentals playlist**  
   https://www.youtube.com/playlist?list=PLunhqkrRNRhYYCaSTVP-qJnyUPkTxJnBt

**Move on when:** ROS can send velocity commands to the physical robot and receive encoder/IMU/range telemetry continuously, with a working command-loss watchdog.

---

# Stage 08 — Offline iPhone LiDAR Mapping

Open: [08-offline-iphone-lidar-mapping.md](./08-offline-iphone-lidar-mapping.md)

Remember: the phone is a **mapping instrument only**. It does not remain on the cleaner.

### iPhone LiDAR scanning

1. ⭐ **Polycam — Ultimate iPhone LiDAR Guide for Scanning**  
   https://www.youtube.com/watch?v=7yXDY25C0hI

### Point clouds / processing

2. ⭐ **Open3D Point Cloud Processing Tutorial**  
   https://www.youtube.com/watch?v=2bVdvgzYLeQ
3. 🧩 **A Gentle Introduction to Open3D Point Clouds**  
   https://www.youtube.com/watch?v=UBRMzuTjE4E
4. 🧩 **Articulated Robotics — LiDAR in ROS**  
   https://www.youtube.com/watch?v=eJZXRncGaGM

### Optional developer videos if we build our own iPhone capture app

5. 🧩 **Apple WWDC — Explore ARKit 4**  
   https://developer.apple.com/videos/play/wwdc2020/10611/
6. 🧩 **Apple WWDC — RoomPlan: Create Parametric 3D Room Scans**  
   https://developer.apple.com/videos/play/wwdc2022/10127/

**Move on when:** we have a correctly scaled, cleaned-up `room.pgm` + `room.yaml`, verified against real room measurements, and a precisely marked home pose.

---

# Stage 09 — Phone-Free Localization

Open: [09-phone-free-localization.md](./09-phone-free-localization.md)

This is one of the **most important stages in the project** because the iPhone is now gone.

### Main learning

1. ⭐ **Automatic Addison — Sensor Fusion & Robot Localization Using ROS 2 Jazzy**  
   https://www.youtube.com/watch?v=XOQTF38lmtE
2. ▶ **Brian Douglas — Sensor Fusion and Tracking playlist**  
   https://www.youtube.com/playlist?list=PLn8PRpmsu08ryYoBpEKzoMOveSTyS-h4a
3. 🧩 **Aleksandar Haber — Differential Drive Forward Kinematics**  
   https://www.youtube.com/watch?v=fx6bxPJ6BEs
4. 🧩 **Articulated Robotics — TF2**  
   https://www.youtube.com/watch?v=QyvHhY4Y_Y8

### Cheap fallback if dead-reckoning drift is too high

5. ⭐ **AprilTag Detection and Pose Estimation Tutorial**  
   https://www.youtube.com/watch?v=fZ92_VMxxyo

That fallback lets us use a cheap ordinary camera + printed landmarks for occasional global correction instead of buying a LiDAR.

**Do not continue just because the RViz arrow looks plausible.** Measure straight-line, turn, square-path and full-cleaning-route drift as described in Stage 09.

---

# Stage 10 — Nav2 Navigation & Obstacle Avoidance

Open: [10-nav2-navigation.md](./10-nav2-navigation.md)

### Main Nav2 learning

1. ⭐ **Articulated Robotics — Nav2 Overview / Autonomous Navigation**  
   https://www.youtube.com/watch?v=jkoGkAd0GYk
2. ⭐ **Automatic Addison — Install / Get Started with Nav2 on ROS 2 Jazzy**  
   https://www.youtube.com/watch?v=gyskLlvX3oI
3. ▶ **MathWorks — Autonomous Navigation playlist**  
   https://www.youtube.com/playlist?list=PLn8PRpmsu08rLRGrnF-S6TyGrmcA2X7kg

### Supporting robot/navigation videos

4. 🧩 **Aleksandar Haber — ROS 2 Jazzy + Gazebo Harmonic Mobile Robot**  
   https://www.youtube.com/watch?v=campHHAPtyg
5. 🧩 **Aleksandar Haber — ROS 2 Jazzy LiDAR Mobile-Robot Simulation**  
   https://www.youtube.com/watch?v=wOa1m8hzrgQ
6. 🧩 **Articulated Robotics — Mobile Robot build series**  
   https://www.youtube.com/playlist?list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT

**Focus on:** map server, costmaps, robot footprint, inflation, planner, controller, behavior tree, goal checking and safe velocity limits. In our final robot, ultrasonic data supplies local obstacle information; the static map supplies known walls.

**Move on when:** repeated point-to-point navigation works on the real robot without needing the iPhone.

---

# Stage 11 — Coverage Planning & Cleaning

Open: [11-coverage-and-cleaning.md](./11-coverage-and-cleaning.md)

Nav2 gets the robot **from A to B**. This stage turns that into **clean the whole floor**.

### Watch first

1. ⭐ ▶ **MathWorks — Autonomous Navigation playlist**  
   https://www.youtube.com/playlist?list=PLn8PRpmsu08rLRGrnF-S6TyGrmcA2X7kg
2. ⭐ **Coverage Path Planning Example / Demonstration**  
   https://www.youtube.com/watch?v=trwM8ocZuMM
3. 🧩 **Articulated Robotics — Nav2 Overview**  
   https://www.youtube.com/watch?v=jkoGkAd0GYk

### Practical strategy for our first version

Do not begin with an advanced research coverage algorithm. First implement a rectangular/zone-based **boustrophedon (back-and-forth lawnmower)** path and send successive poses to Nav2. Once that is reliable, study the OpenNav Coverage code linked in the detailed stage guide.

**Move on when:** the robot covers a defined test area systematically, tracks completed lanes and operates the vacuum/brush during the run.

---

# Stage 12 — Integration, Debugging & Evaluation

Open: [12-integration-testing.md](./12-integration-testing.md)

A final-year demo needs repeatability, not just one lucky run.

### ROS debugging / recording

1. ⭐ **Robotics Back-End — ROS 2 rosbag2 Tutorial**  
   https://www.youtube.com/watch?v=a-O1qM9_S7k
2. ⭐ **ROS 2 `rqt_graph` / Graph Debugging Tutorial**  
   https://www.youtube.com/watch?v=xLthJMv7QYA
3. ⭐ **RViz2 Basics / Visualization Tutorial**  
   https://www.youtube.com/watch?v=L9lD4QQJ8Cw
4. 🧩 **The Construct — rosbag ROS 2 Live Class**  
   https://youtu.be/SZGYvfNL7iU
5. ▶ **ROS 2 Comprehensive Tutorial playlist — revisit debugging sections**  
   https://www.youtube.com/watch?v=bDmjX1bXVk0&list=PL8MgID9MCju0GMQDTWzYmfiU3wY_Zdjl5

**Measure:** endpoint error, square-path drift, navigation success rate, collision/near-collision count, area coverage percentage, cleaning time, battery runtime and watchdog behavior.

**Move on when:** multiple runs give similar results and you have logs/plots/tables to prove performance.

---

# Stage 13 — Novelty / AI Extensions

Open: [13-novelty-ai.md](./13-novelty-ai.md)

Only start this after the base autonomous cleaner is reliable.

## Dirt-aware computer vision — best novelty option

1. ⭐ **freeCodeCamp — OpenCV Course: Full Tutorial with Python**  
   https://www.youtube.com/watch?v=oXlwWbU8l2o
2. ▶ **freeCodeCamp — OpenCV Python: Computer Vision & AI Course**  
   https://www.youtube.com/watch?v=P4Z8_qe2Cu0
3. ⭐ **Roboflow — YOLO11 Custom Object Detection, Step by Step**  
   https://www.youtube.com/watch?v=etjkjZoG2F0
4. 🧩 **Ultralytics — Train YOLO11 on a Custom Dataset**  
   https://www.youtube.com/watch?v=ZN3nRZT7b24

A realistic novelty pipeline is:

```text
cheap RGB camera
      -> dirt / debris detector
      -> detections projected into robot/map coordinates
      -> dirt heatmap
      -> coverage planner adds slower or repeated passes
```

## Optional high-level AI supervisor

5. 🧩 **The Construct — ChatGPT for Robot Programming (ROS 2 Open Class)**  
   https://www.youtube.com/watch?v=qei2-K3tA5o

Use an LLM only to select **constrained high-level actions** such as `CLEAN_ZONE`, `RETRY_ZONE`, `PAUSE` and `RETURN_HOME`. Never allow it to bypass Nav2/Pico safety or directly choose motor PWM.

---

# Suggested Watch Order for the Whole Project

If you want one simple sequence instead of jumping between links:

1. The Construct Linux for Robotics.
2. LearnLinuxTV Linux Crash Course as needed.
3. Articulated Robotics ROS 2 Fundamentals playlist.
4. The Construct Jazzy ROS videos.
5. Articulated Robotics Building a Mobile Robot playlist.
6. Jazzy + Gazebo Harmonic mobile-robot video.
7. Pico beginner course.
8. Pico motor + encoder videos.
9. Brian Douglas PID playlist.
10. Automatic Addison sensor-fusion/Jazzy localization video.
11. Build our real ROS↔Pico drive base.
12. Polycam iPhone LiDAR scanning guide.
13. Open3D point-cloud videos.
14. Complete the offline-map stage.
15. Complete the phone-free drift tests.
16. Articulated Robotics Nav2 + Automatic Addison Jazzy Nav2.
17. Coverage-path material.
18. rosbag/RViz/rqt debugging resources.
19. Only then study OpenCV/YOLO/agent material.

## Important rule

**Videos teach the idea; our stage guide defines what we actually build.** If a video is using ROS 2 Humble, Foxy, Gazebo Classic or an older package name, do not blindly copy the command. Use the Jazzy/Harmonic references already included in the corresponding stage file.
