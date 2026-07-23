# FAST-LIO2 LiDAR-Inertial SLAM Project

## Selected System

I selected **Option C: FAST-LIO2 / FAST_LIO_ROS2**

## Repository

FAST_LIO_ROS2 repository:

https://github.com/Ericsii/FAST_LIO_ROS2

## Why I Chose This System

I chose Option C, FAST-LIO2, because it seemed like the most challenging and fun option. Unlike Option A and B, it incorporates not only LIDAR data but also IMU data, making it the most advanced option and giving me the opportunity to explore how IMU data contributes to LIDAR-based SLAM. I also liked the fact that Option C connects most directly to real-world robotics systems which would help me with future projects that involve SLAM integration. 

## System Requirements

| Label | Version |
|---|---|
| Ubuntu | >= 20.04 |
| ROS 2 | >= Foxy (Humble recommended, but I used Jazzy for this tutorial) |
| PCL | >= 1.8 |
| Eigen | >= 3.3.4 |
| Livox SDK | Livox SDK2 |
| Livox driver | livox_ros_driver2 |
| SLAM package | FAST_LIO_ROS2 |

## Installation Steps

### 1. Source ROS 2 Jazzy

I first sourced ROS 2 Jazzy and confirmed the ROS distribution.

```bash
source /opt/ros/jazzy/setup.bash
echo $ROS_DISTRO
```

Expected output:

```text
jazzy
```

### 2. Install Dependencies

I installed the main dependencies needed for Livox, PCL, RViz, and ROS 2 builds.

```bash
sudo apt update

sudo apt install -y \
  git \
  cmake \
  build-essential \
  python3-colcon-common-extensions \
  ros-jazzy-rviz2 \
  ros-jazzy-pcl-conversions \
  ros-jazzy-pcl-ros \
  ros-jazzy-ament-cmake-auto \
  libpcl-dev
```

I also checked the Eigen version:

```bash
pkg-config --modversion eigen3
```

### 3. Livox SDK2 Installation

I installed Livox SDK2 because it is required before building `livox_ros_driver2`.

```bash
cd ~/Desktop/Livox-SDK2/Livox-SDK2
mkdir build
cd build
cmake ..
make -j1
sudo make install
sudo ldconfig
```

The original build command used:

```bash
make -j
```

However, this caused my Ubuntu virtual machine to freeze or run out of memory. I rebuilt using:

```bash
make -j1
```

This completed successfully.

I also had to add:

```cpp
#include <cstdint>
```

to two Livox SDK2 header files because the build failed on missing integer types such as `uint8_t`, `uint16_t`, and `uint64_t`.

The files were:

```text
sdk_core/comm/define.h
sdk_core/logger_handler/file_manager.h
```

Result:

```text
Livox SDK2 built to 100% and installed successfully.
```

I did not modify the Livox SDK2 network configuration because this project uses recorded ROS 2 bag data instead of a live Livox LiDAR. The SDK2 network configuration would become important later when connecting to a physical Livox sensor, since it controls LiDAR IP addresses, host IP addresses, and data ports.

### 4. livox_ros_driver2 Installation

I created a ROS 2 workspace for the Livox driver.

```bash
mkdir -p ~/ws_livox/src
cd ~/ws_livox/src
git clone https://github.com/Livox-SDK/livox_ros_driver2.git
```

During the first `livox_ros_driver2` build attempt, this command failed:

```bash
colcon build --symlink-install
```

The failure happened because `package.xml` was missing. The repository contained separate package files for ROS 1 and ROS 2:

```text
package_ROS1.xml
package_ROS2.xml
```

Since I was building with ROS 2, I copied the ROS 2 package file:

```bash
cd ~/ws_livox/src/livox_ros_driver2
cp package_ROS2.xml package.xml
```

Then I rebuilt from the workspace root.

```bash
cd ~/ws_livox
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --parallel-workers 1 --cmake-args -DROS_EDITION=ROS2 -DDISTRO_ROS=jazzy
```

