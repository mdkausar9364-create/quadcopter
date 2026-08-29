# Autonomous Quadcopter Framework

An open-source, full-stack autonomous aerial navigation framework developed by the **Robotronics Club at IIT Mandi**. Designed specifically for rugged mountain environments, this system integrates high-performance state estimation, trajectory planning, and computer vision onboard standard multirotor platforms.

![ROS 2 Humble](https://img.shields.io/badge/ROS_2-Humble-22314E?style=flat-square&logo=ros&logoColor=white)
![PX4](https://img.shields.io/badge/PX4-v1.14+-8A2BE2?style=flat-square)
![NVIDIA Jetson](https://img.shields.io/badge/NVIDIA-Jetson-76B900?style=flat-square&logo=nvidia&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.10-3776AB?style=flat-square&logo=python&logoColor=white)
![C++](https://img.shields.io/badge/C%2B%2B-17-00599C?style=flat-square&logo=cplusplus&logoColor=white)
![OpenCV](https://img.shields.io/badge/OpenCV-4.8+-5C3EE8?style=flat-square&logo=opencv&logoColor=white)
![CUDA](https://img.shields.io/badge/CUDA-11.4-76B900?style=flat-square)
![TensorRT](https://img.shields.io/badge/TensorRT-8.x-76B900?style=flat-square)
![YOLOv8](https://img.shields.io/badge/YOLOv8-Detection-00CFFF?style=flat-square)
![Gazebo](https://img.shields.io/badge/Gazebo-Ignition-15C46B?style=flat-square)
![Git](https://img.shields.io/badge/Git-Friendly-F05032?style=flat-square&logo=git&logoColor=white)

---

## Key Features

* **Autonomous Navigation:** Real-time 3D path planning using dynamic occupancy grid mapping and trajectory optimization.
* **State Estimation:** Visual-Inertial Odometry (VIO) integrated with extended Kalman filtering (EKF) for GPS-denied environments.
* **Target Detection & Tracking:** Edge-computed YOLOv8 pipeline for object detection, precise landing zone recognition, and visual tracking.
* **Fail-Safe Systems:** Automated Return-to-Launch (RTL), obstacle fallback avoidance, and low-battery management triggers.
* **Simulation Support:** Full Software-In-The-Loop (SITL) environment integrated with Gazebo and PX4.

---

## Hardware Architecture

The platform is built on a **Holybro X500 V2** 500 mm carbon-fiber airframe
with a Pixhawk 6C flight controller (PX4) and an NVIDIA Jetson companion
computer running the ROS 2 autonomy stack.

| Doc | Contents |
| --- | -------- |
| [Hardware](hardware/README.md) | Frame, propulsion, layout, mechanical specs |
| [Electronics](electronics/README.md) | System architecture & data flow |
| [Bill of Materials](electronics/bom.md) | Full parts list |
| [Wiring](electronics/wiring.md) | Connections, motor order, checklist |

---

## Software Stack

* **Operating System:** Ubuntu 22.04 LTS / JetPack 5.x
* **Middleware:** ROS 2 (Humble Hawksbill)
* **Flight Stack:** PX4 Autopilot (v1.14+) via `px4_ros_com` and MicroXRCE-DDS
* **Computer Vision:** OpenCV, CUDA, TensorRT
* **Simulation:** Gazebo Ignition

---

## Getting Started

### Prerequisites

Ensure you have ROS 2 Humble installed on your host or companion computer.

```bash
# Clone the repository
git clone https://github.com/the-robotronics-club/quadcopter
cd quadcopter

# Install dependencies
rosdep update
rosdep install --from-paths src --ignore-src -r -y
```

### Build

```bash
cd src
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source install/setup.bash
```

---

## MicroXRCE-DDS Agent

PX4 and ROS 2 communicate over **uXRCE-DDS** (micro-ROS): the flight controller
runs a micro XRCE-DDS *client* while the companion computer runs the
**MicroXRCE-DDS Agent** (`MicroXRCEAgent`), which bridges PX4 uORB topics onto
the ROS 2 graph as `/fmu/*` topics.

### Install

```bash
# Option A - prebuilt Debian package (simplest)
sudo apt install ros-humble-micro-ros-agent

# Option B - build from source (latest)
git clone https://github.com/eProsima/Micro-XRCE-DDS-Agent.git
cd Micro-XRCE-DDS-Agent && mkdir build && cd build
cmake ..
make
sudo make install
sudo ldconfig
```

### Run

| Transport | Use case                   | Command                                              |
| --------- | -------------------------- | ---------------------------------------------------- |
| UDP       | SITL simulation (loopback) | `MicroXRCEAgent udp4 -p 8888 -v`                     |
| Serial    | Pixhawk over UART (TELEM2) | `MicroXRCEAgent serial --dev /dev/ttyTHS1 -b 921600` |
| Ethernet  | Pixhawk on the network     | `MicroXRCEAgent udp4 -p 2019 -r 2020`                |

> **Hardware:** set PX4 `UXRCE_DDS_CFG` to TELEM2 and `SER_TEL2_BAUD` to
> `921600` in QGroundControl before first use.
> **SITL:** the PX4 SITL firmware starts its own uXRCE-DDS client on UDP
> port `8888`, so the agent must listen on that port.

Verify the agent is up and PX4 is connected:

```bash
ros2 daemon stop
ros2 topic list | grep -E 'fmu|vehicle'
```

If no `/fmu/*` topics appear, the agent is not running or the client is not
connected.

---

## Running the System (ROS 2)

### Simulation (SITL)

```bash
# Terminal 1 - MicroXRCE-DDS agent over UDP loopback
MicroXRCEAgent udp4 -p 8888 -v

# Terminal 2 - PX4 SITL + Gazebo (from the PX4-Autopilot source directory)
make px4_sitl gazebo

# Terminal 3 - ROS 2 nodes
cd src && source install/setup.bash
ros2 run quadcopter_bringup teleop
```

### Hardware

1. Power the airframe; confirm the Pixhawk boots and links over UART.
2. Verify telemetry in QGroundControl (`UXRCE_DDS_CFG` = TELEM2).
3. Start the agent and the ROS 2 node:

```bash
MicroXRCEAgent serial --dev /dev/ttyTHS1 -b 921600

cd src && source install/setup.bash
ros2 run quadcopter_bringup teleop
```

### Inspect the ROS 2 Graph

```bash
ros2 node list
ros2 topic list
ros2 topic echo /fmu/out/vehicle_attitude
```

### px4_ros_com Example Nodes

```bash
# Sensor data listeners
ros2 run px4_ros_com sensor_combined_listener
ros2 run px4_ros_com vehicle_gps_position_listener

# Offboard control example (CAUTION: engages motors)
ros2 run px4_ros_com offboard_control

# Launched versions
ros2 launch px4_ros_com sensor_combined_listener.launch.py
```

---

## Controlling the Quadcopter

Use the keyboard teleop node from `quadcopter_bringup`:

```bash
ros2 run quadcopter_bringup teleop
```

| Key      | Action                    |
| -------- | ------------------------- |
| `w`      | Move forward (+X)         |
| `s`      | Move back (-X)            |
| `a`      | Move left (+Y)            |
| `d`      | Move right (-Y)           |
| `r`      | Move up (-Z, NED)         |
| `f`      | Move down (+Z, NED)       |
| `x`      | Stop / hover in place     |
| `q`      | Increase max speed by 10% |
| `z`      | Decrease max speed by 10% |
| `Ctrl+C` | Exit safely               |

The node publishes `OffboardControlMode` and `TrajectorySetpoint` at 20 Hz,
switches PX4 to **Offboard** mode after ~15 setpoints, and **arms** after ~30.
Always keep the RC transmitter in an override-capable flight mode and test in
SITL before any hardware trial.

---

## Repository Structure

```text
quadcopter/
|-- README.md                    # Project overview
|-- INSTRUCTIONS.md              # Build / launch / control guide
|-- hardware/                    # Mechanical / airframe docs
|-- electronics/                 # Wiring, BOM, system architecture
|-- src/                         # ROS 2 colcon workspace
|   |-- quadcopter_bringup/      # Autonomy package (teleop control node)
|   |-- px4_ros_com/             # PX4 <-> ROS 2 bridge + example nodes
|   |-- px4_msgs/                # PX4 uORB message definitions
|   `-- worlds/                  # Simulation world files
`-- website/                     # Project website
```

---

## Contributing

Contributions are welcome! Please follow these steps:
1. Fork the project repository.
2. Create your Feature Branch (`git checkout -b feature/AwesomeFeature`).
3. Commit your changes (`git commit -m 'Add some AwesomeFeature'`).
4. Push to the branch (`git push origin feature/AwesomeFeature`).
5. Open a Pull Request detailing your changes.



---

## Contact & Acknowledgments

* **Club:** Robotronics Club, IIT Mandi
* **Website:** [robotronics.iitmandi.ac.in](https://robotronics.iitmandi.ac.in)
* **Email:** robotronics@iitmandi.ac.in
