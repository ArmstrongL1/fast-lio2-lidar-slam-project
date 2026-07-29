# FAST-LIO2 LiDAR-Inertial SLAM on ROS 2 Jazzy

This project demonstrates how to install, configure, and run **FAST-LIO2** on **Ubuntu 24.04 with ROS 2 Jazzy** using recorded LiDAR and IMU data.

The most important issue in this project was a message-format mismatch. The recorded bag publishes LiDAR data as `sensor_msgs/msg/PointCloud2`, while the original Livox FAST-LIO configuration expected `livox_ros_driver2/msg/CustomMsg`. The key fix was:

```yaml
preprocess:
  lidar_type: 4
```

After this change, FAST-LIO initialized successfully, published odometry and `/cloud_registered`, and displayed the reconstructed environment in RViz.

## Main Takeaways

- Check the bag's actual topic names and message types before configuring FAST-LIO.
- Use `lidar_type: 4` when the LiDAR topic publishes `sensor_msgs/msg/PointCloud2`.
- Start FAST-LIO before playing the bag.
- Play the bag once instead of using `--loop` because restarting timestamps can interrupt FAST-LIO output.
- Verify input and output topics separately. Active sensor topics do not guarantee that FAST-LIO is producing output.

## Documentation

- [`commands.md`](commands.md) — short reference for the most important commands
- [`troubleshooting.md`](troubleshooting.md) — errors encountered and their fixes

# Tutorial

## 1. Tested Environment

| Component | Version or format |
| --- | --- |
| Operating system | Ubuntu 24.04 |
| ROS distribution | ROS 2 Jazzy |
| Visualization | RViz2 |
| Point-cloud library | PCL |
| LiDAR topic | `/livox/lidar` |
| LiDAR message | `sensor_msgs/msg/PointCloud2` |
| IMU topic | `/livox/imu` |
| IMU message | `sensor_msgs/msg/Imu` |
| Dataset | Recorded ROS 2 bag |

> This tutorial uses recorded data. It does not cover Ethernet, IP-address, or `MID360_config.json` configuration for a physical Livox LiDAR.

FAST-LIO requires these components to be built in order:

1. Livox SDK2
2. `livox_ros_driver2`
3. `FAST_LIO_ROS2`

## 2. Source ROS 2 Jazzy

Open a terminal and run:

```bash
source /opt/ros/jazzy/setup.bash
echo $ROS_DISTRO
```

Expected output:

```text
jazzy
```

## 3. Install Dependencies

```bash
sudo apt update

sudo apt install -y \
  git \
  cmake \
  build-essential \
  python3-colcon-common-extensions \
  python3-rosdep \
  ros-jazzy-rviz2 \
  ros-jazzy-pcl-conversions \
  ros-jazzy-pcl-ros \
  ros-jazzy-ament-cmake-auto \
  libpcl-dev \
  libeigen3-dev
```

Check the installed Eigen version:

```bash
pkg-config --modversion eigen3
```

## 4. Install Livox SDK2

Clone and build Livox SDK2:

```bash
cd ~

git clone https://github.com/Livox-SDK/Livox-SDK2.git
cd ~/Livox-SDK2

mkdir -p build
cd build

cmake ..
make -j1
sudo make install
sudo ldconfig
```

`make -j1` limits the build to one worker. It is slower than using every CPU core, but it is safer for virtual machines with limited memory.

## 5. Build livox_ros_driver2

Create a separate ROS 2 workspace for the Livox driver:

```bash
mkdir -p ~/ws_livox/src
cd ~/ws_livox/src

git clone https://github.com/Livox-SDK/livox_ros_driver2.git
```

Install dependencies and build:

```bash
cd ~/ws_livox

source /opt/ros/jazzy/setup.bash

rosdep install --from-paths src --ignore-src -r -y

colcon build \
  --symlink-install \
  --parallel-workers 1
```

Source the completed workspace:

```bash
source ~/ws_livox/install/setup.bash
```

Confirm that ROS 2 can find the driver:

```bash
ros2 pkg list | grep livox
```

Expected relevant output:

```text
livox_ros_driver2
```

## 6. Build FAST_LIO_ROS2

Create a workspace for FAST-LIO:

```bash
mkdir -p ~/fastlio_ws/src
cd ~/fastlio_ws/src

git clone https://github.com/Ericsii/FAST_LIO_ROS2.git --recursive
```

The `--recursive` option is required because FAST-LIO uses Git submodules, including `ikd-Tree`.

Install dependencies and build:

