# Stage 02 — ROS 2 Fundamentals

## Goal

Understand ROS 2 well enough that later packages such as Nav2, RTAB-Map and our Pico bridge stop looking like magic.

At the end you should understand:

- nodes;
- topics and messages;
- publishers/subscribers;
- services;
- actions;
- parameters;
- packages/workspaces;
- launch files;
- QoS at a practical level;
- rosbag2;
- basic ROS debugging commands.

---

## 1. Mental Model: ROS Is a Graph

A ROS robot is not one giant program. It is many processes communicating using standardized interfaces.

Our final graph will look roughly like:

```text
Nav2 ---- /cmd_vel ----> cleanbot_bridge ---- serial ----> Pico
                             |
                             +---- /wheel/odom
                             +---- /imu/data
                             +---- /range/front
                             +---- /ir_array

wheel odom + IMU ----> robot_localization ----> /odometry/filtered

map_server ----> /map ----> Nav2
```

The most useful question while debugging ROS is:

> Which node should publish this data, what topic/type is it using, and which node should consume it?

---

## 2. Best Learning Sequence

### Primary resource

Work through the official beginner tutorials in order:

- https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools.html
- https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries.html

### Video companion

Articulated Robotics ROS 2 Fundamentals playlist:

- https://www.youtube.com/playlist?list=PLunhqkrRNRhYYCaSTVP-qJnyUPkTxJnBt

Use videos to understand the idea; use **Jazzy documentation** for current commands/API details.

---

## 3. Nodes and Topics

Run the standard talker/listener example in two terminals.

Terminal 1:

```bash
ros2 run demo_nodes_cpp talker
```

Terminal 2:

```bash
ros2 run demo_nodes_py listener
```

Inspect it:

```bash
ros2 node list
ros2 topic list
ros2 topic info /chatter
ros2 topic echo /chatter
ros2 topic hz /chatter
ros2 node info /talker
```

Understand:

```text
publisher -> topic -> subscriber
```

A topic has a **message type**. Nodes do not simply send arbitrary Python objects.

Useful message definitions for this project:

- `geometry_msgs/msg/Twist` or `TwistStamped` — commanded robot velocity;
- `nav_msgs/msg/Odometry` — robot motion estimate;
- `sensor_msgs/msg/Imu` — MPU6050 data;
- `sensor_msgs/msg/Range` — ultrasonic reading;
- `nav_msgs/msg/OccupancyGrid` — 2D map;
- `geometry_msgs/msg/PoseStamped` — goal/pose;
- `sensor_msgs/msg/Image`, `CameraInfo`, `PointCloud2` — mapping data.

Inspect message definitions:

```bash
ros2 interface show geometry_msgs/msg/Twist
ros2 interface show nav_msgs/msg/Odometry
ros2 interface show sensor_msgs/msg/Range
```

---

## 4. Services vs Actions

### Services

Use when you ask for one quick request/response.

Example future uses:

```text
reset_odometry
set_vacuum_enabled
save_map
```

Learn:

```bash
ros2 service list
ros2 service type <service>
ros2 service call ...
```

### Actions

Use for tasks that take time, provide feedback and can be cancelled.

Nav2 navigation is action-based because “drive to this location” takes time.

Examples:

```text
NavigateToPose
FollowWaypoints
```

Learn:

```bash
ros2 action list
ros2 action info <action>
```

Official concept guide:

- https://docs.ros.org/en/jazzy/How-To-Guides/Topics-Services-Actions.html

---

## 5. Create Our First Package

Create a Python package:

```bash
cd ~/cleanbot_ws/src
ros2 pkg create --build-type ament_python cleanbot_core --dependencies rclpy std_msgs geometry_msgs
```

Build:

```bash
cd ~/cleanbot_ws
colcon build --symlink-install
source install/setup.bash
```

The package should contain roughly:

```text
cleanbot_core/
├── package.xml
├── setup.py
├── setup.cfg
├── resource/
└── cleanbot_core/
    ├── __init__.py
    ├── velocity_test_pub.py
    └── velocity_test_sub.py
```

---

## 6. Write a Minimal Publisher

Create `cleanbot_core/velocity_test_pub.py`:

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist


class VelocityTestPublisher(Node):
    def __init__(self):
        super().__init__('velocity_test_publisher')
        self.pub = self.create_publisher(Twist, '/test_cmd_vel', 10)
        self.timer = self.create_timer(0.5, self.tick)

    def tick(self):
        msg = Twist()
        msg.linear.x = 0.10
        msg.angular.z = 0.0
        self.pub.publish(msg)


