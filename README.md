# ranger_mini_deploy

Robonix deploy manifest for the AgileX Ranger Mini robot at SysWonder
lab. Hardware:

- Jetson Orin (aarch64, Tegra-special CUDA stack)
- AgileX Ranger Mini v2 chassis (CAN bus, renamed `can_ranger`)
- Livox MID-360 3D LiDAR + integrated 6-axis IMU (Ethernet, 192.168.1.161)
- Intel RealSense D435i RGBD camera (USB 3.0, with internal IMU)

## Bring-up sequence

The first goal is **rtabmap fusing lidar + RGBD + IMU**, with the
chassis powered off (no driving yet). All package URLs in
`robonix_manifest.yaml` resolve from this enkerewpo GitHub org:

| Package                  | Repo                                             | Owns                  |
| ------------------------ | ------------------------------------------------ | --------------------- |
| `mid360_lidar_rbnx`      | enkerewpo/mid360_lidar_rbnx                      | primitive/lidar/*     |
| `mid360_imu_rbnx`        | enkerewpo/mid360_imu_rbnx                        | primitive/imu/*       |
| `realsense_camera_rbnx`  | enkerewpo/realsense_camera_rbnx                  | primitive/camera/*    |
| `ranger_chassis_rbnx`    | enkerewpo/ranger_chassis_rbnx (disabled)         | primitive/chassis/*   |
| `mapping_rbnx`           | enkerewpo/mapping_rbnx                           | service/map/*         |

```bash
# on the Jetson, in this directory:
rbnx build .         # clones each url: package and runs its build.sh
rbnx boot  .         # spawns each one and runs Driver(CMD_INIT, config)
```

`rbnx build` writes everything to `rbnx-build/cache/<name>/` so
`~/wheatfox/packages/` (the working dir on the Jetson) is never touched.

## URDF — required, not shipped

Soma needs a Ranger Mini URDF (`urdf_path` in the system.soma block).
The URDF must include:

- `base_link` (chassis frame; convention: ground projection of the
  geometric centre, X forward, Z up)
- `livox_frame` mount transform from `base_link`
- `camera_link` and `camera_color_optical_frame` mount transforms

Until a calibrated URDF is in hand, an interim path is to launch
`static_transform_publisher` for each frame manually. Sketch (drop in
a side-launch, e.g. `~/wheatfox/static_tfs.launch.xml`):

```xml
<!-- replace x y z and roll pitch yaw with your measured mount values -->
<launch>
  <node pkg="tf2_ros" exec="static_transform_publisher" name="tf_lidar"
        args="0.20 0 0.40  0 0 0  base_link livox_frame"/>
  <node pkg="tf2_ros" exec="static_transform_publisher" name="tf_camera"
        args="0.30 0 0.35  0 0 0  base_link camera_link"/>
</launch>
```

Then leave `system.soma` commented out in the manifest until the URDF
is ready, and run that side-launch in another shell.

## Network — MID-360

Lidar firmware-configured at `192.168.1.161`. Jetson NIC must live on
`192.168.1.50/24`. NetworkManager helper on the robot:
`~/wheatfox/scripts/nm-mid360-static-ip.sh` (sets a profile that
survives reboots; auto-picks the first `enx*` interface unless
`MID360_IFACE=…` is set).

## Verifying the bring-up

After `rbnx boot`:

```bash
# Lidar publishing?
ros2 topic hz /scanner/cloud   # ~10 Hz

# IMU?
ros2 topic hz /livox/imu       # ~200 Hz

# RealSense?
ros2 topic hz /camera_435i/color/image_raw                    # ~30 Hz
ros2 topic hz /camera_435i/aligned_depth_to_color/image_raw   # ~30 Hz

# rtabmap output?
ros2 topic hz /map             # 1 Hz-ish (OccupancyGrid)
ros2 topic echo /robonix/map/pose --once

# What atlas knows about?
rbnx caps                      # should list all the contracts above
```

Open RViz and load the rtabmap visualization config to see the map
build up.

## Defer / boot sequencing

The deploy manifest is an unordered list. Boot ordering happens at
runtime via the defer protocol: a package whose dep isn't ready
returns `Driver_Response(state="deferred")` and `rbnx boot` retries it
periodically until the system reaches steady state. There's no
explicit dep graph in the manifest — each package only declares what
it needs at the moment its Init runs.

Concretely, the cascade for this stack:

```
mid360_lidar.Init    →  spawns livox driver, declares lidar3d
                        (also makes /livox/imu live on the bus)
mid360_imu.Init      →  defers if /livox/imu silent; succeeds on retry
                        once mid360_lidar's launch is publishing
realsense_camera.Init→  spawns realsense, declares rgb + depth
mapping.Init         →  queries atlas for lidar3d, rgb, depth, imu;
                        defers any not yet present; succeeds when all are
```

## License

Manifest + this README: MulanPSL-2.0.
Each `url:` package retains its own license.
