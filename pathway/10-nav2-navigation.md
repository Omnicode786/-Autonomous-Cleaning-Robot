# Stage 10 — Nav2 Navigation & Obstacle Avoidance

## Goal

Make the physical robot navigate from one point to another on the saved room map **without the iPhone or LiDAR attached**.

At the end of this stage:

- Nav2 loads the static room map;
- our Stage 9 localization provides the required TF chain;
- Nav2 sends velocity commands to the Pico bridge;
- ultrasonic sensors update the local costmap;
- the robot can reach selected goals while avoiding newly placed obstacles;
- velocities and clearances are tuned conservatively for the real robot.

Do not begin this stage until phone-free localization passed its drift tests.

---

# 1. Understand the Nav2 Data Flow

For our robot:

```text
maps/room.yaml
      |
      v
  map_server ----------------------------> /map
                                             |
                                             v
                                      global costmap
                                             |
known home + odometry ----------------> TF: map -> odom
EKF ----------------------------------> TF: odom -> base_link
URDF ---------------------------------> TF: base_link -> sensors
                                             |
ultrasonic Range topics --------------> local costmap
                                             |
                                             v
                                       Nav2 planner
                                             |
                                       Nav2 controller
                                             |
                                             v
                                          /cmd_vel
                                             |
                                             v
                                      cleanbot_bridge
                                             |
                                          USB serial
                                             |
                                             v
                                            Pico
```

The key idea is that **Nav2 consumes our localization; it does not create it for us**.

---

# 2. Learn Nav2 Before Configuring It

Use these as the primary resources:

- Nav2 Getting Started: https://docs.nav2.org/jazzy/getting_started/
- Nav2 Navigation Concepts: https://docs.nav2.org/jazzy/getting_started/navigation_concepts/index.html
- Nav2 First-Time Robot Setup Guide: https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/index.html
- Nav2 configuration guide: https://docs.nav2.org/jazzy/configuration/index.html
- Articulated Robotics Nav2 overview: https://www.youtube.com/watch?v=jkoGkAd0GYk

Before touching the real robot, run a standard Nav2 simulation example so you understand:

```text
map
costmap
goal
planner
controller
behavior tree
cmd_vel
```

Do not copy a complete TurtleBot configuration and assume it fits our robot.

---

# 3. Install Nav2

On ROS 2 Jazzy, use the official package instructions.

Typical package names are:

```bash
sudo apt update
sudo apt install ros-jazzy-navigation2 ros-jazzy-nav2-bringup
```

Confirm packages:

```bash
ros2 pkg list | grep nav2
```

If package names ever differ, follow the current Jazzy docs rather than old tutorials.

---

# 4. Important: Do NOT Launch AMCL in Our Baseline

Most Nav2 tutorials use:

```text
static map + AMCL + 2D LaserScan
```

Our runtime robot intentionally has no LiDAR/laser scan.

Therefore our baseline is:

```text
static map + known-home localization + encoder/IMU dead reckoning
```

AMCL documentation shows it consumes a laser scan input:

- https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/others/configuring_amcl.html

So our bringup must **not accidentally launch AMCL**.

If using a Nav2 bringup launch that normally bundles localization, either:

- use the navigation-only launch path; or
- configure the launch so AMCL/localization is disabled; or
- write our own small bringup launch that starts exactly the nodes we need.

Verify:

```bash
ros2 node list | grep amcl
```

For the baseline design this should return nothing.

---

# 5. Create `cleanbot_navigation`

Recommended package structure:

```text
src/cleanbot_navigation/
├── package.xml
├── setup.py or CMakeLists.txt
├── launch/
│   └── navigation.launch.py
├── config/
│   ├── nav2.yaml
│   └── controller_limits.yaml
└── rviz/
    └── navigation.rviz
```

The launch should eventually bring together:

```text
map_server
lifecycle manager
Nav2 planner/controller/BT/navigation servers
local/global costmaps
our existing TF/localization providers
```

Keep map path and robot parameters configurable.

---

# 6. Robot Footprint

Nav2 needs the physical footprint of the robot.

Measure the real robot:

```text
length
width
bumper overhang
brush overhang if it can collide
```

For a circular approximation:

```yaml
robot_radius: 0.18
```

For a rectangular robot, use a polygon footprint.

Example concept for a 0.32 m x 0.28 m body:

```yaml
footprint: "[[0.16, 0.14], [0.16, -0.14], [-0.16, -0.14], [-0.16, 0.14]]"
```

