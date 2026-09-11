# Stage 09 — Phone-Free Localization on the Static Map

## Goal

Make the robot use the room map **without carrying the iPhone or any LiDAR**.

This is the engineering stage that decides whether the project idea is viable in the real room.

At the end we need:

- a repeatable physical home/start pose;
- encoder + MPU6050 fused odometry;
- a valid `map -> odom -> base_link` TF chain;
- a measured drift rate;
- a practical method for resetting/correcting drift;
- enough repeatability to complete one cleaning run.

If this stage fails, do not continue to Nav2 tuning. Navigation cannot compensate for a robot that does not know where it is.

---

# 1. The Important Difference: Map vs Localization

A **map** answers:

> What does the room look like?

Localization answers:

> Where is the robot in that map right now?

The iPhone solves the map problem once.

After we remove the iPhone, we still need the second problem.

Our runtime sensors are:

```text
wheel encoders
MPU6050
ultrasonic
IR array
```

Encoders + IMU can estimate motion, but that estimate slowly drifts.

---

# 2. Why We Are Not Using AMCL

Nav2's standard AMCL implementation is a 2D probabilistic localizer that uses a laser scan topic.

Its configuration includes:

```text
map
odometry
LaserScan
```

Our robot has **no runtime LiDAR**, so feeding AMCL fake or extremely sparse ultrasonic “laser scans” is not the baseline design.

AMCL docs:

- https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/others/configuring_amcl.html

Do not create a synthetic 360° scan from one ultrasonic sensor just to make AMCL start. That makes the architecture look standard while giving bad localization.

---

# 3. What Nav2 Actually Requires

Nav2 needs a valid TF chain:

```text
map -> odom -> base_link -> sensor frames
```

The normal roles are:

```text
map -> odom       global localization correction
odom -> base_link continuous local odometry
```

Official Nav2 state-estimation guide:

- https://docs.nav2.org/jazzy/getting_started/navigation_concepts/state_estimation.html

Transformation setup:

- https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/transformation/setup_transforms.html

The Nav2 docs explicitly do not require LiDAR specifically; another valid positioning system can provide `map -> odom`. Our baseline uses a known start and accepts dead-reckoning drift for a limited-duration controlled demo.

---

# 4. Baseline Runtime Localization Strategy

For the first successful system:

```text
PHYSICAL HOME POSE
        |
reset/alignment at start
        |
        v
map -> odom initialized
        |
        +-------------------+
                            |
encoders + MPU6050 --> EKF --> odom -> base_link
                            |
                            v
                        robot pose
```

The robot is physically placed at the same home pose every run.

At the beginning of a run:

1. wheels are stationary;
2. IMU bias calibration runs;
3. encoder counters/odometry are reset;
4. software knows the corresponding pose in the saved map;
5. `map` and `odom` are aligned from that known relationship.

From then on, the robot dead-reckons using encoders + IMU until the next correction/reset.

---

# 5. Make Home Pose Mechanically Repeatable

Do not rely on a human visually guessing the same start orientation every time.

Cheap methods:

### Option A — Floor tape outline

Mark:

```text
left wheel position
right wheel position
front direction
```

### Option B — Small physical docking guide

Use MDF/acrylic/foam pieces so the chassis is placed against repeatable stops.

### Option C — IR home stripe

If the 5-channel array reliably detects contrasting tape, place a known stripe across the home/dock approach.

Best result: combine A/B with C.

A 2–3° starting-heading error can become a large position error after several metres, so physical repeatability matters.

---

# 6. Choose the Map/Home Coordinate Relationship

Suppose `config/home_pose.yaml` contains:

```yaml
x: 0.50
y: 0.50
yaw: 0.0
```

This means that when the robot is physically at the home marker, its `base_link` pose in `map` is known.

There are two ways to make the runtime TF relationship simple.

## Method A — Design odometry origin to match home

At startup, reset odometry so:

```text
odom pose = 0,0,0
```

Then `map -> odom` is the transform from the map origin to the known home pose.

## Method B — Design map coordinates so home is map zero

If convenient, process the map so home is:

```text
map pose = 0,0,0
```

then at startup:

```text
map -> odom = identity
```

Method B is conceptually easiest, but only use it if map-image/origin handling remains clear.

---

# 7. First Test With Static `map -> odom`

If map and odom are aligned at startup and there is no global correction during the run, `map -> odom` can initially be fixed.

Conceptual command for an identity transform:

```bash
ros2 run tf2_ros static_transform_publisher \
  --x 0 --y 0 --z 0 \
  --yaw 0 --pitch 0 --roll 0 \
  --frame-id map \
  --child-frame-id odom
```

