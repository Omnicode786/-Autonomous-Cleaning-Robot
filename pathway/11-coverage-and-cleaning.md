# Stage 11 — Coverage Planning & Cleaning System

## Goal

Turn point-to-point navigation into an actual cleaning robot.

At the end of this stage:

- the robot systematically covers a selected floor area instead of wandering randomly;
- coverage paths respect wall/obstacle clearance;
- the vacuum and brush are controlled by software;
- temporary obstacles are handled by Nav2;
- the robot can return to home after a cleaning zone;
- coverage percentage, overlap and cleaning time can be measured.

---

# 1. Navigation Is Not Coverage

Nav2 solves:

```text
A -> B
```

Cleaning requires:

```text
visit nearly all reachable floor area
```

A simple vacuum pattern is **boustrophedon / lawnmower coverage**:

```text
+--------------------------------+
| >>>>>>>>>>>>>>>>>>>>>>>>>>>>>> |
|                              v |
| <<<<<<<<<<<<<<<<<<<<<<<<<<<<<< |
| v                              |
| >>>>>>>>>>>>>>>>>>>>>>>>>>>>>> |
|                              v |
| <<<<<<<<<<<<<<<<<<<<<<<<<<<<<< |
+--------------------------------+
```

For a university project, start with deterministic parallel lanes. It is easier to explain, debug and evaluate than random movement.

---

# 2. Start With One Simple Rectangular Zone

Do not immediately try to automatically segment an arbitrary room.

Create a manually defined cleaning zone in map coordinates.

Example:

```yaml
# config/cleaning_zones.yaml
zones:
  demo_zone:
    min_x: 0.7
    max_x: 4.2
    min_y: 0.6
    max_y: 3.2
```

These are only example coordinates.

Choose the real zone by viewing the map in RViz and measuring coordinates.

The zone should exclude:

- walls;
- fixed furniture;
- known unreachable areas;
- a safety margin around the map boundary.

Once rectangular coverage works reliably, extend to multiple polygons/rooms.

---

# 3. Determine Effective Cleaning Width

Measure the actual width that one pass cleans.

Suppose:

```text
brush/suction effective width = 0.24 m
```

Do not set lane spacing exactly to 0.24 m. Small localization errors could leave strips uncleaned.

A useful starting overlap is roughly 10–25%.

Example:

```text
cleaning width = 0.24 m
overlap = 20%
lane spacing = 0.24 * 0.80 = 0.192 m
```

Round to something practical such as 0.19–0.20 m.

Measure cleaning width experimentally with visible debris rather than only measuring the brush diameter.

---

# 4. Generate Lawnmower Lanes

For a rectangular zone:

1. choose sweep direction along the longer room dimension;
2. inset boundaries by robot footprint + safety margin;
3. create parallel lines separated by `lane_spacing`;
4. alternate direction of each line;
5. connect endpoints using safe turns/Nav2 goals.

Conceptual Python logic:

```python
waypoints = []
y = min_y
forward = True

while y <= max_y:
    if forward:
        waypoints.append((min_x, y))
        waypoints.append((max_x, y))
    else:
        waypoints.append((max_x, y))
        waypoints.append((min_x, y))

    forward = not forward
    y += lane_spacing
```

Real implementation must also include:

- yaw orientation;
- obstacle/free-space checking;
- robot-radius/footprint margins;
- valid map cells only.

Do not send waypoints through occupied cells just because they fit inside a rectangle.

---

# 5. Use Nav2 for Motion Between Coverage Points

Do not write your own low-level path follower for coverage.

Coverage planner decides **where to clean**.

Nav2 decides **how to move there safely**.

Use Nav2 Simple Commander or the Waypoint Follower interface.

Resources:

- Nav2 Simple Commander API: https://docs.nav2.org/commander_api/index.html
- Waypoint Follower: https://docs.nav2.org/jazzy/configuration/packages/configuring-waypoint-follower.html

Conceptually:

```python
navigator.followWaypoints(poses)
```

or send lane endpoints one at a time using `goToPose()`.

For early testing, one-at-a-time goals are easier to debug.

---

# 6. Recommended `cleanbot_coverage` Package

```text
src/cleanbot_coverage/
├── package.xml
├── setup.py
├── config/
│   └── cleaning_zones.yaml
├── launch/
│   └── coverage.launch.py
└── cleanbot_coverage/
    ├── coverage_planner.py
    ├── cleaning_manager.py
    └── coverage_tracker.py
```

Responsibilities:

### `coverage_planner.py`

```text
zone polygon/rectangle
+ robot clearance
+ lane spacing
-> ordered lane waypoints
```

### `cleaning_manager.py`

Controls the cleaning state machine and sends navigation tasks.