Use **your measured values**, not this example.

Nav2 footprint guide:

- https://docs.nav2.org/jazzy/configuration/packages/configuring-costmaps.html

The collision footprint should represent parts that must not hit objects, not merely the wheel-to-wheel width.

---

# 7. Global Costmap

The global costmap is based mainly on the saved room map.

For our baseline, think in terms of:

```text
Static Layer
Inflation Layer
```

### Static Layer

Uses `/map` from the map server.

Docs:

- https://docs.nav2.org/jazzy/configuration/packages/costmap-plugins/static.html

### Inflation Layer

Adds safety cost around obstacles/walls.

Docs:

- https://docs.nav2.org/jazzy/configuration/packages/costmap-plugins/inflation.html

Start conservative so the robot does not skim walls.

Do not make the inflation radius huge just to stop collisions; if the footprint or odometry is wrong, fix those first.

---

# 8. Local Costmap With Ultrasonic Sensors

The saved map does not know about temporary obstacles such as:

```text
person
bag
moved chair
partly closed door
```

Our ultrasonic sensors provide local obstacle information.

Nav2 includes a **Range Sensor Layer** designed for 1D range sensors such as sonar/IR:

- https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/costmap_2d/costmap_plugins/range.html

Each sensor must already publish a correct:

```text
sensor_msgs/msg/Range
```

with a valid TF frame.

Conceptual local-costmap structure:

```yaml
local_costmap:
  local_costmap:
    ros__parameters:
      rolling_window: true
      width: 3.0
      height: 3.0
      resolution: 0.05

      plugins:
        - range_layer
        - inflation_layer

      range_layer:
        plugin: "nav2_costmap_2d::RangeSensorLayer"
        topics:
          - /range/front
          - /range/left
          - /range/right

      inflation_layer:
        plugin: "nav2_costmap_2d::InflationLayer"
```

Treat this as a **configuration pattern**. Check the current Jazzy Range Layer documentation for exact parameter names and plugin syntax before finalizing the YAML.

If we only own one ultrasonic sensor, begin with `/range/front` and add additional cheap sensors later only if blind spots become a real problem.

---

# 9. Visualize Costmaps in RViz

Add displays for:

```text
Map
TF
RobotModel
Odometry
Global Costmap
Local Costmap
Path
Range
Footprint
```

Before autonomous motion, verify:

1. robot footprint is centered correctly;
2. fixed walls align with robot position;
3. ultrasonic obstacle appears in front of the actual sensor;
4. the obstacle disappears/clears according to Range Layer behavior;
5. global and local frames are correct.

If an obstacle appears behind the robot when it is physically in front, the sensor TF is wrong.

---

# 10. Conservative Velocity Limits

This robot relies on dead reckoning, so high speed is the enemy.

Start roughly around:

```text
max linear velocity:   0.10–0.20 m/s
max angular velocity:  0.4–0.7 rad/s
```

These are starting ranges, not final requirements.

Use gentle acceleration/deceleration.

Benefits:

- less wheel slip;
- better encoder accuracy;
- more time for ultrasonic sensing;
- shorter stopping distance;
- safer lab testing;
- lower current spikes.

Only increase speed after repeatable navigation works.

---

# 11. Planner vs Controller

Understand the difference.

### Planner

Creates a path through the map:

```text
current pose -> goal pose
```

### Controller

Generates velocity commands that follow that path.

Do not spend the first week trying every planner/controller plugin.

Start with the standard/recommended Jazzy setup from the Nav2 examples and tune only what the tests prove necessary.

Useful docs:

- Planner server: https://docs.nav2.org/jazzy/configuration/packages/configuring-planner-server.html
- Controller server: https://docs.nav2.org/jazzy/configuration/packages/configuring-controller-server.html
- Nav2 tuning guide: https://docs.nav2.org/jazzy/tuning/index.html

---

# 12. First Navigation Test — RViz Goal

Do **not** start with full room coverage.

### Test sequence

1. Put robot at physical home.
2. Initialize localization exactly as Stage 9.
3. Verify map/odom/base TF in RViz.
4. Put wheels on floor.
5. Keep emergency stop access ready.
6. Set one goal only 0.5–1.0 m away.
7. Observe path and motion.
8. Stop immediately if localization visibly diverges.

Gradually test:

```text
straight goal
slightly offset goal
90-degree-turn goal
longer goal
goal near but safely away from wall
```

