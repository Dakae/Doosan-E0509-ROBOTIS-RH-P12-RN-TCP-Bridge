# Doosan E0509 - ROBOTIS RH-P12-RN TCP Bridge

[English](./README.md) | [한국어](./README.ko.md)

---

## Introduction

This repository provides a ROS 2 TCP bridge for controlling and monitoring a
**ROBOTIS RH-P12-RN(A)** gripper mounted on a **Doosan E0509** robot.

The project runs a DRL-side TCP server on the Doosan controller, communicates
with the gripper through flange RS-485 Modbus RTU, and exposes the gripper to
ROS 2 as reusable nodes, topics, services, and an action interface.

The original standalone Python prototype is preserved under `old/`. The main
implementation is now the ROS 2 package layout under:

- `dsr_gripper_tcp/`
- `dsr_gripper_tcp_interfaces/`

---

## Why This Project Exists

On the target Doosan E0509 setup, directly reading gripper feedback from ROS 2
through DRL was not reliable enough for closed-loop control.

This bridge solves the problem by:

- running a TCP server inside a DRL script on the Doosan controller,
- using DRL to communicate with the gripper over flange RS-485 Modbus RTU,
- sending lightweight binary command/response packets between ROS 2 and DRL,
- publishing gripper state back to ROS 2 for robot task coordination.

---

## Key Features

- **Bidirectional gripper control**
  - Position move
  - Torque ON/OFF
  - Motion profile configuration
  - State readback

- **ROS 2 service/action server**
  - A single node owns the TCP bridge
  - Robot task nodes can control the gripper through stable service/action APIs

- **Safe grasp action**
  - Performs one closing motion
  - Detects grasp success using current feedback
  - Provides action feedback and result for robot task logic

- **Live web dashboard**
  - Browser-based monitoring and manual control
  - Position, current, velocity, temperature, torque state, and moving state

- **Controller recovery helpers**
  - DRL start retry
  - TCP reconnect
  - Gripper initialize retry
  - Flange serial recovery logic

---

## System Architecture

```text
                  ROS 2 robot task/action node
                              |
                              | service / action / topic
                              v
                    [ gripper_service_node ]
                              |
                    DoosanGripperTcpBridge
                              |
                         TCP socket
                              |
                [ Doosan controller DRL script ]
                              |
                    Flange serial Modbus RTU
                              |
                    [ ROBOTIS RH-P12-RN(A) ]
```

The web dashboard can also own the bridge directly:

```text
Browser <-- SocketIO --> web_dashboard_node --> TCP bridge --> DRL --> Gripper
```

> Do not run `gripper_service_node` and `web_dashboard_node` at the same time
> unless the dashboard has been changed to use the service node as a client.
> Only one process should own the TCP bridge.

---

## Repository Layout

```text
.
├── dsr_gripper_tcp/
│   ├── dsr_gripper_tcp/
│   │   ├── gripper_tcp_protocol.py
│   │   ├── gripper_tcp_bridge.py
│   │   ├── example_gripper_tcp.py
│   │   ├── web_dashboard.py
│   │   ├── web_dashboard_node.py
│   │   └── gripper_service_node.py
│   ├── launch/
│   │   ├── web_dashboard_node.launch.py
│   │   └── gripper_service_node.launch.py
│   ├── package.xml
│   ├── setup.py
│   └── README.md
├── dsr_gripper_tcp_interfaces/
│   ├── msg/GripperState.msg
│   ├── srv/
│   │   ├── GetState.srv
│   │   ├── GetPosition.srv
│   │   ├── SetPosition.srv
│   │   ├── GetMotionProfile.srv
│   │   ├── SetMotionProfile.srv
│   │   └── SetTorque.srv
│   ├── action/SafeGrasp.action
│   ├── CMakeLists.txt
│   └── package.xml
└── old/
    └── legacy standalone prototype files
```

---

## Requirements

- Ubuntu 22.04
- ROS 2 Humble
- Doosan ROS 2 packages, including `dsr_msgs2`
- Python 3.10+
- Python packages:

```bash
pip install flask flask-socketio
```

---

## Build

