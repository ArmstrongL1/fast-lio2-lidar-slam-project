# FAST-LIO2 Important Commands

This file is a short reference for the commands used most often or changed specifically for this project. See [`README.md`](README.md) for the complete tutorial and [`troubleshooting.md`](troubleshooting.md) for fixes.

## Source the Workspaces

Run in every new FAST-LIO terminal:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash
```

ROS 2 uses `install/setup.bash`, not `devel/setup.bash`.

## Important Configuration

```yaml
common:
  lid_topic: "/livox/lidar"
  imu_topic: "/livox/imu"

preprocess:
  lidar_type: 4
```

`lidar_type: 4` is required because this bag publishes `sensor_msgs/msg/PointCloud2`.

## Rebuild FAST-LIO After Configuration Changes

```bash
cd ~/fastlio_ws

source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash

colcon build \
  --symlink-install \
  --parallel-workers 1 \
  --packages-select fast_lio

source ~/fastlio_ws/install/setup.bash
```

## Run the Project

Start FAST-LIO before playing the bag.

### Terminal 1: FAST-LIO

```bash
cd ~/fastlio_ws

source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash

ros2 launch fast_lio mapping.launch.py \
  config_file:=livox_pointcloud2.yaml
```

### Terminal 2: ROS 2 Bag

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash

ros2 bag play ~/lidar_bags/lidar_bag
```

Do not use `--loop` for the normal workflow.

### Terminal 3: Verify Inputs and Output

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash

ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
ros2 topic hz /cloud_registered
```

## Inspect the Bag

```bash
ros2 bag info ~/lidar_bags/lidar_bag
ros2 topic list -t
ros2 topic echo --once /livox/lidar --field header.frame_id
```

## Expected Success Messages

```text
p_pre->lidar_type 4
IMU Initial Done
Initialize the map kdtree
```

Use `camera_init` as the RViz fixed frame for FAST-LIO output.
