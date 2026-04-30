# Doosan E0509 - ROBOTIS RH-P12-RN TCP Bridge

ROS 2 control bridge for a **ROBOTIS RH-P12-RN(A) gripper** mounted on a
**Doosan E0509** robot.

This project runs a DRL-side TCP server on the Doosan controller and exposes the
gripper to ROS 2 as reusable Python nodes, services, topics, and an action
interface. It also includes a Flask/SocketIO web dashboard for live monitoring
and manual control.

> The previous standalone Python prototype is preserved under `old/`. The main
> implementation is now the ROS 2 package layout under `dsr_gripper_tcp/` and
> `dsr_gripper_tcp_interfaces/`.

## Why This Exists

On the target E0509 setup, direct DRL/ROS 2 integration was not enough for
reliable gripper state feedback. The bridge solves this by:

- starting a DRL TCP server on the Doosan controller,
- communicating with the RH-P12-RN(A) over flange serial Modbus RTU inside DRL,
- sending command/response packets between ROS 2 and DRL over a lightweight TCP
  binary protocol,
- exposing state and commands through ROS 2 service/action/topic interfaces.

## Features

- **Bidirectional gripper control**: position, torque, motion profile, and state
  readback.
- **Service/action server node**: a single ROS 2 node owns the TCP bridge and
  exposes stable APIs for robot task nodes.
- **Safe grasp action**: performs one closing motion and judges grasp success
  using current feedback.
- **Live web dashboard**: monitor position, current, velocity, temperature, and
  torque state in a browser.
- **Controller recovery helpers**: retries DRL start, gripper initialize, and
  TCP reconnect paths for common controller/serial startup issues.
- **Legacy archive**: original standalone scripts are kept in `old/` for
  reference.

## Architecture

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

Do not run `gripper_service_node` and `web_dashboard_node` at the same time
unless the dashboard has been changed to use the service node as a client. Only
one node should own the TCP bridge.

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

## Requirements

- ROS 2 Humble or compatible ROS 2 distribution
- Doosan ROS 2 packages, including `dsr_msgs2`
- Python packages:

```bash
pip install flask flask-socketio
```

## Build

Place this repository in your ROS 2 workspace `src/` directory.

```bash
cd ~/ros2_ws
colcon build --packages-select dsr_gripper_tcp_interfaces dsr_gripper_tcp
source install/setup.bash
```

## Quick Start

### 1. Service/action server for robot integration

Use this when another robot control node should command the gripper.

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

Torque on:

```bash
ros2 service call /gripper_service/set_torque \
  dsr_gripper_tcp_interfaces/srv/SetTorque "{enabled: true}"
```

Open:

```bash
ros2 service call /gripper_service/set_position \
  dsr_gripper_tcp_interfaces/srv/SetPosition "{position: 0, timeout_sec: 5.0}"
```

Move to position:

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

### 2. Web dashboard node

Use this when you want browser-based manual control and telemetry.

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

### 3. CLI example

Simple close/open example:

```bash
ros2 run dsr_gripper_tcp example_gripper_tcp \
  --controller-host 110.120.1.56 \
  --namespace dsr01 \
  --service-prefix ""
```

## Safe Grasp Behavior

`SafeGrasp.action` performs a single closing motion to `target_position`. It does
not step the gripper incrementally. After the motion completes, grasp success is
judged using current feedback:

- success if `abs(final_current) >= max_current`
- success if the current increase from the start is greater than or equal to
  `current_delta_threshold`

The DRL-side move logic also treats a high current condition as a valid grasp
completion signal, so the gripper can stop before reaching the final close
position when it contacts an object.

## Service Node State Feedback

`/gripper_service/state` publishes `dsr_gripper_tcp_interfaces/msg/GripperState`.
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
motions and abort/recover if `object_lost` becomes true.

## Legacy Files

The original standalone prototype files are kept under `old/`:

- `old/example_gripper_tcp.py`
- `old/gripper_tcp_bridge.py`
- `old/gripper_tcp_protocol.py`
- `old/web_dashboard.py`
- `old/README.md`

They are kept for reference only. New development should use the ROS 2 packages.

## More Documentation

See package-level documentation:

- `dsr_gripper_tcp/README.md`
- `dsr_gripper_tcp_interfaces/msg/GripperState.msg`
- `dsr_gripper_tcp_interfaces/action/SafeGrasp.action`

## Status

This project is actively being developed and tested on a Doosan E0509 +
ROBOTIS RH-P12-RN(A) setup.
