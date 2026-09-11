# Autonomous Cleaning Robot - Learning Roadmap and Project Plan

A low-cost university project for building an autonomous floor-cleaning robot using a Raspberry Pi Pico, an iPhone LiDAR camera, a laptop running ROS 2, and inexpensive sensors and motors.

This README is deliberately organized as a learning path. Do the sections in order. Do not start with SLAM, Nav2, or AI until the basic robot and simulation are working.

## 1. Project Goal

Build a robot that can:

- Drive using two DC geared motors.
- Receive high-level velocity commands from ROS 2.
- Read ultrasonic distance sensors.
- Read an MPU6050 IMU.
- Read the existing 5-element IR array.
- Use an iPhone LiDAR-capable rear camera for RGB-D perception and ARKit visual-inertial pose estimation.
- Build a map of an indoor environment.
- Localize itself in that map.
- Navigate autonomously while avoiding obstacles.
- Cover a room systematically for cleaning.
- Switch a vacuum/blower and brush motor on and off.
- Monitor battery state and stop safely on communication failure.

## 2. Hardware We Already Have

| Part | Status | Planned role |
|---|---|---|
| DC geared motors, no encoders | Have | Differential drive |
| Ultrasonic sensor(s) | Have | Near-field obstacle detection |
| MPU6050 | Have | IMU / motion sensing |
| 5-element IR sensor array | Have | Floor/cliff/edge experiments; depends on module |
| Battery cells | Have | Main power source |
| iPhone with LiDAR | Have | RGB-D + ARKit pose |
| Laptop | Have | Ubuntu + ROS 2 + SLAM + Nav2 |

Likely additional low-cost hardware:

- Raspberry Pi Pico RP2040.
- Dual DC motor driver.
- BMS appropriate for the battery pack.
- 5 V / 3.3 V regulator or buck converter as required.
- MOSFET or suitable driver for the vacuum/blower motor.
- Brush motor and brush mechanism.
- Chassis, caster, wheels, dust chamber/filter, switches, fuse, wiring and connectors.

## 3. Recommended System Architecture

```text
                 iPhone
       ARKit + Camera + LiDAR
      RGB + depth + camera pose
                    |
             USB or Wi-Fi
                    |
                    v
        +-----------------------+
        |        Laptop         |
        | Ubuntu 24.04          |
        | ROS 2 Jazzy           |
        |                       |
        | iPhone bridge         |
        | RTAB-Map              |
        | Nav2                  |
        | Coverage planner      |
        | RViz2 / rosbag2       |
        +-----------+-----------+
                    |
          USB Serial / micro-ROS
                    |
                    v
        +-----------------------+
        | Raspberry Pi Pico     |
        |                       |
        | Motor PWM/direction   |
        | Ultrasonic            |
        | MPU6050               |
        | IR array              |
        | Battery measurement   |
        | Vacuum/brush control  |
        | Command watchdog      |
        +-----------+-----------+
                    |
            Motor / MOSFET
                    |
        Wheels + Vacuum + Brush
```

## 4. Software Stack We Will Target

Use this stack unless a dependency forces a change:

- Ubuntu 24.04 LTS on the laptop.
- ROS 2 Jazzy Jalisco.
- Python 3 for most ROS nodes and rapid prototyping.
- C/C++ Pico SDK for the final Pico firmware if using micro-ROS.
- MicroPython only for quick standalone Pico sensor/motor experiments if desired.
- Gazebo Harmonic for simulation.
- RViz2 for visualization and debugging.
- URDF/Xacro for the robot model.
- TF2 for coordinate frames.
- RTAB-Map for RGB-D SLAM with iPhone data.
- Nav2 for autonomous navigation.
- rosbag2 for recording experiments.
- Git and GitHub for version control.

Why Jazzy instead of simply choosing the newest ROS release? Jazzy is an LTS ROS 2 release, has excellent Ubuntu 24.04 support, and has a mature combination of Nav2 and Gazebo Harmonic. For a university project, stability and tutorial availability matter more than using the newest feature set.

Official release/support reference:
https://www.ros.org/reps/rep-2000.html

ROS getting started page:
https://www.ros.org/blog/getting-started/

## 5. The Most Important Design Decision: No Wheel Encoders

Our current DC motors do not have encoders.

This is acceptable for the first prototype, but it changes the architecture.

Without encoders:

- The Pico can command motor PWM but cannot know the true wheel speed.
- Equal PWM does not guarantee equal wheel velocity.
- Carpet, wheel friction and battery voltage will change the motion.
- Traditional wheel odometry will not be trustworthy.

For the first version, use ARKit visual-inertial odometry from the iPhone as the main motion estimate. RTAB-Map can use external odometry, and ARKit pose can conceptually supply the `odom -> base_link` motion estimate.

