# FAST-LIO2 Troubleshooting

This file contains the problems encountered while installing and running FAST-LIO2 on Ubuntu 24.04 with ROS 2 Jazzy.

For the main project overview, see [`README.md`](README.md). For commonly used commands, see [`commands.md`](commands.md).

## Quick Diagnostic Order

When FAST-LIO is not producing output, check these items in order:

1. Confirm that `/livox/lidar` is publishing.
2. Confirm that `/livox/imu` is publishing.
3. Confirm that the configuration uses the correct topic names.
4. Confirm that `preprocess.lidar_type` is `4`.
5. Confirm that the terminal reports `p_pre->lidar_type 4`.
6. Confirm that the terminal reports `IMU Initial Done`.
7. Confirm that `/cloud_registered` is publishing.
8. Confirm that RViz uses `camera_init` for FAST-LIO output.
9. Restart FAST-LIO before replaying the bag from the beginning.

## Livox SDK2 Build Freezes

Using all available CPU cores may consume too much memory in a virtual machine.

Instead of:

```bash
make -j
```

use:

```bash
make -j1
```

## Missing Integer-Type Errors

Some older Livox SDK2 versions may fail with errors involving:

```text
uint8_t
uint16_t
uint32_t
uint64_t
```

Only when those errors appear, add:

```cpp
#include <cstdint>
```

to the affected header files.

During this project, the affected files were:

```text
sdk_core/comm/define.h
sdk_core/logger_handler/file_manager.h
```

Rebuild Livox SDK2:

```bash
cd ~/Livox-SDK2/build

make -j1
sudo make install
sudo ldconfig
```

Do not modify these files unless the missing-integer-type error actually occurs.

## livox_ros_driver2 Is Missing package.xml

Some older repository versions contained:

```text
package_ROS1.xml
package_ROS2.xml
```

but did not contain the expected `package.xml`.

Only when `package.xml` is missing, run:

```bash
cd ~/ws_livox/src/livox_ros_driver2
cp package_ROS2.xml package.xml
```

Then rebuild:

```bash
cd ~/ws_livox

source /opt/ros/jazzy/setup.bash

colcon build \
  --symlink-install \
  --parallel-workers 1
```

## Jazzy Driver Build Error

An older driver version produced an error involving:

```text
LIVOX_INTERFACES_INCLUDE_DIRECTORIES
```

When the standard build fails with that specific error, rebuild with explicit ROS 2 Jazzy arguments:

```bash
cd ~/ws_livox

source /opt/ros/jazzy/setup.bash

colcon build \
  --symlink-install \
  --parallel-workers 1 \
  --cmake-args \
  -DROS_EDITION=ROS2 \
  -DDISTRO_ROS=jazzy
```

## devel/setup.bash Does Not Exist

Some FAST-LIO instructions reference:

```bash
source <livox-workspace>/devel/setup.bash
```

That is a ROS 1 workspace path.

For ROS 2, use:

```bash
source ~/ws_livox/install/setup.bash
```

The full sourcing order is:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash
```

## FAST-LIO Loads lidar_type 1

If the terminal shows:

```text
p_pre->lidar_type 1
```

FAST-LIO is still using a Livox `CustomMsg` configuration.

Confirm that the configuration contains:

```yaml
preprocess:
  lidar_type: 4
```

Then rebuild and source the workspace:

```bash
cd ~/fastlio_ws

colcon build \
  --symlink-install \
  --parallel-workers 1 \
  --packages-select fast_lio

source ~/fastlio_ws/install/setup.bash
```

Relaunch FAST-LIO:

```bash
ros2 launch fast_lio mapping.launch.py \
  config_file:=livox_pointcloud2.yaml
```

## Configuration File Is Not Found

Confirm that the source file exists:

```bash
ls ~/fastlio_ws/src/FAST_LIO_ROS2/config/livox_pointcloud2.yaml
```

Rebuild FAST-LIO:

```bash
cd ~/fastlio_ws

colcon build \
  --symlink-install \
  --parallel-workers 1 \
  --packages-select fast_lio
```

Check the installed configuration directory:

```bash
ls ~/fastlio_ws/install/fast_lio/share/fast_lio/config
```

As a fallback, copy the file manually:

```bash
cp ~/fastlio_ws/src/FAST_LIO_ROS2/config/livox_pointcloud2.yaml \
   ~/fastlio_ws/install/fast_lio/share/fast_lio/config/
```

## No Output on /cloud_registered

First, confirm that both input topics are active:

```bash
ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
```

Then check:

1. The configuration uses `/livox/lidar` and `/livox/imu`.
2. `preprocess.lidar_type` is set to `4`.
3. The FAST-LIO terminal reports `p_pre->lidar_type 4`.
4. The terminal reports `IMU Initial Done`.
5. FAST-LIO was launched before the bag started.
6. The bag is being played from the beginning.

Restart FAST-LIO and play the bag once:

```bash
ros2 bag play ~/lidar_bags/lidar_bag
```

## RViz Reports a Missing Fixed Frame

For FAST-LIO output, set:

```text
Fixed Frame: camera_init
```

The frame may not exist until FAST-LIO begins receiving and processing data.

Start FAST-LIO first, then play the bag.

For raw `/livox/lidar` data, use the frame from the LiDAR message header, such as `livox_frame`.

## Point Cloud Appears Very Small

This is usually an RViz camera issue rather than a mapping failure.

Try:

- **Focus Camera**
- Zooming in
- Panning toward the cloud
- Rotating the view
- Increasing the point size in the point-cloud display

## Output Stops After the Bag Restarts

During loop playback, the bag timestamps return to the beginning of the recording.

FAST-LIO maintains an internal LiDAR-inertial state, so the timestamp reset may cause mapped output to stop even though the input topics remain active.

Use this recovery sequence:

1. Stop the bag.
2. Stop FAST-LIO.
3. Relaunch FAST-LIO.
4. Play the bag once.
5. Record results before the bag ends.

Use:

```bash
ros2 bag play ~/lidar_bags/lidar_bag
```

instead of:

```bash
ros2 bag play ~/lidar_bags/lidar_bag --loop
```

## Main Fixes Applied

| Problem | Solution |
| --- | --- |
| SDK build consumed too much memory | Used `make -j1` |
| Older SDK missed integer definitions | Added `<cstdint>` only where required |
| Older driver lacked `package.xml` | Copied `package_ROS2.xml` |
| Older driver had a Jazzy CMake error | Added explicit ROS 2 Jazzy arguments |
| Instructions referenced a ROS 1 setup path | Used `install/setup.bash` |
| Bag used standard PointCloud2 messages | Changed `lidar_type` from `1` to `4` |
| Custom configuration was not installed | Rebuilt FAST-LIO or copied the file |
| RViz used the wrong fixed frame | Used `camera_init` |
| Bag loops interrupted mapped output | Played the bag once after a fresh launch |