def main():
    rclpy.init()
    node = VelocityTestPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

Add the executable entry point to `setup.py`, rebuild, then run it.

Follow the official Python publisher/subscriber tutorial for the exact `setup.py` format:

- https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html

Inspect:

```bash
ros2 topic echo /test_cmd_vel
ros2 topic hz /test_cmd_vel
```

---

## 7. Parameters

Parameters are configuration values owned by a node.

Future examples:

```text
wheel_radius
wheel_separation
serial_port
encoder_counts_per_rev
max_linear_speed
vacuum_pwm
```

Learn:

```bash
ros2 param list
ros2 param get <node> <parameter>
ros2 param set <node> <parameter> <value>
```

Prefer parameters/config files over hardcoding robot dimensions into many files.

---

## 8. Launch Files

A real robot requires many nodes. Do not start every node manually.

A launch file should eventually start:

```text
robot_state_publisher
Pico bridge
EKF
map server
Nav2
coverage node
```

Learn Python launch files here:

- https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Launch-Main.html

Create a simple `bringup_test.launch.py` that starts your publisher and subscriber together.

Important lesson: **launch files start/configure systems; nodes should still be independently testable.**

---

## 9. QoS — Learn Only What We Need

ROS 2 Quality of Service controls how messages are delivered.

Understand these terms:

- reliability: reliable vs best effort;
- durability: volatile vs transient local;
- history/depth;
- deadline/liveliness conceptually.

Typical intuition:

```text
commands/configuration -> reliable
high-rate sensor streams -> often best effort
static map -> transient-local behavior is useful
```

Do not randomly change QoS to fix a broken topic. First check publisher/subscriber compatibility.

Official guide:

- https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Quality-of-Service-Settings.html

Inspect QoS with:

```bash
ros2 topic info /topic_name --verbose
```

---

## 10. rosbag2

rosbag2 records ROS topics so we can reproduce tests without the physical hardware.

This will be extremely useful for:

- iPhone mapping data;
- encoder/IMU calibration;
- ultrasonic debugging;
- final demonstration evidence.

Practice:

```bash
ros2 bag record /test_cmd_vel
```

Stop with Ctrl+C, then:

```bash
ros2 bag info <bag_folder>
ros2 bag play <bag_folder>
```

Documentation:

- https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data.html

---

## 11. Debugging Toolbox

Memorize these:

```bash
ros2 node list
ros2 node info /node
ros2 topic list
ros2 topic type /topic
ros2 topic info /topic --verbose
ros2 topic echo /topic
ros2 topic hz /topic
ros2 service list
ros2 action list
ros2 param list
rqt_graph
```

When something fails, check in this order:

1. Is the expected node running?
2. Is the topic present?
3. Is the message type correct?
4. Is data actually arriving?
5. Is its rate sensible?
6. Are frame IDs/timestamps valid?
7. Is QoS compatible?

This process will save days later.

---

## 12. Exercises

### Exercise A

Create two nodes:

```text
/fake_distance_publisher -> sensor_msgs/Range
/range_warning_node -> prints warning if distance < 0.30 m
```

### Exercise B

Turn the warning distance into a ROS parameter instead of a hardcoded value.

### Exercise C

Start both nodes from one launch file.

### Exercise D

Record the range topic with rosbag2 and replay it while the publisher is stopped.

If you can do these without copying every line blindly, you understand enough ROS to proceed.

---

## Stage Acceptance Checklist

- [ ] Understand nodes/topics/messages.
- [ ] Can write Python publisher/subscriber nodes.
- [ ] Understand when to use services vs actions.
- [ ] Can build packages with `colcon`.
- [ ] Can use ROS parameters.
- [ ] Can write a basic launch file.
- [ ] Can inspect a topic's type/rate/QoS.
- [ ] Can record and replay a rosbag.
- [ ] Can use `rqt_graph` to inspect the ROS graph.

---

## Best References

- ROS 2 Jazzy tutorials: https://docs.ros.org/en/jazzy/Tutorials.html
- ROS 2 concepts: https://docs.ros.org/en/jazzy/Concepts.html
- Articulated Robotics ROS 2 playlist: https://www.youtube.com/playlist?list=PLunhqkrRNRhYYCaSTVP-qJnyUPkTxJnBt
- ROS message definitions: https://docs.ros.org/en/jazzy/p/common_interfaces/

**Next:** [Stage 03 — Robot Model, TF & Simulation](./03-robot-model-tf-simulation.md)