Later, adding inexpensive magnetic or optical wheel encoders is one of the highest-value upgrades. Encoders make low-level velocity PID control and dead reckoning dramatically better.

Do not fake encoder odometry just to satisfy Nav2. A wrong odometry source is often worse than a carefully integrated external pose source.

## 6. Learning Order - Overview

Follow this order:

1. Linux terminal basics.
2. Git and GitHub basics.
3. Python basics.
4. Basic electronics and Pico programming.
5. Differential-drive robotics math.
6. ROS 2 fundamentals.
7. TF2, URDF/Xacro and RViz2.
8. Gazebo simulation.
9. Pico motor and sensor firmware.
10. Laptop-to-Pico communication.
11. iPhone ARKit + LiDAR data acquisition.
12. ROS representation of RGB, depth, pose and point clouds.
13. RTAB-Map RGB-D SLAM.
14. Nav2 autonomous navigation.
15. Ultrasonic/IR safety integration.
16. Coverage-path planning for cleaning.
17. Vacuum/brush control and power management.
18. Logging, testing and evaluation.
19. Novelty/AI additions only after the base robot works.

---

# PHASE A - Linux, Git and Python Foundations

## A1. Linux terminal

You do not need to become a Linux administrator. You need to be comfortable with:

- `pwd`, `ls`, `cd`
- `mkdir`, `cp`, `mv`, `rm`
- `cat`, `less`, `grep`
- pipes `|` and redirection `>`
- `sudo`
- `apt`
- environment variables
- `.bashrc`
- process basics: `ps`, `top`, `kill`
- device files such as `/dev/ttyACM0`
- file permissions and `chmod`
- USB serial permissions
- SSH basics

Best beginner video for this exact robotics path:

The Construct - Linux for Robotics Basics, ROS 2 Jazzy Learning Week Day 1
https://www.youtube.com/watch?v=ZD1WHWE0504

Official Ubuntu command-line tutorial:
https://ubuntu.com/desktop/docs/en/latest/tutorial/the-linux-command-line-for-beginners/

### A1 deliverable

You should be able to create folders, install packages, edit files, run Python files and understand why `source /opt/ros/jazzy/setup.bash` is needed.

## A2. Git and GitHub

Learn:

- repository
- clone
- status
- add
- commit
- push
- pull
- branch
- merge
- `.gitignore`
- README Markdown
- issues
- pull requests

GitHub's official interactive beginner exercise:
https://github.com/skills/introduction-to-github

GitHub Git overview:
https://docs.github.com/en/get-started/using-git/about-git

### A2 deliverable

Make a test repository, create a branch, modify a README, commit it and merge it.

## A3. Python

Python will be our main language for high-level ROS nodes.

Learn only the parts we need:

- variables and basic types
- lists, tuples and dictionaries
- conditions
- loops
- functions
- modules/imports
- classes and objects
- exceptions
- file I/O
- `venv`
- basic NumPy arrays
- sockets/WebSockets later if required

The Construct - Python for Robotics, Jazzy Learning Week Day 2
https://www.youtube.com/watch?v=VeFnm7bLhsM

Official Python tutorial:
https://docs.python.org/3/tutorial/

### A3 deliverable

Write a Python program that reads simulated left/right motor commands, calculates differential-drive motion values, and logs them to a file.

---

# PHASE B - Basic Robotics Math You Actually Need

Do not spend weeks on advanced mathematics. Learn these specific concepts.

## B1. Differential-drive kinematics

Understand:

- wheel radius `r`
- wheel separation / track width `L`
- left wheel velocity `v_l`
- right wheel velocity `v_r`
- robot linear velocity `v`
- robot angular velocity `omega`

Core relationships:

```text
v     = (v_r + v_l) / 2
omega = (v_r - v_l) / L

v_r = v + omega * L / 2
v_l = v - omega * L / 2
```

This is how a ROS `cmd_vel` command eventually becomes left/right wheel commands.

## B2. Coordinate frames

Understand:

- x, y, z axes
- translation
- rotation
- yaw, pitch, roll
- quaternion at a practical level
- parent and child frames
- homogeneous transforms conceptually

Important frames we will eventually use:

```text
map
 |
odom
 |
base_link
 |-- iphone_link
 |-- imu_link
 |-- ultrasonic_front_link
 |-- ultrasonic_left_link
 |-- ultrasonic_right_link
```

## B3. Sensor basics

Learn:

- noise
- bias
- drift
- sampling rate
- timestamping
- calibration
- low-pass filtering
- why sensors need a common coordinate frame

## B4. PID control

Learn the idea of:

- proportional P
- integral I
- derivative D
- setpoint
- measured value
- error

