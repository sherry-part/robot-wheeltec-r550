# Wheeltec R550 mini_tank

Robonix deployment for the SysWonder Wheeltec R550 mini_tank: Jetson Orin,
LSLIDAR N10P, Orbbec Astra S, RTAB-Map, Scene, Nav2, Explore, and VLM pilot.

The deployment uses native ROS 2 packages on Jetson. Each package entry selects
its own `package_manifest*.yaml`, so architecture and native/container choices are
package properties rather than global environment variables.

## Hardware

| Component | Model | Deployment configuration |
| --- | --- | --- |
| Mobile base | Wheeltec R550 mini_tank | Differential-drive tracked chassis; `base_link` footprint 0.55 m × 0.40 m; built-in 6-axis IMU |
| Compute | NVIDIA Jetson Orin | aarch64 Jetson-native packages, ROS 2 Humble, and `rmw_fastrtps_cpp` |
| 2D lidar | LSLIDAR N10P | 360° 2D laser scanner; publishes `/scan` with `frame_id=laser` |
| RGB-D camera | Orbbec Astra S | 640×480 at 30 FPS for RGB and depth; USB 3.0; provides snapshot MCP tools for VLM perception |

The exact provider IDs, device addresses, sensor profiles, and runtime options
are defined in [`robonix_manifest.yaml`](robonix_manifest.yaml). The robot body,
component hierarchy, footprint, and provider-to-component mapping are defined
in [`soma.yaml`](soma.yaml).

Robot-specific algorithm configuration is also deployment-owned:

- [`config/rtabmap_params.yaml`](config/rtabmap_params.yaml) contains the full
  R550 RTAB-Map parameter set.
- [`config/param_mini_tank.yaml`](config/param_mini_tank.yaml) contains the
  complete R550 Nav2 configuration.
- [`config/navigate.xml`](config/navigate.xml) contains the R550 navigation
  BehaviorTree.
- [`urdf/mini_tank_robot.urdf`](urdf/mini_tank_robot.urdf) is the kinematic
  model consumed by `robot_description`. The manifest references this
  deployment asset through absolute path so a clone does not depend on a
  cache path or another checkout.

The manifest references these files with paths relative to this repository.
The Mapping and Navigation provider repositories contain templates only; do
not move R550 dimensions, sensor limits, or controller policy upstream.

Package target selection and algorithm configuration are separate. A package's
`manifest:` chooses a build/start implementation such as Jetson native or a
container. Robot-specific runtime values remain in this repository's parameter
files (or documented config overrides) and do not create a new upstream profile.

Scene is pinned to the front Astra S provider `astra_s_camera`. That provider
supplies RGB, depth, and camera intrinsics for visual scene queries. Scene
obtains the robot's globally corrected pose from the Mapping
`robonix/service/map/pose` contract and combines it with the complete URDF
camera transform published by `robot_description`.

## Prepare

```bash
# 1. Install Robonix toolchain
cd ~/robonix
git switch dev
git pull --ff-only origin dev
make install

# 2. Install ROS 2 system dependencies
sudo apt install ros-humble-rtabmap-ros \
  ros-humble-navigation2 ros-humble-nav2-bringup \
  ros-humble-imu-filter-madgwick

# 3. Source the Wheeltec ROS 2 workspace
source ~/wheeltec_ros2/install/setup.bash
```

Set VLM credentials in your shell before boot (or edit `boot.sh`):

```bash
export VLM_BASE_URL="https://your-llm-endpoint/v1"
export VLM_API_KEY="sk-..."
export VLM_MODEL="qwen-vl-max"
```

## Build and boot

```bash
bash boot.sh
```

The wrapper sources ROS Humble, runs pre-flight checks (`rbnx` CLI, ROS 2,
`turn_on_wheeltec_robot`, `rtabmap_slam`, `imu_filter_madgwick`), and then
execs `rbnx boot -f robonix_manifest.yaml`. `--no-build` skips the build step
for repeated boots. `--shutdown` tears down a running stack.

Operator pages:

- Mapping: `http://<robot-host>:8091/`
- Liaison for Robonix Client: `<robot-host>:50081`

## RViz

The deploy keeps a complete RViz configuration with laser scans, map, costmap,
TF tree, and robot model displays.

```bash
source /opt/ros/humble/setup.bash
rviz2 -d config/wheeltec.rviz
```

## Safety and bring-up order

The checked-in full manifest includes the chassis, lidar, camera, mapping,
Nav2, and the Explore skill; starting it exposes physical motion capabilities.
Keep the hardware emergency stop available and clear the workspace before full
bring-up.

Explore is a lazy-activate skill — it stays INACTIVE at boot and only activates
when the LLM/pilot invokes `robonix/skill/explore/explore`. Once active, it
drives the robot autonomously through frontier-based exploration. Ensure mapping
and Nav2 are healthy before triggering Explore.

After the chassis is powered on:

1. Verify odometry is publishing: `ros2 topic hz /odom_combined`
2. Send zero Twist and confirm the watchdog holds the base stopped.
3. Use a low-speed, short-duration command in a clear area.
4. Verify Mapping pose and Nav2 costmaps before sending a nearby goal.
5. Only then test Explore via `rbnx chat` ("explore this room").

