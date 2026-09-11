# Autonomous Cleaning Robot — Learning Pathway

This folder is the detailed build-and-learning sequence for the project. Work through the files **in order**. Each stage has a specific output and acceptance test so we do not jump ahead with half-working subsystems.

## Final Architecture We Are Building

The iPhone is **not mounted on the robot**.

### Mapping day

1. Walk around the room with the iPhone LiDAR.
2. Capture ARKit LiDAR/depth/pose data.
3. Build or export a 3D scan.
4. Convert it into a ROS 2D occupancy map.
5. Validate scale and mark a known robot home pose.
6. Save the map.
7. Put the iPhone away.

### Cleaning day

The robot starts from the same physical home pose and uses:

- wheel encoders for distance/rotation;
- MPU6050 gyro for angular-rate correction;
- an EKF for continuous odometry;
- ultrasonic sensors for local obstacle detection;
- the IR array for floor safety and/or a known home marker;
- Nav2 on the saved occupancy map;
- a coverage planner to clean the complete area.

This is viable for a **small, controlled university environment**, but it is dead-reckoning based after the phone is removed. It will accumulate drift. Stage 9 is therefore mandatory and contains the rules for deciding whether the drift is acceptable and the cheapest fallback if it is not.

---

## Pathway

| # | Stage | Main result |
|---:|---|---|
| 01 | [Linux, Git & Workspace](./01-linux-git-workspace.md) | Stable Ubuntu/ROS development environment |
| 02 | [ROS 2 Fundamentals](./02-ros2-fundamentals.md) | Understand ROS graph and write basic packages |
| 03 | [Robot Model, TF & Simulation](./03-robot-model-tf-simulation.md) | Working differential-drive simulation |
| 04 | [Pico, Electronics & Drive Base](./04-pico-electronics-drive-base.md) | Safe low-level motor control |
| 05 | [Encoders, PID & Wheel Odometry](./05-encoders-pid-odometry.md) | Measured wheel motion and closed-loop speed |
| 06 | [Sensors & Sensor Fusion](./06-sensors-and-fusion.md) | MPU6050, ultrasonic and IR integrated correctly |
| 07 | [Pico ↔ ROS 2 Bridge](./07-pico-ros2-bridge.md) | Laptop controls real base over USB and receives telemetry |
| 08 | [Offline iPhone LiDAR Mapping](./08-offline-iphone-lidar-mapping.md) | `room.yaml` + `room.pgm` static map |
| 09 | [Phone-Free Localization](./09-phone-free-localization.md) | Known-start encoder/IMU navigation with measured drift |
| 10 | [Nav2 Navigation](./10-nav2-navigation.md) | Point-to-point autonomous navigation with range obstacles |
| 11 | [Coverage & Cleaning](./11-coverage-and-cleaning.md) | Systematic room coverage + vacuum/brush |
| 12 | [Integration & Testing](./12-integration-testing.md) | Repeatable demo with measured performance |
| 13 | [Novelty / AI](./13-novelty-ai.md) | Optional innovation after the core system works |

---

## Golden Rules

1. **Do not buy a LiDAR.** The project is specifically designed around one-time iPhone scanning.
2. **Do buy encoders.** Runtime localization depends heavily on good wheel feedback.
3. **Do not use AMCL without a real runtime scan source.** Our base design does not have one after the iPhone is removed.
4. **Do not treat ultrasonic as a replacement for LiDAR localization.** Ultrasonic is excellent for local collision avoidance but sparse/noisy for global localization.
5. **Always start normal cleaning from the same known home pose.** That is what aligns the saved map with encoder/IMU odometry.
6. **Measure drift before integrating Nav2.** If the robot cannot repeat a simple square path, navigation will not magically fix it.
7. **Keep the Pico responsible for real-time safety.** A lost serial command must stop the wheels even if ROS crashes.
8. **Use simulation first.** Every navigation concept should work in Gazebo before debugging the real robot.
9. **Keep velocities low.** This improves encoder accuracy, stopping distance and demo safety.
10. **Add AI last.** A reliable robot is a much stronger project than an unreliable robot with an LLM attached.

---

## Minimum Success Criteria for the Final Project

A good university-demo target is:

- static room map created once with iPhone LiDAR;
- iPhone removed during all autonomous cleaning demonstrations;
- robot starts from a repeatable marked home pose;
- wheel velocity PID works on both motors;
- fused encoder/IMU odometry remains usable for one cleaning run;
- robot follows at least several Nav2 goals without collision;
- ultrasonic obstacles appear in the local costmap;
- coverage planner follows parallel cleaning lanes;
- vacuum/brush can be enabled and disabled by software;
- communication loss stops the robot;
- repeatability and drift are measured rather than guessed.

If those work, the project already demonstrates embedded systems, control, ROS 2, mapping, navigation and autonomous planning without needing an expensive onboard LiDAR.
