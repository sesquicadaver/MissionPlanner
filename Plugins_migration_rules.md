# ТЗ: перенос плагінів Mission Planner на Linux-native GCS

## 1. Мета

Створити механізм коректного переносу функціональності плагінів Mission Planner на нову платформу:

```text
Rust daemon + Qt/QML UI + Plugin SDK
```

Перенос виконується не як пряме копіювання C#/WinForms-коду, а як міграція функцій плагінів у нову модульну, безпечну, Linux-native архітектуру.

---

## 2. Базовий принцип

```text
Переноситься функціональність, а не legacy-реалізація.
```

Заборонено:

```text
- тягнути WinForms
- тягнути Windows API
- давати плагінам прямий доступ до MAVLink/serial/filesystem
- виконувати небезпечні дії в обхід policy-gate
- створювати власну мову плагінів
```

---

## 3. Цільова архітектура

```text
Qt/QML UI
  └─ Plugin UI host
        │
        ▼
Rust daemon
  ├─ Plugin Manager
  ├─ Capability Gate
  ├─ Policy Gate
  ├─ MAVLink API
  ├─ Telemetry API
  ├─ Params API
  ├─ Mission API
  ├─ Logs API
  └─ Audit Logger

Plugin runtimes
  ├─ Rust native plugins
  ├─ WASM sandbox plugins
  ├─ Python engineering plugins
  └─ QML UI plugins
```

---

## 4. Класифікація плагінів MP

Кожен плагін Mission Planner перед переносом класифікується.

| Тип             | Приклади                                | Цільова реалізація              |
| --------------- | --------------------------------------- | ------------------------------- |
| Critical/system | flashing, calibration, motor test       | Rust native                     |
| Mission tools   | validators, route tools, terrain checks | Rust або WASM                   |
| Analysis/log    | BIN/TLOG parsing, graphs, diagnostics   | Python/WASM/Rust                |
| UI panels       | custom dashboards, forms, map overlays  | QML                             |
| Automation      | checks, scripts, workflows              | WASM/Python                     |
| Legacy/obsolete | застарілі інтеграції                    | не переносити без обґрунтування |

---

## 5. Обов’язковий процес міграції

Для кожного плагіна:

```text
1. Інвентаризація
2. Аналіз залежностей
3. Визначення capabilities
4. Визначення API calls
5. Вибір runtime
6. Проєктування нового модуля
7. Реалізація
8. Тести
9. SITL/HITL перевірка
10. Документація
11. Верифікація сумісності
```

---

## 6. Plugin Manifest

Кожен плагін має мати `plugin.toml`.

```toml
id = "battery_health"
name = "Battery Health"
version = "0.1.0"
api_version = "1"
runtime = "wasm"

[capabilities]
telemetry.read = true
params.read = true
params.write = false
mission.read = false
mission.write = false
mavlink.raw_send = false
filesystem.read = false
filesystem.write = false
network.access = false

[ui]
dashboard_widget = true
map_overlay = false
settings_page = true

[safety]
requires_explicit_confirmation = false
allowed_vehicle_states = ["disarmed", "armed"]
```

---

## 7. Capability model

Плагін декларує всі потрібні права явно.

Основні capabilities:

```text
telemetry.read
telemetry.subscribe
params.read
params.write
mission.read
mission.write
logs.read
logs.write
map.overlay
ui.widget
mavlink.command
mavlink.raw_send
vehicle.arm
vehicle.disarm
vehicle.set_mode
firmware.flash
device.access
filesystem.read
filesystem.write
network.access
```

За замовчуванням:

```text
deny-by-default
```

---

## 8. Policy-gate

Усі небезпечні операції проходять через єдиний pipeline:

```text
plugin request
→ schema validation
→ capability check
→ policy check
→ vehicle state check
→ execution
→ audit log
→ result
```

Обов’язково через policy-gate:

```text
arm/disarm
mode change
mission upload
parameter write
firmware flash
motor test
actuator test
reboot
raw MAVLink send
```

---

## 9. API для плагінів

### Telemetry API

```text
subscribe_telemetry(topic)
get_vehicle_state()
get_link_status()
get_battery_status()
get_gps_status()
get_ekf_status()
```

### Params API

```text
list_params()
get_param(name)
set_param(name, value)
diff_params(snapshot)
validate_param_write(name, value)
```

### Mission API

```text
download_mission()
validate_mission(mission)
upload_mission(mission)
diff_mission(local, remote)
```

### Logs API

```text
list_logs()
read_log(log_id)
stream_tlog()
add_log_marker()
export_report()
```

### UI API

```text
register_dashboard_widget()
register_map_overlay()
register_settings_page()
show_notification()
request_user_confirmation()
```

### MAVLink API

```text
send_command_long()
send_command_int()
subscribe_message(message_id)
```

`raw_send` заборонений за замовчуванням.

---

## 10. Runtime-стратегія

## 10.1 Rust native plugins

Використовувати для:

```text
critical/system/safety-sensitive plugins
```

Вимоги:

```text
cargo fmt
cargo clippy
cargo test
no unwrap
no unsafe без review
```

---

## 10.2 WASM plugins

Використовувати для:

```text
mission validators
automation
portable analysis logic
sandboxed extensions
```

Вимоги:

```text
memory limit
execution timeout
no direct filesystem
no direct network
host API only
```

---

## 10.3 Python plugins

Використовувати для:

```text
engineering tools
diagnostics
log analysis
rapid prototyping
```

Заборонено для:

```text
firmware flashing
arming
motor tests
direct MAVLink control
```

---

## 10.4 QML plugins

Використовувати тільки для UI.

QML-плагін:

```text
не має business logic
не має MAVLink доступу
не має serial доступу
не пише параметри напряму
```

---

## 11. Legacy compatibility adapter

Створити таблицю відповідності:

```text
Mission Planner Plugin API → Linux GCS Plugin API
```

Приклад:

```text
MainV2.comPort.MAV.param["X"]
→ params.get_param("X")

MainV2.comPort.setParam("X", value)
→ params.set_param("X", value)

MainV2.comPort.doCommand(...)
→ mavlink.send_command_long(...)
```

C#-код не виконується напряму. Adapter використовується як міграційна документація і tooling для semi-automatic porting.

---

## 12. Інвентаризація плагінів

Для кожного MP-плагіна створити запис:

```yaml
id:
source_path:
purpose:
current_language: C#
ui_dependency: true/false
windows_dependency: true/false
mavlink_access:
param_access:
mission_access:
filesystem_access:
network_access:
risk_level: low/medium/high/critical
target_runtime:
migration_status:
test_requirements:
```

---

## 13. Критерії переносу

Плагін переноситься, якщо:

```text
- функція актуальна для Linux GCS
- немає дубля в core
- є чіткий API boundary
- можна протестувати через unit/integration/SITL
- capabilities можуть бути формально описані
```

Плагін не переноситься, якщо:

```text
- прив’язаний до Windows-only UI
- дублює core-функцію
- не має реального сценарію використання
- небезпечний і не піддається policy-gate
```

---

## 14. Тестування

Для кожного перенесеного плагіна обов’язково:

```text
unit tests
API contract tests
capability tests
policy-gate tests
failure tests
SITL tests, якщо є взаємодія з vehicle
UI tests, якщо є QML
```

Критичні сценарії:

```text
link loss
vehicle disconnected
parameter timeout
COMMAND_ACK failure
mission upload interruption
invalid telemetry
permission denied
plugin crash
```

---

## 15. Безпека

Плагін не може:

```text
- самостійно відкривати serial port
- напряму писати MAVLink frame
- обходити daemon API
- змінювати config без дозволу
- читати довільні файли
- писати в довільні директорії
- відкривати network socket без capability
```

Усі дії логуються:

```text
plugin_id
operation
capability
vehicle_id
timestamp
result
error
trace_id
```

---

## 16. UI-вимоги

UI-плагіни мають підтримувати:

```text
dashboard widget
map overlay
settings page
tool panel
status indicator
```

Вимоги:

```text
QML ≤ 250 рядків на компонент
business logic у daemon/plugin backend
UI не блокує main thread
усі команди async
усі помилки показуються явно
```

---

## 17. Версіонування API

Plugin API має семантичне версіонування:

```text
1.0.0
```

Правила:

```text
major — breaking changes
minor — backward-compatible API
patch — bugfix
```

Плагін вказує:

```toml
api_version = "1"
```

Несумісний плагін не завантажується.

---

## 18. Структура plugin SDK

```text
plugin-sdk/
  rust/
  wasm/
  python/
  qml/
  schemas/
  examples/
  docs/
```

Приклади обов’язкові:

```text
battery_health
mission_validator
log_analyzer
map_overlay
params_readonly_panel
```

---

## 19. Definition of Done для перенесеного плагіна

Плагін вважається перенесеним, якщо:

```text
manifest створено
capabilities описано
legacy behavior задокументовано
нову реалізацію завершено
тести проходять
SITL перевірка пройдена
policy-gate покрито тестами
немає direct MAVLink/serial bypass
логування реалізовано
документація оновлена
приклади використання додано
```

---

## 20. Roadmap

### Phase 1 — Inventory

```text
знайти всі MP plugins
класифікувати
оцінити ризики
створити migration matrix
```

### Phase 2 — Plugin SDK MVP

```text
plugin.toml
capability model
telemetry read API
params read API
QML widget registration
audit logging
```

### Phase 3 — Representative ports

Перенести 3–5 типових плагінів:

```text
1 UI widget
1 telemetry analysis
1 mission validator
1 params tool
1 log analyzer
```

### Phase 4 — Critical plugins

```text
calibration
mission upload helpers
firmware-related tools
safety checks
```

### Phase 5 — Ecosystem

```text
plugin registry
plugin signing
version compatibility
package manager
developer docs
```

---

## 21. Головне правило

```text
Плагін — це не довірений код всередині GCS.
Плагін — це обмежений учасник системи, який працює тільки через API, capabilities і policy-gate.
```