Do not test a tight doorway before open-space navigation is reliable.

---

# 13. Goal Command Interfaces

During development, the easiest method is clicking **2D Goal Pose / Nav2 Goal** in RViz.

Later, coverage code can use Nav2's Python API:

- Simple Commander API: https://docs.nav2.org/commander_api/index.html

Useful calls include concepts such as:

```text
goToPose()
followWaypoints()
followPath()
cancelTask()
```

The exact API may evolve, so use the installed Jazzy docs/source when implementing.

---

# 14. Dynamic Obstacle Test

After basic goals work:

1. send robot through open space;
2. place a large cardboard box or solid obstacle in the planned path;
3. ultrasonic should detect it;
4. local costmap should mark the obstacle;
5. Nav2 should stop or replan depending on configuration;
6. remove obstacle;
7. observe whether the costmap clears appropriately.

Do not use a human leg as the first obstacle test.

Test with something large and easy for ultrasonic to reflect from.

---

# 15. Ultrasonic Safety Layer vs Pico Emergency Stop

Use **both**.

### Nav2 Range Layer

Provides intelligent local obstacle avoidance.

### Pico hard safety threshold

The Pico may also enforce a very-close emergency condition such as:

```text
if front_range < emergency_distance:
    wheel targets = 0
```

This prevents a laptop/ROS failure from continuing to drive into something.

Be careful: bad ultrasonic spikes can cause false stops, so require a sensible filtering/confirmation rule.

The exact emergency distance should exceed the robot's stopping distance at maximum allowed speed.

---

# 16. Tuning Order

If navigation is poor, tune in this order:

1. **TF correctness**
2. **wheel/IMU odometry accuracy**
3. **map scale and alignment**
4. **robot footprint**
5. **velocity/acceleration limits**
6. **costmap sensors/inflation**
7. **controller parameters**
8. **planner parameters**

Do not start at step 8.

Many “Nav2 problems” are actually bad odometry or TF.

---

# 17. Measure Navigation Success

Create:

```text
experiments/navigation/navigation_results.csv
```

Suggested fields:

```text
run_id
start_pose
goal_pose
straight_line_distance_m
nav_time_s
success
final_position_error_m
final_yaw_error_deg
minimum_obstacle_clearance_m
recovery_count
notes
```

Perform at least:

```text
10 simple open-space goals
10 mixed turning goals
5 dynamic-obstacle tests
```

The exact count can be adjusted to your course requirements, but repeatability matters.

---

# 18. Stage Deliverable

By the end, the repository should contain roughly:

```text
src/cleanbot_navigation/
├── launch/
│   └── navigation.launch.py
├── config/
│   └── nav2.yaml
└── rviz/
    └── navigation.rviz

experiments/navigation/
└── navigation_results.csv
```

The bringup procedure should be documented so another team member can reproduce it.

---

## Stage Acceptance Checklist

- [ ] Static map loads correctly.
- [ ] AMCL is not accidentally running in the no-LiDAR baseline.
- [ ] `map -> odom -> base_link` stays connected during motion.
- [ ] Robot footprint matches physical dimensions.
- [ ] Global static costmap aligns with the room.
- [ ] Ultrasonic data appears correctly in the local costmap.
- [ ] One short open-space goal succeeds repeatedly.
- [ ] Turning/multi-direction goals succeed.
- [ ] A newly placed obstacle causes safe stop/replanning.
- [ ] Navigation speeds are low enough to avoid excessive slip.
- [ ] Pico command timeout/emergency stop still works.
- [ ] Navigation success/error is recorded over repeated trials.

---

## Best References

- Nav2 Getting Started: https://docs.nav2.org/jazzy/getting_started/
- Nav2 concepts: https://docs.nav2.org/jazzy/getting_started/navigation_concepts/index.html
- First-Time Robot Setup: https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/index.html
- Costmaps: https://docs.nav2.org/jazzy/configuration/packages/configuring-costmaps.html
- Range Sensor Layer: https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/costmap_2d/costmap_plugins/range.html
- Nav2 tuning guide: https://docs.nav2.org/jazzy/tuning/index.html
- Nav2 Simple Commander: https://docs.nav2.org/commander_api/index.html
- Articulated Robotics Nav2 overview: https://www.youtube.com/watch?v=jkoGkAd0GYk

**Next:** [Stage 11 — Coverage Planning & Cleaning](./11-coverage-and-cleaning.md)