### `coverage_tracker.py`

Tracks where the robot has travelled and calculates coverage metrics.

Keep these responsibilities separate so each can be tested independently.

---

# 7. Cleaning State Machine

Use an explicit state machine instead of a giant loop.

Recommended states:

```text
IDLE
  |
  v
PREPARE
  |
  v
NAVIGATE_TO_ZONE
  |
  v
CLEANING
  |
  +----> OBSTACLE / RECOVERY ----+
  |                               |
  +<------------------------------+
  |
  v
RETURN_HOME
  |
  v
DONE

Any state -> FAULT
```

### IDLE

Waiting for a cleaning request.

### PREPARE

Check:

```text
localization ready
battery okay
Pico connected
vacuum/brush responsive
```

### NAVIGATE_TO_ZONE

Move to the beginning of the first lane with cleaning motor off or low.

### CLEANING

Turn on brush/vacuum and execute coverage waypoints.

### RETURN_HOME

Return to the known home area after zone completion or low battery.

### FAULT

Stop wheels and cleaning hardware safely.

This makes final demonstration behavior predictable and reportable.

---

# 8. Vacuum and Brush Control

The Pico should control high-current cleaning loads through proper drivers.

Conceptually:

```text
Pico GPIO/PWM
      |
      v
logic-level MOSFET / motor driver
      |
      v
vacuum blower / brush motor
```

Never connect cleaning motors directly to Pico GPIO.

### Vacuum

If only ON/OFF is needed, use a suitable logic-level MOSFET/load switch rated above the blower's current.

If PWM speed control is useful, confirm the motor/load is compatible with that drive method.

### Brush motor

For one-direction brushing, a MOSFET may be enough.

If reversing is required to clear jams, use a motor driver/H-bridge.

For inductive brushed motors, provide appropriate flyback/transient suppression according to the driver topology.

---

# 9. Measure Current Before Finalizing Power Electronics

The vacuum motor can draw much more current than the Pico electronics.

Measure or find:

```text
normal running current
startup current
stall/blockage current if safely testable
```

Select:

```text
MOSFET/driver
BMS
wires
connectors
fuse
```

with margin.

Do not size the battery system based only on average current.

---

# 10. Software Cleaning Commands

Useful ROS interfaces:

```text
/cleanbot/vacuum_enable
/cleanbot/brush_enable
/cleanbot/cleaning_state
```

For a first system, Boolean topics are fine.

A cleaner long-term API is a service/action such as:

```text
StartCleaning(zone_name)
StopCleaning
```

or a custom action with progress feedback.

Do not delay the project by designing custom interfaces too early. Working behavior first.

---

# 11. Handling Dynamic Obstacles During Coverage

Coverage code should **not** personally control emergency steering.

When a person/box/chair appears:

```text
ultrasonic -> local costmap -> Nav2 controller/replanner
```

Coverage manager should:

1. wait while Nav2 handles the local obstacle;
2. detect navigation failure/timeout;
3. retry the current waypoint if safe;
4. skip an unreachable waypoint after a limited number of retries;
5. continue the remaining coverage;
6. record missed regions.

Never retry forever.

Example policy:

```text
max retries per waypoint = 2
if still unreachable -> mark skipped -> continue
```

The exact number can be tuned.

---

# 12. Coverage Around Static Obstacles

The rectangular-lane generator will eventually encounter fixed furniture in the occupancy map.

Three levels of sophistication:

### Level 1 — Manual zones

Divide the room into simple rectangles around obstacles.

Example:

```text
zone A: left side of table
zone B: right side of table
zone C: doorway section
```

**Recommended first implementation.**

### Level 2 — Grid clipping

Generate lanes and split/remove portions that cross occupied/inflated map cells.

### Level 3 — Full polygon coverage planner

Use a specialized coverage planning library.

OpenNav Coverage is a modern ROS 2/Nav2 project for complete coverage planning:

- Repository: https://github.com/open-navigation/opennav_coverage
- Nav2 Coverage Server docs: https://docs.nav2.org/rolling/configuration_and_development/configuration_guide/others/configuring_coverage_server/

Use it as an advanced upgrade after our simple lane planner works. Do not make the whole project dependent on it from day one.

---

# 13. Coverage Tracker

We should prove how much floor was covered.

Create an internal grid aligned with the occupancy map.

For each robot pose:

1. transform robot footprint/cleaning footprint into map coordinates;
2. mark those free cells as cleaned;
3. accumulate only cells inside selected cleaning zone.

Then:

```text
coverage % = cleaned free cells / total target free cells * 100
```

Also calculate:

```text
overlap %
cleaning time
travel distance
number of skipped waypoints
number of recoveries
```

