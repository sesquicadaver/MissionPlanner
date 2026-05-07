# Технічне завдання

## Проєкт: Linux-native Mission Planner-compatible Ground Control Station

## Архітектура: Rust daemon + Qt/QML UI

---

# 1. Мета проєкту

Розробити Linux-native Ground Control Station (GCS) для екосистеми [ArduPilot](https://ardupilot.org/?utm_source=chatgpt.com) з повною підтримкою MAVLink, сумісністю з функціональністю [Mission Planner](https://missionplanner.com/?utm_source=chatgpt.com) на рівні протоколів, форматів і workflows, але без використання WinForms/Windows-залежностей.

Система повинна:

* працювати нативно на Linux;
* підтримувати desktop і companion computer deployment;
* мати daemon-oriented architecture;
* підтримувати headless mode;
* забезпечувати безпечну роботу через policy-gate;
* бути придатною для SITL/HITL;
* підтримувати розширення через plugin API.

---

# 2. Цільові платформи

## Desktop

* Ubuntu LTS
* Debian
* Arch Linux
* Fedora

## Embedded

* Raspberry Pi 5
* x86 mini-PC
* Jetson
* Industrial SBC

---

# 3. Основні вимоги

## Обов’язкові

* Linux-native
* MAVLink v2
* ArduPilot compatibility
* daemon/UI separation
* asynchronous architecture
* hotplug support
* offline operation
* deterministic behavior
* structured logging
* crash recovery
* headless operation

## Заборонено

* залежність від WinForms
* Mono/Wine runtime
* прямий доступ UI до serial/MAVLink
* Windows-only API
* blocking IO
* shared mutable global state

---

# 4. Архітектура

```text
+-------------------+
| Qt/QML UI         |
+-------------------+
          │
          │ gRPC / Unix Socket
          ▼
+-------------------+
| Rust Daemon       |
+-------------------+
│ MAVLink Router    |
│ Vehicle Manager   |
│ Params Service    |
│ Mission Service   |
│ Log Service       |
│ Link Manager      |
│ Safety Policy     |
│ Device Manager    |
└───────────────────┘
          │
          ▼
+-------------------+
| Vehicle / SITL    |
+-------------------+
```

---

# 5. Архітектурні принципи

## 5.1 Process isolation

UI і backend працюють як окремі процеси.

## 5.2 Capability model

Усі дії описуються як Effects/Capabilities.

Приклад:

```text
Capability::Arm
Capability::WriteMission
Capability::SetMode
Capability::FlashFirmware
```

## 5.3 Policy-gate

Усі effects проходять через:

```text
request
→ validation
→ policy
→ execution
→ audit log
```

deny-by-default.

## 5.4 Event-driven

Уся система працює через async events.

---

# 6. Технологічний стек

# Backend

## Мова

* Rust stable

## Runtime

* tokio

## IPC

* gRPC (tonic)
* Unix domain sockets

## Serialization

* protobuf
* serde

## MAVLink

* mavlink crate

## Logging

* tracing
* tracing-subscriber

## Storage

* sqlite
* sqlx

---

# Frontend

## UI

* Qt 6
* QML

## Rendering

* OpenGL/Vulkan

## Maps

* MapLibre GL
* offline tile cache

## Charts

* QtCharts

---

# 7. Backend subsystem requirements

# 7.1 Link Manager

## Supported transports

* UART
* USB CDC ACM
* TCP
* UDP
* multicast UDP
* serial radio

## Features

* auto reconnect
* heartbeat monitor
* timeout detection
* link quality metrics
* multi-link support
* redundancy

## Requirements

* non-blocking IO
* async reconnect
* hotplug detection
* configurable retry policy

---

# 7.2 MAVLink Router

## Features

* MAVLink v2
* multi-vehicle routing
* signing support
* stream rate control
* message filtering
* replay protection

## Supported messages

* HEARTBEAT
* SYS_STATUS
* ATTITUDE
* GLOBAL_POSITION_INT
* GPS_RAW_INT
* PARAM_VALUE
* MISSION_ITEM_INT
* COMMAND_ACK
* STATUSTEXT
* LOG_ENTRY
* LOG_DATA
* AUTOPILOT_VERSION
* BATTERY_STATUS

---

# 7.3 Vehicle Manager

## Features

* vehicle discovery
* vehicle registry
* type detection
* state tracking
* failsafe state monitoring
* EKF/GPS health
* mode tracking

---

# 7.4 Parameters Service

## Features

* full parameter sync
* cache
* diff detection
* validation
* batch update
* rollback

## Requirements

* atomic updates
* checksum validation
* parameter schema support

---

# 7.5 Mission Service

## Features

* upload/download
* mission validation
* geofence support
* rally points
* mission diff

## Supported mission items

* WAYPOINT
* TAKEOFF
* LAND
* RTL
* LOITER
* DO_SET_SERVO
* DO_CHANGE_SPEED

---

# 7.6 Log Service

## Formats

* BIN
* TLOG

## Features

* live recording
* replay
* telemetry timeline
* metadata indexing
* corruption detection

---

# 7.7 Firmware Service

## Features

* firmware download
* board detection
* flashing
* bootloader mode
* firmware validation

## Requirements

* safe flashing
* rollback detection
* CRC verification

---

# 7.8 Device Manager

## Features

* udev integration
* hotplug
* serial detection
* permission validation

## Supported devices

* FC
* telemetry radios
* GPS
* joystick
* gamepad

---

# 8. UI requirements

# 8.1 Dashboard

## Widgets

* artificial horizon
* GPS status
* battery
* EKF
* RC status
* mode indicator
* link quality
* arming state

---

# 8.2 Map View

## Features

* offline maps
* live tracking
* mission editing
* geofence visualization
* rally points
* home position

---

# 8.3 Mission Planner

## Features

* drag-and-drop waypoints
* terrain awareness
* altitude profiles
* spline editing
* mission validation

---

# 8.4 Parameter Editor

## Features

* grouped parameters
* search
* diff mode
* unsafe parameter warning
* staged apply

---

# 8.5 Calibration Wizards

## Required

* accelerometer
* compass
* radio
* ESC
* level horizon

---

# 8.6 Log Viewer

## Features

* synchronized graphs
* timeline
* event markers
* FFT visualization
* export

---

# 9. CLI requirements

```text
gcsctl connect
gcsctl status
gcsctl params pull
gcsctl params set
gcsctl mission upload
gcsctl mission download
gcsctl logs list
gcsctl logs fetch
```

---

# 10. SITL/HITL

## Supported

* ArduPilot SITL
* HITL
* multi-vehicle simulation

## Features

* simulation orchestration
* fault injection
* telemetry replay

---

# 11. Safety requirements

## Mandatory

* deny-by-default
* policy-gate
* audit logging
* capability validation
* emergency disconnect

## Dangerous operations

Require explicit confirmation:

* ARM
* DISARM
* firmware flash
* reboot
* parameter reset

---

# 12. Logging requirements

## Structured logs

JSON format.

## Levels

* TRACE
* DEBUG
* INFO
* WARN
* ERROR

## Requirements

* rotation
* crash-safe flushing
* per-subsystem filtering

---

# 13. Performance requirements

## Telemetry latency

≤ 20 ms internal pipeline latency.

## UI refresh

≥ 60 FPS map/dashboard.

## Memory

Stable under long-duration operation.

## CPU

No busy loops.

---

# 14. Reliability requirements

## Required

* watchdog
* crash recovery
* daemon restart safety
* corrupted log handling
* stale link detection

---

# 15. Security requirements

## IPC

Authenticated local IPC.

## MAVLink

Signing support.

## Filesystem

Least privilege.

## Device access

Controlled через udev groups.

---

# 16. Plugin system

## Supported

* telemetry processors
* custom widgets
* mission validators
* exporters

## Isolation

Plugins sandboxed.

---

# 17. Testing requirements

# Unit tests

Coverage mandatory.

# Integration tests

SITL-driven.

# Stress tests

Long-duration telemetry sessions.

# Fault injection

Required.

---

# 18. Packaging

## Formats

* AppImage
* deb
* rpm
* tarball

## Services

systemd integration.

---

# 19. Deployment modes

## Desktop mode

Full UI.

## Headless mode

Daemon only.

## Embedded mode

Reduced UI + remote dashboard.

---

# 20. Roadmap

# Phase 1

Core daemon:

* MAVLink
* links
* telemetry
* params
* missions

# Phase 2

Qt/QML UI:

* dashboard
* map
* params
* mission editor

# Phase 3

Logs/calibration/flashing.

# Phase 4

Plugins/SITL orchestration/HITL.

# Phase 5

Distributed multi-vehicle support.
