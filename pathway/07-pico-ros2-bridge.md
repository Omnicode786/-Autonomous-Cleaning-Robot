# Stage 07 — Pico ↔ ROS 2 Bridge

## Goal

Connect the real Pico drive base to the ROS 2 computer using a simple, debuggable USB serial protocol.

At the end of this stage:

- ROS velocity commands control the physical wheels;
- Pico telemetry becomes normal ROS topics;
- encoders, IMU, ultrasonic, IR and battery data reach ROS;
- a lost ROS/USB connection stops the robot;
- recorded ROS data can reproduce sensor-side debugging.

For the first version, **simple USB serial is preferred over micro-ROS**. micro-ROS remains a later upgrade.

---

## 1. Why Start With a Serial Bridge?

A custom serial bridge is easy to inspect:

```text
ROS node -> plain text packet -> Pico
Pico -> plain text telemetry -> ROS node
```

When something breaks, we can test each side independently with a serial terminal.

micro-ROS is powerful, but it adds:

- build-system complexity;
- agent configuration;
- transport setup;
- more difficult first-time debugging.

We should first make the robot reliable, then decide whether micro-ROS adds enough value.

Reference for later:

- micro-ROS Pico SDK example: https://github.com/micro-ROS/micro_ros_raspberrypi_pico_sdk
- micro-ROS tutorials: https://micro.ros.org/docs/tutorials/core/first_application_linux/

---

## 2. Install Python Serial Support

On the laptop, use `pyserial`.

Documentation:

- https://pyserial.readthedocs.io/en/latest/

Install using the package method appropriate for the ROS environment. Avoid mixing system Python/pip packages carelessly on modern Ubuntu.

Check the Pico port:

```bash
ls /dev/ttyACM*
dmesg | tail
```

Typical:

```text
/dev/ttyACM0
```

Test manually with a serial tool before writing ROS code.

---

## 3. Define One Protocol and Freeze It

Do not invent a new packet format every week.

Use newline-delimited ASCII for version 1.

### Laptop -> Pico command

```text
C,<seq>,<left_target_mps>,<right_target_mps>,<vacuum>,<brush>
```

Example:

```text
C,1042,0.150,0.150,1,1
```

Meaning:

```text
packet type: C
sequence: 1042
left target: 0.15 m/s
right target: 0.15 m/s
vacuum: on
brush: on
```

Stop:

```text
C,1043,0.000,0.000,0,0
```

### Pico -> Laptop telemetry

```text
S,<seq>,<pico_ms>,<left_count>,<right_count>,<left_mps>,<right_mps>,<gyro_z>,<range_front>,<ir_mask>,<battery_v>
```

Example:

```text
S,5521,237844,10325,10298,0.149,0.151,-0.004,0.82,4,11.72
```

If we have multiple ultrasonic sensors, add clearly named/ordered fields and document the packet version.

---

## 4. Protocol Rules

Every packet should follow these rules:

1. Exactly one line per packet.
2. Every packet begins with a type letter.
3. Every packet has a sequence number.
4. Invalid packets are ignored, never partially applied.
5. Motion commands have range checks.
6. The Pico stores time of last valid command.
7. Missing commands trigger stop.
8. Debug messages use a different prefix, for example `D,`.
9. Version changes are documented.

Later, if corruption becomes a real problem, add a checksum/CRC. Do not add complexity before evidence says it is needed.

---

## 5. Decide Where Kinematics Run

Recommended division:

### Laptop

Receives robot-level command:

```text
linear.x = v
angular.z = omega
```

Converts it to wheel target speeds:

```text
left  = v - omega * L/2
right = v + omega * L/2
```

Sends wheel targets to Pico.

### Pico

Runs:

```text
left target -> left PID -> left PWM
right target -> right PID -> right PWM
```

This keeps ROS conventions and robot geometry parameters on the laptop while the real-time loop stays on the microcontroller.

---

## 6. ROS Bridge Package

Create:

```text
src/cleanbot_bridge/
├── package.xml
├── setup.py
├── setup.cfg
├── resource/
├── launch/
│   └── bridge.launch.py
├── config/
│   └── bridge.yaml
└── cleanbot_bridge/
    ├── __init__.py
    ├── serial_bridge.py
    └── odometry.py
```

Suggested parameters:

```yaml
serial_port: /dev/ttyACM0
baud_rate: 115200
wheel_radius: 0.033
wheel_separation: 0.160
command_rate_hz: 20.0
command_timeout_s: 0.5
```

Use measured/calibrated values, not these placeholders.

---

## 7. ROS Interfaces

The bridge should eventually interact with:

### Subscriptions

```text
/cmd_vel                     robot velocity command
/cleanbot/vacuum_enable      cleaning command
/cleanbot/brush_enable       cleaning command
```

Confirm the exact `/cmd_vel` message type used by your Nav2 configuration with:

```bash
ros2 topic info /cmd_vel
```

Write the bridge around the actual interface rather than assuming an old tutorial's type.

### Publications

```text
/wheel/odom                  nav_msgs/Odometry
/imu/data                    sensor_msgs/Imu
/range/front                 sensor_msgs/Range
/range/left                  sensor_msgs/Range (if present)
/range/right                 sensor_msgs/Range (if present)
/ir_array                    custom/simple topic
/battery_voltage             std_msgs/Float32 or BatteryState later
/cleanbot/status             diagnostics/status
```

A cleaner future battery interface is `sensor_msgs/msg/BatteryState`.

---

## 8. Serial Reader Architecture

Do not block the ROS executor indefinitely waiting for one serial line.

Options:

- background Python thread reads serial and pushes parsed packets into a thread-safe queue;
- non-blocking serial polling from a timer;
- asyncio if the team is comfortable with it.