```bash
cd ~/fastlio_ws

source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash

rosdep install --from-paths src --ignore-src -r -y

colcon build \
  --symlink-install \
  --parallel-workers 1
```

Source the workspace:

```bash
source ~/fastlio_ws/install/setup.bash
```

Confirm that the FAST-LIO package is available:

```bash
ros2 pkg list | grep fast
```

Expected relevant output:

```text
fast_lio
```

## 7. Source the Workspaces

ROS 2 workspaces use `install/setup.bash`, not the ROS 1 path `devel/setup.bash`.

Run these commands in every new terminal that uses FAST-LIO:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash
```

You can optionally add them to `~/.bashrc`:

```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
echo "source ~/ws_livox/install/setup.bash" >> ~/.bashrc
echo "source ~/fastlio_ws/install/setup.bash" >> ~/.bashrc
```

Reload the shell:

```bash
source ~/.bashrc
```

## 8. Inspect the ROS 2 Bag

The primary recorded bag used by this project was stored at:

```text
~/lidar_bags/lidar_bag
```

Inspect its metadata:

```bash
source /opt/ros/jazzy/setup.bash
ros2 bag info ~/lidar_bags/lidar_bag
```

Play the bag in one terminal:

```bash
ros2 bag play ~/lidar_bags/lidar_bag
```

While the bag is playing, open another terminal and list its topics and message types:

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic list -t
```

The important topics should include:

```text
/livox/lidar [sensor_msgs/msg/PointCloud2]
/livox/imu [sensor_msgs/msg/Imu]
```

Check their publication rates:

```bash
ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
```

Inspect the LiDAR frame:

```bash
ros2 topic echo --once /livox/lidar --field header.frame_id
```

Inspect the fields in the point-cloud message:

```bash
ros2 topic echo --once /livox/lidar --field fields
```

The important discovery was that `/livox/lidar` uses:

```text
sensor_msgs/msg/PointCloud2
```

rather than:

```text
livox_ros_driver2/msg/CustomMsg
```

## 9. Create a PointCloud2 Configuration

Move to the FAST-LIO configuration directory:

```bash
cd ~/fastlio_ws/src/FAST_LIO_ROS2/config
```

Copy the existing Livox configuration:

```bash
cp avia.yaml livox_pointcloud2.yaml
```

Open the new file:

```bash
nano livox_pointcloud2.yaml
```

Set the input topics:

```yaml
common:
  lid_topic: "/livox/lidar"
  imu_topic: "/livox/imu"
```

Change the LiDAR input type:

```yaml
preprocess:
  lidar_type: 4
```

The value `4` tells FAST-LIO to use its standard `sensor_msgs/msg/PointCloud2` input callback.

The original Livox setting normally uses:

```yaml
preprocess:
  lidar_type: 1
```

That value expects `livox_ros_driver2/msg/CustomMsg`, so it does not match this dataset.

Save the file in Nano:

```text
Ctrl+O
Enter
Ctrl+X
```

## 10. Rebuild FAST-LIO

After creating or changing the configuration file, rebuild the package so the configuration is installed into the package share directory:

```bash
cd ~/fastlio_ws

source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash

colcon build \
  --symlink-install \
  --parallel-workers 1 \
  --packages-select fast_lio
```

Source the rebuilt workspace:

```bash
source ~/fastlio_ws/install/setup.bash
```

Verify that the configuration file was installed:

```bash
ls ~/fastlio_ws/install/fast_lio/share/fast_lio/config
```

The output should include:

```text
livox_pointcloud2.yaml
```

## 11. Run FAST-LIO2

Use three terminals for the most reliable workflow. Start FAST-LIO before playing the bag.

### Terminal 1: Launch FAST-LIO

```bash
cd ~/fastlio_ws

source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash

ros2 launch fast_lio mapping.launch.py \
  config_file:=livox_pointcloud2.yaml
```

When the correct configuration loads, the terminal should show:

```text
p_pre->lidar_type 4
```

Other successful initialization messages may include:

```text
Multi thread started
Node init finished.
IMU Initial Done
Initialize the map kdtree
```

`IMU Initial Done` indicates that FAST-LIO received enough IMU data to initialize its motion estimate.

### Terminal 2: Play the Bag

Open a second terminal:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash

ros2 bag play ~/lidar_bags/lidar_bag
```

Play the bag once rather than using loop playback.

Avoid:

```bash
ros2 bag play ~/lidar_bags/lidar_bag --loop
```

Loop playback resets the timestamps when the recording restarts. FAST-LIO maintains an internal LiDAR-inertial state, so that timestamp reset may cause mapped output to stop.

### Terminal 3: Monitor the Data

Open a third terminal:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash
```