We cannot properly close the wheel-speed loop without encoders, so PID is not essential for the first motor version. Learn it because it becomes important immediately if encoders are added.

---

# PHASE C - ROS 2 Fundamentals

This is the core of the project.

## C1. Install ROS 2 Jazzy

Target platform:

- Ubuntu 24.04
- ROS 2 Jazzy Desktop

Official installation guide:
https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html

Install the desktop version because we want RViz and common development tools.

## C2. Learn ROS 2 concepts

Learn in this order:

1. ROS graph
2. nodes
3. topics
4. messages
5. publishers/subscribers
6. services
7. actions
8. parameters
9. packages
10. workspaces
11. `colcon`
12. launch files
13. namespaces/remapping
14. QoS basics
15. `rqt_graph`
16. rosbag2

Best free Jazzy-oriented beginner sequence:

Day 3 - ROS 2 Jazzy Basics and First Program
https://www.youtube.com/watch?v=DnWu6iVcVRU

Day 4 - ROS 2 Topics
https://www.youtube.com/watch?v=Yn2nbiPNMkE

Day 5 - Hands-On ROS 2 Robot Project
https://www.youtube.com/watch?v=-roYHqI_Vc4

Official ROS 2 beginner CLI tutorials:
https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools.html

Official guide to topics, services and actions:
https://docs.ros.org/en/jazzy/How-To-Guides/Topics-Services-Actions.html

Additional practical ROS learning:
https://roboticsbackend.com/how-to-learn-ros2/

### C deliverable

Create a package named something like `cleanbot_bringup` and write:

- one Python publisher
- one Python subscriber
- one service
- one launch file

Then inspect it with `ros2 node list`, `ros2 topic list`, `ros2 topic echo`, `ros2 topic hz` and `rqt_graph`.

---

# PHASE D - Build a Mobile Robot in Simulation First

This phase will save enormous debugging time later.

## D1. Primary video series

If you watch only one long project series, use:

Articulated Robotics - Building a Mobile Robot
https://youtube.com/playlist?list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT

It covers the architecture of a ROS 2 mobile robot, robot description, simulation, control and navigation. Some individual commands may target an older ROS/Gazebo release, so use the concepts from the series and the Jazzy/Harmonic official documentation for current package names and commands.

Written Articulated Robotics ROS overview:
https://articulatedrobotics.xyz/tutorials/ready-for-ros/ros-overview/

## D2. Learn URDF and Xacro

Learn:

- links
- joints
- visual geometry
- collision geometry
- inertial properties
- sensor frames
- wheel joints
- fixed joints
- Xacro variables and macros

Do not try to create a beautiful CAD model first. Start with cylinders and boxes.

### D2 deliverable

Your robot must appear correctly in RViz2 with:

- base
- two wheels
- caster
- iPhone frame
- IMU frame
- ultrasonic frames

## D3. Learn TF2

TF2 is not optional. A large fraction of ROS navigation problems are actually transform problems.

You must understand:

```text
map -> odom -> base_link -> sensors
```

Learn how to inspect transforms with RViz and TF command-line tools.

## D4. Gazebo Harmonic

For ROS 2 Jazzy use modern Gazebo / Gazebo Harmonic rather than old Gazebo Classic tutorials wherever possible.

Official ROS-Gazebo integration tutorial:
https://gazebosim.org/docs/harmonic/ros2_integration/

Official Gazebo installation with ROS:
https://gazebosim.org/docs/harmonic/ros_installation/

Nav2 custom robot setup guide for Gazebo:
https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/gazebo/

### D deliverable

Before touching SLAM, your simulated robot must:

- spawn in Gazebo
- appear in RViz
- have a valid TF tree
- receive `cmd_vel`
- drive forward/backward
- rotate left/right
- be controllable with keyboard teleop

---

# PHASE E - Raspberry Pi Pico and Electronics

## E1. Pico fundamentals

Learn:

- GPIO input/output
- PWM
- ADC
- I2C
- UART
- timers
- interrupts conceptually
- USB serial
- non-blocking program loops
- watchdog behavior

Excellent free Pico course:

Core Electronics - Pico Course for Beginners
https://www.youtube.com/watch?v=Ic4ExTusoTw

You do not need every chapter. Prioritize GPIO, PWM, I2C and UART.

## E2. Motor control

Understand the motor driver's:

- PWM / enable inputs
- direction inputs
- motor voltage
- logic voltage
- current rating
- heat dissipation

Practice sequence:

1. Spin left motor slowly.
2. Spin right motor slowly.
3. Reverse each motor.
4. Ramp PWM instead of instantly jumping to maximum.
5. Make helper functions for forward, reverse and stop.

Important: the Pico GPIO must never directly power a DC motor.