If home has a non-zero map pose, publish the calculated transform instead.

Check your installed Jazzy command help before copying options:

```bash
ros2 run tf2_ros static_transform_publisher --help
```

This transform being static means:

> We currently have no global sensor correcting odometry drift.

That is intentional for the baseline test.

---

# 8. `odom -> base_link` Comes From the EKF

Use Stage 6 `robot_localization` output for:

```text
odom -> base_link
```

Do not allow both the raw wheel-odometry node and EKF to publish the same transform.

Recommended ownership:

```text
wheel odom node -> /wheel/odom message only
IMU node        -> /imu/data
EKF             -> /odometry/filtered + odom->base_link
```

Inspect:

```bash
ros2 run tf2_ros tf2_echo odom base_link
ros2 topic echo /odometry/filtered
```

---

# 9. Measure Drift Before Navigation

Create a dedicated test area with tape marks.

We care about:

```text
position drift per metre
heading drift per turn
final error after a representative cleaning path
repeatability between runs
```

## Test A — 1 m straight

Run 10 times from the same start.

Record:

```text
estimated final x/y/yaw
measured final x/y/yaw
```

## Test B — 3 m straight

Longer distance reveals scale errors.

## Test C — 360° spin

Repeat clockwise and counter-clockwise.

## Test D — square

Use four equal sides and 90° turns.

## Test E — realistic 5-minute cleaning pattern

Drive parallel lanes and turns similar to the final cleaner.

At the end, compare actual robot position to ROS estimated position.

This test matters more than an impressive RViz screenshot.

---

# 10. Record a Drift Table

Create:

```text
experiments/localization/drift_results.csv
```

Suggested columns:

```text
run_id
path_type
distance_m
turns
runtime_s
estimated_x
estimated_y
estimated_yaw
actual_x
actual_y
actual_yaw
position_error_m
yaw_error_deg
battery_v
floor_type
notes
```

Plot error vs distance/time.

This gives the final report a real engineering evaluation.

---

# 11. Improve Dead-Reckoning Accuracy

Do these before adding new hardware.

## A. Recalibrate wheel radius

A scale error causes distance drift.

## B. Recalibrate effective wheel separation

A separation error causes turning drift.

## C. Slow down turns

Fast skid turns often cause wheel slip.

## D. Limit acceleration

Sudden acceleration produces slip and current spikes.

## E. Balance chassis weight

One wheel carrying much more weight can behave differently.

## F. Improve caster alignment

Caster drag can create systematic turning.

## G. Improve encoder mounting

Loose/slipping discs create unfixable odometry errors.

## H. Revisit PID

Bad wheel-speed tracking produces different left/right travel.

## I. Improve IMU mounting

Rigid mount, low vibration and correct frame orientation.

---

# 12. Use the IR Array as a Cheap Re-Zero Landmark

This is the best use of the existing IR array if it behaves reliably on the demo floor.

Place a deliberate high-contrast home strip or pattern.

When the robot returns to home:

1. approach slowly;
2. detect the known pattern;
3. stop at a repeatable physical position;
4. align chassis heading using the docking geometry;
5. reset odometry to the known home pose.

This does not make localization globally perfect everywhere, but it prevents drift from carrying over between cleaning jobs/zones.

### Zone strategy

Instead of one 30-minute run:

```text
home -> clean zone A -> home/reset
home -> clean zone B -> home/reset
home -> clean zone C -> home/reset
```

This is much more realistic for dead-reckoning-only localization.

---

# 13. Optional Wall-Based Correction

Ultrasonic can provide **limited** pose correction near known flat walls.

### With one ultrasonic sensor

You can correct distance to a wall when the robot is already known to face approximately perpendicular to it.

Example:

```text
expected wall distance = 0.20 m
measured = 0.27 m
```

This provides one geometric constraint.

### With two separated front sensors

If available, comparing two ranges can estimate whether the robot is angled relative to a wall.

Then a wall-alignment routine can:

1. approach slowly;
2. rotate until left/right wall distances match;
3. stop at a known distance;
4. use the known wall geometry to correct one position axis + heading.

### Limitations

Ultrasonic reflections are noisy. Only use wall correction:

- near large flat walls;
- at low speed;
- after repeated/filtered measurements;
- when the robot already knows which wall it is approaching.

Do not use a single ultrasonic reading to teleport the robot pose.

---

# 14. Why Ultrasonic Is Still Valuable

Even when it does not localize globally, ultrasonic can keep the robot safe when the static map is outdated.

Examples:

```text
chair moved since mapping
person walks into path
bag placed on floor
door partly closed
```

