# FAST-LIO2 LiDAR-Inertial SLAM on ROS 2 Jazzy

This project demonstrates how to install, configure, and run **FAST-LIO2** on **Ubuntu 24.04 with ROS 2 Jazzy** using recorded LiDAR and IMU data.

The provided ROS 2 bag publishes:

| Topic          | Message type                  | Purpose                 |
| -------------- | ----------------------------- | ----------------------- |
| `/livox/lidar` | `sensor_msgs/msg/PointCloud2` | LiDAR point-cloud input |
| `/livox/imu`   | `sensor_msgs/msg/Imu`         | IMU motion input        |

The primary challenge was that the original FAST-LIO Livox configuration expected `livox_ros_driver2/msg/CustomMsg`, while the recorded bag used the standard ROS 2 `PointCloud2` format.

The main configuration fix was:

```yaml
preprocess:
  lidar_type: 4
```

After this change, FAST-LIO successfully produced registered point-cloud and odometry output in RViz.

---

## Project Results

The completed system successfully:

* Processed recorded LiDAR and IMU data
* Initialized the IMU
* Generated a registered 3D point cloud
* Estimated sensor motion through odometry
* Displayed the reconstructed environment in RViz
* Published mapped cloud output on `/cloud_registered`

This project was evaluated visually in RViz as a qualitative SLAM demonstration.

---

## System Overview

FAST-LIO2 is a LiDAR-inertial odometry and mapping system that combines two sensor inputs:

* **LiDAR** measures the surrounding 3D environment.
* **IMU** measures acceleration and rotation.
* **FAST-LIO2** combines both inputs to estimate motion and construct a map.

This project uses the ROS 2 port maintained at:

