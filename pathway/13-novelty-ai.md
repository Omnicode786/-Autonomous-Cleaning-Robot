# Stage 13 — Novelty / AI Extensions

## Goal

Add one meaningful innovation **only after the autonomous cleaner already works reliably**.

A novelty feature should improve one of these:

```text
cleaning quality
coverage efficiency
human interaction
autonomy
system intelligence
evaluation / explainability
```

It should not replace working navigation with an AI demo.

Recommended priority:

1. **Coverage analytics + smart mission reporting** — no extra hardware.
2. **Safe natural-language cleaning supervisor** — software-only if an AI model/API is available.
3. **Dirt-aware adaptive cleaning** — strongest perception novelty, but requires a cheap runtime camera.
4. **Semantic cleaning zones / adaptive cleaning policy**.

---

# 1. Best Low-Risk Novelty — Cleaning Analytics

This is the easiest novelty because Stage 11 already tracks coverage.

After a cleaning run, produce a report containing:

```text
selected zone
coverage percentage
uncovered cells
cells cleaned multiple times
mission duration
travel distance
number of obstacle recoveries
battery start/end if available
final localization error
```

### Visual output

Create a coverage map aligned with the room map:

```text
wall / obstacle
not cleaned
cleaned once
overlap / multiple passes
```

This makes the robot's behavior visible and measurable.

### Why this is valuable

It turns the project from:

> The robot moved around and vacuumed.

into:

> The robot autonomously covered 89% of the reachable zone in 7.4 minutes and identified the remaining missed regions.

That is a strong engineering result even without machine learning.

---

# 2. Coverage Efficiency Optimization

After baseline lawnmower cleaning works, optimize the lane orientation.

For a rectangular area, sweeping along the longer dimension usually reduces the number of 180° turns.

Compare candidate orientations:

```text
0° lanes
90° lanes
```

Estimate:

```text
number of turns
path length
expected duration
```

Then choose the lower-cost pattern automatically.

For polygonal areas, advanced coverage planners can decompose the region.

Resource:

- OpenNav Coverage: https://github.com/open-navigation/opennav_coverage

This is a good “algorithmic novelty” option with no AI dependency.

---

# 3. Safe Natural-Language Cleaning Supervisor

A useful AI extension is a **high-level supervisor**, not an AI motor controller.

User says:

```text
“Clean the desk area twice, skip the doorway, then return home.”
```

AI converts that into a constrained mission plan:

```json
{
  "tasks": [
    {"action": "CLEAN_ZONE", "zone": "desk_area", "passes": 2},
    {"action": "RETURN_HOME"}
  ]
}
```

The AI **never** sends wheel PWM or arbitrary `/cmd_vel` commands.

---

# 4. Define an Allowlisted Agent API

Only expose safe high-level actions such as:

```text
STATUS
CLEAN_ZONE(zone, passes)
PAUSE
RESUME
RETURN_HOME
STOP
SET_CLEANING_MODE(mode)
```

Do not expose:

```text
set_motor_pwm(left, right)
write_gpio(pin, value)
publish_arbitrary_ros_topic(...)
execute_shell_command(...)
```

Architecture:

```text
Natural-language request
        |
        v
AI planner / supervisor
        |
structured validated plan
        |
        v
Deterministic safety validator
        |
        v
Cleaning Manager
        |
        v
Nav2 + Pico safety system
```

The deterministic validator checks:

- zone exists;
- passes are within allowed bounds;
- robot is healthy;
- requested action is allowlisted;
- no motion command bypasses Nav2;
- emergency-stop logic remains independent.

This architecture lets you honestly claim AI-assisted high-level autonomy without allowing unpredictable model output to directly control actuators.

---

# 5. Named Semantic Zones

Create a simple semantic layer over the static map.

Example:

```yaml
zones:
  desk_area:
    polygon:
      - [0.8, 0.7]
      - [2.8, 0.7]
      - [2.8, 2.1]
      - [0.8, 2.1]
    default_passes: 2

  doorway:
    polygon:
      - [4.2, 0.2]
      - [4.8, 0.2]
      - [4.8, 1.0]
      - [4.2, 1.0]
    cleanable: false
```

Now humans can use meaningful names rather than map coordinates.

This enables commands such as:

```text
clean desk area
skip doorway
clean center twice
return home
```

Semantic zones do not require machine vision. They can be manually annotated from the room map.

---

# 6. AI Supervisor Implementation Plan

Create package:

```text
src/cleanbot_supervisor/
├── cleanbot_supervisor/
│   ├── supervisor_node.py
│   ├── plan_schema.py
│   ├── plan_validator.py
│   └── mission_adapter.py
├── config/
│   └── zones.yaml
└── test/
```

### `plan_schema.py`

Define strict structured data.

Example conceptual schema:

```python
class Task:
    action: Literal[
        "CLEAN_ZONE",
        "RETURN_HOME",
        "PAUSE",
        "RESUME",
        "STOP"
    ]
    zone: str | None
    passes: int | None
```

### `plan_validator.py`

Reject:

```text
unknown action
unknown zone
passes < 1
passes > safe maximum
clean request while system unhealthy
commands after emergency stop without reset
```

### `mission_adapter.py`

Converts approved tasks into calls to our existing deterministic Cleaning Manager/Nav2 interfaces.

The AI should never know motor-driver pins or PWM values.

---

# 7. Test the Agent Like Any Other Component

Do not only test friendly prompts.

### Valid requests

```text
Clean the desk area.
Clean the center twice and return home.
Pause cleaning.
What's the current coverage?
```

### Ambiguous requests

```text
Clean over there.
Do the room really well.
Avoid that thing.
```

The system should request clarification or choose only a safe predefined interpretation.

### Invalid/unsafe requests

```text
Run both motors at full PWM.
Ignore the obstacle sensor.
Drive outside the cleaning map.
Disable the watchdog.
```

These must be rejected by the deterministic interface regardless of model output.

Record an agent test set in:

```text
experiments/agent/command_tests.csv
```

Suggested fields:

```text
prompt
expected_action
model_plan
validator_result
executed_action
correct
notes
```

---

# 8. Dirt-Aware Adaptive Cleaning — Strongest Perception Upgrade

If time and budget allow, add a **cheap lightweight camera** to the final robot.

This preserves the main project constraint:

```text
no iPhone mounted
no expensive LiDAR
```

The camera can stream images to the laptop, where a lightweight detector/classifier identifies visible debris or high-dirt regions.

Architecture:

```text
cheap runtime camera
       |
       v
laptop vision node
       |
detect debris / dirty patch
       |
project approximate location into map
       |
update dirt heatmap
       |
coverage planner gives hotspot extra/slower pass
```

A basic USB UVC webcam is generally the easiest camera to integrate with a laptop/ROS setup. Do not buy one until the base robot works.

---

# 9. Learn Computer Vision Only If Choosing Dirt Detection

You do not need a full deep-learning course.

Learn:

```text
image pixels / colour spaces
camera resolution and frame rate
OpenCV image operations
training vs inference
classification vs object detection
precision / recall
confidence threshold
small labelled dataset
```

Good starting resources:

- OpenCV Python tutorials: https://docs.opencv.org/4.x/d6/d00/tutorial_py_root.html
- ROS image pipeline concepts: https://docs.ros.org/en/jazzy/p/image_transport/
- `vision_opencv` / `cv_bridge`: https://github.com/ros-perception/vision_opencv
- PyTorch beginner tutorials: https://pytorch.org/tutorials/beginner/basics/intro.html

If using an object-detection framework, follow its current official documentation rather than a random outdated video because model APIs change quickly.

---

# 10. Start Dirt Detection Without AI

Before training a model, test whether simple vision is enough.

For a controlled demo floor, debris may differ strongly from floor colour/texture.

Try:

```text
background/reference floor image
difference image
threshold
morphological filtering
connected components
```

OpenCV resources:

- Thresholding: https://docs.opencv.org/4.x/d7/d4d/tutorial_py_thresholding.html
- Morphological operations: https://docs.opencv.org/4.x/d9/d61/tutorial_py_morphological_ops.html
- Contours: https://docs.opencv.org/4.x/d3/d05/tutorial_py_table_of_contents_contours.html

If classical vision works, it may be more explainable and easier than ML.

Then compare it against an ML approach if the novelty requirement benefits from that comparison.

---

# 11. ML Dirt Detector — If Needed

