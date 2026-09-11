# Stage 08 — Offline iPhone LiDAR Room Mapping

## Goal

Create the room map **once**, save it to the repository, then remove the iPhone completely from the robot system.

At the end of this stage we need:

```text
maps/room.pgm
maps/room.yaml
config/home_pose.yaml
```

The saved 2D map must have correct:

- scale;
- wall/obstacle locations;
- orientation;
- origin;
- known robot home/start pose.

This stage is intentionally **offline mapping**, not onboard LiDAR navigation.

---

# 1. Why This Works

Nav2 does not care whether a static occupancy map came from:

```text
2D LiDAR SLAM
depth camera
hand-drawn floorplan
3D scanner
iPhone LiDAR
```

Once the map is saved as `nav_msgs/OccupancyGrid` / map-server files, Nav2 can load it later.

Apple ARKit scene reconstruction can create a polygonal mesh of the physical environment using LiDAR-capable devices.

Official Apple resources:

- ARKit scene reconstruction: https://developer.apple.com/documentation/arkit/arworldtrackingconfiguration/scenereconstruction
- Reconstructed scene example: https://developer.apple.com/documentation/arkit/visualizing-and-interacting-with-a-reconstructed-scene
- Scene depth / point cloud example: https://developer.apple.com/documentation/arkit/displaying-a-point-cloud-using-scene-depth

Nav2 map server:

- https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/map_server/configuring_map_server/

---

# 2. Important: A 3D Scan Is Not Yet a Robot Map

The iPhone can give us a mesh/point cloud like:

```text
walls
floor
ceiling
chairs
tables
other 3D geometry
```

Nav2 wants a 2D occupancy map like:

```text
white  = free floor
black  = occupied
unknown/gray = unknown
```

So the mapping pipeline is:

```text
iPhone LiDAR
    |
3D mesh / point cloud
    |
clean + align + crop
    |
keep obstacles at robot collision height
    |
project from top into 2D
    |
room.pgm + room.yaml
```

---

# 3. Define the Home Pose BEFORE Scanning

This is essential because after the phone is removed we no longer have a strong global localization sensor.

Create a repeatable physical start location.

Recommended:

1. Put tape on the floor near a wall or corner.
2. Mark both **position and forward direction**.
3. Make a small docking jig/guide if possible so wheel position is repeatable.
4. If the IR array reliably detects black/white tape, design a home stripe the robot can detect.
5. Photograph and measure the home location relative to two room walls.

Example:

```text
wall
=================================

       0.40 m
         |
         v
        [ H ] ----> robot forward
         ^
         |
       0.30 m from side wall
```

The physical home marker is part of the localization system. Do not treat it as decoration.

---

# 4. Measure Reference Distances Manually

Before scanning, use a tape measure to record at least:

```text
room wall-to-wall length
room wall-to-wall width
home-to-wall distance
one large obstacle dimension
```

Store these in:

```text
hardware/room_reference_measurements.md
```

Why?

Because a beautiful 3D scan with the wrong scale is useless for navigation.

After map generation, we compare the digital distances against these measurements.

---

# 5. Recommended Capture Route — No Custom iOS App First

The easiest route is to use an iPhone LiDAR scanning app that exports a normal 3D mesh.

## Option A — Polycam Space Mode

As of September 2026, Polycam supports LiDAR Space Mode on compatible iPhones and its free tier supports GLTF export. Export options/plans can change, so check the app before depending on a paid-only format.

Resources:

- Space Mode guide: https://learn.poly.cam/hc/en-us/articles/36655587097620-How-to-Use-Space-Mode-LiDAR-Devices
- Export guide: https://learn.poly.cam/hc/en-us/articles/29647691255316-How-to-Export-Polycam-Captures
- Polycam open-source raw-data tooling (`polyform`): https://github.com/PolyCam/polyform

GLTF is enough because Blender can import it and export other formats locally.

## Option B — Another app with free mesh export

For example, the ScanDo project advertises free OBJ/DAE/STL export:

- https://github.com/TortoiseWolfe/ScanDo

We are **not** architecturally tied to one scanning app. We need a metric room mesh we can export.

---

# 6. How to Scan the Room Well

Prepare the room:

- remove people/pets;
- move temporary clutter that should not be part of the permanent map;
- leave fixed furniture that the robot cannot drive through;
- turn lights on evenly;
- avoid mirrors/glass where possible;
- open/close doors in the state expected for the demo.

During scanning:

1. Start near the physical home marker.
2. Hold phone steadily.
3. Walk slowly around the perimeter.
4. Capture every wall/corner.
5. Capture fixed furniture near floor level.
6. Loop back to already scanned regions.
7. Do not whip/rotate the phone quickly.
8. Make sure floor/wall geometry is complete.
9. Finish near the starting area if convenient.

