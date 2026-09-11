# Stage 03 — Robot Model, TF & Simulation

## Goal

Build the robot in software before wiring the real robot.

At the end of this stage:

- the robot has a URDF/Xacro description;
- all coordinate frames are correct;
- the model appears in RViz2;
- it spawns in Gazebo Harmonic;
- `/cmd_vel` can drive it as a differential-drive robot;
- odometry is visible;
- you understand the `map -> odom -> base_link` frame chain.

This stage removes software uncertainty before hardware debugging begins.

---

## 1. Learn Differential-Drive Kinematics

Our robot has two powered wheels separated by distance `L`.

Let:

- `v` = robot forward velocity in m/s;
- `omega` = robot angular velocity in rad/s;
- `v_l` = left wheel linear velocity;
- `v_r` = right wheel linear velocity.

Then:

```text
v_l = v - omega * L/2
v_r = v + omega * L/2

v     = (v_r + v_l)/2
omega = (v_r - v_l)/L
```

If wheel radius is `r`:

```text
wheel angular velocity = wheel linear velocity / r
```

This relationship is the bridge between ROS robot commands and the individual wheel motors.

### Good reference

- ROS 2 Control wheeled-mobile-robot kinematics: https://control.ros.org/jazzy/doc/ros2_controllers/doc/mobile_robot_kinematics.html
- `diff_drive_controller`: https://control.ros.org/jazzy/doc/ros2_controllers/diff_drive_controller/doc/userdoc.html

Do not memorize the equations only — draw the robot and understand why different wheel speeds cause rotation.

---

## 2. Understand ROS Coordinate Frames

The minimum frame tree for navigation is conceptually:

```text
map
 |
odom
 |
base_link
 |-- base_footprint
 |-- left_wheel_link
 |-- right_wheel_link
 |-- imu_link
 |-- ultrasonic_front_link
 |-- ir_array_link
```

### Meaning

**`map`**

Long-term/global coordinate system of the saved room map.

**`odom`**

Continuous local coordinate system from wheel/IMU odometry. It can drift.

**`base_link`**

Robot body frame.

**Sensor frames**

Describe exactly where sensors are relative to the body.

### Required Nav2 concept

Nav2 expects a connected transform tree containing:

```text
map -> odom -> base_link -> sensors
```

Official Nav2 transformation guide:

- https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/transformation/setup_transforms.html

ROS coordinate conventions:

- REP 103: https://www.ros.org/reps/rep-0103.html
- REP 105: https://www.ros.org/reps/rep-0105.html

Use SI units: metres, radians, seconds.

---

## 3. Learn URDF and Xacro

URDF describes links and joints.

Start with simple geometry. Do not spend days designing a perfect visual model.

Create package:

```bash
cd ~/cleanbot_ws/src
ros2 pkg create --build-type ament_cmake cleanbot_description
```

Recommended structure:

```text
cleanbot_description/
├── CMakeLists.txt
├── package.xml
├── urdf/
│   ├── cleanbot.urdf.xacro
│   ├── materials.xacro
│   └── gazebo.xacro
├── launch/
│   └── display.launch.py
└── rviz/
    └── model.rviz
```

### Initial model

Use:

- one box/cylinder for chassis;
- two cylinders for drive wheels;
- one caster approximation;
- small boxes for sensors.

Parameterize important physical dimensions:

```xml
<xacro:property name="wheel_radius" value="0.033"/>
<xacro:property name="wheel_width" value="0.020"/>
<xacro:property name="wheel_separation" value="0.160"/>
```

These are placeholders. Replace them with measured dimensions later.

### Learn these URDF elements

```text
<link>
<joint>
<visual>
<collision>
<inertial>
<origin>
<geometry>
```

Joint types needed:

```text
continuous -> drive wheels
fixed      -> sensors
```

### Resources

- ROS URDF tutorials: https://docs.ros.org/en/jazzy/Tutorials/Intermediate/URDF/URDF-Main.html
- Xacro tutorial: https://docs.ros.org/en/jazzy/Tutorials/Intermediate/URDF/Using-Xacro-to-Clean-Up-a-URDF-File.html
- Articulated Robotics mobile robot playlist: https://www.youtube.com/playlist?list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT

The Articulated Robotics series may use older ROS/Gazebo commands in some episodes. Use it for the explanation and use Jazzy/Harmonic docs for current syntax.

---

## 4. Display the Robot in RViz

Install/use `robot_state_publisher` and `joint_state_publisher` as appropriate.

Your launch file should:

1. process the Xacro file;
2. publish `robot_description`;
3. start `robot_state_publisher`;
4. launch RViz.

Useful commands:

```bash
ros2 topic echo /robot_description --once
ros2 topic list
ros2 run tf2_tools view_frames
```

Open the resulting TF graph PDF or inspect in RViz.

### RViz displays to add

- RobotModel;
- TF;
- Grid;
- Odometry later.

### Checks

- wheel cylinders rotate around the correct axis;
- sensors are not inside the floor;
- `x` is forward;
- `z` is up;
- left/right are not swapped;
- dimensions look physically plausible.

---

## 5. Inertial and Collision Properties

