# ROS2 Multi-Machine Communication & Hand-Eye Calibration Guide
<table style="width: 100%;">
  <tr>
    <td>
      <img src="https://github.com/user-attachments/assets/26c3d77c-3e1b-4ea2-97b5-cf1fa91241fc" alt="image 1" style="width: 100%;" />
    </td>
    <td>
      <img src="https://github.com/user-attachments/assets/f12f8394-62c7-4990-b4ce-e43155a8ac6b" alt="image 2" style="width: 100%;" />
    </td>
  </tr>
</table>
<div align="center">
  <img style="width: 60%;" alt="resized image 60%" src="https://github.com/user-attachments/assets/f69365a5-55ec-4e87-aa25-a4a8b0f4e5d8" />
</div>




This repository provides a step-by-step workflow for setting up high-performance ROS2 Humble communication between a **Vision Computer** and a **Robot Computer** (Franka Panda), followed by a hand-eye calibration procedure using `easy_handeye2`.

**Vision computer (shared) already has everything installed; you can skip to usage.**

## 1. Network Configuration
We use a static IP setup over a direct Ethernet connection.

### Vision Computer Setup
1. Identify your ethernet interface name: `ip link show`
2. Create and activate a static connection:
```bash
sudo nmcli con add type ethernet con-name ros-direct ifname <YOUR_INTERFACE> ip4 192.168.1.11/24
sudo nmcli con up ros-direct
```

### Robot Computer Setup
1. Identify the interface (e.g., the long USB-to-Ethernet name starting with `enx...`).
2. Create and activate the connection:
```bash
sudo nmcli con add type ethernet con-name vision-link ifname <YOUR_INTERFACE> ip4 192.168.1.10/24
sudo nmcli con up vision-link
```
*Note: Verify connectivity by `ping 192.168.1.11` from the robot and vice versa.*

---

## 2. ROS2 Environment Setup
To synchronize the two machines, add the following to the end of your `sudo nano ~/.bashrc` on **both** computers.

**Robot Computer (`amp@panda01`):**
```bash
# >>> ROS2 Ethernet Communication Setup >>>
export ROS_DOMAIN_ID=42
export ROS_IP=192.168.1.10
# Optional: export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
# <<< ROS2 Ethernet Communication Setup <<<
```

**Vision Computer:**
```bash
# >>> ROS2 Ethernet Communication Setup >>>
export ROS_DOMAIN_ID=42
export ROS_IP=192.168.1.11
# <<< ROS2 Ethernet Communication Setup <<<
```

---

## 3. Software Installation
Run these commands on the **Vision Computer**:

### Dependencies
```bash
sudo apt update
sudo apt install python3-transforms3d
sudo apt install ros-humble-rqt ros-humble-rqt-common-plugins ros-humble-rqt-gui ros-humble-rqt-gui-py
```

### Drivers & Calibration Package
```bash
# Clone and build easy_handeye2
cd <your_workspace>/src
git clone https://github.com/marcoesposito1988/easy_handeye2
cd ..
colcon build --symlink-install
source install/setup.bash

# Install RealSense and ArUco drivers
sudo apt install ros-humble-realsense2-camera
sudo apt install ros-humble-aruco-opencv ros-humble-aruco-opencv-msgs
```

---

## 4. Calibration Workflow

### Step 1: Detect the ArUco Marker
1. Print a marker from `DICT_4X4_50` (e.g., ID 4).
2. Start the camera:
   ```bash
   ros2 launch realsense2_camera rs_launch.py
   ```
3. Start the ArUco tracker (with remapping for RealSense topics):
   ```bash
   ros2 run aruco_opencv aruco_tracker_autostart --ros-args \
     -p marker_size:=0.04 \
     -p aruco_dictionary_name:=DICT_4X4_50 \
     --remap /camera/camera_info:=/camera/camera/color/camera_info \
     --remap /camera/image_raw:=/camera/camera/color/image_raw
   ```
If you experience lag with image topics over the network, optimize the FastDDS buffer.

4. Create `~/fastdds.xml`:
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<profiles xmlns="http://eprosima.com/XMLSchemas/fastRTPS_Profiles">
    <transport_descriptors>
        <transport_descriptor>
            <transport_id>custom_udp_transport</transport_id>
            <type>UDPv4</type>
            <sendBufferSize>8388608</sendBufferSize>
            <receiveBufferSize>8388608</receiveBufferSize>
            <non_blocking_send>true</non_blocking_send>
        </transport_descriptor>
    </transport_descriptors>
    <participant profile_name="udp_high_bandwidth_participant" is_default_profile="true">
        <rtps>
            <userTransports>
                <transport_id>custom_udp_transport</transport_id>
            </userTransports>
            <useBuiltinTransports>false</useBuiltinTransports>
        </rtps>
    </participant>
</profiles>
```

5. Export the profile in `~/.bashrc`:
```bash
export FASTRTPS_DEFAULT_PROFILES_FILE=~/fastdds.xml
```

### Step 2: Verify Robot TF
On the **Robot Computer**, start the Franka controller to publish the robot state:
```bash
cd <workspace>
source install/setup.bash
ros2 launch franka_bringup example.launch.py controller_name:=gravity_compensation_example_controller
```
Verify the TF tree using `ros2 run tf2_tools view_frames`.

### Step 3: Run Calibration
Launch the calibration GUI. This example uses `eye_on_base` (tracking marker is on the robot hand, camera is stationary).

```bash
ros2 launch easy_handeye2 calibrate.launch.py \
  name:=fr3_realsense_calib \
  calibration_type:=eye_on_base \
  robot_base_frame:=fr3_link0 \
  robot_effector_frame:=fr3_hand_tcp \
  tracking_base_frame:=camera_color_optical_frame \
  tracking_marker_frame:=marker_4 \
  freehand_robot_movement:=true
```
You can also do eye in hand by using
``calibration_type:=eye_on_hand`` 

If eye_on_base, stick the tag to the end effector, move the arm around, and click on ``take sample``, repeat many times until the result converges. The GUI will show the transformation from base-link to camera.

If eye_on_hand, stick the tag on the table.

<img width="1280" height="625" alt="image" src="https://github.com/user-attachments/assets/8233747f-0a89-4a16-a424-df79dcaa9e29" />