Make **at least 2 independent room scans**. Never trust the only scan you have.

Choose the cleaner one after comparison.

---

# 7. Export the Mesh

Preferred file:

```text
room_scan.glb
```

or:

```text
room_scan.gltf
room_scan.obj
```

Store raw scans outside Git if they are huge, or use Git LFS later. Keep at least a copy in project storage.

Recommended project structure:

```text
mapping/
├── raw/
│   └── room_scan.glb
├── processed/
│   └── room_cleaned.blend
└── notes/
    └── scan_session.md
```

Do not overwrite the raw scan while cleaning it.

---

# 8. Clean the Scan in Blender

Blender is free and excellent for this job:

- https://www.blender.org/
- Blender manual: https://docs.blender.org/manual/en/latest/

Import the GLTF/OBJ.

### First checks

1. Is the floor horizontal?
2. Which axis is up?
3. Is the model in metres?
4. Do measured wall distances match reality?
5. Are there duplicated/floating surfaces?

### Align the model

Make the floor approximately:

```text
Z = 0
```

Align major room walls with X/Y where practical. This makes map generation and home-pose definition simpler.

### Clean only obvious scan garbage

Remove:

- floating isolated fragments;
- duplicated accidental geometry;
- ceiling if it blocks top view;
- geometry far outside the room.

Do not “beautify” the room into a different shape.

---

# 9. Build a Robot-Height Obstacle Slice

A cleaning robot does not care about the ceiling.

It cares about objects intersecting its body height.

Measure actual robot body height later. For initial processing, think in terms of a vertical slice such as:

```text
Z = 0.02 m to robot_height
```

Example if robot is ~0.20 m high:

```text
keep occupied geometry from ~0.02 m to ~0.22 m
```

Why start slightly above zero?

Because the floor itself should not become an obstacle.

The final values depend on:

- chassis height;
- bumper/brush geometry;
- clearance under furniture.

Important examples:

```text
wall -> occupied
chair leg -> occupied
cabinet base -> occupied
floor -> free
table top 0.75 m high -> not itself an obstacle
but table legs -> occupied
```

---

# 10. Convert the Slice to a 2D Occupancy Image

For a university project, use a simple top-down rasterization workflow.

## Recommended resolution

Start with:

```text
0.05 m/pixel
```

meaning each pixel represents 5 cm.

For a 6 m x 5 m room:

```text
width  = 6 / 0.05 = 120 pixels
height = 5 / 0.05 = 100 pixels
```

This is sufficient for a small cleaning robot and keeps maps manageable.

### Map colours

Standard ROS map convention:

```text
black  -> occupied
white  -> free
gray   -> unknown
```

### Practical Blender workflow

1. Position an orthographic camera directly above the room.
2. Point it down perpendicular to the floor.
3. Set orthographic bounds to known room dimensions.
4. Set render pixel dimensions from chosen map resolution.
5. Render obstacle geometry as solid black.
6. Render known interior floor as white.
7. Leave regions that truly were not observed as gray if desired.
8. Export losslessly as PNG.
9. Convert PNG to grayscale PGM if needed.

You can also automate this later with Python/Blender scripting once the manual process is understood.

For our first map, **manual inspection is more important than automation**.

---

# 11. Create the ROS Map YAML

Example `maps/room.yaml`:

```yaml
image: room.pgm
mode: trinary
resolution: 0.05
origin: [0.0, 0.0, 0.0]
negate: 0
occupied_thresh: 0.65
free_thresh: 0.25
```

Meaning:

### `resolution`

Metres per pixel.

### `origin`

Pose of the image's lower-left pixel in the ROS `map` frame:

```text
[x, y, yaw]
```

### `occupied_thresh` / `free_thresh`

Thresholds used to classify pixels.

Nav2 map-server docs:

- https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/map_server/configuring_map_server/

---

# 12. Make Home Pose Simple

The cleanest project setup is to design the map so the physical home pose is easy to express.

For example:

```yaml
# config/home_pose.yaml
x: 0.50
y: 0.50
yaw: 0.0
```

or choose map coordinates so home is effectively:

```text
x = 0
y = 0
yaw = 0
```

Do not force origin placement if it makes the map awkward, but **document exactly how physical home maps to ROS coordinates**.

Take a photo showing robot placement and forward direction.

---

# 13. Load the Map in ROS

Install Nav2 map server with your ROS Jazzy setup.

Launch `map_server` using the official Nav2 instructions/configuration.

The result should publish:

```text
/map
```

message type:

```text
nav_msgs/msg/OccupancyGrid
```