This produces a very strong final project metric.

---

# 14. Visualize Coverage as a Heatmap

You can publish a second occupancy-style grid such as:

```text
/cleanbot/coverage_map
```

or save an image after each run.

Concept:

```text
0     = not cleaned
1     = cleaned once
2+    = overlapping passes
```

Render it later as:

```text
unvisited
visited once
visited multiple times
```

This does not require AI and is already a good novelty/analytics feature.

---

# 15. Test Cleaning Width With Real Debris

Create a controlled test strip.

Use safe visible material appropriate for the vacuum, for example:

```text
small paper confetti
light crumbs / cereal-like particles
```

Avoid powders that may damage electronics/motors or create respiratory/dust hazards.

Test:

1. one straight pass;
2. photograph before/after;
3. measure actually cleaned width;
4. repeat at multiple speeds;
5. choose final lane spacing.

This converts cleaning width from a guess into an experiment.

---

# 16. Choose Cleaning Speed

Navigation speed and effective cleaning speed may differ.

Test something like:

```text
0.08 m/s
0.12 m/s
0.16 m/s
```

Measure debris pickup and localization drift.

A slower robot may clean better and localize better, which is ideal for a university demo.

Store the result as a parameter:

```yaml
cleaning_speed_mps: 0.12
```

Use your measured best value.

---

# 17. Return-to-Home Strategy

After each zone:

```text
coverage complete
      |
turn vacuum/brush off or reduce power
      |
Nav2 goal -> near home approach pose
      |
slow docking/home-marker routine
      |
IR/physical alignment if available
      |
reset/re-zero localization
```

Because our runtime localization is dead-reckoning based, zone-by-zone home resets are particularly valuable.

For a first final demo, cleaning one moderate-size zone and returning home reliably is better than attempting the entire building.

---

# 18. Low-Battery Behavior

If battery voltage measurement is available, define conservative thresholds from the actual battery chemistry/BMS rather than arbitrary numbers.

Possible state behavior:

```text
battery normal -> continue
battery low -> stop cleaning -> return home
battery critical -> stop motors/cleaning safely
```

Do not use a single instantaneous voltage sample because motor startup causes voltage sag.

Filter voltage over time.

Battery-state estimation from voltage alone is approximate, especially under load. Document that limitation.

---

# 19. Coverage Experiments

Create:

```text
experiments/coverage/coverage_results.csv
```

Suggested columns:

```text
run_id
zone
lane_spacing_m
cleaning_speed_mps
target_area_m2
coverage_percent
overlap_percent
runtime_s
path_length_m
skipped_waypoints
recoveries
final_position_error_m
battery_start_v
battery_end_v
notes
```

Run the same zone several times.

We want repeatability, not one perfect screenshot.

---

# 20. Cleaning Effectiveness Experiment

For a controlled test area:

1. distribute a repeatable amount/count of visible debris;
2. photograph/count before;
3. run one cleaning mission;
4. measure remaining debris;
5. calculate a simple pickup metric.

Example:

```text
initial pieces = 50
remaining = 8
pickup rate = (50 - 8) / 50 * 100 = 84%
```

A basic quantified experiment is much stronger than saying “the vacuum works.”

---

## Stage Acceptance Checklist

- [ ] At least one cleaning zone is defined in map coordinates.
- [ ] Effective cleaning width measured experimentally.
- [ ] Lane spacing includes deliberate overlap.
- [ ] Lawnmower/boustrophedon waypoints are generated correctly.
- [ ] Waypoints stay inside safe free space.
- [ ] Nav2 executes coverage waypoints.
- [ ] Vacuum can be enabled/disabled through the Pico.
- [ ] Brush can be enabled/disabled through the Pico.
- [ ] Dynamic obstacle can interrupt/replan safely.
- [ ] Failed waypoint has bounded retry behavior.
- [ ] Robot returns toward home after cleaning.
- [ ] Coverage percentage is measured.
- [ ] Cleaning effectiveness is measured on a controlled test area.
- [ ] Several repeated coverage runs are recorded.

---

## Best References

- Nav2 Simple Commander API: https://docs.nav2.org/commander_api/index.html
- Nav2 Waypoint Follower: https://docs.nav2.org/jazzy/configuration/packages/configuring-waypoint-follower.html
- Nav2 navigation concepts: https://docs.nav2.org/jazzy/getting_started/navigation_concepts/index.html
- OpenNav Coverage: https://github.com/open-navigation/opennav_coverage
- Nav2 Coverage Server docs: https://docs.nav2.org/rolling/configuration_and_development/configuration_guide/others/configuring_coverage_server/

**Next:** [Stage 12 — Integration, Testing & Evaluation](./12-integration-testing.md)
