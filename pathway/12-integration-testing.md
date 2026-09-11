# Stage 12 — Integration, Testing & Evaluation

## Goal

Turn all individually working pieces into one repeatable autonomous-cleaning system and generate the evidence needed for a strong university demonstration/report.

At the end of this stage:

- one launch procedure starts the complete system;
- startup and shutdown happen safely;
- all important parameters are frozen/documented;
- localization, navigation, obstacle avoidance and cleaning are tested together;
- failure cases are intentionally tested;
- performance metrics are recorded;
- another team member can reproduce the final demo from the repository.

A project is not finished because it worked once. It is finished when the same procedure works repeatedly and failures are understood.

---

# 1. Freeze the Final Architecture

Before final integration, write down exactly which component owns each responsibility.

Recommended final ownership:

```text
Laptop / ROS 2
--------------
map_server              -> static room map
home localization       -> map -> odom initialization
robot_localization EKF  -> odom -> base_link
cleanbot_bridge         -> ROS <-> Pico serial conversion
Nav2                    -> path planning + path following
coverage manager        -> cleaning lanes / mission sequence
coverage tracker        -> coverage metrics
robot_state_publisher   -> base_link -> sensor/wheel transforms

Pico
----
encoder counting
wheel velocity PID
motor PWM + direction
MPU6050 acquisition
ultrasonic acquisition
IR acquisition
battery ADC if used
vacuum/brush switching
serial protocol
command watchdog
hard close-obstacle stop if implemented
```

Make sure only one node publishes each important TF.

Especially:

```text
map -> odom       one provider only
odom -> base_link one provider only
```

---

# 2. Create a Single Bringup Package

Recommended:

```text
src/cleanbot_bringup/
├── package.xml
├── launch/
│   ├── robot.launch.py
│   ├── navigation.launch.py
│   └── full_system.launch.py
├── config/
│   ├── robot.yaml
│   ├── ekf.yaml
│   ├── nav2.yaml
│   └── cleaning.yaml
└── README.md
```

`full_system.launch.py` should eventually start or include:

```text
robot_state_publisher
cleanbot_bridge
EKF
static map server
home/map->odom initializer
Nav2
coverage/cleaning manager
RViz optionally
```

Do not hide every subsystem inside one huge Python launch file. Use smaller launch files and include them.

ROS launch documentation:

- https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Launch-Main.html

---

# 3. Centralize Parameters

Avoid different copies of the same value in five packages.

Create a canonical robot configuration containing measured values such as:

```yaml
wheel_radius_m: 0.0328
wheel_separation_m: 0.163
encoder_counts_per_rev: 40
max_linear_speed_mps: 0.15
max_angular_speed_rps: 0.55
command_timeout_s: 0.50
cleaning_lane_spacing_m: 0.19
home_x: 0.50
home_y: 0.50
home_yaw: 0.0
```

Those numbers are examples only.

The final values must come from your calibration experiments.

Maintain a simple parameter record:

```text
hardware/calibration/final_parameters.md
```

For each important value record:

```text
value
units
how it was measured
measurement date
which stage/test produced it
```

---

# 4. Define the Final Startup Procedure

The project should have one documented sequence.

Recommended:

```text
1. Inspect robot for loose wires / mechanical obstruction.
2. Place robot in the physical home fixture/mark.
3. Switch robot power on.
4. Keep robot stationary while MPU6050 bias calibrates.
5. Connect/check Pico USB.
6. Reset encoder counters / odometry state.
7. Start ROS 2 bringup.
8. Confirm Pico bridge connected.
9. Confirm /wheel/odom and /imu/data are updating.
10. Confirm EKF /odometry/filtered is healthy.
11. Load room map.
12. Initialize map -> odom from known home pose.
13. Verify TF and robot pose in RViz.
14. Verify ultrasonic ranges.
15. Verify battery status if available.
16. Arm/enable autonomous motion.
17. Select cleaning zone.
18. Start cleaning mission.
```

Do not allow the robot to begin driving immediately when ROS starts.

Use an explicit **armed/ready** state.

---

# 5. Define the Shutdown Procedure

Normal shutdown:

```text
1. Stop cleaning mission.
2. Command zero velocity.
3. Switch vacuum and brush off.
4. Confirm wheels stopped.
5. Stop ROS nodes.
6. Disconnect USB if needed.
7. Switch robot power off.
8. Charge battery using correct charger/procedure.
```

Emergency shutdown must be simpler:

```text
physical main/E-stop -> motor power removed / safe state
```

Do not rely on closing a terminal window as the only emergency stop.

---

# 6. Add Health Checks

At minimum, the system should know whether these are alive:

```text
Pico serial link
encoder telemetry
IMU telemetry
ultrasonic telemetry
EKF output
map topic
Nav2 action server
battery voltage if measured
```

A lightweight health node can periodically check topic age.

Example logic:

```text
if /imu/data older than 0.5 s -> NOT READY
if Pico telemetry older than 0.5 s -> FAULT
if /odometry/filtered missing -> NOT READY
if map unavailable -> NOT READY
```

The exact thresholds depend on topic rates.