## E3. MPU6050

Learn:

- I2C addressing
- accelerometer readings
- gyroscope readings
- units
- gyro bias
- why yaw drifts if integrated by itself

Practical Pico + MPU6050 video:
https://www.youtube.com/watch?v=HezXoT12E40

## E4. Ultrasonic sensors

Learn:

- trigger pulse
- echo pulse
- time-of-flight
- conversion to metres
- min/max range
- timeout handling
- filtering obvious outliers

If using HC-SR04-style hardware, verify its echo voltage before connecting it to a 3.3 V Pico input. Use appropriate level shifting/division when necessary.

## E5. 5-element IR array

Identify exactly what the array is designed for.

If it is a reflective line-following array, it may be useful for:

- floor contrast detection
- cliff/drop experiments if mounted downward
- boundary-marker experiments

But it is not automatically a reliable cliff sensor. Calibrate it on the actual floor surfaces used for the demonstration.

## E6. Power electronics

Learn enough to safely build:

- battery pack + BMS
- master fuse
- master switch
- motor power rail
- logic power rail
- buck conversion
- common ground
- MOSFET switching for vacuum/brush
- decoupling capacitors
- keeping motor noise away from logic

### E deliverable

A standalone Pico program should be able to:

- command both motors
- read ultrasonic data
- read MPU6050 data
- read IR data
- report battery voltage if hardware is available
- switch vacuum/brush outputs
- immediately stop motors when a software watchdog expires

---

# PHASE F - Connect the Pico to ROS 2

There are two possible approaches.

## Route 1 - Recommended first: simple USB serial bridge

This is the easiest for a university prototype.

Pico sends lines/packets such as:

```text
IMU,ax,ay,az,gx,gy,gz
RANGE,front,0.42
IR,1,0,0,1,1
BATTERY,11.4
```

Laptop ROS node converts them into standard ROS messages.

Laptop sends commands such as:

```text
VEL,0.15,0.30
VACUUM,1
BRUSH,1
STOP
```

Advantages:

- very easy to debug
- Pico firmware stays small
- no complex embedded ROS build system initially

## Route 2 - Optional upgrade: micro-ROS

Once the serial prototype works, migrate to micro-ROS if time allows.

Best tutorial:

Articulated Robotics - Beginner micro-ROS Tutorial with Raspberry Pi Pico
https://www.youtube.com/watch?v=MBKAZ_2P1Sk

Official/community Pico micro-ROS integration repository:
https://github.com/micro-ROS/micro_ros_raspberrypi_pico_sdk

### Suggested ROS topics

Pico or serial bridge -> ROS:

```text
/imu/data
/range/front
/range/left
/range/right
/ir_array
/battery_voltage
/cleaning/status
```

ROS -> Pico:

```text
/cmd_vel
/cleaning/vacuum_enable
/cleaning/brush_enable
/emergency_stop
```

### Critical safety behavior

If the Pico has not received a valid velocity/heartbeat command for a short timeout, it must set wheel PWM to zero. Do this on the Pico itself rather than trusting a high-level ROS process to stop it.

### F deliverable

Use keyboard teleop in ROS 2 to drive the real robot through the Pico.

Do not proceed until this is reliable.

---

# PHASE G - iPhone LiDAR and ARKit

This is the most project-specific part.

## G1. What we want from the iPhone

The ideal data stream is:

- RGB image
- depth image
- camera intrinsics
- ARKit camera pose / VIO
- timestamps
- optional confidence map

Apple ARKit scene-depth documentation:
https://developer.apple.com/documentation/arkit/arframe/scenedepth

Apple example - displaying a point cloud using scene depth:
https://developer.apple.com/documentation/ARKit/displaying-a-point-cloud-using-scene-depth

ARKit smoothed scene depth:
https://developer.apple.com/documentation/arkit/arframe/smoothedscenedepth

## G2. Learn these ARKit concepts

- `ARSession`
- `ARWorldTrackingConfiguration`
- `ARFrame`
- `capturedImage`
- `sceneDepth`
- `smoothedSceneDepth`
- depth map
- confidence map
- camera intrinsics
- camera transform
- timestamp synchronization

## G3. Important coordinate-frame work

ARKit and ROS use different coordinate conventions.

You must explicitly convert:

- ARKit camera position/orientation
- camera optical frame
- robot `base_link`

Do not fix this by trial-and-error signs until the map "looks okay". Document the transform mathematically and test axes one at a time.

## G4. Existing projects to study

Very relevant open-source example:

`iPhone-lidar-slam-playground`
https://github.com/MatthewKazan/iPhone-lidar-slam-playground

It contains a Swift iPhone app using ARKit to capture point clouds and upload scans to ROS 2 through rosbridge. Study it rather than designing the entire iOS-to-ROS path from zero.