Check the input topics:

```bash
ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
```

Check the main FAST-LIO output:

```bash
ros2 topic hz /cloud_registered
```

The expected data flow is:

```text
/livox/lidar ─┐
              ├──> FAST-LIO2 ───> /cloud_registered
/livox/imu ───┘
```

If the input topics are active but `/cloud_registered` is not publishing, FAST-LIO is not processing the data correctly. See [`troubleshooting.md`](troubleshooting.md).

## 12. Configure RViz

RViz should open automatically with the FAST-LIO launch file.

Set the RViz fixed frame to:

```text
camera_init
```

This fixed frame is used for FAST-LIO's processed mapping output.

The fixed frame for raw LiDAR visualization may instead be the frame from the LiDAR message header, such as:

```text
livox_frame
```

| Visualization | Fixed frame |
| --- | --- |
| Raw `/livox/lidar` topic | LiDAR message frame |
| FAST-LIO registered cloud and odometry | `camera_init` |

Useful RViz displays include:

| Display | Purpose |
| --- | --- |
| Registered cloud | Reconstructed environment |
| Map cloud | Accumulated map |
| Effected cloud | Points used during processing |
| Odometry | Estimated sensor position |
| Path | Estimated sensor trajectory |

If the map appears very small, use **Focus Camera**, zoom in, pan toward the cloud, or increase the point size.

## 13. Verify a Successful Run

### Check the Inputs

```bash
ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
```

Both commands should report incoming messages.

### Check the FAST-LIO Output

```bash
ros2 topic hz /cloud_registered
```

This should report registered point-cloud messages.

### Check the Loaded Configuration

The FAST-LIO terminal should report:

```text
p_pre->lidar_type 4
```

### Check IMU Initialization

The terminal should report:

```text
IMU Initial Done
```

### Check RViz

RViz should show:

```text
Global Status: Ok
Fixed Frame: OK
```

with the fixed frame set to:

```text
camera_init
```

# Project Information

## Project Results

The completed system successfully:

- Processed recorded LiDAR and IMU data
- Initialized the IMU
- Generated a registered 3D point cloud
- Estimated sensor motion through odometry
- Displayed the reconstructed environment in RViz
- Published mapped cloud output on `/cloud_registered`

The output was evaluated visually in RViz as a qualitative SLAM demonstration.

## How the System Works

FAST-LIO2 combines two sensor inputs:

- **LiDAR** measures the surrounding 3D environment.
- **IMU** measures acceleration and rotation.
- **FAST-LIO2** combines both inputs to estimate motion and construct a map.

This project uses:

- [Ericsii/FAST_LIO_ROS2](https://github.com/Ericsii/FAST_LIO_ROS2)
- [Livox-SDK/Livox-SDK2](https://github.com/Livox-SDK/Livox-SDK2)
- [Livox-SDK/livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2)

## Limitations

1. **Recorded data only**  
   The system was tested using ROS 2 bag data rather than a live physical Livox LiDAR.

2. **No live network configuration**  
   The project did not test LiDAR IP addresses, host IP settings, Ethernet communication, or `MID360_config.json`.

3. **Qualitative evaluation**  
   The output was evaluated visually in RViz. The estimated trajectory was not compared with ground-truth data.

4. **Loop playback instability**  
   FAST-LIO did not reliably continue publishing `/cloud_registered` after the bag restarted in loop mode.

5. **Project-specific configuration**  
   The `lidar_type: 4` setting is required because this dataset publishes `sensor_msgs/msg/PointCloud2`. A bag using Livox `CustomMsg` would require a different setting.

The final result should be understood as a working LiDAR-inertial SLAM integration and visualization demonstration rather than a quantitatively evaluated SLAM benchmark.

## References

- [FAST-LIO2: Fast Direct LiDAR-Inertial Odometry](https://arxiv.org/abs/2107.06829)
- [FAST_LIO_ROS2](https://github.com/Ericsii/FAST_LIO_ROS2)
- [Livox SDK2](https://github.com/Livox-SDK/Livox-SDK2)
- [Livox ROS Driver 2](https://github.com/Livox-SDK/livox_ros_driver2)

## Acknowledgments

This project uses open-source work from the FAST-LIO and Livox development teams. Their repositories provided the LiDAR-inertial odometry implementation, Livox SDK, ROS 2 message definitions, and driver integration used in this project.