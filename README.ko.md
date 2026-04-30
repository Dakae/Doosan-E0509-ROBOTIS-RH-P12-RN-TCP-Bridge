# Doosan E0509 - ROBOTIS RH-P12-RN TCP Bridge

[English](./README.md) | [한국어](./README.ko.md)

---

## 소개

이 저장소는 **Doosan E0509** 로봇에 장착된
**ROBOTIS RH-P12-RN(A)** 그리퍼를 ROS 2에서 제어하고 상태를 읽기 위한
TCP 브리지 프로젝트입니다.

Doosan 컨트롤러에서는 DRL 스크립트가 TCP 서버를 실행하고, DRL 내부에서
Flange RS-485 Modbus RTU로 그리퍼와 통신합니다. ROS 2 쪽에서는 이 TCP
브리지를 통해 그리퍼를 노드, 토픽, 서비스, 액션 인터페이스로 사용할 수
있습니다.

기존 단일 Python 프로토타입은 `old/` 폴더에 보관되어 있습니다. 현재 메인
구현은 아래 ROS 2 패키지 구조입니다.

- `dsr_gripper_tcp/`
- `dsr_gripper_tcp_interfaces/`

---

## 개발 배경

대상 Doosan E0509 환경에서는 DRL만으로 그리퍼 상태를 ROS 2에 안정적으로
피드백하기 어려웠습니다. 특히 그리퍼 명령은 가능하지만, 위치/전류/온도 같은
상태를 ROS 2 쪽에서 신뢰성 있게 읽어오는 경로가 필요했습니다.

이 브리지는 다음 방식으로 문제를 해결합니다.

- Doosan 컨트롤러 내부 DRL 스크립트에서 TCP 서버 실행
- DRL에서 Flange RS-485 Modbus RTU로 RH-P12-RN(A) 그리퍼와 통신
- ROS 2와 DRL 사이에 경량 binary command/response TCP 프로토콜 사용
- 그리퍼 상태를 ROS 2 topic/service/action으로 노출

---

## 주요 기능

- **양방향 그리퍼 제어**
  - 위치 이동
  - 토크 ON/OFF
  - 모션 프로파일 설정
  - 상태 읽기

- **ROS 2 서비스/액션 서버**
  - 하나의 노드가 TCP bridge를 단일 소유
  - 로봇 작업 노드는 안정적인 service/action API로 그리퍼 제어 가능

- **안전 파지 액션**
  - 한 번에 닫는 모션 수행
  - 전류 피드백으로 파지 성공 여부 판단
  - 로봇 작업 로직에서 사용할 수 있는 action feedback/result 제공

- **실시간 웹 대시보드**
  - 브라우저 기반 모니터링 및 수동 제어
  - 위치, 전류, 속도, 온도, 토크, 이동 상태 확인

- **컨트롤러 복구 보조 로직**
  - DRL start 재시도
  - TCP 재연결
  - 그리퍼 initialize 재시도
  - Flange serial 복구 로직

---

## 시스템 구조

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

웹 대시보드는 bridge를 직접 소유하는 방식으로 실행할 수도 있습니다.

```text
Browser <-- SocketIO --> web_dashboard_node --> TCP bridge --> DRL --> Gripper
```

> `gripper_service_node`와 `web_dashboard_node`는 동시에 실행하지 않는 것을
> 권장합니다. 둘 다 TCP bridge를 직접 소유하기 때문입니다. 동시에 사용하려면
> 웹 대시보드를 service node의 client 구조로 변경해야 합니다.

---

## 저장소 구조

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

## 요구 사항

- Ubuntu 22.04
- ROS 2 Humble
- Doosan ROS 2 패키지 (`dsr_msgs2` 포함)
- Python 3.10+
- Python 패키지:

```bash
pip install flask flask-socketio
```

---

## 빌드

이 저장소를 ROS 2 workspace의 `src/` 아래에 위치시킨 뒤 빌드합니다.

```bash
cd ~/ros2_ws
colcon build --packages-select dsr_gripper_tcp_interfaces dsr_gripper_tcp
source install/setup.bash
```

---

## 빠른 실행

### 1. 서비스/액션 서버 실행

다른 로봇 제어 노드가 그리퍼를 제어해야 할 때 사용하는 권장 방식입니다.