Alternative for early prototyping:

Record3D library
https://github.com/marek-simonik/record3d

Record3D can stream RGB-D and camera pose from supported iOS devices to a computer, which can be useful for proving the ROS/SLAM side before writing a custom iPhone app.

## G5. Preferred ROS message representation

Try to publish standard messages:

```text
/camera/color/image_raw          sensor_msgs/Image
/camera/depth/image_raw          sensor_msgs/Image
/camera/color/camera_info        sensor_msgs/CameraInfo
/iphone/odom                     nav_msgs/Odometry
/tf                              TF transforms
```

PointCloud2 is useful for visualization and obstacles, but for RTAB-Map it may be cleaner to preserve synchronized RGB + registered depth + camera info + odometry rather than unnecessarily converting everything into a huge point cloud stream first.

### G deliverable

With the phone mounted rigidly on the stationary robot:

- show RGB in RViz
- show depth data
- visualize a point cloud if desired
- move the robot/phone and confirm the ARKit pose axes behave correctly
- verify timestamps and update rates

---

# PHASE H - SLAM with RTAB-Map

Because our main ranging sensor is an iPhone RGB-D system, use RTAB-Map rather than beginning with a 2D LaserScan-only SLAM pipeline.

RTAB-Map ROS repository:
https://github.com/introlab/rtabmap_ros

RTAB-Map supports RGB-D input, external odometry and ROS 2 integration.

Install binaries first rather than building from source unless a missing feature forces you to build:

```bash
sudo apt install ros-jazzy-rtabmap-ros
```

## H1. Learn these SLAM concepts

- odometry
- localization
- mapping
- keyframes
- loop closure
- pose graph
- occupancy grid
- point cloud
- drift
- map frame vs odom frame

## H2. Our intended data flow

```text
iPhone RGB -----------+
iPhone depth ---------+----> RTAB-Map ----> map / occupancy grid
iPhone camera_info ---+
ARKit VIO odometry ---+
```

Useful background discussion from the RTAB-Map maintainer about the iOS architecture:
https://github.com/introlab/rtabmap/discussions/1155

### H deliverable

Push the robot manually around a small room and produce a repeatable map with at least one successful loop closure.

Record rosbag files while testing so the same sensor run can be replayed without physically driving the robot every time.

---

# PHASE I - Autonomous Navigation with Nav2

Do this only after mapping and transforms are stable.

## I1. Best starting resources

Nav2 Getting Started:
https://docs.nav2.org/jazzy/getting_started/

Nav2 Quickstart:
https://docs.nav2.org/jazzy/getting_started/quickstart/quickstart/

Nav2 Navigation Concepts:
https://docs.nav2.org/jazzy/getting_started/navigation_concepts/

Nav2 First-Time Robot Setup Guide:
https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/

Excellent one-hour conceptual/practical introduction:

Robotics Back-End - ROS2 Nav2 Navigation Stack in 1 Hour
https://www.youtube.com/watch?v=idQb2pB-h2Q

The video uses an older ROS release, so use it to understand the workflow and use current Jazzy docs for exact commands.

## I2. Learn these Nav2 concepts

- global planner
- local controller
- global costmap
- local costmap
- static layer
- obstacle/voxel layers
- inflation layer
- robot footprint
- behavior trees
- recovery behaviors
- goal poses
- velocity limits
- controller tuning

## I3. Ultrasonic integration

Publish ultrasonic readings as `sensor_msgs/Range`.

Nav2 has a Range Sensor costmap layer specifically for sonar, IR and other 1-D range sensors:
https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/costmap_2d/costmap_plugins/range/

## I4. Depth data for obstacle avoidance

The iPhone depth/point cloud can be fed to an appropriate obstacle or voxel representation after filtering and frame conversion.

Nav2 sensor setup guide:
https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/sensors/setup_sensors_gz/

Nav2 Voxel Layer:
https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/costmap_2d/costmap_plugins/voxel/

## I5. Collision monitor

For a physical student robot, add an independent high-level stop/slow layer in addition to ordinary planning.

Nav2 Collision Monitor tutorial:
https://docs.nav2.org/jazzy/tutorials/general_tutorials/using_collision_monitor/using_collision_monitor/

Range/sonar/IR inputs can be used by Collision Monitor.

### I deliverable

The robot should:

1. start at a known pose
2. accept a goal in RViz
3. plan a path
4. drive toward it
5. avoid a chair/box added after mapping
6. stop rather than collide if a close obstacle appears

---

# PHASE J - Cleaning Coverage Planning

Nav2 answers:

"How do I get from the current pose to a goal?"

A vacuum also needs to answer:

"Which goals should I visit so I cover the entire cleanable floor?"

That is coverage-path planning.

## J1. First simple algorithm

Do not begin with an advanced research planner.

Start with a boustrophedon/lawnmower pattern:

```text
>>>>>>>>>>>>>>>>>>>>
                    v
<<<<<<<<<<<<<<<<<<<<
v
>>>>>>>>>>>>>>>>>>>>
                    v
<<<<<<<<<<<<<<<<<<<<
```

Pipeline:

1. obtain a 2D occupancy grid
2. shrink free space by robot radius / safety margin
3. choose a sweep direction
4. generate parallel lanes
5. split lanes where obstacles interrupt them
6. convert lane endpoints into Nav2 poses
7. execute them sequentially
8. track which cells/lanes were completed

## J2. Learn

- occupancy-grid indexing
- world coordinates vs grid coordinates
- morphological inflation/erosion conceptually
- connected free space
- waypoint generation
- path ordering
- coverage percentage

## J3. Cleaning state machine

Implement a simple state machine:

```text
IDLE
  -> MAPPING
  -> READY
  -> NAVIGATING_TO_CLEAN_AREA
  -> CLEANING
  -> OBSTACLE_RECOVERY
  -> LOW_BATTERY
  -> FINISHED
  -> ERROR
```

### J deliverable

Display a coverage path in RViz and have the robot drive parallel cleaning lanes in a simple rectangular room.

---

# PHASE K - Cleaning Mechanism

The cleaning hardware does not need to match a commercial vacuum.

For a university proof-of-concept, demonstrate:

- airflow/suction
- debris collection
- brush agitation or sweeping
- autonomous on/off control

## K1. Control interface

Recommended commands:

```text
/cleaning/vacuum_enable
/cleaning/brush_enable
```

Later add:

```text
/cleaning/suction_level
/cleaning/brush_speed
```

## K2. Electrical rules

- Never power the vacuum motor from the Pico.
- Use a properly rated MOSFET/driver.
- Use a common reference ground where the circuit requires it.
- Add a fuse.
- Keep logic supply stable when the blower starts.
- Expect large current spikes from motors.
- Verify motor/driver temperatures during testing.

### K deliverable

ROS command turns the cleaning mechanism on, the robot follows a short path, then ROS turns the mechanism off.

---

# PHASE L - Testing and Engineering Evidence

This is what turns a cool demo into a strong university project.

## L1. Learn debugging tools

Use:

- `ros2 topic list`
- `ros2 topic echo`
- `ros2 topic hz`
- `ros2 node info`
- `ros2 param list`
- `rqt_graph`
- RViz2
- TF tree inspection
- rosbag2
- Python logging
- Git commits/issues

## L2. Measure performance

Record at least:

- mapping repeatability
- pose drift before/after loop closure
- obstacle detection distance
- navigation success rate
- collision count
- coverage percentage
- cleaning time per square metre
- battery runtime
- CPU usage on laptop
- iPhone stream rate
- command/communication latency

## L3. Failure tests

Deliberately test:

- disconnect Pico USB
- stop iPhone stream
- put obstacle directly in front
- lose Wi-Fi if Wi-Fi is used
- battery voltage sag
- IMU unplugged
- ultrasonic invalid reading
- ROS node crash

The wheels should stop safely when command communication fails.

---

# 7. Optional ros2_control Learning

`ros2_control` is important in professional ROS robots, but it is not required for our first simple serial-Pico prototype.

Study it after basic motion works.

Documentation:
https://control.ros.org/jazzy/

Differential drive controller:
https://control.ros.org/jazzy/doc/ros2_controllers/diff_drive_controller/doc/userdoc.html

Important limitation for our current hardware: the standard differential-drive controller can calculate odometry from hardware feedback, but our wheels currently do not provide encoder feedback. Do not force this architecture until there is a useful feedback signal.

---

# 8. Recommended Repository Structure Once Coding Starts

