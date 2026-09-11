# Stage 06 — Sensors & Sensor Fusion

## Goal

Integrate the sensors we already own in a way that actually helps navigation.

At the end of this stage:

- MPU6050 data is calibrated and published as `sensor_msgs/Imu`;
- ultrasonic sensors are published as `sensor_msgs/Range`;
- the 5-channel IR array has a measured, documented purpose;
- wheel odometry + IMU angular rate are fused with `robot_localization`;
- `odom -> base_link` is smooth and stable;
- sensors have correct frames and timestamps.

---

# Part A — MPU6050

## 1. What the MPU6050 Can and Cannot Do

The MPU6050 contains:

```text
3-axis accelerometer
3-axis gyroscope
```

It does **not** contain a magnetometer.

For our floor robot, the most useful measurement is usually:

```text
gyroscope Z angular velocity
```

because it helps estimate turning rate.

Do not assume that the MPU6050 gives a permanently accurate absolute yaw heading. Gyroscope integration drifts over time.

Our initial fusion plan is therefore:

```text
encoders -> forward velocity + wheel-derived yaw rate
MPU6050 -> independent gyro Z rate
                  |
                  v
          robot_localization EKF
                  |
                  v
          continuous odometry
```

---

## 2. Wire the MPU6050

Use I2C.

Typical connection:

```text
Pico 3.3 V -> module VCC if module supports it
Pico GND   -> module GND
Pico GP0   -> SDA
Pico GP1   -> SCL
```

Some MPU6050 breakout boards contain regulators/pullups and some clones differ. Check the actual module before applying voltage.

Learn I2C basics:

- device address;
- SDA/SCL;
- register read/write;
- bus pullups;
- units and scaling.

Useful resources:

- Pico I2C API: https://www.raspberrypi.com/documentation/pico-sdk/hardware.html#hardware_i2c
- MPU6050 register map/datasheet: https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf
- Pico examples repository: https://github.com/raspberrypi/pico-examples

---

## 3. Gyroscope Bias Calibration

When stationary, gyro output will not be exactly zero.

At startup:

1. keep robot still;
2. collect around 2–5 seconds of gyro samples;
3. average them;
4. store the average as bias;
5. subtract bias from later readings.

Concept:

```text
gyro_corrected = gyro_raw - gyro_bias
```

Do this for all axes, especially `gz`.

If the robot moves during calibration, reject/restart calibration.

Record the bias from several power cycles. If it varies wildly, investigate sensor/power/noise issues.

---

## 4. Axis Direction

The MPU board's printed axes may not match the robot frame.

ROS convention:

```text
x forward
y left
z up
```

Place the sensor consistently and define `imu_link` in URDF.

Test:

- rotate robot counter-clockwise viewed from above;
- verify `angular_velocity.z` sign matches ROS right-hand-rule convention.

Do not fix an incorrect sign by randomly changing EKF settings. Fix frame orientation or conversion explicitly.

---

# Part B — Ultrasonic Sensors

## 5. Role of Ultrasonic

Ultrasonic is used for:

```text
local obstacle detection
near-field stopping
wall-distance experiments
```

It is **not** our primary global localization sensor.

Ultrasonic problems include:

- wide beam;
- specular reflection;
- soft surfaces;
- angled walls;
- cross-talk between multiple sensors;
- occasional no-return values.

We handle these by filtering and conservative safety thresholds.

---

## 6. Measurement

For HC-SR04-style sensors:

```text
trigger pulse -> sound pulse -> echo time
```

Distance:

```text
distance = speed_of_sound * echo_time / 2
```

The division by 2 is because sound travels to the obstacle and back.

### Important electrical warning

Many modules produce a 5 V echo. Protect Pico 3.3 V GPIO using a voltage divider/level shifter.

### Multiple sensors

Do not trigger all sensors simultaneously.

Example schedule:

```text
t = 0 ms    front trigger
t = 30 ms   left trigger
t = 60 ms   right trigger
```

Adjust spacing to sensor range/environment.