```bash
ros2 launch dsr_gripper_tcp gripper_service_node.launch.py \
  controller_host:=110.120.1.56 \
  namespace:=dsr01 \
  service_prefix:=
```

주요 인터페이스:

- `/gripper_service/state`
- `/gripper_service/joint_state`
- `/gripper_service/get_state`
- `/gripper_service/get_position`
- `/gripper_service/set_position`
- `/gripper_service/set_motion_profile`
- `/gripper_service/get_motion_profile`
- `/gripper_service/set_torque`
- `/gripper_service/safe_grasp`

토크 ON:

```bash
ros2 service call /gripper_service/set_torque \
  dsr_gripper_tcp_interfaces/srv/SetTorque "{enabled: true}"
```

열기:

```bash
ros2 service call /gripper_service/set_position \
  dsr_gripper_tcp_interfaces/srv/SetPosition "{position: 0, timeout_sec: 5.0}"
```

특정 위치로 이동:

```bash
ros2 service call /gripper_service/set_position \
  dsr_gripper_tcp_interfaces/srv/SetPosition "{position: 700, timeout_sec: 5.0}"
```

안전 파지:

```bash
ros2 action send_goal /gripper_service/safe_grasp \
  dsr_gripper_tcp_interfaces/action/SafeGrasp \
  "{target_position: 700, max_current: 400, current_delta_threshold: 120, timeout_sec: 8.0}" \
  --feedback
```

상태 모니터링:

```bash
ros2 topic echo /gripper_service/state
```

### 2. 웹 대시보드 실행

브라우저에서 그리퍼를 모니터링하고 수동 제어하고 싶을 때 사용합니다.

```bash
ros2 launch dsr_gripper_tcp web_dashboard_node.launch.py \
  controller_host:=110.120.1.56 \
  namespace:=dsr01 \
  service_prefix:= \
  web_port:=5000
```

브라우저에서 접속:

```text
http://localhost:5000
```

### 3. CLI 예제 실행

간단한 close/open 예제입니다.

```bash
ros2 run dsr_gripper_tcp example_gripper_tcp \
  --controller-host 110.120.1.56 \
  --namespace dsr01 \
  --service-prefix ""
```

---

## 안전 파지 동작

`SafeGrasp.action`은 `target_position`까지 한 번에 닫는 모션을 수행합니다.
그리퍼를 step 단위로 조금씩 움직이지 않습니다.

모션 완료 후 아래 조건으로 파지 성공을 판단합니다.

- `abs(final_current) >= max_current`
- 시작 전류 대비 증가량이 `current_delta_threshold` 이상

DRL 내부 move 로직도 전류가 충분히 높아지면 물체를 잡은 것으로 판단하고
정상 완료 신호를 반환합니다. 따라서 물체에 닿으면 최종 close 위치에 도달하기
전에 멈출 수 있습니다.

---

## 상태 피드백

`/gripper_service/state`는
`dsr_gripper_tcp_interfaces/msg/GripperState`를 publish합니다.

중요 필드:

- `present_position`: 현재 그리퍼 위치 pulse
- `goal_position`: 마지막 목표 위치
- `present_current`: 현재 전류
- `present_velocity`: 현재 속도
- `torque_enabled`: 토크 상태
- `grasp_detected`: 전류 기반 파지 감지
- `object_lost`: 파지 후 물체 놓침 의심
- `status_text`: 노드 내부 상태 메시지

로봇 작업 action server는 로봇팔 이동 중 이 topic을 subscribe하고,
`object_lost`가 true가 되면 작업을 중단하거나 복구 동작을 수행할 수 있습니다.

---

## Legacy 파일

초기 단일 파일 프로토타입은 `old/` 아래에 보관되어 있습니다.

- `old/example_gripper_tcp.py`
- `old/gripper_tcp_bridge.py`
- `old/gripper_tcp_protocol.py`
- `old/web_dashboard.py`
- `old/README.md`

참고용으로만 보관하며, 새 개발은 ROS 2 패키지 구조를 기준으로 진행합니다.

---

## 추가 문서

- `dsr_gripper_tcp/README.md`
- `dsr_gripper_tcp_interfaces/msg/GripperState.msg`
- `dsr_gripper_tcp_interfaces/action/SafeGrasp.action`

---

## 상태

이 프로젝트는 Doosan E0509 + ROBOTIS RH-P12-RN(A) 환경에서 개발 및 테스트
중입니다.