```text
autonomous-cleaning-robot/
|
|-- README.md
|-- docs/
|   |-- architecture.md
|   |-- wiring.md
|   |-- calibration.md
|   |-- testing.md
|   `-- report-notes.md
|
|-- firmware/
|   `-- pico/
|
|-- ros2_ws/
|   `-- src/
|       |-- cleanbot_bringup/
|       |-- cleanbot_description/
|       |-- cleanbot_hardware_bridge/
|       |-- cleanbot_sensors/
|       |-- cleanbot_navigation/
|       |-- cleanbot_coverage/
|       `-- cleanbot_cleaning/
|
|-- iphone/
|   `-- lidar_bridge_app/
|
|-- simulation/
|   |-- worlds/
|   `-- models/
|
|-- config/
|   |-- nav2/
|   |-- rtabmap/
|   `-- robot_localization/
|
|-- bags/
|   `-- README.md
|
|-- scripts/
|-- tests/
`-- media/
```

Do not commit huge rosbag files directly to the repository. Keep only small test data or use an external storage/release mechanism.

---

# 9. Minimum Viable Project Milestones

## Milestone 1 - ROS basics complete

- ROS 2 installed.
- Publisher/subscriber written.
- Workspace and launch files understood.

## Milestone 2 - Simulated base

- URDF works.
- TF works.
- Robot moves in Gazebo from `cmd_vel`.

## Milestone 3 - Real base teleoperation

- Pico drives both motors.
- ROS keyboard teleop drives real robot.
- Communication watchdog works.

## Milestone 4 - Real sensors

- MPU6050 visible in ROS.
- Ultrasonic visible as Range messages.
- IR array visible.

## Milestone 5 - iPhone bridge

- RGB visible in ROS.
- Depth visible in ROS.
- ARKit pose visible in ROS/TF.

## Milestone 6 - SLAM

- RTAB-Map generates a usable indoor map.
- Loop closure works.

## Milestone 7 - Autonomous navigation

- Nav2 accepts goals.
- Robot reaches goals.
- Dynamic obstacle avoidance works.

## Milestone 8 - Autonomous cleaning

- Coverage path generated.
- Vacuum/brush controlled by ROS.
- Robot covers a simple room automatically.

At this point the core university project is complete.

---

# 10. Suggested Learning Schedule

This is only an ordering guide; adapt it to the semester.

| Week | Main target |
|---|---|
| 1 | Linux + Git + Python basics |
| 2 | ROS 2 nodes/topics/services/actions/packages |
| 3 | TF + URDF + RViz + simulated differential drive |
| 4 | Gazebo + teleop + basic Nav2 simulation |
| 5 | Pico motors + ultrasonic + MPU6050 + IR |
| 6 | ROS-to-Pico serial bridge + real teleoperation |
| 7 | iPhone ARKit RGB-D + pose bridge |
| 8 | RTAB-Map and rosbag testing |
| 9 | Nav2 on real robot + obstacle layers |
| 10 | Coverage planner + cleaning hardware |
| 11 | Reliability, tuning and measurements |
| 12 | Novelty feature + final demo/report |

If time is short, prioritize milestones 1 through 8 before adding anything called AI.

---

# 11. Compact Resource List - Watch/Read in This Order

## Essential videos

1. Linux for Robotics - The Construct
   https://www.youtube.com/watch?v=ZD1WHWE0504

2. Python for Robotics - The Construct
   https://www.youtube.com/watch?v=VeFnm7bLhsM

3. ROS 2 Jazzy Basics - The Construct
   https://www.youtube.com/watch?v=DnWu6iVcVRU

4. ROS 2 Topics - The Construct
   https://www.youtube.com/watch?v=Yn2nbiPNMkE

5. ROS 2 Hands-On Robot Project - The Construct
   https://www.youtube.com/watch?v=-roYHqI_Vc4

6. Building a Mobile Robot - Articulated Robotics playlist
   https://youtube.com/playlist?list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT

7. Raspberry Pi Pico Course - Core Electronics
   https://www.youtube.com/watch?v=Ic4ExTusoTw

8. MPU6050 with Raspberry Pi Pico
   https://www.youtube.com/watch?v=HezXoT12E40

9. micro-ROS with Raspberry Pi Pico - Articulated Robotics
   https://www.youtube.com/watch?v=MBKAZ_2P1Sk

10. Nav2 Crash Course - Robotics Back-End
    https://www.youtube.com/watch?v=idQb2pB-h2Q

## Essential documentation

ROS 2 Jazzy installation:
https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html

ROS 2 beginner tutorials:
https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools.html

ROS topics/services/actions:
https://docs.ros.org/en/jazzy/How-To-Guides/Topics-Services-Actions.html

Gazebo Harmonic ROS integration:
https://gazebosim.org/docs/harmonic/ros2_integration/

Nav2:
https://docs.nav2.org/jazzy/

Nav2 first robot setup:
https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/

RTAB-Map ROS 2:
https://github.com/introlab/rtabmap_ros

Apple ARKit scene depth:
https://developer.apple.com/documentation/arkit/arframe/scenedepth

Apple ARKit point cloud example:
https://developer.apple.com/documentation/ARKit/displaying-a-point-cloud-using-scene-depth

Pico micro-ROS integration:
https://github.com/micro-ROS/micro_ros_raspberrypi_pico_sdk

Relevant iPhone-to-ROS project:
https://github.com/MatthewKazan/iPhone-lidar-slam-playground

Record3D alternative/prototyping route:
https://github.com/marek-simonik/record3d

GitHub beginner exercise:
https://github.com/skills/introduction-to-github

---

# 12. Novelty / AI Extensions - Only After the Core Robot Works

The best novelty feature is not "put ChatGPT on the robot". It should improve cleaning behavior and be measurable.

## Novelty Option A - AI Dirt Detection and Dirt Heatmap

Use the iPhone RGB camera or another inexpensive camera to detect visibly dirty regions.

Pipeline:

```text
camera image
   |