The `livox_ros_driver2` build also had a ROS 2 Jazzy compatibility issue. The repository documentation recommends ROS 2 Foxy or Humble, but my VM used ROS 2 Jazzy. After fixing the missing `package.xml` issue, the build still failed because `LIVOX_INTERFACES_INCLUDE_DIRECTORIES` was not set. Building with the ROS 2 and Jazzy CMake arguments fixed the issue.

After building, I sourced the workspace:

```bash
source ~/ws_livox/install/setup.bash
```

I checked that the package was visible:

```bash
ros2 pkg list | grep livox
```

Expected output:

```text
livox_ros_driver2
```

### 5. FAST_LIO_ROS2 Installation

I created a FAST-LIO workspace and cloned the repository with submodules.

```bash
mkdir -p ~/fastlio_ws/src
cd ~/fastlio_ws/src
git clone https://github.com/Ericsii/FAST_LIO_ROS2.git --recursive
```

Then I built FAST-LIO.

```bash
cd ~/fastlio_ws
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
rosdep install --from-paths src --ignore-src -y
colcon build --symlink-install --parallel-workers 1
```

After building, I sourced the FAST-LIO workspace:

```bash
source ~/fastlio_ws/install/setup.bash
```

I checked that the FAST-LIO package was visible:

```bash
ros2 pkg list | grep fast
```

Expected relevant output:

```text
fast_lio
```

### 6. ROS 1 vs ROS 2 Setup Issue

The FAST_LIO_ROS2 README says to source:

```bash
source $Livox_ros_driver_dir$/devel/setup.bash
```

However, `devel/setup.bash` is used in ROS 1 workspaces. My Livox driver was built as a ROS 2 workspace, so I used:

```bash
source ~/ws_livox/install/setup.bash
```

instead.

This was an important difference because ROS 2 workspaces use:

```text
install/setup.bash
```

not:

```text
devel/setup.bash
```

## Build Steps

## Build Steps

This project required building three main components:

1. Livox SDK2
2. `livox_ros_driver2`
3. FAST_LIO_ROS2

---

### 1. Build Livox SDK2

Livox SDK2 had to be built first because `livox_ros_driver2` depends on it.

```bash
cd ~/Desktop/Livox-SDK2/Livox-SDK2
mkdir -p build
cd build
cmake ..
make -j1
sudo make install
sudo ldconfig
```

I used:

```bash
make -j1
```

instead of:

```bash
make -j
```

because `make -j` caused my Ubuntu virtual machine to freeze or run out of memory.

I also had to add:

```cpp
#include <cstdint>
```

to these Livox SDK2 header files:

```text
sdk_core/comm/define.h
sdk_core/logger_handler/file_manager.h
```

This fixed missing integer type errors involving types such as:

```text
uint8_t
uint16_t
uint32_t
uint64_t
```

After these fixes, Livox SDK2 built to 100% and installed successfully.

---

### 2. Build livox_ros_driver2

The Livox ROS 2 driver was built in:

```text
~/ws_livox
```

Commands:

```bash
mkdir -p ~/ws_livox/src
cd ~/ws_livox/src
git clone https://github.com/Livox-SDK/livox_ros_driver2.git
```

The first build attempt failed because `package.xml` was missing. The repository included separate package files for ROS 1 and ROS 2:

```text
package_ROS1.xml
package_ROS2.xml
```

Since this project used ROS 2, I copied the ROS 2 package file:

```bash
cd ~/ws_livox/src/livox_ros_driver2
cp package_ROS2.xml package.xml
```

Then I built from the workspace root:

```bash
cd ~/ws_livox
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --parallel-workers 1 --cmake-args -DROS_EDITION=ROS2 -DDISTRO_ROS=jazzy
```

I used the Jazzy CMake arguments because the driver originally had a ROS 2 Jazzy compatibility issue involving:

```text
LIVOX_INTERFACES_INCLUDE_DIRECTORIES
```

After building, I sourced the workspace:

```bash
source ~/ws_livox/install/setup.bash
```

I confirmed the package was available:

```bash
ros2 pkg list | grep livox
```

Expected output:

```text
livox_ros_driver2
```

---

### 3. Build FAST_LIO_ROS2

FAST_LIO_ROS2 was built in:

```text
~/fastlio_ws
```

Commands:

```bash
mkdir -p ~/fastlio_ws/src
cd ~/fastlio_ws/src
git clone https://github.com/Ericsii/FAST_LIO_ROS2.git --recursive
```

Then I built the workspace:

```bash
cd ~/fastlio_ws
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
rosdep install --from-paths src --ignore-src -y
colcon build --symlink-install --parallel-workers 1
```

After building, I sourced the FAST-LIO workspace:

```bash
source ~/fastlio_ws/install/setup.bash
```

I confirmed the package was available:

```bash
ros2 pkg list | grep fast
```

Expected relevant output:

```text
fast_lio
```

---

### 4. Important ROS 2 Build Note

The FAST_LIO_ROS2 README referenced:

```bash
source $Livox_ros_driver_dir$/devel/setup.bash
```

However, that is a ROS 1 workspace path. Since this project used ROS 2, I used:

```bash
source ~/ws_livox/install/setup.bash
```

instead.

ROS 2 workspaces use:

```text
install/setup.bash
```

not:

```text
devel/setup.bash
```

## Input Bag Information

## Input Bag Information

The provided class bags were stored locally in:

```text
~/lidar_bags
```

The bags I worked with were:

```text
lidar_bag
lidar_bag2
lidar_bag3
```

The main bag used for the FAST-LIO2 test was:

```text
~/lidar_bags/lidar_bag
```

I inspected the bag before running FAST-LIO:

```bash
source /opt/ros/jazzy/setup.bash
ros2 bag info ~/lidar_bags/lidar_bag
```

To play the bag, I used:

```bash
ros2 bag play ~/lidar_bags/lidar_bag
```

The bag published the following important topics:

| Topic | Type | Purpose |
|---|---|---|
| `/livox/lidar` | `sensor_msgs/msg/PointCloud2` | LiDAR point cloud input |
| `/livox/imu` | `sensor_msgs/msg/Imu` | IMU motion input |

I confirmed the active topics while the bag was playing:

```bash
ros2 topic list -t
```

I also checked the LiDAR topic rate:

```bash
ros2 topic hz /livox/lidar
```

and the IMU topic rate:

```bash
ros2 topic hz /livox/imu
```

To inspect the LiDAR message frame and fields, I used:

```bash
ros2 topic echo --once /livox/lidar --field header
ros2 topic echo --once /livox/lidar --field header.frame_id
ros2 topic echo --once /livox/lidar --field fields
```

The important discovery was that `/livox/lidar` was published as:

```text
sensor_msgs/msg/PointCloud2
```

not:

```text
livox_ros_driver2/msg/CustomMsg
```

This mattered because the original FAST-LIO Livox configuration expected Livox custom messages. Since the bag used standard `PointCloud2` messages, I had to change the FAST-LIO configuration to use:

```yaml
preprocess:
  lidar_type: 4
```

This allowed FAST-LIO to process the provided bag correctly.

## Topic and Frame Checks

## Topic and Frame Checks

While the bag was playing, I checked the input topics and frame information to make sure FAST-LIO was receiving the correct data.

The LiDAR input topic was:

```text
/livox/lidar
```

The IMU input topic was:

```text
/livox/imu
```

I checked the topic names and message types with:

```bash
ros2 topic list -t
```

The important topics were:

| Topic | Type |
|---|---|
| `/livox/lidar` | `sensor_msgs/msg/PointCloud2` |
| `/livox/imu` | `sensor_msgs/msg/Imu` |

I checked the LiDAR message header with:

```bash
ros2 topic echo --once /livox/lidar --field header
```

I checked the LiDAR frame ID with:

```bash
ros2 topic echo --once /livox/lidar --field header.frame_id
```

I also checked the PointCloud2 fields with:

```bash
ros2 topic echo --once /livox/lidar --field fields
```

This confirmed that the bag was publishing a standard ROS 2 `PointCloud2` message instead of a Livox custom message.

For raw LiDAR visualization in RViz, the fixed frame would normally be the LiDAR frame from the message header, such as:

```text
livox_frame
```

However, this project was not only visualizing the raw `/livox/lidar` topic. I was running FAST-LIO and visualizing FAST-LIO's processed SLAM output.

For the FAST-LIO output, the correct RViz fixed frame was:

```text
camera_init
```

This frame worked because RViz showed:

```text
Global Status: Ok
Fixed Frame: OK
```

The difference is:

| Situation | RViz display | Fixed frame |
|---|---|---|
| Raw LiDAR bag visualization | `/livox/lidar` | LiDAR message frame, such as `livox_frame` |
| FAST-LIO output visualization | `/cloud_registered`, odometry, map/path outputs | `camera_init` |

Using `camera_init` allowed RViz to display the FAST-LIO registered point cloud and odometry output correctly.

## Configuration Changes

## Configuration Changes

The most important configuration change was switching FAST-LIO from Livox custom message mode to standard `PointCloud2` mode.

The original FAST-LIO Livox configuration expected the LiDAR input to use:

```text
livox_ros_driver2/msg/CustomMsg
```

However, the provided class bags publish `/livox/lidar` as:

```text
sensor_msgs/msg/PointCloud2
```

Because of this, FAST-LIO did not process the bag correctly with the original Livox setting.

I created a new configuration file:

```text
~/fastlio_ws/src/FAST_LIO_ROS2/config/livox_pointcloud2.yaml
```

This file was copied from the original FAST-LIO config and modified for the provided bags.

The main changes were:

| Parameter | Original value | New value | Reason |
|---|---|---|---|
| `preprocess.lidar_type` | `1` | `4` | The bag publishes `/livox/lidar` as `sensor_msgs/msg/PointCloud2`, not `livox_ros_driver2/msg/CustomMsg` |
| `common.lid_topic` | existing/default value | `/livox/lidar` | Matches the provided bag LiDAR topic |
| `common.imu_topic` | existing/default value | `/livox/imu` | Matches the provided bag IMU topic |

The most important setting was:

```yaml
preprocess:
  lidar_type: 4
```

This told FAST-LIO to use the standard `sensor_msgs/msg/PointCloud2` callback instead of the Livox custom message callback.

The topic settings were:

```yaml
common:
  lid_topic: "/livox/lidar"
  imu_topic: "/livox/imu"
```

After creating the modified config file, I copied it into the installed FAST-LIO package config directory:

```bash
cp ~/fastlio_ws/src/FAST_LIO_ROS2/config/livox_pointcloud2.yaml \
   ~/fastlio_ws/install/fast_lio/share/fast_lio/config/
```

This was necessary because the launch file searched for config files inside the installed package directory.

When FAST-LIO launched correctly, the terminal showed:

```text
p_pre->lidar_type 4
```

This confirmed that the modified config file was being used.

When FAST-LIO launched incorrectly, the terminal showed:

```text
p_pre->lidar_type 1
```

That meant it was still using the original Livox custom message configuration, which was wrong for the provided bag.

## Running FAST-LIO2

## Running FAST-LIO2

The reliable workflow was to launch FAST-LIO first, then play the bag once.

I did **not** use loop playback for the final FAST-LIO run because FAST-LIO did not reliably continue publishing output after the bag restarted.

---

### Terminal 1: Start FAST-LIO

```bash
cd ~/fastlio_ws
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash
ros2 launch fast_lio mapping.launch.py config_file:=livox_pointcloud2.yaml
```

When the correct config loaded, the terminal showed:

```text
p_pre->lidar_type 4
```

This confirmed that FAST-LIO was using the `PointCloud2` configuration.

Other successful terminal messages included:

```text
Multi thread started
Node init finished.
IMU Initial Done
Initialize the map kdtree
```

The line:

```text
IMU Initial Done
```

showed that FAST-LIO received and initialized the IMU data from `/livox/imu`.

---

### Terminal 2: Play the Bag

In a second terminal, I played the bag:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
ros2 bag play ~/lidar_bags/lidar_bag
```

For reliable testing, I played the bag once instead of using:

```bash
ros2 bag play ~/lidar_bags/lidar_bag --loop
```

Loop playback caused problems after the bag restarted.

---

### Terminal 3: Monitor Topics

In a third terminal, I checked the input and output topics:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ws_livox/install/setup.bash
source ~/fastlio_ws/install/setup.bash

ros2 topic hz /livox/lidar
ros2 topic hz /livox/imu
ros2 topic hz /cloud_registered
```