## Robot description

`soma.yaml` and `urdf/mini_tank_robot.urdf` are served by Soma. The description
contains the body footprint and sensor tree used by Pilot and other consumers.
Mount transforms remain calibration-sensitive; update the body URDF after
physical measurement rather than compensating in Scene or Mapping.

The component tree in `soma.yaml` nests sensors under their physical mount:

```
base (mobile_base, base_link)
  ├── lidar (lidar_2d, laser)
  ├── imu (imu, gyro_link)
  └── front_rgbd (rgbd_camera, camera_link)
```

## Capability contracts

| Contract | Owner | Transport |
| --- | --- | --- |
| `robonix/primitive/robot_description/driver` | robot_description | gRPC |
| `robonix/primitive/chassis/driver` | r550_chassis | gRPC |
| `robonix/primitive/chassis/odom` | r550_chassis | ROS 2 `/odom_combined` |
| `robonix/primitive/chassis/move` | r550_chassis | gRPC |
| `robonix/primitive/chassis/twist_in` | r550_chassis | ROS 2 `/cmd_vel` |
| `robonix/primitive/imu/imu` | r550_chassis | ROS 2 `/imu_data` |
| `robonix/primitive/lidar/lidar` | n10p_lslidar | ROS 2 `/scan` |
| `robonix/primitive/camera/rgb` | astra_s_camera | ROS 2 `/camera/color/image_raw` |
| `robonix/primitive/camera/depth` | astra_s_camera | ROS 2 `/camera/depth/image_raw` |
| `robonix/primitive/camera/intrinsics` | astra_s_camera | ROS 2 `/camera/color/camera_info` |
| `robonix/primitive/camera/snapshot` | astra_s_camera | MCP |
| `robonix/primitive/camera/depth_snapshot` | astra_s_camera | MCP |
| `robonix/service/map/occupancy_grid` | mapping | ROS 2 `/map` |
| `robonix/service/map/pointcloud` | mapping | ROS 2 `/rtabmap/cloud_map` |
| `robonix/service/map/pose` | mapping | ROS 2 `/robonix/map/pose` |
| `robonix/service/map/odom` | mapping | ROS 2 `/rtabmap/odom` |
| `robonix/service/map/save_map` | mapping | gRPC + MCP |
| `robonix/service/map/load_map` | mapping | gRPC + MCP |
| `robonix/service/map/pose_estimate` | mapping | gRPC + MCP |
| `robonix/service/navigation/navigate` | nav2 | gRPC + MCP |
| `robonix/service/navigation/navigate/status` | nav2 | gRPC + MCP |
| `robonix/service/navigation/navigate/cancel` | nav2 | gRPC + MCP |
| `robonix/skill/explore/driver` | explore | gRPC |
| `robonix/skill/explore/explore` | explore | MCP |
| `robonix/skill/explore/explore/status` | explore | MCP |
| `robonix/skill/explore/explore/cancel` | explore | MCP |

## Provider dependency graph

```
robot_description                     ← no deps, must boot first
    │  /robot_description (URDF) → TF tree
    ▼
r550_chassis                          ← needs /robot_description for joint states
    │  /odom_combined  /imu_data  /cmd_vel
    │  TF: base_footprint → base_link, base_footprint → gyro_link
    │
    ├──► n10p_lslidar                 ← needs base_link TF
    │       /scan (LaserScan, frame_id=laser)
    │
    └──► astra_s_camera               ← needs camera_link TF
            /camera/color/image_raw   /camera/depth/image_raw
            + snapshot MCP tools
            │
            ├──► mapping              ← lidar2d + odom
            │       /map  /rtabmap/cloud_map  /robonix/map/pose
            │       Web UI → :8091
            │
            ├──► nav2                 ← map + odom + scan
            │       navigation/{navigate, status, cancel}
            │
            ├──► scene                ← camera (astra_s_camera)
            │       scene/{list_objects, goal_near}
            │
            └──► explore              ← map + nav2 (lazy-activate)
                    skill/explore/{explore, status, cancel}
```

Boot order follows this dependency chain: system caps first (atlas → soma →
executor → pilot → liaison → scene), then primitives in declaration order
(robot_description → chassis → lidar → camera), then services (mapping →
nav2), then skills (explore — lazy-activate, stays INACTIVE until LLM
invokes it). Services use the Atlas defer-queue: nav2 waits for mapping, which
waits for lidar + chassis.

## Verification

```bash
# Capability status — all non-skill caps should show ACTIVE
rbnx caps -v

# Registered contracts
rbnx contracts

# Sensor health
ros2 topic hz /scan                    # ~10 Hz
ros2 topic hz /odom_combined           # ~30 Hz
ros2 topic hz /imu_data                # ~200 Hz
ros2 topic hz /camera/color/image_raw  # ~30 Hz
ros2 topic hz /camera/depth/image_raw  # ~30 Hz
ros2 topic hz /map                     # ~1 Hz

# Pose
ros2 topic echo /robonix/map/pose --once

# Chat
rbnx chat
```