small vision model / classifier
   |
probability of dirt
   |
project detection onto map
   |
dirt heatmap
   |
coverage planner gives dirty zones extra passes
```

Possible behavior:

- normal area: one cleaning pass
- medium-dirt area: slower pass
- high-dirt area: two or three passes with stronger suction

Why this is a strong novelty feature:

- it has a clear AI component
- it affects actual robot decisions
- it is easy to explain in a final presentation
- it can be quantitatively evaluated against fixed-pattern cleaning

Measure precision/recall of dirty-area detection and whether adaptive cleaning improves debris pickup or reduces unnecessary coverage time.

## Novelty Option B - Semantic Map

Use object detection to label objects/regions such as:

- desk
- chair
- door
- sofa
- trash bin

Attach semantic information to the geometric map.

Then the robot can understand commands like:

```text
Clean around the desk.
Avoid the cable area.
Do another pass near the dining table.
```

The important engineering rule: a language model should never directly output raw motor PWM. It should select high-level, validated robot actions or navigation goals. Nav2 and the Pico remain responsible for safe movement.

## Novelty Option C - Secondary Supervisor Agent

Add a high-level software agent on the laptop that observes:

- battery voltage
- cleaning coverage
- sensor health
- repeated navigation failures
- vacuum/brush status
- map position

The agent can choose among safe predefined actions:

```text
START_CLEANING(zone)
PAUSE
RETURN_TO_START
RETRY_ZONE(zone)
REDUCE_SPEED
REQUEST_HUMAN_HELP(reason)
CREATE_MAINTENANCE_REPORT
```

This is much more defensible than giving an LLM unrestricted robot control.

Example:

```text
Human: Clean the room but spend extra time near the desks.

Agent:
1. identifies desk-labelled regions
2. generates priority cleaning zones
3. sends waypoints to the coverage/Nav2 layer
4. monitors progress
5. reports completion and missed areas
```

## Novelty Option D - Learning from Previous Cleaning Runs

Store statistics for each map cell/zone:

- how often dirt is detected
- how often navigation fails
- average time required
- battery cost

Future cleaning runs can prioritize historically dirty areas and reduce unnecessary repeated passes in consistently clean areas.

This can be implemented first with simple statistics. Machine learning can be added later if the dataset becomes large enough.

## Novelty Option E - Automatic Cleaning Report

After a run, automatically generate:

- map image
- path taken
- area covered
- missed zones
- cleaning duration
- battery used
- obstacle events
- dirt hotspots
- errors/recoveries

This is comparatively easy to build and makes the final demonstration look polished.

## Recommended novelty choice

For this project, the strongest final combination would be:

1. Core autonomous cleaning robot.
2. Dirt-detection heatmap.
3. Adaptive extra cleaning passes in dirty regions.
4. Optional natural-language supervisor that converts commands into safe predefined cleaning tasks.

That gives the project a meaningful AI component without making the entire robot depend on an unreliable AI model.

---

# 13. Final Target Demo

A strong final university demonstration would look like this:

1. Place robot at the start position.
2. Open RViz on laptop.
3. iPhone streams RGB-D and ARKit pose.
4. Robot displays/localizes in the mapped room.
5. User selects "Clean Room".
6. Robot generates a coverage plan.
7. Vacuum and brush turn on.
8. Robot follows systematic cleaning lanes.
9. iPhone depth + ultrasonic sensors detect obstacles.
10. Nav2 routes around obstacles.
11. Robot stops safely for very close hazards.
12. Optional AI detects a dirty region and schedules another pass.
13. Robot completes the region and turns cleaning motors off.
14. Laptop displays a cleaning report and coverage map.

That is already a substantial robotics project: embedded control, Linux, ROS 2, RGB-D sensing, VIO, SLAM, navigation, planning, hardware integration and an optional AI layer.

---

# 14. Rule for the Team

Build one vertical slice at a time.

Bad approach:

```text
Buy/assemble everything -> write all nodes -> start SLAM -> debug 15 failures at once
```

Good approach:

```text
ROS basics
-> simulated drive
-> real teleop
-> sensors
-> iPhone stream
-> mapping
-> navigation
-> coverage
-> cleaning
-> novelty
```

Every phase should have a visible demonstration before moving to the next phase.