The input topics were:

```text
/livox/lidar
/livox/imu
```

The main FAST-LIO output topic I checked was:

```text
/cloud_registered
```

If `/livox/lidar` and `/livox/imu` were publishing but `/cloud_registered` was not, then the bag was active but FAST-LIO was not producing mapped cloud output.

---

### RViz Settings

RViz opened from the FAST-LIO launch file.

The fixed frame was set to:

```text
camera_init
```

This worked because RViz showed:

```text
Global Status: Ok
Fixed Frame: OK
```

The main RViz displays I used were:

| Display | Meaning |
|---|---|
| `CloudRegistered` | Registered point cloud output |
| `CloudMap` | Map point cloud display |
| `CloudEffected` | Processed/effected cloud points |
| `Odometry` | Estimated LiDAR/IMU sensor pose |
| `Path` | Estimated trajectory over time |

In RViz:

```text
Colored point cloud = reconstructed environment
Red odometry line = estimated sensor movement
```

The red line came from the odometry display. It showed FAST-LIO's estimate of how the LiDAR/IMU sensor moved through the environment.

## Results



## Problems and Troubleshooting

### Problem 1: Livox SDK2 build froze the VM

The original build command was:

```bash
make -j
```

This caused my Ubuntu virtual machine to freeze or run out of memory.

Solution:

```bash
make -j1
```

Using `make -j1` limited the build to one thread and allowed Livox SDK2 to build successfully.

---

### Problem 2: Missing integer type errors in Livox SDK2

The Livox SDK2 build failed with missing integer type errors involving types such as:

```text
uint8_t
uint16_t
uint32_t
uint64_t
```

Solution:

I added:

```cpp
#include <cstdint>
```

to the affected Livox SDK2 header files:

```text
sdk_core/comm/define.h
sdk_core/logger_handler/file_manager.h
```

After adding this include, the SDK built successfully.

---

### Problem 3: livox_ros_driver2 was missing package.xml

The first `livox_ros_driver2` build attempt failed because `package.xml` was missing.

The repository contained separate package files:

```text
package_ROS1.xml
package_ROS2.xml
```

Since this project used ROS 2, I copied the ROS 2 package file:

```bash
cd ~/ws_livox/src/livox_ros_driver2
cp package_ROS2.xml package.xml
```

Then I rebuilt the workspace.

---

### Problem 4: livox_ros_driver2 had a ROS 2 Jazzy compatibility issue

After fixing the missing `package.xml`, the driver still failed to build on ROS 2 Jazzy. The error involved:

```text
LIVOX_INTERFACES_INCLUDE_DIRECTORIES
```

The repository documentation recommends ROS 2 Foxy or Humble, but this project used ROS 2 Jazzy.

Solution:

I rebuilt using ROS 2 and Jazzy CMake arguments:

```bash
cd ~/ws_livox
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --parallel-workers 1 --cmake-args -DROS_EDITION=ROS2 -DDISTRO_ROS=jazzy
```

After this, `livox_ros_driver2` built successfully.

---

### Problem 5: FAST_LIO_ROS2 README used ROS 1 workspace instructions

The FAST_LIO_ROS2 README referenced:

```bash
source $Livox_ros_driver_dir$/devel/setup.bash
```

However, `devel/setup.bash` is used in ROS 1 workspaces.

This project used ROS 2, so the correct setup file was:

```bash
source ~/ws_livox/install/setup.bash
```

ROS 2 workspaces use:

```text
install/setup.bash
```

not:

```text
devel/setup.bash
```

---

### Problem 6: FAST-LIO initially used the wrong LiDAR message type

FAST-LIO originally launched with:

```text
p_pre->lidar_type 1
```

This was wrong for the provided bags.

The provided bags publish `/livox/lidar` as:

```text
sensor_msgs/msg/PointCloud2
```

but `lidar_type: 1` expects Livox custom messages:

```text
livox_ros_driver2/msg/CustomMsg
```

Solution:

I created a modified config file and changed:

```yaml
preprocess:
  lidar_type: 4
```

This made FAST-LIO use the standard `PointCloud2` input.