The mission manager should refuse to start if critical dependencies are unhealthy.

---

# 7. Logging Strategy

Use three kinds of logs.

### ROS console logs

For human-readable events:

```text
mission started
serial disconnected
waypoint failed
returning home
vacuum enabled
```

### rosbag2

Record time-series data:

```text
/cmd_vel
/wheel/odom
/imu/data
/odometry/filtered
/range/*
/map
/cleanbot/cleaning_state
/cleanbot/coverage_map
/battery topic
```

Do not record enormous unnecessary streams in every run.

### CSV experiment files

Use concise summary metrics so results are easy to compare.

Recommended folders:

```text
experiments/
├── odometry/
├── localization/
├── navigation/
├── coverage/
├── cleaning/
└── final_runs/
```

---

# 8. Version Your Test Configuration

Every important run should be reproducible.

Record:

```text
Git commit SHA
map version
Nav2 config version
Pico firmware version
battery state
floor type
robot hardware revision
```

At major milestones create Git tags, for example:

```text
v0.1-simulation
v0.2-drive-base
v0.3-odometry
v0.4-static-map
v0.5-navigation
v0.6-coverage
v1.0-demo
```

Git tagging reference:

- https://git-scm.com/book/en/v2/Git-Basics-Tagging

---

# 9. Test Matrix

Do not jump straight to a full cleaning run after integration.

Use this order.

## Test Group A — Boot / readiness

Test at least several times:

```text
cold boot
USB already connected
USB connected after robot power
ROS restarted while Pico stays powered
Pico restarted while ROS stays running
```

Expected result: system returns to a safe ready/not-ready state without unexpected wheel motion.

---

## Test Group B — Manual driving

With vacuum off:

```text
forward
reverse
left turn
right turn
spin
stop
```

Verify:

- correct wheel direction;
- smooth PID;
- no serial dropouts;
- odometry direction is correct;
- ultrasonic values remain stable while motors run.

---

## Test Group C — Localization repeatability

From home:

```text
straight path
square path
representative cleaning path
return toward home
```

Compare actual and estimated pose.

If integration has made drift worse than Stage 9, stop and investigate.

---

## Test Group D — Navigation

Use several predefined goals:

```text
G1 open straight
G2 requires turn
G3 near wall with safe clearance
G4 opposite side of demo zone
G5 return-home approach
```

Run each multiple times.

Record success/failure and final error.

---

## Test Group E — Dynamic obstacles

Use large safe obstacles.

Test:

```text
obstacle present before mission
obstacle placed after motion begins
obstacle removed after stop
blocked waypoint
```

Expected:

```text
no collision
bounded recovery/retry
mission either continues or fails safely
```

---

## Test Group F — Cleaning

Test:

```text
vacuum only
brush only
vacuum + brush
cleaning while stationary
cleaning while moving
full lane coverage
```

Watch for electrical noise/resets when cleaning loads start.

---

# 10. Fault-Injection Testing

This is one of the strongest things you can show in a robotics project.

Intentionally test failures under controlled conditions.

## Fault 1 — Unplug Pico USB

Robot wheels must stop because of Pico watchdog.

Expected:

```text
< 1 s safe stop, depending on configured timeout
ROS reports connection loss
```

## Fault 2 — Kill the ROS bridge process

Again, Pico watchdog must stop motion.

## Fault 3 — Kill Nav2

Bridge/Pico should not keep an old nonzero command indefinitely.

## Fault 4 — Block a wheel gently during a bench test

Observe:

```text
encoder speed drops
PID output rises
```

Do not hold a stalled motor long enough to overheat it.

An advanced version can detect a persistent commanded-speed vs measured-speed mismatch and report a motor stall.

## Fault 5 — Disconnect MPU6050

Mission should not silently pretend localization is normal.

## Fault 6 — Invalid ultrasonic data

One invalid reading should not crash the system or create permanent false obstacles.

## Fault 7 — Put a large obstacle very close ahead

Hard safety stop/local costmap should prevent forward motion.

## Fault 8 — Low battery / voltage sag

If battery monitoring exists, test the low-voltage policy using a controlled safe approach rather than intentionally deeply discharging lithium cells.

---

# 11. Measure Final Performance

Choose metrics that match the claims in your report.

## Mapping

```text
map dimension error (%)
wall/obstacle alignment error
resolution
```

## Odometry/localization

```text
straight distance error (%)
360-degree yaw error
square-loop closure error
position drift after representative mission
```

## Navigation

```text
goal success rate (%)
average final position error
average mission time
collision count
recovery count
```

## Coverage

```text
coverage percentage
overlap percentage
path length
area cleaned per minute
missed cells/waypoints
```

## Cleaning

```text
debris pickup percentage
best cleaning speed
best lane spacing
```

## Reliability

```text
successful full runs / attempted runs
serial packet drop rate
unexpected resets
watchdog-stop success rate
```

## Power

If available:

```text
battery voltage before/after
runtime
approximate energy/current measurements if instrumented
```

---

# 12. Suggested Final Success Targets

Set final targets after early experiments, but a realistic controlled-demo target might be:

```text
Full mission success:       >= 8/10 repeated trials
Coverage:                   >= 85–90% of selected reachable zone
Collision count:            0
Watchdog failure test:      100% safe stops
Final localization error:   within clearance budget for the selected room
Cleaning pickup:            quantified and repeatable
```

Do not use these numbers blindly. If your room geometry requires tighter localization, use a tighter threshold.

It is better to report an honest 86% measured coverage rate than claim “100% autonomous coverage” without evidence.

---

# 13. Build a Final Demo Scenario

Keep the final demo controlled and meaningful.

Recommended:

```text
1. Show saved map in RViz.
2. Point out that iPhone/LiDAR is not on the robot.
3. Place robot at marked home.
4. Launch system and show healthy sensor topics.
5. Start a selected cleaning zone.
6. Robot executes parallel cleaning lanes.
7. Place one large temporary obstacle.
8. Robot stops/replans safely.
9. Robot continues coverage.
10. Robot returns toward home.
11. Show final coverage map/percentage.
12. Show before/after cleaning result.
```

This demonstrates the project's real contributions clearly.

Do not improvise a brand-new room layout on presentation day.

---

# 14. Prepare a Demo Checklist

Print or keep a simple checklist:

```text
[ ] battery charged
[ ] wheels/encoders secure
[ ] vacuum chamber empty
[ ] home marker intact
[ ] USB cable secured
[ ] correct map selected
[ ] correct Git/config version
[ ] Pico connected
[ ] IMU calibrated
[ ] odometry reset
[ ] TF correct
[ ] ultrasonic okay
[ ] emergency stop accessible
[ ] test obstacle ready
[ ] debris test prepared
```

This is boring and extremely useful.

---

# 15. Final Documentation Structure

By project completion, repository should contain:

```text
README.md
pathway/
firmware/
src/
config/
maps/
hardware/
simulation/
experiments/
```

Add a final `docs/` folder later with:

```text
docs/
├── architecture.md
├── wiring.md
├── calibration.md
├── operating-procedure.md
├── results.md
└── troubleshooting.md
```

The pathway explains how you built it; `docs/` explains the finished system.

---

# 16. Troubleshooting Method

When the integrated robot fails, do not change five parameters at once.

Ask which layer failed:

```text
Power?
Pico firmware?
Encoder/PID?
Serial bridge?
Sensor topic?
TF?
EKF?
Static map?
Nav2 costmap?
Nav2 controller?
Coverage manager?
Cleaning hardware?
```

Then isolate it.

Useful commands:

```bash
ros2 node list
ros2 topic list
ros2 topic hz /topic
ros2 topic echo /topic
ros2 topic info /topic --verbose
ros2 run tf2_tools view_frames
ros2 run tf2_ros tf2_echo map base_link
rqt_graph
```

Use saved rosbags to reproduce software failures without moving the real robot whenever possible.

---

# 17. Report / Presentation Story

A strong technical narrative is:

```text
Problem:
Commercial autonomous vacuums commonly rely on onboard perception hardware.

Constraint:
Build a low-cost academic prototype without carrying an expensive LiDAR or iPhone.

Approach:
Use iPhone LiDAR once for metric room mapping, then remove it.
Use low-cost encoders + IMU for runtime dead reckoning.
Use ultrasonic/IR for local safety.
Use ROS 2/Nav2 for navigation and deterministic coverage planning.

Engineering challenge:
Localization drift after the global mapping sensor is removed.

Solution:
Careful wheel calibration, encoder PID, IMU fusion, known home pose,
short controlled missions and periodic home/landmark re-zeroing.

Evaluation:
Measure drift, navigation success, coverage, cleaning effectiveness and fault safety.
```

That limitation is not something to hide. It is one of the most interesting engineering parts of the project.

---

## Stage Acceptance Checklist

- [ ] One documented full-system launch exists.
- [ ] System does not move automatically on startup.
- [ ] Final calibrated parameters are recorded centrally.
- [ ] Startup procedure works repeatedly.
- [ ] Shutdown/emergency stop procedure is tested.
- [ ] Health checks detect critical missing components.
- [ ] Full ROS bag can be recorded for final missions.
- [ ] USB/ROS failure triggers safe Pico watchdog stop.
- [ ] Dynamic obstacle tests pass without collision.
- [ ] Repeated complete cleaning missions are measured.
- [ ] Coverage percentage is reported.
- [ ] Cleaning effectiveness is reported.
- [ ] Localization drift is reported.
- [ ] Final demo procedure is rehearsed.
- [ ] Repository contains enough information for another team member to reproduce the demo.

---

## Best References

- ROS 2 launch: https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Launch-Main.html
- rosbag2: https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data.html
- ROS 2 logging: https://docs.ros.org/en/jazzy/Tutorials/Demos/Logging-and-logger-configuration.html
- Nav2 tuning: https://docs.nav2.org/jazzy/tuning/index.html
- TF2 debugging/tutorials: https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Tf2/Tf2-Main.html
- Git tags: https://git-scm.com/book/en/v2/Git-Basics-Tagging

**Next:** [Stage 13 — Novelty / AI Extensions](./13-novelty-ai.md)
