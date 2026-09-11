# Stage 05 — Encoders, PID & Wheel Odometry

## Goal

Turn the drive base from an open-loop toy car into a measurable mobile robot.

This is one of the most important stages because after the iPhone is removed, **wheel encoders become the main source of translation information**.

At the end of this stage:

- both wheels have reliable encoder feedback;
- the Pico measures wheel speed;
- each motor has closed-loop speed PID;
- wheel radius and wheel separation are calibrated;
- encoder data produces differential-drive odometry;
- repeated straight/rotation tests give acceptable error.

Do not move to Nav2 while this stage is weak.

---

## 1. Encoder Hardware First

Our existing motors have no built-in encoders, so mechanical compatibility matters.

Possible encoder types:

### A. Optical encoder disc

A slotted/striped disc rotates with the motor/wheel shaft and an optical sensor counts transitions.

Good when:

- there is accessible shaft space;
- the disc can be mounted concentrically;
- the sensor can be rigidly fixed.

### B. Magnetic encoder

A magnet attaches to the rotating shaft and a Hall/magnetic sensor measures rotation.

Often mechanically cleaner if there is a rear shaft.

### C. Replace only the two motors with encoder motors

If the existing motors cannot accept a reliable encoder, this may be the cheapest engineering solution.

A badly mounted Rs 600 encoder that slips is worse than a slightly more expensive encoder motor.

### Before buying

Check:

```text
shaft diameter
shaft accessibility
wheel attachment
available rear-shaft space
encoder disc bore
sensor mounting distance
expected pulses per revolution
logic voltage
```

Take photos and measurements before ordering.

---

## 2. Understand PPR / CPR

Manufacturers may specify:

- PPR — pulses per revolution;
- CPR — counts per revolution;
- counts per motor revolution;
- counts per gearbox output revolution.

These are not always the same.

For a geared motor, the most important value is:

```text
counts per WHEEL revolution
```

If you do not trust the datasheet, measure it.

### Manual measurement

1. Mark the wheel at one point.
2. Reset encoder count to zero.
3. Rotate the wheel exactly 10 full revolutions slowly.
4. Read total counts.
5. Divide by 10.

```text
counts_per_wheel_rev = total_counts / 10
```

Using 10 revolutions reduces measurement error.

---

## 3. Single-Channel vs Quadrature

### Single-channel

One signal pulses as the wheel rotates.

You know:

```text
how much rotation occurred
```

but not direction from the encoder alone.

For a simple robot, direction can be inferred from the commanded motor direction.

### Quadrature

Two channels A/B are phase shifted.

You can determine:

```text
rotation amount
rotation direction
```

Quadrature is better if available.

If using quadrature, understand x1/x2/x4 counting before choosing CPR values.

---

## 4. Count Pulses Reliably on the Pico

Do not poll an encoder slowly in the main loop; pulses may be missed.

Use:

- GPIO interrupts for normal encoder rates;
- PIO only if needed for high-rate/complex quadrature counting.

The RP2040 is excellent for this type of job.

Pico references:

- GPIO API: https://www.raspberrypi.com/documentation/pico-sdk/hardware.html#hardware_gpio
- PIO documentation: https://www.raspberrypi.com/documentation/pico-sdk/hardware.html#hardware_pio
- Pico examples: https://github.com/raspberrypi/pico-examples

### Basic counter

Concept:

```c
volatile int32_t left_count = 0;
volatile int32_t right_count = 0;
```

Increment/decrement in the encoder callback.

Use sufficiently large signed counters and handle direction consistently.

---

## 5. Convert Counts to Wheel Motion

Let:

```text
N = counts per wheel revolution
C = counts measured during sample
r = wheel radius in metres
T = sample duration in seconds
```

Wheel angle change:

```text
delta_theta = C * 2*pi / N
```

Wheel travel:

```text
distance = r * delta_theta
```

Wheel angular velocity:

```text
wheel_rad_s = delta_theta / T
```

Wheel linear velocity:

```text
wheel_m_s = distance / T
```

Example:

```text
N = 40 counts/rev
r = 0.033 m
C = 4 counts in 0.1 s

wheel rotations = 4/40 = 0.1 rev
angle = 0.1 * 2*pi = 0.628 rad
speed = 0.628 / 0.1 = 6.28 rad/s
linear speed = 6.28 * 0.033 = 0.207 m/s
```

---

## 6. Low-Speed Quantization

Cheap encoders may produce few pulses per wheel revolution.

At low speeds you may see:

```text
sample 1: 0 counts
sample 2: 1 count
sample 3: 0 counts
sample 4: 1 count
```

Instantaneous speed will look noisy.

Solutions:

- calculate over a slightly longer rolling window;
- measure time between pulses;
- use a moving average for reported velocity;
- keep the PID sample rate sensible relative to encoder resolution.

Do not smooth so aggressively that motor response becomes delayed.

---

## 7. Closed-Loop Wheel Speed PID

Open-loop PWM says:

> Apply 35% power.

Closed-loop speed control says:

> Maintain 0.20 m/s.

That is what we need.

For each wheel:

```text
error = target_speed - measured_speed
```

PID output:

```text
u = Kp*error + Ki*integral(error) + Kd*derivative(error)
```

Clamp `u` to the allowed motor command range.

### Best PID learning resource

Brian Douglas — Understanding PID Control playlist:

- https://www.youtube.com/playlist?list=PLn8PRpmsu08pQBgjxYFXSsODEF3Jqmm-y

You mainly need:

- proportional term;
- integral term;
- derivative term;
- saturation;
- integral windup;
- sample time.

---

## 8. Practical PID Tuning Procedure

Tune each wheel independently with the robot lifted first, then on the floor.

### Step 1 — Kp only

Set:

```text
Ki = 0
Kd = 0
```

Increase `Kp` gradually until speed responds quickly but does not oscillate badly.

### Step 2 — Add small Ki

Add enough integral action to remove steady-state speed error.

Too much `Ki` causes slow oscillation/windup.

### Step 3 — Add Kd only if needed

For noisy cheap encoders, derivative can amplify noise. You may find PI control is sufficient.

### Step 4 — Add output limits

```text
-1.0 <= motor_command <= 1.0
```

### Step 5 — Anti-windup

Do not let the integral term grow indefinitely while output is already saturated.

### Step 6 — Test multiple targets

Test:

```text
0.05 m/s
0.10 m/s
0.15 m/s
0.20 m/s
```

and reverse speeds.

Save plots/logs of target vs measured speed.

---

## 9. Straight-Line Matching

Even with the same target speed, motors differ.

Closed-loop PID should make:

```text
left measured speed ~= right measured speed
```

Run a 2 m straight test.

Measure:

- commanded distance;
- actual forward distance;
- sideways deviation;
- final heading error.

Repeat at least 5 times.

Do not tune based on one lucky run.

---

## 10. Differential-Drive Odometry

Over a time step, measure left/right wheel distance:

```text
dl = left wheel travel
dr = right wheel travel
L  = wheel separation
```

Robot center travel:

```text
ds = (dr + dl) / 2
```

Heading change:

```text
dtheta = (dr - dl) / L
```

For small steps:

```text
x += ds * cos(theta + dtheta/2)
y += ds * sin(theta + dtheta/2)
theta += dtheta
```

Normalize heading as needed.

This estimate becomes `nav_msgs/Odometry` on the laptop or can be computed directly in firmware and transmitted.

For our architecture, a good division is:

```text
Pico: count encoders + PID + report counts/speeds
Laptop: calculate/publish ROS odometry
```

That keeps ROS frame/timestamp handling on the computer.

---

## 11. Wheel Radius Calibration

A wheel advertised as 65 mm may not have an effective rolling diameter of exactly 65 mm under load.

### Procedure

1. Put robot on the real floor.
2. Command a long straight movement based on encoder distance.
3. Compare estimated distance with physically measured distance.
4. Adjust effective wheel radius.

If estimated travel is 2.00 m but actual is 1.90 m:

```text
new_radius = old_radius * actual / estimated
```

Repeat until distance error is acceptable.

Do this on the floor used for the final demo if possible.

---

## 12. Wheel Separation Calibration

Wheel separation controls rotation calculation.

### Procedure

1. Command the robot to rotate several full turns based on encoder odometry.
2. Measure actual total rotation.
3. Adjust effective wheel separation until estimated/actual rotation match.

Use multiple rotations, e.g. 5 x 360°, because measuring one turn is noisy.

The effective separation can differ slightly from ruler measurement because of wheel scrub/slip.

---

## 13. ROS Odometry Message

Eventually publish:

```text
Topic: /wheel/odom
Type: nav_msgs/msg/Odometry
header.frame_id = odom
child_frame_id = base_link
```

It should contain:

- x/y pose estimate;
- yaw orientation as quaternion;
- forward velocity;
- angular velocity;
- realistic covariance values.

Do not publish impossible certainty (all-zero covariance) if downstream filters depend on uncertainty.

Nav2 odometry guide:

- https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/odom/setup_odom.html

`diff_drive_controller` reference:

- https://control.ros.org/jazzy/doc/ros2_controllers/diff_drive_controller/doc/userdoc.html

We may implement our own small odometry node first rather than building a full `ros2_control` hardware interface.

---

## 14. Calibration Experiments

Create an `experiments/odometry/` folder and record results.

### Test A — straight 1 m

Run at least 5 times.

Record:

```text
actual distance
estimated distance
lateral error
heading error
battery voltage
floor type
```

### Test B — straight 2 m

Repeat.

### Test C — 360° rotation

Run clockwise and counterclockwise.

### Test D — square

Drive:

```text
1 m forward
90° turn
1 m forward
90° turn
1 m forward
90° turn
1 m forward
90° turn
```

Final robot pose should be close to start.

The square test is extremely important because it exposes systematic wheel/turning error.

---

## 15. Initial Acceptance Targets

These are project targets, not universal robotics standards.

Before Stage 6, aim for roughly:

```text
2 m straight distance error: <= 5–10%
360° rotation error:        <= ~10°
wheel speed tracking:       stable without sustained oscillation
missed encoder counts:      none during normal speed
```

Before phone-free Nav2, Stage 9 will require stronger repeatability.

If errors are bad, fix them now.

Common causes:

- loose encoder disc;
- wheel slip;
- wrong counts/rev;
- wrong radius;
- wrong wheel separation;
- encoder interrupt loss;
- motor PID too aggressive;
- chassis weight imbalance;
- caster drag;
- battery voltage sag.

---

## Stage Acceptance Checklist

- [ ] Encoders are mechanically rigid and do not slip.
- [ ] Counts per wheel revolution measured.
- [ ] Direction/sign is correct for both wheels.
- [ ] Encoder counts do not drop at maximum planned speed.
- [ ] Left wheel speed PID works.
- [ ] Right wheel speed PID works.
- [ ] Straight-line closed-loop driving is repeatable.
- [ ] Wheel radius calibrated.
- [ ] Effective wheel separation calibrated.
- [ ] `/wheel/odom` can be generated from measured motion.
- [ ] 1 m / 2 m / rotation / square experiments are recorded.

---

## Best References

- Brian Douglas PID playlist: https://www.youtube.com/playlist?list=PLn8PRpmsu08pQBgjxYFXSsODEF3Jqmm-y
- Nav2 odometry setup: https://docs.nav2.org/jazzy/configuration_and_development/first_time_robot_setup_guide/odom/setup_odom.html
- ROS 2 `diff_drive_controller`: https://control.ros.org/jazzy/doc/ros2_controllers/diff_drive_controller/doc/userdoc.html
- Pico GPIO API: https://www.raspberrypi.com/documentation/pico-sdk/hardware.html#hardware_gpio
- Pico PIO API: https://www.raspberrypi.com/documentation/pico-sdk/hardware.html#hardware_pio

**Next:** [Stage 06 — Sensors & Sensor Fusion](./06-sensors-and-fusion.md)