When the correct config loaded, the terminal showed:

```text
p_pre->lidar_type 4
```

---

### Problem 7: Modified FAST-LIO config file was not found

At first, FAST-LIO warned that the modified config file path was not a file. This happened because the launch file looked for config files inside the installed package directory.

Solution:

I copied the modified config file into the installed FAST-LIO config folder:

```bash
cp ~/fastlio_ws/src/FAST_LIO_ROS2/config/livox_pointcloud2.yaml \
   ~/fastlio_ws/install/fast_lio/share/fast_lio/config/
```

After this, I relaunched FAST-LIO with:

```bash
ros2 launch fast_lio mapping.launch.py config_file:=livox_pointcloud2.yaml
```

Then the terminal correctly showed:

```text
p_pre->lidar_type 4
```

---

### Problem 8: RViz fixed frame confusion

In earlier raw LiDAR visualization tutorials, the RViz fixed frame was set to the LiDAR frame from the message header, such as:

```text
livox_frame
```

However, this project visualized FAST-LIO output, not only the raw `/livox/lidar` topic.

For FAST-LIO output, the correct fixed frame was:

```text
camera_init
```

This worked because RViz showed:

```text
Global Status: Ok
Fixed Frame: OK
```

The difference was:

| Situation | Fixed frame |
|---|---|
| Raw `/livox/lidar` visualization | LiDAR message frame, such as `livox_frame` |
| FAST-LIO output visualization | `camera_init` |

---

### Problem 9: RViz point cloud looked very small

At first, the mapped point cloud appeared very small in RViz.

This was a visualization issue, not a SLAM failure.

Solution:

I adjusted the RViz camera view by zooming, panning, rotating, and using Focus Camera. I also adjusted the point cloud display settings.

Useful RViz controls:

```text
Mouse wheel = zoom in/out
Middle mouse drag = pan
Left mouse drag = rotate
Focus Camera = center the view on the selected cloud
```

---

### Problem 10: Red line in RViz was confusing

A red line appeared in RViz. At first I thought it might be the Path display, but after turning displays on and off, I confirmed it came from the Odometry display.

The red line represented FAST-LIO's estimated sensor movement.

In RViz:

```text
Colored point cloud = reconstructed environment
Red odometry line = estimated LiDAR/IMU movement
```

This was useful because it showed that FAST-LIO was producing odometry, not only a point cloud.

---

### Problem 11: Bag loop playback caused FAST-LIO output to stop

When I replayed the bag multiple times without restarting FAST-LIO, RViz did not always update correctly.

I confirmed that the input bag topics were still publishing:

```text
/livox/lidar
/livox/imu
```

However, FAST-LIO stopped publishing:

```text
/cloud_registered
```

This showed that the rosbag was still active, but FAST-LIO was no longer producing registered cloud output after the bag looped or restarted.

The likely reason is that loop playback causes the bag timestamps to jump back to the beginning. FAST-LIO maintains an internal LiDAR-inertial motion estimate, so the timestamp reset can cause the pipeline to stop updating correctly.

Reliable solution:

```text
Restart FAST-LIO/RViz.
Play the bag once.
Take screenshots while the output appears.
Restart FAST-LIO before running the bag again.
```

For reliable evaluation, I used:

```bash
ros2 bag play ~/lidar_bags/lidar_bag
```

instead of:

```bash
ros2 bag play ~/lidar_bags/lidar_bag --loop
```

---

### Summary of Main Fixes

| Problem | Fix |
|---|---|
| Livox SDK2 froze VM | Built with `make -j1` |
| Missing integer types | Added `#include <cstdint>` |
| Missing `package.xml` | Copied `package_ROS2.xml` to `package.xml` |
| Jazzy driver build issue | Built with `-DROS_EDITION=ROS2 -DDISTRO_ROS=jazzy` |
| ROS 1 setup path | Used `install/setup.bash` instead of `devel/setup.bash` |
| Wrong FAST-LIO LiDAR type | Changed `lidar_type` from `1` to `4` |
| Config not found | Copied config into installed FAST-LIO share directory |
| RViz fixed frame confusion | Used `camera_init` for FAST-LIO output |
| Bag loop issue | Played bag once from a fresh FAST-LIO launch |

