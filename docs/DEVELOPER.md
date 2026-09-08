# Developer guide — `so_arm101_urdf`

ROS 2 **Jazzy** description package for the **SO-ARM101** 6-DOF follower arm.
Bringup, hardware plugin, and cameras live in **`lucy_ros_packages`**.

---

## 1. Role of this package

| Concern | Lives in |
|---------|----------|
| Robot description (xacro, meshes, joint limits, collisions) | `description/` |
| ros2_control hardware interfaces (real / mock) | `description/ros2_control/` |
| Gazebo physics + generated plugin | `description/gazebo/` |
| Hardware mapping (board, actuators, encoder placeholders) | `config/hardware/active.yaml` |
| Controller parameters | `config/controllers.yaml` (generated) |
| RViz layout | `config/so_arm101_rviz.rviz` |
| Launches | `launch/` |

**Robot selection:** `robot_package:=so_arm101_urdf` or `LUCY_ROBOT_PACKAGE=so_arm101_urdf`.

Bringup and `lucy_config_pipeline` both read `config/control.launch.yaml` for the
entry xacro / base path / controllers file (do not hardcode `inmoov.urdf.xacro`).

**Control-panel 3D viewer:** meshes are served by `/mesh/get`. STL payloads use
`encoding=zlib_base64` (zlib then base64). Rebuild `lucy_msgs` +
`lucy_config_pipeline` and refresh the panel after mesh-service changes.

---

## 2. Layout

```text
so_arm101_urdf/
├── package.xml / CMakeLists.txt / version.txt
├── docs/DEVELOPER.md
├── launch/                 # control, gazebo, joint_preview, rviz, rviz_standalone
├── config/
│   ├── control.launch.yaml
│   ├── controllers.yaml    # generated
│   ├── so_arm101_rviz.rviz
│   └── hardware/
│       ├── active.yaml
│       ├── active_meta.yaml
│       └── configs/default.yaml
├── description/
│   ├── urdf/so_arm101.urdf.xacro
│   ├── robot_description/
│   │   ├── urdf/properties.xacro
│   │   ├── urdf/robot_description.urdf.xacro
│   │   └── meshes/stl/     # 13 STL meshes
│   ├── ros2_control/so_arm101_ros2_control.xacro   # generated
│   └── gazebo/
│       ├── so_arm101_gazebo_physics.xacro
│       ├── gazebo.xacro                            # generated
│       └── gazebo_bridge.yaml                      # generated
├── worlds/default.sdf
└── test/test_xacro_smoke.py
```

---

## 3. Joints and hardware

| Joint | Approx. range | Servo |
|-------|---------------|-------|
| `Rotation` | ±110° | STS3215 |
| `Pitch` | ±100° | STS3215 |
| `Elbow` | −100° … 90° | STS3215 |
| `Wrist_Pitch` | ±95° | STS3215 |
| `Wrist_Roll` | ±160° | STS3215 |
| `Jaw` | −10° … 100° | STS3215 |

Kinematics come from the calibrated `so101_new_calib.urdf` (SO-ARM100 / onshape-to-robot lineage).

**Lucy schema note:** `servo_type` must be `180` / `270` / `300`. Hardware YAML uses `'300'` as a stand-in until bus-servo support exists. Pin numbers and serial IDs are placeholders for hardware integration.

---

## 4. Joint-state feedback (blue dots)

Today Lucy is **open-loop** for real hardware:

- `LucySystemHardware::read()` echoes the last commanded position
- Control-panel blue dots subscribe to `/joint_states` (command echo, not encoders)

STS3215 servos **do** have magnetic encoders (`Present_Position`, 4096 ticks/rev). This package prepares disabled `type: encoder` sensor rows in `active.yaml`. Enabling closed-loop feedback later requires:

1. Bus-servo driver (half-duplex serial)
2. Firmware publishing measured positions on `sensors/so_arm`
3. `LucySystemHardware::read()` consuming measured joints
4. Schema / firmware templates that understand encoder sensors

Until then, mock / Gazebo behave like other Lucy robots: mock echoes commands; Gazebo physics can lag.

---

## 5. Launch files

| Launch | Purpose |
|--------|---------|
| `joint_preview.launch.py` | RSP + JSP GUI + RViz (no HW) |
| `control.launch.py` | `lucy_control_supervisor` (real or `use_mock_hardware:=true`) |
| `gazebo.launch.py` | gz-sim + spawn + controllers |
| `rviz.launch.py` / `rviz_standalone.launch.py` | RViz only |

```bash
ros2 launch so_arm101_urdf joint_preview.launch.py
ros2 launch so_arm101_urdf control.launch.py use_mock_hardware:=true
LUCY_ROBOT_PACKAGE=so_arm101_urdf pixi run core
```

---

## 6. Xacro entry

`description/urdf/so_arm101.urdf.xacro`:

- Includes properties + body
- Unless `use_gazebo_sim`: includes generated `so_arm101_ros2_control.xacro`
- If `use_gazebo_sim`: includes physics + generated `gazebo.xacro`

Standalone expand:

```bash
ros2 run xacro xacro description/urdf/so_arm101.urdf.xacro \
  base_path:=$(pwd)/description \
  controller_config:=$(pwd)/config/controllers.yaml \
  use_mock_hardware:=true
```

---

## 7. Regenerating configs

From `lucy_ws` (with workspace sourced / pixi shell):

```bash
# Generate into a temp dir, then install into package paths
OUT=/tmp/so_arm101_gen
mkdir -p "$OUT"
generate_config \
  --input src/so_arm101_urdf/config/hardware/active.yaml \
  --urdf src/so_arm101_urdf/description/urdf/so_arm101.urdf.xacro \
  --base-path src/so_arm101_urdf/description \
  --controller-config src/so_arm101_urdf/config/controllers.yaml \
  --output-dir "$OUT" \
  --targets all

cp "$OUT"/so_arm101_ros2_control.xacro \
  src/so_arm101_urdf/description/ros2_control/
cp "$OUT"/controllers.yaml src/so_arm101_urdf/config/
cp "$OUT"/gazebo.xacro "$OUT"/gazebo_bridge.yaml \
  src/so_arm101_urdf/description/gazebo/
```

Or use the control-panel **VALIDATE → ACTIVATE → RELOAD** pipeline with `robot_package:=so_arm101_urdf`.

---

## 8. Build and test

```bash
colcon build --symlink-install --packages-select so_arm101_urdf
source install/setup.bash
colcon test --packages-select so_arm101_urdf --event-handlers console_direct+
```

No `package.xml` dependency on `lucy_ros2_control` (avoid cycles). Keep both packages built in the same workspace for control launches.

---

## 9. Differences from InMoov / Thais

| | InMoov / Thais | SO-ARM101 |
|--|----------------|-----------|
| Form factor | Full humanoid | Single 6-DOF arm |
| Entry xacro | `inmoov.urdf.xacro` | `so_arm101.urdf.xacro` |
| Meshes | Collada DAE | STL |
| Servos | PWM hobby | STS3215 bus (schema still PWM-typed) |
| Controllers | left/right arm + torso_head | `so_arm_controller` |
| Encoders | None | Prepared (disabled) |

---

## 10. Future work

- Hardware serial IDs / bus-servo firmware
- Closed-loop encoder → `/joint_states`
- Leader → follower teleoperation bridge
- Dual-arm (leader + follower) configs