Check:

```bash
ros2 topic echo /map --once
ros2 topic info /map
```

Display the map in RViz.

---

# 14. Map Validation — Mandatory

Do not accept the map because it “looks right.”

### Scale validation

Pick two points on a wall and compare:

```text
real distance
digital map distance
```

Aim for only a few percent error.

### Obstacle validation

Check that:

- walls are continuous;
- robot-sized gaps are realistic;
- fixed furniture bases appear;
- phantom scan artifacts are removed;
- free corridors are not accidentally occupied.

### Robot footprint validation

In RViz, compare robot footprint to doorway/gap sizes.

A map can be geometrically correct but still unusable if a 40 cm-wide robot is asked to enter a 35 cm gap.

---

# 15. Optional Advanced ROS Mapping Route

If the team wants deeper SLAM work for academic value, use the iPhone data directly in ROS during the one-time scan.

Useful reference project:

- iPhone LiDAR + ROS 2 SLAM playground: https://github.com/MatthewKazan/iPhone-lidar-slam-playground

The project contains an iOS ARKit app that sends point clouds to ROS 2 through rosbridge. It is a **reference implementation**, not guaranteed to be drop-in for Jazzy; its README documents its tested environment.

Useful packages:

- rosbridge suite: https://github.com/RobotWebTools/rosbridge_suite
- RTAB-Map ROS 2: https://github.com/introlab/rtabmap_ros/tree/ros2
- OctoMap ROS mapping: https://github.com/OctoMap/octomap_mapping/tree/ros2

RTAB-Map can build/export occupancy grids from RGB-D, stereo or LiDAR data. OctoMap can consume `PointCloud2` and provide a projected 2D occupancy map.

If this route works cleanly, save the resulting 2D map with Nav2 map saver.

Map saver docs:

- https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/map_server/configuring_map_saver/

Typical CLI concept:

```bash
ros2 run nav2_map_server map_saver_cli -f maps/room
```

If saving another map topic such as OctoMap's projection, remap the `map` input appropriately.

### Which route should we use?

For finishing the robot:

```text
mesh export -> 2D occupancy map
```

is the baseline.

For extra research/SLAM content:

```text
ARKit/ROS point clouds -> RTAB-Map/OctoMap -> occupancy map
```

is the advanced path.

Do not block the whole project on the advanced mapping path.

---

# 16. What Gets Committed

Commit:

```text
maps/room.pgm
maps/room.yaml
config/home_pose.yaml
hardware/room_reference_measurements.md
mapping/notes/scan_session.md
```

If the 3D scan is large, do not commit it normally to GitHub. Store it separately or use Git LFS later.

`scan_session.md` should record:

```text
date
room
scan app/method
iPhone model
map resolution
reference measurements
map-generation procedure
known limitations
```

---

## Stage Acceptance Checklist

- [ ] Physical home pose exists and is repeatable.
- [ ] At least two room reference distances measured manually.
- [ ] At least two LiDAR scans captured.
- [ ] Best scan exported in a usable metric format.
- [ ] Scan cleaned/aligned without changing room geometry.
- [ ] Robot-height obstacle slice selected.
- [ ] 2D occupancy image created.
- [ ] `room.yaml` has correct resolution/origin.
- [ ] `/map` loads successfully in ROS/RViz.
- [ ] Digital distances match physical measurements acceptably.
- [ ] Doors/gaps/walls look correct relative to robot footprint.
- [ ] Home pose in the map is documented.
- [ ] The iPhone can now be completely removed from the runtime robot.

---

## Best References

- Apple ARKit scene reconstruction: https://developer.apple.com/documentation/arkit/arworldtrackingconfiguration/scenereconstruction
- Apple reconstructed-scene sample: https://developer.apple.com/documentation/arkit/visualizing-and-interacting-with-a-reconstructed-scene
- Polycam Space Mode: https://learn.poly.cam/hc/en-us/articles/36655587097620-How-to-Use-Space-Mode-LiDAR-Devices
- Polycam `polyform`: https://github.com/PolyCam/polyform
- Blender manual: https://docs.blender.org/manual/en/latest/
- Nav2 Map Server: https://docs.nav2.org/jazzy/configuration_and_development/configuration_guide/core_servers/map_server/configuring_map_server/
- RTAB-Map ROS 2: https://github.com/introlab/rtabmap_ros/tree/ros2
- OctoMap mapping: https://github.com/OctoMap/octomap_mapping/tree/ros2
- iPhone LiDAR ROS reference: https://github.com/MatthewKazan/iPhone-lidar-slam-playground

**Next:** [Stage 09 — Phone-Free Localization](./09-phone-free-localization.md)