## Limitations

## Limitations

The main limitation was looped rosbag playback. FAST-LIO worked reliably when the bag was played once from a fresh launch, but it did not reliably continue publishing `/cloud_registered` after the bag restarted in loop mode.

During loop playback, I confirmed that the input topics were still active:

```text
/livox/lidar
/livox/imu
```

However, FAST-LIO stopped publishing:

```text
/cloud_registered
```

This showed that the bag was still playing, but FAST-LIO was no longer producing registered point cloud output after the loop restarted.

For the most reliable result, I used this workflow:

```text
Restart FAST-LIO/RViz.
Play the bag once.
Capture screenshots while the output appears.
Restart FAST-LIO before running the bag again.
```

Another limitation is that this project used recorded ROS 2 bag data instead of a live physical Livox LiDAR scanner. Because of that, I did not test real-time Ethernet setup, LiDAR IP configuration, host IP configuration, or live sensor communication.

A real Livox Mid-360 setup would require additional steps, including:

```text
powering the LiDAR,
connecting Ethernet,
setting the host IP address,
checking the LiDAR IP address,
editing MID360_config.json,
launching livox_ros_driver2,
and verifying live /livox/lidar and /livox/imu topics.
```

This project was also completed on ROS 2 Jazzy, even though the FAST_LIO_ROS2 documentation recommends ROS 2 Humble. Because of that, some build and compatibility fixes were required.

Finally, the result was evaluated visually in RViz rather than with numerical accuracy metrics. I confirmed that FAST-LIO produced point cloud and odometry output, but I did not compare the estimated trajectory against ground truth.

Overall, the system was successful for integration and visualization, but the final result should be understood as a working qualitative SLAM demo rather than a fully optimized or quantitatively evaluated SLAM benchmark.

## What I Learned

This project taught me that LiDAR SLAM integration is not only about launching a ROS package. The system has to match the dataset’s topic names, message types, frames, and timing behavior.

The most important thing I learned was that FAST-LIO2 is a LiDAR-inertial system, not a LiDAR-only system. The LiDAR provides the 3D point cloud of the environment, while the IMU provides motion information such as turning, acceleration, and vibration. FAST-LIO uses both sensors together to estimate the movement of the sensor and build a map.

In simple terms:

```text
LiDAR sees the environment.
IMU feels the movement.
FAST-LIO combines both to estimate motion and build the map.
```

I also learned that message type matters a lot. The provided bags published `/livox/lidar` as:

```text
sensor_msgs/msg/PointCloud2
```

but the original FAST-LIO Livox configuration expected:

```text
livox_ros_driver2/msg/CustomMsg
```

Because of that, FAST-LIO did not work correctly until I changed:

```yaml
preprocess:
  lidar_type: 4
```

This was the most important configuration fix in the project.

I also learned that RViz fixed frames depend on what is being visualized. When visualizing the raw LiDAR topic, the fixed frame should usually match the LiDAR message frame. However, when visualizing FAST-LIO output, the correct fixed frame was:

```text
camera_init
```

This worked because RViz showed:

```text
Global Status: Ok
Fixed Frame: OK
```

Another important thing I learned was that ROS 1 and ROS 2 workspaces use different setup files. Some repository instructions referenced:

```text
devel/setup.bash
```

but this project used ROS 2, so the correct workspace setup file was:

```text
install/setup.bash
```

I also learned that looped bag playback can cause problems for systems that maintain motion history. FAST-LIO worked reliably when the bag was played once from a fresh launch, but it did not reliably keep publishing `/cloud_registered` after the bag restarted in loop mode. This showed that even when the input topics are still publishing, the SLAM system itself may stop producing output.

Overall, I learned how to:

- build Livox SDK2,
- build `livox_ros_driver2`,
- build FAST_LIO_ROS2,
- inspect ROS 2 bag topics,
- check LiDAR and IMU message types,
- modify FAST-LIO configuration files,
- debug RViz frame issues,
- verify IMU data,
- identify FAST-LIO output topics,
- and document integration problems clearly.

The main lesson was that a successful SLAM project depends on matching the algorithm’s assumptions to the actual data being used.
