# Modular Robotics Platform (ROS 2 + ESP32)

A reusable ROS 2-based pattern for controlling robots through a Steam Deck
and Foxglove Studio, starting with a 4-wheel differential-drive rover. Each
robot's firmware keeps speaking whatever native protocol it already uses
(UDP, WebSocket, etc.); a small robot-specific ROS 2 "bridge" node is the
only thing that changes per robot type. Foxglove becomes a single control
and telemetry surface for every robot on the platform.

## Architecture

```
Foxglove Studio
      |  (WebSocket)
Foxglove Bridge          [foxglove-bridge.service, on Steam Deck]
      |
ROS 2                    [topics: /cmd_vel, /wheel_cmd, ...]
      |
robot-specific bridge node   e.g. rover_udp_bridge
      |  (native protocol: UDP, WebSocket, etc.)
Robot firmware (ESP32)
      |
Motor drivers / servos / sensors
```

ROS 2 is the layer everything is built around: every robot on this platform
exposes standard ROS 2 topics and message types (`geometry_msgs/Twist` for
velocity control, `sensor_msgs/JointState` for servo-driven robots, and so
on), and Foxglove talks to ROS 2, never to a robot's firmware directly. The
firmware itself never runs ROS; it only needs to speak its own native wire
protocol, and a ROS 2 node on the Steam Deck handles the translation in
both directions.

## What's in this repo

| Path | What it is |
|---|---|
| [`rover/`](rover/README.md) | The first robot on the platform: an ESP32-based 4WD differential-drive rover. Hardware, firmware, networking, and status. |
| [`controller/`](controller/README.md) | The plan (and prior, pre-ROS implementation) for a universal ROS 2 controller pattern usable across future robot types. |
| [`ros2_ws/src/rover_udp_bridge/`](ros2_ws/src/rover_udp_bridge) | The ROS 2 package: two nodes that bridge ROS 2 topics to the rover's UDP motor protocol. |

## ROS 2 package: `rover_udp_bridge`

This is the only ROS 2 code in the repo so far, and it is the concrete
example the rest of the platform's "bridge node per robot" pattern is
modeled on.

| Node | Subscribes to | Message type | Role |
|---|---|---|---|
| `raw_wheel_udp_node` | `wheel_cmd` | `std_msgs/Int32MultiArray` | Direct per-wheel PWM, `[FL, FR, RL, RR]`, no kinematics. Used for diagnostics and isolating ROS/UDP/motor issues. |
| `cmd_vel_to_udp_node` | `cmd_vel` | `geometry_msgs/Twist` | Standard ROS 2 mobile-robot control interface. Converts linear/angular velocity into per-wheel PWM via differential-drive kinematics. Same message type Foxglove's Teleop panel publishes, and the same message type a future Nav2 stack would use. |

Both nodes re-publish the last command at a fixed rate and auto-stop if no
new ROS 2 message arrives within ~0.5s, mirroring the ESP32 firmware's own
failsafe so the two layers can't disagree about whether the rover should be
moving.

### Build and run (ROS 2, via colcon)

```bash
cd ros2_ws
colcon build --packages-select rover_udp_bridge
source install/setup.bash

# Twist-based teleop / Nav2-compatible control:
ros2 launch rover_udp_bridge cmd_vel_launch.py

# Raw per-wheel diagnostics:
ros2 launch rover_udp_bridge raw_wheel_launch.py
```

> If you're on a non-Ubuntu Linux distribution (SteamOS/Arch, for example)
> with no system-wide `apt`-installed ROS 2, install ROS 2 via
> [Pixi/RoboStack](https://robostack.github.io/) and run the above inside a
> `pixi shell` session.

Quick manual test that bypasses Foxglove entirely:

```bash
ros2 topic pub --once /wheel_cmd std_msgs/msg/Int32MultiArray \
"{layout: {dim: [], data_offset: 0}, data: [128, 128, 128, 128]}"
```

## Project status

- ESP32 firmware, motors, Wi-Fi, OTA: working and stable.
- `rover_udp_bridge` ROS 2 package: built and confirmed working end-to-end via CLI (`ros2 topic pub`).
- Foxglove's Publish panel into `/wheel_cmd`: currently broken. CLI publish works; Foxglove's does not, even though the bridge node appears to register the publisher without error. Suspected `ROS_DOMAIN_ID` / `RMW_IMPLEMENTATION` mismatch between the interactive shell and the `foxglove-bridge.service` systemd environment.
- `cmd_vel_to_udp_node` (Twist-based teleop): built, not yet verified in practice, blocked on the Foxglove issue above.
- Odometry / sensor feedback: not started.
- Universal controller pattern (`controller/`): documented plan only, not yet implemented.

See [`rover/README.md`](rover/README.md) for full hardware, networking, and firmware details, and [`controller/README.md`](controller/README.md) for the multi-robot ROS 2 pattern this rover is the first instance of.

## Roadmap

### Phase 1: Rover drivetrain and ROS 2 bridge (current)
- [x] Assemble 4WD chassis, wire dual L298N drivers to ESP32
- [x] ESP32 firmware: Wi-Fi, OTA, UDP motor protocol, failsafe stop
- [x] `rover_udp_bridge` ROS 2 package: `raw_wheel_udp_node` and `cmd_vel_to_udp_node`
- [x] Verify end-to-end control via `ros2 topic pub`
- [ ] Fix Foxglove Publish panel -> ROS 2 topic issue
- [ ] Verify `cmd_vel` / Twist teleop end-to-end through Foxglove

### Phase 2: Sensing
- [ ] Connect ultrasonic sensors, publish as a ROS 2 topic (e.g. `sensor_msgs/Range`)
- [ ] Connect IR sensors for line tracking / edge detection
- [ ] Visualize live sensor topics in Foxglove Studio
- [ ] Basic odometry

### Phase 3: Autonomy
- [ ] Obstacle avoidance behavior as a ROS 2 node
- [ ] Line-following mode
- [ ] Head swivel (pan servo) integration
- [ ] Evaluate Nav2 for autonomous navigation on top of `cmd_vel`

### Phase 4: Platform / multi-robot
- [ ] Implement the `controller/` pattern for a second, servo-driven robot
- [ ] OTA firmware updates managed from the ROS 2 side
- [ ] Standardized ROS 2 topic/message conventions across robot types
- [ ] Multi-robot control from a single Foxglove/ROS 2 session

### Phase 5: Integration
- [ ] Home automation integration (e.g. Home Assistant bridge)
- [ ] Persistent logging / telemetry storage

## Repo structure

```
.
├── README.md                          # this file
├── rover/
│   └── README.md                      # rover hardware, firmware, networking, status
├── controller/
│   └── README.md                      # universal ROS 2 controller pattern (plan)
└── ros2_ws/
    └── src/
        └── rover_udp_bridge/           # ROS 2 package (ament_python)
            ├── package.xml
            ├── setup.py
            ├── setup.cfg
            ├── resource/
            │   └── rover_udp_bridge
            ├── launch/
            │   ├── cmd_vel_launch.py
            │   └── raw_wheel_launch.py
            └── rover_udp_bridge/
                ├── __init__.py
                ├── raw_wheel_udp_node.py
                └── cmd_vel_to_udp_node.py
```

## License

MIT, see [`LICENSE`](LICENSE).