Place this repository in your ROS 2 workspace `src/` directory.

```bash
cd ~/ros2_ws
colcon build --packages-select dsr_gripper_tcp_interfaces dsr_gripper_tcp
source install/setup.bash
```

---

## Quick Start

### 1. Start the Service/Action Server

Use this node when another robot control node should command the gripper.

```bash
ros2 launch dsr_gripper_tcp gripper_service_node.launch.py \
  controller_host:=110.120.1.56 \
  namespace:=dsr01 \
  service_prefix:=
```

Main interfaces:

- `/gripper_service/state`
- `/gripper_service/joint_state`
- `/gripper_service/get_state`
- `/gripper_service/get_position`
- `/gripper_service/set_position`
- `/gripper_service/set_motion_profile`
- `/gripper_service/get_motion_profile`
- `/gripper_service/set_torque`
- `/gripper_service/safe_grasp`

Torque ON:

```bash
ros2 service call /gripper_service/set_torque \
  dsr_gripper_tcp_interfaces/srv/SetTorque "{enabled: true}"
```

Open:

```bash
ros2 service call /gripper_service/set_position \
  dsr_gripper_tcp_interfaces/srv/SetPosition "{position: 0, timeout_sec: 5.0}"
```

Move to a target position:

```bash
ros2 service call /gripper_service/set_position \
  dsr_gripper_tcp_interfaces/srv/SetPosition "{position: 700, timeout_sec: 5.0}"
```

Safe grasp:

```bash
ros2 action send_goal /gripper_service/safe_grasp \
  dsr_gripper_tcp_interfaces/action/SafeGrasp \
  "{target_position: 700, max_current: 400, current_delta_threshold: 120, timeout_sec: 8.0}" \
  --feedback
```

Monitor state:

```bash
ros2 topic echo /gripper_service/state
```

### 2. Start the Web Dashboard

Use this node for browser-based monitoring and manual control.

```bash
ros2 launch dsr_gripper_tcp web_dashboard_node.launch.py \
  controller_host:=110.120.1.56 \
  namespace:=dsr01 \
  service_prefix:= \
  web_port:=5000
```

Open:

```text
http://localhost:5000
```

### 3. Run the CLI Example

```bash
ros2 run dsr_gripper_tcp example_gripper_tcp \
  --controller-host 110.120.1.56 \
  --namespace dsr01 \
  --service-prefix ""
```

---

## Safe Grasp Behavior

`SafeGrasp.action` performs a single closing motion to `target_position`.
It does not move the gripper step-by-step.

After the motion completes, grasp success is judged using current feedback:

- success if `abs(final_current) >= max_current`
- success if the current increase from the start is greater than or equal to
  `current_delta_threshold`

The DRL-side move logic also treats a high-current condition as a valid grasp
completion signal, so the gripper can stop before reaching the final close
position when it contacts an object.

---

## State Feedback

`/gripper_service/state` publishes
`dsr_gripper_tcp_interfaces/msg/GripperState`.

Important fields:

- `present_position`: current gripper position pulse
- `goal_position`: last commanded target position
- `present_current`: measured gripper current
- `present_velocity`: measured velocity
- `torque_enabled`: torque state
- `grasp_detected`: current-based grasp detection
- `object_lost`: possible object loss after a grasp
- `status_text`: node-side status string

A robot task action server can subscribe to this topic while executing arm
motions and abort or recover if `object_lost` becomes true.

---

## Legacy Files

The original standalone prototype files are kept under `old/`:

- `old/example_gripper_tcp.py`
- `old/gripper_tcp_bridge.py`
- `old/gripper_tcp_protocol.py`
- `old/web_dashboard.py`
- `old/README.md`

They are kept for reference only. New development should use the ROS 2 packages.

---

## More Documentation

- `dsr_gripper_tcp/README.md`
- `dsr_gripper_tcp_interfaces/msg/GripperState.msg`
- `dsr_gripper_tcp_interfaces/action/SafeGrasp.action`

---

## Status

This project is actively being developed and tested on a Doosan E0509 +
ROBOTIS RH-P12-RN(A) setup.
