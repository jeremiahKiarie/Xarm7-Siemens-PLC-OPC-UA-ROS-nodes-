# XArm7 Siemens PLC OPC-UA ROS2 Integration

## Table of Contents
1. [Project Overview](#project-overview)
2. [Prerequisites](#prerequisites)
3. [Repository Setup](#repository-setup)
4. [Execution Order](#execution-order)
5. [Component Details](#component-details)

---

## Project Overview

This project integrates a **UFactory XArm7 collaborative robotic arm** with **Siemens PLC systems** through **OPC-UA (Open Platform Communications Unified Architecture)** using ROS2. The system enables autonomous pick-and-place operations with real-time joint state monitoring and PLC communication.

### Key Components:
- **Motion Planning**: MoveIt configuration for XArm7 motion planning and execution
- **OPC-UA Server**: Exposes robot state and PLC signals for Siemens integration
- **OPC-UA Client**: Reads PLC commands and writes joint states
- **Planning Scene Management**: Dynamic environment obstacle tracking
- **Gripper Control**: Integrated end-effector actuation

---

## Prerequisites

Before starting, ensure your system has:

### System Requirements
- **ROS2** (Humble, Foxy, Galactic, or compatible version)
- **Ubuntu 20.04+** or equivalent
- **Python 3.8+**
- **Network connectivity** to the robot at `172.16.40.20`
- Network connectivity to PLC/OPC-UA server at `172.16.40.3` and `172.16.40.116`

### Required Packages
```bash
sudo apt install git python3-colcon-common-extensions -y
```

### Additional Dependencies
```bash
pip install opcua python3-opcua
```

---

## Repository Setup

### Step 1: Clone and Build Development Workspace (dev_ws)

```bash
# Clone the repository
git clone <your-repository-link> dev_ws
cd dev_ws

# Build the workspace
colcon build

# Source the workspace
source install/setup.bash
```

The dev_ws contains:
- `xarm_moveit_config` - MoveIt configuration for XArm7
- `moveit_nodes_pkg` - Pick & place, planning scene, and motion planning nodes
- `xarm_api` - XArm hardware interface and APIs
- `xarm_controller` - Joint and gripper controllers

### Step 2: Clone and Build ROS2-OPC-UA Workspace (ros2_ws)

```bash
# Open a new terminal
git clone <your-repository-link> ros2_ws
cd ros2_ws

# Build the workspace
colcon build

# Source the workspace
source install/setup.bash
```

The ros2_ws contains:
- `minimal_opcua_test` - OPC-UA server and client nodes for Siemens integration

---

## Execution Order

**Open 5 terminals** and execute the following commands **in this exact order**. Source the appropriate workspace in each terminal.

### Terminal 1: Launch MoveIt with XArm
```bash
cd dev_ws
source install/setup.bash

ros2 launch xarm_moveit_config xarm7_moveit_realmove.launch.py add_gripper:=true robot_ip:=172.16.40.20
```
**Purpose**: Initializes the MoveIt motion planning framework and connects to the physical XArm7 robot arm with gripper attachment.

**Expected Output**: MoveIt interface ready, robot joint states publishing, RViz visualization available.

---

### Terminal 2: Run Planning Scene Updater
```bash
cd dev_ws
source install/setup.bash

ros2 run moveit_nodes_pkg update_planning_scene_node
```
**Purpose**: Maintains the planning scene with obstacles and collision geometry from the environment.

**Expected Output**: Planning scene continuously updated with current collision objects.

---

### Terminal 3: Start OPC-UA Server
```bash
cd ros2_ws
source install/setup.bash

ros2 run minimal_opcua_test opc_ua_server1
```
**Purpose**: Establishes the OPC-UA server that exposes robot state variables to Siemens PLC systems (`172.16.40.3:4840`).

**Expected Output**: 
```
ROS 2 OPC UA Server Started
Registered Namespace 'http://robotics.digitaltwins.robot' at index 3
Server listening on opc.tcp://172.16.40.116:4840
```

**OPC-UA Variables Exposed**:
- `activate_plc` - Boolean signal to activate PLC mode
- `activate_robot` - Boolean signal to activate robot operations
- `gripper_arm` - Gripper state variable
- `joint_pos_1` through `joint_pos_7` - XArm joint positions
- `joint_vel_1` through `joint_vel_7` - XArm joint velocities

---

### Terminal 4: Run Pick and Place Node
```bash
cd dev_ws
source install/setup.bash

ros2 run moveit_nodes_pkg pick_and_place_xarm
```
**Purpose**: Executes autonomous pick-and-place operations using MoveIt motion planning.

**Expected Output**: 
- Robot moves to predefined pick positions
- Gripper actuates to grasp objects
- Robot moves to place positions
- Gripper releases objects
- Returns to home position

---

### Terminal 5: Run OPC-UA Client (Joint State Writer)
```bash
cd ros2_ws
source install/setup.bash

ros2 run minimal_opcua_test min_write_jointstates4
```
**Purpose**: OPC-UA client that reads PLC commands from the server and writes current joint states back to the OPC-UA namespace for Siemens PLC monitoring.

**Expected Output**: Joint states continuously written to OPC-UA server, PLC synchronization active.

---

## Component Details

### MoveIt Configuration (xarm_moveit_config)
Handles motion planning and execution with collision detection. The launch file `xarm7_moveit_realmove.launch.py` accepts parameters:
- `add_gripper=true` - Includes gripper in planning group
- `robot_ip=172.16.40.20` - IP address of the XArm7 robot

### Planning Scene Updater (update_planning_scene_node)
Subscribes to `/planning_scene` topic and manages collision objects to prevent planned motions from intersecting with obstacles.

### Pick and Place Node (pick_and_place_xarm)
Main execution node for autonomous operations:
- Reads target poses from configuration
- Plans motion trajectories using MoveIt
- Actuates gripper to grasp/release
- Executes planned trajectories on physical arm

### OPC-UA Server (opc_ua_server1)
Provides Siemens PLC integration through standard OPC-UA protocol:
- Endpoint: `opc.tcp://172.16.40.116:4840`
- Namespace: `http://robotics.digitaltwins.robot`
- Exposes all robot state variables for monitoring and control

### OPC-UA Client (min_write_jointstates4)
Establishes bidirectional communication with the OPC-UA server:
- Reads control signals from PLC (e.g., start/stop commands)
- Writes current XArm joint positions and velocities
- Maintains real-time synchronization with PLC system

---

## Troubleshooting

### Network Connectivity Issues
- Verify robot is reachable: `ping 172.16.40.20`
- Verify PLC/OPC-UA server is reachable: `ping 172.16.40.116` and `ping 172.16.40.3`
- Check firewall rules for ports 4840 (OPC-UA)

### MoveIt not responding
- Ensure Terminal 1 (MoveIt launch) completed initialization
- Check robot connection: `ros2 topic list | grep robot_states`

### OPC-UA Connection Failed
- Verify OPC-UA server is running (Terminal 3)
- Check endpoint configuration in client: `opc.tcp://172.16.40.116:4840`

### Pick and Place Node not executing
- Confirm Terminals 1-3 are running successfully
- Check planning scene is updated (Terminal 2 output)
- Verify robot joint states are publishing: `ros2 topic echo /joint_states`

---

## Additional Resources

- **XArm ROS2 Documentation**: [XArm Developer Guide](./dev_ws/src/xarm_ros2/ReadMe.md)
- **MoveIt Documentation**: [MoveIt Official Docs](https://moveit.ros.org/)
- **OPC-UA Specification**: [IEC 62541 Standard](https://en.wikipedia.org/wiki/OPC_Unified_Architecture)