---

## 7. Publish `sensor_msgs/Range`

For each ultrasonic sensor, publish one ROS topic such as:

```text
/range/front
/range/left
/range/right
```

Message type:

```text
sensor_msgs/msg/Range
```

Set correctly:

- `header.frame_id`;
- radiation type = ultrasound;
- field of view;
- minimum range;
- maximum range;
- current range.

Message documentation:

- https://docs.ros.org/en/jazzy/p/sensor_msgs/msg/Range.html

Later Nav2 can consume these through the **Range Sensor Layer**:

- https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/costmap_2d/costmap_plugins/range.html

---

## 8. Filter Invalid Range Data

Define rules for:

```text
timeout / no echo
below minimum range
above maximum range
single unrealistic jump
```

Do not replace every bad measurement with `0`, because zero can look like an obstacle touching the sensor.

Use the correct convention for invalid/max range expected by the ROS message/layer and document it.

A simple median filter over 3–5 recent valid readings can reduce spikes.

---

# Part C — 5-Channel IR Array

## 9. First Identify the Exact Module

A “5-array IR sensor” is often a line-following reflectance array.

Before assigning it a job, determine:

- digital or analog outputs;
- active-high or active-low;
- sensor spacing;
- operating voltage;
- detection distance;
- response on the actual floor.

### Possible uses

**1. Home marker** — recommended if reliable.

Place a deliberate black/white tape stripe at the known home pose. The array detects the marker so the robot can reset/re-align at the start or between cleaning zones.

**2. Cliff/drop detection** — only after real testing.

A reflectance sensor can sometimes detect floor disappearance, but surface colour/shine strongly affects it.

**3. Boundary line** — useful for demo zones.

Do not label it a “cliff sensor” in the report unless the experiments prove it works across the demonstration surfaces.

---

## 10. IR Calibration Procedure

Collect readings for:

```text
normal floor
black tape
white tape
floor edge / drop if safely testable
bright room lighting
dim lighting
```

Record values into:

```text
experiments/ir_calibration.csv
```

For each channel store:

```text
surface, sensor0, sensor1, sensor2, sensor3, sensor4
```

Choose thresholds from data rather than guesswork.

If using it as a home marker, test at different approach angles and speeds.

---

# Part D — ROS Frames and Timestamps

## 11. Sensor Frames

URDF should define frames such as:

```text
base_link
 |-- imu_link
 |-- ultrasonic_front_link
 |-- ultrasonic_left_link
 |-- ultrasonic_right_link
 |-- ir_array_link
```

Every sensor ROS message must use the corresponding frame ID.

Check transforms:

```bash
ros2 run tf2_ros tf2_echo base_link imu_link
```

In RViz, display TF and Range data to verify direction/position visually.

---

## 12. Timestamps

Sensor fusion is time-sensitive.

For the first USB-serial version, the laptop bridge can timestamp incoming measurements when a complete telemetry packet is received.

Better later:

- Pico includes monotonic microsecond/millisecond sample timestamp;
- bridge estimates mapping to ROS time;
- sensor values from one telemetry packet share a consistent acquisition timestamp.

For a uni prototype, receive-time stamping is acceptable if serial latency is small and consistent, but record this limitation in the report.

---

# Part E — Encoder + IMU Fusion

## 13. Why Fuse Sensors?

Wheel encoders:

```text
excellent short-term distance
affected by wheel slip
turning error accumulates
```

MPU6050 gyro:

```text
excellent short-term angular-rate measurement
gyro bias/drift exists
no absolute heading reference
```

Combining them gives a better continuous estimate than using either alone.

Use:

- `robot_localization` EKF;
- `two_d_mode: true`.

Package documentation:

- https://docs.ros.org/en/jazzy/p/robot_localization/
- Project/wiki source: https://github.com/cra-ros-pkg/robot_localization

---

## 14. Recommended First EKF Inputs

Start conservatively.

### Wheel odometry

Use measured forward velocity and wheel-derived yaw rate.

### IMU