Gazebo needs more than visual shapes.

Each moving body needs sensible:

- collision geometry;
- mass;
- inertia.

For a first model, approximate the chassis as a box and wheels as cylinders.

Bad inertia values cause:

- robot exploding/jittering;
- wheels sinking;
- unrealistic acceleration;
- unstable contacts.

Do not use zero mass or zero inertia.

A useful learning reference is the Gazebo/URDF part of the Articulated Robotics mobile robot series.

---

## 6. Gazebo Harmonic

Use modern Gazebo Harmonic with ROS 2 Jazzy.

Resources:

- Gazebo Harmonic docs: https://gazebosim.org/docs/harmonic/getstarted/
- ROS 2 integration: https://gazebosim.org/docs/harmonic/ros2_integration/
- ROS/Gazebo version pairing: https://gazebosim.org/docs/harmonic/ros_installation/

Install using the supported ROS/Jazzy instructions rather than old `gazebo11` / Gazebo Classic tutorials.

---

## 7. Simulate Differential Drive

There are two reasonable routes.

### Route A — Simple Gazebo DiffDrive system

Best for learning and first simulation.

Configure the Gazebo differential-drive system with:

- left wheel joint;
- right wheel joint;
- wheel separation;
- wheel radius;
- command topic;
- odometry topic/frame.

Then bridge required Gazebo/ROS topics with `ros_gz_bridge` if needed.

### Route B — ros2_control

More realistic/professional, but more setup.

Use later if the team wants a unified simulated/physical hardware interface.

For this university project, **Route A is enough initially**. The real Pico will use our own USB bridge later.

---

## 8. Test `/cmd_vel`

Install a teleoperation package if needed:

```bash
sudo apt install ros-jazzy-teleop-twist-keyboard
```

Run:

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Check the command topic:

```bash
ros2 topic echo /cmd_vel
```

If your simulated drive plugin expects another topic, remap it rather than changing every upstream node.

Test:

```text
forward
reverse
turn left
turn right
spin in place
stop
```

Use low speeds.

---

## 9. Verify Odometry

The simulated drivetrain should publish odometry.

Inspect:

```bash
ros2 topic list | grep odom
ros2 topic echo /odom
ros2 topic hz /odom
```

Add Odometry display in RViz.

Understand that this simulation odometry is easier than real hardware. Stage 5 will make the real robot produce similar data from encoder counts.

---

## 10. Build a Simple Test World

Create a Gazebo world with:

- floor;
- four walls;
- one box obstacle;
- one narrow gap wider than the robot;
- one open area for turning.

Do not recreate the real room yet. The goal is to debug motion and geometry.

Recommended structure:

```text
simulation/
├── worlds/
│   └── test_room.sdf
└── launch/
    └── sim.launch.py
```

---

## 11. Measure the Real Robot Early

Even before full assembly, collect these dimensions and store them in a config/hardware note:

```text
wheel diameter / radius
wheel separation (center-to-center)
chassis width
chassis length
chassis height
sensor positions
caster location
```

Later navigation accuracy depends on these numbers.

---

## 12. Common Problems

### Robot turns when commanded straight

Likely wheel joints/orientations are wrong or the left/right joints are swapped.

### Wheels spin but robot does not move

Check collision geometry, friction, wheel contact and drive plugin joint names.

### RViz says “No transform”

Inspect TF. Do not randomly change Fixed Frame until the tree is correct.

### Robot jumps/explodes in Gazebo

Check masses/inertia and overlapping collision geometry.

### Commands exist but Gazebo receives nothing

Check topic names and `ros_gz_bridge` configuration.

---

## Stage Deliverable

Commit:

```text
src/cleanbot_description/
simulation/
```

with a launch command that brings up the robot in Gazebo and RViz.

Record the command in the repo documentation.

---

## Stage Acceptance Checklist

- [ ] Robot appears correctly in RViz.
- [ ] TF tree is valid and understandable.
- [ ] `base_link` has correct orientation.
- [ ] Wheel/sensor frames are positioned correctly.
- [ ] Robot spawns stably in Gazebo Harmonic.
- [ ] `/cmd_vel` drives it forward/reverse/turn/spin.
- [ ] `/odom` changes appropriately.
- [ ] You understand `map`, `odom`, `base_link` and sensor frames.
- [ ] Real wheel radius and wheel separation have been measured or a task exists to measure them.

---

## Best References

- Articulated Robotics — Building a Mobile Robot: https://www.youtube.com/playlist?list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT
- ROS Jazzy URDF tutorials: https://docs.ros.org/en/jazzy/Tutorials/Intermediate/URDF/URDF-Main.html
- TF2 tutorials: https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Tf2/Tf2-Main.html
- Nav2 transform setup: https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/transformation/setup_transforms.html
- Gazebo Harmonic: https://gazebosim.org/docs/harmonic/getstarted/
- `diff_drive_controller`: https://control.ros.org/jazzy/doc/ros2_controllers/diff_drive_controller/doc/userdoc.html

**Next:** [Stage 04 — Pico, Electronics & Drive Base](./04-pico-electronics-drive-base.md)