Nav2's Range Sensor Layer is designed to accept sonar/IR/1D range sensors for collision avoidance:

- https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/costmap_2d/costmap_plugins/range.html

That will be configured in Stage 10.

---

# 15. Acceptance Threshold for Phone-Free Operation

We need a project-specific decision rule.

For a small room, a reasonable first target is:

```text
position error after typical cleaning run: <= 0.15–0.25 m
heading error:                            small enough to stay in lanes
no collision caused by pose drift
robot can reliably return near home
```

The exact allowable error depends on:

- room size;
- robot width;
- obstacle clearance;
- lane spacing;
- run duration.

If your robot is 30 cm wide and a corridor has only 5 cm clearance each side, 20 cm drift is obviously unacceptable.

Use geometry to set the final threshold.

---

# 16. Decision Gate: Is the No-LiDAR Runtime Good Enough?

Proceed to Nav2 if:

- straight-line calibration is stable;
- turns are repeatable;
- square test is repeatable;
- representative path error stays within safe clearance;
- error does not vary wildly run-to-run.

Do **not** proceed if:

- heading wanders randomly;
- encoder counts are inconsistent;
- the robot ends a square far from the start;
- error grows too quickly for one cleaning zone.

---

# 17. Cheapest Fallback If Drift Is Too Large

The project's requirement is “no expensive LiDAR and no iPhone hanging on the robot.”

That does **not** mean we should force dead-reckoning if it clearly fails.

Cheapest robust fallback:

```text
cheap USB / CSI / ESP32-class camera
        +
printed AprilTag / ArUco landmarks
        +
known marker coordinates
        -> periodic global pose correction
```

This can be dramatically cheaper and lighter than a LiDAR.

Potential references:

- AprilTag 3: https://github.com/AprilRobotics/apriltag
- ROS 2 AprilTag wrapper examples: https://github.com/christianrauch/apriltag_ros
- OpenCV ArUco: https://docs.opencv.org/4.x/d5/dae/tutorial_aruco_detection.html

Treat this as **Plan B**, not a required purchase now.

A second fallback is to borrow a LiDAR only during development/demo, but that defeats the intended runtime novelty.

---

# 18. What Not to Do

Do not:

- publish fake perfect odometry in the final demo;
- use commanded wheel speed as if it were encoder measurement;
- hide drift by resetting pose manually every minute without documenting it;
- claim ultrasonic is doing SLAM if it is only preventing collisions;
- use AMCL with fabricated scan data just because tutorials expect it;
- tune Nav2 around wrong odometry.

A transparent controlled-navigation design is academically stronger than pretending to have commercial-grade localization.

---

# 19. Runtime Localization Bring-Up

At the beginning of each run:

```text
1. Place robot at physical HOME fixture/mark.
2. Keep robot still.
3. Start Pico.
4. Calibrate MPU6050 gyro bias.
5. Reset encoder counts.
6. Start ROS bridge.
7. Start EKF.
8. Reset/initialize odometry to home.
9. Publish/initialize map -> odom relationship.
10. Verify TF in RViz.
11. Only then enable autonomous movement.
```

Automate these steps later with a bringup launch/state machine, but understand them manually first.

---

## Stage Acceptance Checklist

- [ ] Physical home pose is repeatable.
- [ ] Home pose coordinates are documented in map frame.
- [ ] Encoder/IMU odometry starts consistently at home.
- [ ] `map -> odom -> base_link` TF tree is valid.
- [ ] AMCL is not being incorrectly used without a scan source.
- [ ] 1 m, 3 m, rotation, square and realistic-path drift tests completed.
- [ ] Drift data is saved to CSV/rosbag.
- [ ] Wheel geometry/PID has been retuned if needed.
- [ ] IR home-marker approach tested if applicable.
- [ ] Ultrasonic wall correction tested only if geometry/sensor count supports it.
- [ ] Typical cleaning-run drift is below the team's defined safe limit.
- [ ] If not, the team has triggered Plan B rather than ignoring the problem.

---

## Best References

- Nav2 state estimation: https://docs.nav2.org/jazzy/getting_started/navigation_concepts/state_estimation.html
- Nav2 transformations: https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/transformation/setup_transforms.html
- Nav2 odometry: https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/odom/setup_odom.html
- AMCL docs — understand why it is not our baseline: https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/others/configuring_amcl.html
- REP 105 coordinate frames: https://www.ros.org/reps/rep-0105.html
- `robot_localization`: https://docs.ros.org/en/jazzy/p/robot_localization/
- Nav2 Range Layer: https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/costmap_2d/costmap_plugins/range.html

**Next:** [Stage 10 — Nav2 Navigation](./10-nav2-navigation.md)