Use gyro Z angular velocity.

Do **not** immediately fuse MPU6050 absolute yaw unless you have a trustworthy orientation-estimation pipeline. The sensor has no magnetometer, so absolute yaw will drift.

An initial configuration concept:

```yaml
ekf_filter_node:
  ros__parameters:
    frequency: 30.0
    two_d_mode: true
    publish_tf: true

    map_frame: map
    odom_frame: odom
    base_link_frame: base_link
    world_frame: odom

    odom0: /wheel/odom
    odom0_config: [false, false, false,
                   false, false, false,
                   true,  false, false,
                   false, false, true,
                   false, false, false]

    imu0: /imu/data
    imu0_config: [false, false, false,
                  false, false, false,
                  false, false, false,
                  false, false, true,
                  false, false, false]
```

The 15 configuration booleans represent:

```text
x, y, z,
roll, pitch, yaw,
vx, vy, vz,
vroll, vpitch, vyaw,
ax, ay, az
```

This is a **starting configuration**, not a tuned final config. Verify frame IDs and covariances before trusting it.

---

## 15. Avoid Double-Counting Data

Wheel odometry position and wheel odometry velocity are derived from the same encoder measurements.

Fusing every field just because it exists can make the filter overconfident.

Start with the few measurements we actually trust:

```text
wheel forward velocity
gyro/wheel angular velocity
```

Then add data only when tests show a benefit.

Good background:

- robot_localization sensor prep: https://docs.ros.org/en/noetic/api/robot_localization/html/preparing_sensor_data.html

The concepts remain useful even where the documentation page comes from an older ROS documentation build.

---

## 16. Output

Our EKF should publish:

```text
/odometry/filtered
```

and:

```text
odom -> base_link
```

Inspect:

```bash
ros2 topic echo /odometry/filtered
ros2 topic hz /odometry/filtered
ros2 run tf2_ros tf2_echo odom base_link
```

Plot raw wheel yaw rate, raw gyro Z and filtered yaw rate during turns.

---

## 17. Experiments

### Stationary IMU

Leave robot stationary 2 minutes.

Check:

- gyro noise;
- bias stability;
- filtered odometry should not rotate significantly.

### Slow 360° turn

Compare:

```text
encoder-only rotation
IMU integration
EKF result
physical rotation
```

### Straight 2 m

Check that IMU fusion does not accidentally cause heading instability.

### Square path

Repeat Stage 5 square test using filtered odometry.

Keep raw bag files:

```bash
ros2 bag record /wheel/odom /imu/data /odometry/filtered /range/front
```

---

## Stage Acceptance Checklist

- [ ] MPU6050 gyro bias is calibrated at startup.
- [ ] IMU axes/signs match ROS conventions.
- [ ] `/imu/data` has correct frame/timestamp.
- [ ] Ultrasonic readings are stable enough for obstacle detection.
- [ ] Ultrasonic topics use `sensor_msgs/Range`.
- [ ] Multiple ultrasonic sensors do not cross-talk badly.
- [ ] IR array is characterized on actual demo floor.
- [ ] A reliable use for the IR array is documented.
- [ ] EKF receives wheel + IMU data.
- [ ] `/odometry/filtered` is smooth and continuous.
- [ ] `odom -> base_link` is published exactly once.
- [ ] Straight/turn/square data has been recorded and inspected.

---

## Best References

- `robot_localization`: https://docs.ros.org/en/jazzy/p/robot_localization/
- `robot_localization` GitHub: https://github.com/cra-ros-pkg/robot_localization
- ROS `Range` message: https://docs.ros.org/en/jazzy/p/sensor_msgs/msg/Range.html
- Nav2 Range Layer: https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/costmap_2d/costmap_plugins/range.html
- Pico I2C API: https://www.raspberrypi.com/documentation/pico-sdk/hardware.html#hardware_i2c
- MPU6050 register map: https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf

**Next:** [Stage 07 — Pico ↔ ROS 2 Bridge](./07-pico-ros2-bridge.md)