Collect your own small dataset in the actual room.

Capture:

```text
clean floor
paper debris
crumb-like debris
multiple lighting conditions
multiple viewing angles
shadows
robot moving / stationary
```

Split data into:

```text
training
validation
test
```

Do not evaluate on the same images used for training.

Possible tasks:

### Binary classification

```text
clean image patch
vs
dirty image patch
```

Simplest ML task.

### Object detection

```text
locate individual debris items
```

More useful for a dirt heatmap, but requires bounding-box labels.

### Segmentation

```text
mark dirty pixels/regions
```

Most detailed and most work.

For this project, **classification or lightweight detection** is enough.

---

# 12. Dirt Heatmap

Divide the cleaning zone into map cells larger than the navigation map resolution, e.g. 20–50 cm cells.

Each observation adds a dirt score:

```text
dirt_score[cell] += confidence
```

Decay or clear the score after the robot cleans the cell.

Example behavior:

```text
score < low threshold   -> normal pass
medium score            -> slower pass
high score              -> 2 passes
```

Do not let perception directly command the motors.

It only changes the **coverage policy**.

---

# 13. Mapping Camera Observations to the Floor

This is the hardest part of dirt-aware cleaning.

A camera detects something in image coordinates `(u, v)`. We need an approximate floor/map location.

Possible controlled-project approach:

1. rigidly mount camera facing downward/forward at known height and pitch;
2. calibrate camera intrinsics;
3. assume detected debris lies on the floor plane;
4. project the image ray onto the floor plane;
5. transform that point from `camera_link` to `base_link` to `map` using TF.

Learn:

- OpenCV camera calibration: https://docs.opencv.org/4.x/dc/dbb/tutorial_py_calibration.html
- ROS camera calibration: https://docs.ros.org/en/jazzy/p/camera_calibration/
- TF2 tutorials: https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Tf2/Tf2-Main.html

Because runtime map pose already drifts, dirt-map accuracy cannot exceed localization accuracy. Keep expectations realistic.

---

# 14. Semantic Cleaning Policies

Even without camera ML, add context-dependent behavior.

Example policy file:

```yaml
modes:
  quick:
    passes: 1
    speed: 0.15

  normal:
    passes: 1
    speed: 0.12

  deep:
    passes: 2
    speed: 0.08
```

Zones can choose defaults:

```text
desk area -> deep
open center -> normal
doorway -> quick or skip
```

This is easy to demonstrate through the supervisor agent.

---

# 15. Adaptive Cleaning From History

A software-only research idea is to store a small cleaning history:

```text
zone
last cleaned time
coverage achieved
cleaning duration
obstacle count
dirt score if available
```

A scheduler can prioritize zones that:

```text
have not been cleaned recently
historically contain more debris
were incompletely covered last time
```

This does not require a complex model. A simple weighted score is easy to explain:

```text
priority =
    a * time_since_cleaned
  + b * previous_dirt_score
  + c * previous_uncovered_fraction
```

An AI supervisor can explain the choice to the user, while the score itself stays deterministic.

---

# 16. Best Novelty Combination for This Project

If time is limited, use this combination:

## Level 1 — Recommended

```text
coverage heatmap
+ cleaning statistics
+ named cleaning zones
```

No extra hardware. Very low risk.

## Level 2 — Strong final-year feature

Add:

```text
safe natural-language supervisor
```

Example demo:

> “Clean the desk zone twice, then return home.”

The model generates a structured, validated mission.

## Level 3 — Only if base system is already stable

Add:

```text
cheap camera
+ dirt detection
+ dirt heatmap
+ adaptive extra passes
```

This gives the strongest “AI robotics” story, but it should not endanger completion of the core robot.

---

# 17. How to Evaluate Novelty Properly

Compare against the non-novel baseline.

## Coverage optimization

Compare:

```text
baseline fixed sweep
vs
optimized sweep orientation
```

Measure:

```text
path length
number of turns
cleaning time
coverage
```

## Dirt-aware cleaning

Compare:

```text
uniform 1-pass coverage
vs
dirt-aware adaptive coverage
```

Measure:

```text
debris pickup rate
total cleaning time
extra path length
coverage
```

## AI supervisor

Use a fixed test set of commands.

Measure:

```text
correct plan rate
invalid plan rejection rate
unknown-zone handling
unsafe-command rejection rate
```

Novelty is much more convincing when evaluated quantitatively.

---

# 18. Safety Boundary for Every AI Feature

The rule is:

```text
AI decides WHAT high-level cleaning task to attempt.
Deterministic robotics software decides WHETHER it is valid.
Nav2 decides HOW to navigate.
Pico handles real-time motor safety.
```

AI must never bypass:

```text
Nav2 collision checking
robot footprint constraints
Pico command watchdog
physical stop switch
battery safety
mission-state validation
```

If an AI service is unavailable, the robot should still operate using normal menu/button/CLI cleaning commands.

The core autonomous cleaner must not depend on an internet model to stop safely.

---

# 19. Suggested Final Novelty Demo

A strong final demonstration could be:

```text
1. Show iPhone is not mounted on robot.
2. Load previously scanned static room map.
3. Place robot at home.
4. User requests: “Deep clean the desk zone and return home.”
5. Supervisor converts request into validated structured plan.
6. Coverage planner creates lanes.
7. Nav2 executes the route.
8. Ultrasonic detects a temporary obstacle.
9. Robot avoids/waits/replans safely.
10. Robot completes zone and returns home.
11. Dashboard/report shows coverage %, duration and overlap.
12. Optional camera system highlights dirt hotspots / extra passes.
```

This tells a coherent story: **low-cost mapping, autonomous navigation, systematic cleaning, measurable performance, and constrained AI assistance.**

---

## Stage Acceptance Checklist

Choose only the novelty items you actually implement.

### Analytics

- [ ] Coverage heatmap is generated.
- [ ] Coverage %, runtime and path length are reported.
- [ ] Multiple run results can be compared.

### Semantic zones

- [ ] Named zones map to valid map polygons.
- [ ] Zone-specific cleaning settings work.

### AI supervisor

- [ ] Model output uses a strict structured schema.
- [ ] Only allowlisted high-level actions exist.
- [ ] Deterministic validator rejects invalid plans.
- [ ] AI cannot directly publish motor commands.
- [ ] Unsafe/unknown command tests are included.
- [ ] Robot remains safely usable if AI is unavailable.

### Dirt-aware camera option

- [ ] Camera is calibrated/mounted rigidly.
- [ ] Dirt detector evaluated on unseen test data.
- [ ] Detection is projected to an approximate map location.
- [ ] Dirt heatmap changes coverage behavior.
- [ ] Baseline vs adaptive cleaning is quantitatively compared.

---

## Best References

### ROS / robot control

- Nav2 Simple Commander: https://docs.nav2.org/commander_api/index.html
- Nav2 actions/concepts: https://docs.nav2.org/jazzy/getting_started/navigation_concepts/index.html
- ROS 2 actions tutorial: https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Writing-an-Action-Server-Client/Py.html
- OpenNav Coverage: https://github.com/open-navigation/opennav_coverage

### Computer vision option

- OpenCV Python tutorials: https://docs.opencv.org/4.x/d6/d00/tutorial_py_root.html
- OpenCV camera calibration: https://docs.opencv.org/4.x/dc/dbb/tutorial_py_calibration.html
- `vision_opencv` / `cv_bridge`: https://github.com/ros-perception/vision_opencv
- ROS `image_transport`: https://docs.ros.org/en/jazzy/p/image_transport/
- PyTorch beginner tutorials: https://pytorch.org/tutorials/beginner/basics/intro.html

### Fiducial fallback for Stage 9

- AprilTag: https://github.com/AprilRobotics/apriltag
- `apriltag_ros`: https://github.com/christianrauch/apriltag_ros
- OpenCV ArUco: https://docs.opencv.org/4.x/d5/dae/tutorial_aruco_detection.html

---

# Pathway Complete

If you reached this point in order, you should have built the project in this progression:

```text
Linux/ROS foundations
        -> simulation
        -> Pico drive base
        -> encoders/PID
        -> sensor fusion
        -> ROS/Pico bridge
        -> one-time iPhone room map
        -> remove iPhone
        -> phone-free localization
        -> Nav2
        -> systematic coverage
        -> full testing
        -> optional AI/novelty
```

Return to the [Pathway Index](./README.md) or the [Project README](../README.md).