* [Ericsii/FAST_LIO_ROS2](https://github.com/Ericsii/FAST_LIO_ROS2)

It also depends on:

* [Livox-SDK/Livox-SDK2](https://github.com/Livox-SDK/Livox-SDK2)
* [Livox-SDK/livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2)

---

## Tested Environment

| Component            | Version                       |
| -------------------- | ----------------------------- |
| Operating system     | Ubuntu 24.04                  |
| ROS distribution     | ROS 2 Jazzy                   |
| Visualization        | RViz2                         |
| Point-cloud library  | PCL                           |
| LiDAR message format | `sensor_msgs/msg/PointCloud2` |
| IMU message format   | `sensor_msgs/msg/Imu`         |
| Dataset type         | Recorded ROS 2 bag            |

> This tutorial uses recorded data. It does not cover the Ethernet or network configuration required for a physical Livox LiDAR.

---

# Installation

FAST-LIO requires three main components to be built in this order:

1. Livox SDK2
2. `livox_ros_driver2`
3. `FAST_LIO_ROS2`

---

## 1. Source ROS 2 Jazzy

Open a terminal and run:

```bash
source /opt/ros/jazzy/setup.bash
echo $ROS_DISTRO
```

Expected output:

```text
jazzy
```

---

## 2. Install Dependencies

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

---

## 3. Install Livox SDK2

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

`make -j1` limits the build to one worker. This is slower than using all available CPU cores, but it is safer for virtual machines with limited memory.

---

## 4. Build livox_ros_driver2

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

---

## 5. Build FAST_LIO_ROS2

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

---

## ROS 2 Workspace Sourcing

ROS 2 workspaces use:

```bash
install/setup.bash
```

They do not use the ROS 1 workspace path:

```bash
devel/setup.bash
```

For this project, source the workspaces in the following order:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash
```

These commands must be run in every new terminal that uses FAST-LIO.

They can optionally be added to `~/.bashrc`:

```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
echo "source ~/ws_livox/install/setup.bash" >> ~/.bashrc
echo "source ~/fastlio_ws/install/setup.bash" >> ~/.bashrc
```

Then reload the shell:

```bash
source ~/.bashrc
```

---

# Inspecting the ROS 2 Bag

The recorded bags used for this project were stored in:

```text
~/lidar_bags
```

The primary bag was:

```text
~/lidar_bags/lidar_bag
```

Inspect its metadata:

```bash
source /opt/ros/jazzy/setup.bash

ros2 bag info ~/lidar_bags/lidar_bag
```

Play the bag in a terminal:

```bash
ros2 bag play ~/lidar_bags/lidar_bag
```

While the bag is playing, open another terminal and check the available topics:

```bash
source /opt/ros/jazzy/setup.bash

ros2 topic list -t
```

The important topics should include:

```text
/livox/lidar [sensor_msgs/msg/PointCloud2]
/livox/imu [sensor_msgs/msg/Imu]
```

Check the publication rates:

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

The important discovery for this project was that `/livox/lidar` used:

```text
sensor_msgs/msg/PointCloud2
```

rather than:

```text
livox_ros_driver2/msg/CustomMsg
```

---

# FAST-LIO Configuration

## 1. Create a PointCloud2 Configuration

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

Update the input topics:

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

The original Livox setting generally uses:

```yaml
preprocess:
  lidar_type: 1
```

That setting expects:

```text
livox_ros_driver2/msg/CustomMsg
```

Because this project's bag uses `PointCloud2`, `lidar_type: 1` is not appropriate.

Save the file in Nano with:

```text
Ctrl+O
Enter
Ctrl+X
```

---

## 2. Rebuild FAST-LIO

After adding the configuration file, rebuild the package so the file is installed into the package share directory:

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

---

# Running FAST-LIO2

Use three terminals for the most reliable workflow.

Start FAST-LIO before playing the bag.

---

## Terminal 1: Launch FAST-LIO

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

---

## Terminal 2: Play the Bag

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

Loop playback can reset timestamps when the recording restarts, which may cause FAST-LIO to stop publishing output correctly.

---

## Terminal 3: Monitor the Data

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

If the two input topics are active but `/cloud_registered` is not publishing, FAST-LIO is not processing the data correctly.

---

# RViz Configuration

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

The correct frame therefore depends on what is being displayed:

| Visualization                          | Fixed frame         |
| -------------------------------------- | ------------------- |
| Raw `/livox/lidar` topic               | LiDAR message frame |
| FAST-LIO registered cloud and odometry | `camera_init`       |

Useful RViz displays include:

| Display          | Purpose                       |
| ---------------- | ----------------------------- |
| Registered cloud | Reconstructed environment     |
| Map cloud        | Accumulated map               |
| Effected cloud   | Points used during processing |
| Odometry         | Estimated sensor position     |
| Path             | Estimated sensor trajectory   |

In this project:

* The colored point cloud represented the reconstructed environment.
* The odometry display represented the estimated movement of the LiDAR and IMU sensor.

If the map appears very small, use **Focus Camera** or adjust the RViz view.

---

# Verifying a Successful Run

A successful run should satisfy the following checks.

## Input Topics

```bash
ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
```

Both commands should report incoming messages.

## FAST-LIO Output

```bash
ros2 topic hz /cloud_registered
```

This should report registered point-cloud messages.

## Correct Configuration

The FAST-LIO terminal should report:

```text
p_pre->lidar_type 4
```

## IMU Initialization

The terminal should report:

```text
IMU Initial Done
```

## RViz Status

RViz should show:

```text
Global Status: Ok
Fixed Frame: OK
```

with the fixed frame set to:

```text
camera_init
```

---

# Troubleshooting

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

---

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

Rebuild Livox SDK2 after making the change:

```bash
cd ~/Livox-SDK2/build

make -j1
sudo make install
sudo ldconfig
```

Do not modify these files unless the missing-integer-type error actually occurs.

---

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

---

## Jazzy Driver Build Error

An older version of the driver produced an error involving:

```text
LIVOX_INTERFACES_INCLUDE_DIRECTORIES
```

If the standard build fails with that specific error, rebuild with explicit ROS 2 Jazzy arguments:

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

---

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

---

## FAST-LIO Loads lidar_type 1

If the terminal shows:

```text
p_pre->lidar_type 1
```

FAST-LIO is still using a Livox CustomMsg configuration.

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

---

## Configuration File Is Not Found

Confirm that the source file exists:

```bash
ls ~/fastlio_ws/src/FAST_LIO_ROS2/config/livox_pointcloud2.yaml
```

Then rebuild FAST-LIO:

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

---

## No Output on /cloud_registered

First, confirm that the input topics are active:

```bash
ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
```

Then check:

1. The configuration uses the correct topic names.
2. `preprocess.lidar_type` is set to `4`.
3. The FAST-LIO terminal reports `p_pre->lidar_type 4`.
4. The IMU initializes successfully.
5. The bag is played after FAST-LIO starts.

Restart FAST-LIO and play the bag from the beginning.

---

## RViz Reports a Missing Fixed Frame

For FAST-LIO output, set:

```text
Fixed Frame: camera_init
```

The frame may not exist until FAST-LIO begins receiving and processing data.

Start FAST-LIO first, then play the bag.

---

## Point Cloud Appears Very Small

This is usually an RViz camera issue rather than a mapping failure.

Try:

* Focus Camera
* Zooming in
* Panning toward the cloud
* Rotating the view
* Increasing the point size in the point-cloud display

---

## Output Stops After the Bag Restarts

During loop playback, the bag's timestamps return to the beginning of the recording.

FAST-LIO maintains an internal LiDAR-inertial state, so this timestamp reset may cause mapped output to stop even though the input topics remain active.

The reliable workflow is:

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

---

# Main Fixes Applied

| Problem                                | Solution                                 |
| -------------------------------------- | ---------------------------------------- |
| SDK build consumed too much memory     | Used `make -j1`                          |
| Older SDK missed integer definitions   | Added `<cstdint>` where required         |
| Older driver lacked `package.xml`      | Copied `package_ROS2.xml`                |
| Older driver had a Jazzy CMake error   | Added explicit ROS 2 Jazzy arguments     |
| README referenced a ROS 1 setup path   | Used `install/setup.bash`                |
| Bag used standard PointCloud2 messages | Changed `lidar_type` from `1` to `4`     |
| Custom configuration was not installed | Rebuilt FAST-LIO or copied the file      |
| RViz used the wrong fixed frame        | Used `camera_init`                       |
| Bag loops interrupted mapped output    | Played the bag once after a fresh launch |

---

# Limitations

This project has several limitations:

1. **Recorded data only**

   The system was tested using ROS 2 bag data rather than a live physical Livox LiDAR.

2. **No live network configuration**

   The project did not test LiDAR IP addresses, host IP settings, Ethernet communication, or `MID360_config.json`.

3. **Qualitative evaluation**

   The output was evaluated visually in RViz. The estimated trajectory was not compared with ground-truth data.

4. **Loop playback instability**

   FAST-LIO did not reliably continue publishing `/cloud_registered` after the bag restarted in loop mode.

5. **Project-specific configuration**

   The `lidar_type: 4` configuration is specifically required because this dataset publishes `sensor_msgs/msg/PointCloud2`. A bag using Livox `CustomMsg` would require a different setting.

The final result should therefore be understood as a working LiDAR-inertial SLAM integration and visualization demonstration rather than a quantitatively evaluated SLAM benchmark.

---

# Key Takeaways

The most important lesson from this project was that running a SLAM package requires more than launching a ROS node.

The algorithm's assumptions must match the actual dataset, including:

* Topic names
* Message types
* Coordinate frames
* IMU availability
* Timestamp behavior
* Configuration values

For this project, the most important correction was matching FAST-LIO to the bag's LiDAR message type:

```text
Bag message type:
sensor_msgs/msg/PointCloud2

Required FAST-LIO setting:
preprocess.lidar_type: 4
```

The project also demonstrated that active input topics do not automatically mean the SLAM pipeline is producing output. Input and output topics must be checked independently.

---

# References

* [FAST-LIO2: Fast Direct LiDAR-Inertial Odometry](https://arxiv.org/abs/2107.06829)
* [FAST_LIO_ROS2](https://github.com/Ericsii/FAST_LIO_ROS2)
* [Livox SDK2](https://github.com/Livox-SDK/Livox-SDK2)
* [Livox ROS Driver 2](https://github.com/Livox-SDK/livox_ros_driver2)

---

# Acknowledgments

This project uses open-source work from the FAST-LIO and Livox development teams. Their repositories provided the LiDAR-inertial odometry implementation, Livox SDK, ROS 2 message definitions, and driver integration used in this project.