For beginners, a dedicated serial-reading thread plus ROS timers is understandable.

Architecture:

```text
serial thread
    |
parse telemetry
    |
thread-safe latest state / queue
    |
ROS timer publishes messages
```

Command path:

```text
/cmd_vel callback
    |
compute wheel targets
    |
store latest desired command
    |
20 Hz serial-write timer
```

A fixed command rate avoids sending unpredictable bursts.

---

## 9. Parsing Safely

Never assume a line is valid.

Validate:

```text
correct prefix
correct number of fields
integer fields parse
float fields parse
ranges are sensible
sequence/time is sensible
```

If a telemetry packet is corrupt:

```text
log warning -> discard packet -> continue
```

Do not crash the whole bridge because one serial line is malformed.

---

## 10. Convert Telemetry to ROS Messages

### Wheel odometry

Use encoder counts/speeds and calibrated geometry from Stage 5.

Publish:

```text
header.frame_id = odom
child_frame_id  = base_link
```

If `robot_localization` will publish `odom -> base_link`, configure the raw wheel odometry node **not** to publish the same TF. Only one node should own each transform.

### IMU

Convert units to SI:

```text
angular velocity -> rad/s
linear acceleration -> m/s^2
```

Set:

```text
header.frame_id = imu_link
```

### Ultrasonic

Publish `sensor_msgs/Range` with each sensor's own frame.

### IR

For the first version, a compact message is acceptable. Options:

```text
std_msgs/UInt8 bit mask
Int32MultiArray
custom CleanbotIr message later
```

Do not over-engineer it unless necessary.

---

## 11. Sequence Numbers and Drop Detection

Sequence numbers help detect missing data.

Example:

```text
100
101
102
105
```

means telemetry packets `103` and `104` were missed.

Track:

```text
packets received
packets dropped
parse errors
last telemetry age
```

Publish diagnostics or log periodically.

This becomes useful evidence in the final report.

---

## 12. Two Independent Safety Timeouts

### Pico-side watchdog — mandatory

If the Pico receives no valid command for ~0.5 s:

```text
wheel targets = 0
vacuum/brush policy = safe state
```

### ROS-side telemetry timeout

If the bridge receives no Pico telemetry for a configured period:

```text
log ERROR
stop sending nonzero commands
report robot disconnected
```

Safety must not depend only on the laptop.

---

## 13. Bring-Up Testing Order

### Test A — fake Pico

Before moving wheels, feed sample telemetry strings into the parser.

Test:

- valid packets;
- missing field;
- invalid number;
- huge value;
- blank line;
- debug line.

### Test B — Pico connected, wheels lifted

Use a manually published command.

Example concept:

```bash
ros2 topic pub ... /cmd_vel ...
```

Use the exact message type shown by `ros2 topic info`.

Verify wheel targets change.

### Test C — watchdog

Send forward command, then stop the ROS bridge.

Wheels must stop automatically.

### Test D — sensor topics

Inspect:

```bash
ros2 topic hz /imu/data
ros2 topic echo /range/front
ros2 topic hz /wheel/odom
```

### Test E — floor motion

Set very low velocity limits and teleoperate the physical robot.

---

## 14. Record a Hardware Bag

Record:

```bash
ros2 bag record \
  /cmd_vel \
  /wheel/odom \
  /imu/data \
  /range/front \
  /odometry/filtered
```

Add other range topics if present.

Then stop the robot and replay the bag for software debugging.

The ability to debug without repeatedly running the physical robot is a major productivity improvement.

---

## 15. Optional Upgrade: micro-ROS

Only consider micro-ROS after the serial architecture works.

Advantages:

- Pico can expose ROS-style publishers/subscribers directly;
- typed messages;
- standard ROS graph integration.

Costs:

- more firmware/build complexity;
- micro-ROS agent required;
- more difficult debugging.

For the final university project, a robust documented serial protocol is completely acceptable.

---

## Common Problems

### Permission denied on `/dev/ttyACM0`

Check `dialout` group and device permissions.

### Pico port number changes

Use a udev rule later to create a stable device symlink such as:

```text
/dev/cleanbot_pico
```

### ROS commands lag

Check:

- serial read blocking;
- command frequency;
- huge debug printing;
- bridge callback design.

### Random resets when motors run

This is almost certainly power/noise/hardware first, not ROS.

### Robot moves after ROS crashes

Pico watchdog is missing or broken. Fix immediately.

---

## Stage Acceptance Checklist

- [ ] Protocol format is documented and versioned.
- [ ] ROS bridge connects to Pico automatically.
- [ ] `/cmd_vel` can move both physical wheels.
- [ ] Wheel targets are converted using correct geometry.
- [ ] Encoder telemetry reaches ROS.
- [ ] `/imu/data` reaches ROS at a stable rate.
- [ ] Ultrasonic data reaches `sensor_msgs/Range` topics.
- [ ] Invalid serial lines do not crash the bridge.
- [ ] Sequence/drop statistics are observable.
- [ ] Pico watchdog stops motors when bridge disappears.
- [ ] ROS side detects missing telemetry.
- [ ] A rosbag of a real driving test has been recorded.

---

## Best References

- pySerial: https://pyserial.readthedocs.io/en/latest/
- ROS 2 Python tutorials: https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries.html
- ROS message types: https://docs.ros.org/en/jazzy/p/common_interfaces/
- micro-ROS Pico reference: https://github.com/micro-ROS/micro_ros_raspberrypi_pico_sdk
- micro-ROS docs: https://micro.ros.org/docs/

**Next:** [Stage 08 — Offline iPhone LiDAR Mapping](./08-offline-iphone-lidar-mapping.md)
