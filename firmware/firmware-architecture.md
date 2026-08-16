# Thermone V1 Firmware Architecture

## Purpose

This document defines the internal software architecture for Thermone V1 firmware.

The firmware runs on the Thermone ESP32 controller and is responsible for:

- Booting reliably
- Managing hardware
- Reading temperature probes
- Managing Wi-Fi and Ethernet
- Providing first-time provisioning
- Registering with Thermone Cloud
- Uploading telemetry
- Sending heartbeats
- Fetching configuration
- Polling commands
- Performing OTA updates
- Buffering telemetry while offline
- Handling recovery
- Managing device state
- Driving the status LED
- Handling the setup/reset button

The firmware architecture should remain modular so that networking, sensors, OTA, storage, and cloud logic do not become tightly coupled.

---

# Core Design Principles

Thermone firmware should follow these rules:

1. Temperature monitoring must continue even when cloud communication fails.
2. Networking must never block sensor monitoring indefinitely.
3. Every physical probe port has its own logical identity.
4. Hardware-specific GPIO details should be isolated from application logic.
5. Device secrets must not be logged.
6. OTA updates must be recoverable.
7. Device configuration must survive normal reboots.
8. The firmware must support factory reset and recovery mode.
9. Cloud outages must not stop local monitoring.
10. Modules should communicate through clear interfaces.

---

# Firmware Technology

Recommended production firmware stack:

```text
ESP-IDF
```

Possible development environment:

```text
PlatformIO
+
ESP-IDF
```

Arduino libraries may be used selectively where appropriate, but the production architecture should be built around ESP-IDF capabilities.

---

# Repository

Firmware lives in:

```text
Thermone/firmware
```

Recommended initial structure:

```text
firmware/
├── CMakeLists.txt
├── sdkconfig.defaults
├── partitions.csv
├── README.md
│
├── main/
│   ├── main.cpp
│   └── CMakeLists.txt
│
├── components/
│   ├── board/
│   ├── sensors/
│   ├── network/
│   ├── provisioning/
│   ├── identity/
│   ├── cloud/
│   ├── telemetry/
│   ├── storage/
│   ├── ota/
│   ├── commands/
│   ├── status/
│   ├── button/
│   ├── time/
│   ├── health/
│   ├── watchdog/
│   └── recovery/
│
├── boards/
│   └── thermone_v1/
│       ├── board.h
│       └── pins.h
│
├── tests/
│
└── scripts/
```

---

# High-Level Firmware Architecture

```text
                   ┌────────────────────┐
                   │       main         │
                   └─────────┬──────────┘
                             │
                             ▼
                   ┌────────────────────┐
                   │  Device Manager    │
                   └─────────┬──────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
       Sensors            Network            Storage
          │                  │                  │
          ▼                  ▼                  ▼
      Telemetry            Cloud              Config
          │                  │
          └──────────┬───────┘
                     │
                     ▼
                 Device State
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
        Status      OTA       Commands
          │
          ▼
       LED / UI
```

---

# Main Entry Point

The firmware entry point should remain small.

Conceptually:

```cpp
void app_main()
{
    initialize_board();
    initialize_storage();
    initialize_identity();
    initialize_status();
    initialize_sensors();
    initialize_network();
    initialize_cloud();
    initialize_tasks();
}
```

The entry point should not contain application business logic.

---

# Device Manager

The Device Manager coordinates global device state.

Responsibilities:

- Track lifecycle state
- Coordinate startup
- Manage transitions between setup and normal operation
- Coordinate recovery
- Notify status subsystem
- Expose device-wide state

Possible states include:

```text
BOOTING
FACTORY_UNPROVISIONED
NETWORK_SETUP
NETWORK_CONNECTING
CLOUD_REGISTERING
ACTIVE
OFFLINE
UPDATING
RECOVERY
FACTORY_RESETTING
ERROR
```

---

# Device State Machine

Example:

```text
BOOT
 │
 ▼
Load Factory Identity
 │
 ▼
Identity Valid?
 │
 ├── NO → RECOVERY
 │
 └── YES
       │
       ▼
Load Runtime Configuration
       │
       ▼
Network Available?
       │
       ├── NO → NETWORK_SETUP / OFFLINE
       │
       └── YES
             │
             ▼
Registered?
             │
        ┌────┴────┐
        │         │
       NO        YES
        │         │
        ▼         ▼
REGISTER       ACTIVE
```

---

# Board Abstraction

The `board` component isolates physical hardware mappings.

Example:

```text
boards/
└── thermone_v1/
    ├── pins.h
    └── board.h
```

The rest of the firmware should reference:

```text
PORT_A01
PORT_A02
```

instead of raw GPIO numbers.

---

# Board Pin Definition

Example:

```cpp
constexpr gpio_num_t PIN_PROBE_A01 = GPIO_NUM_13;
constexpr gpio_num_t PIN_PROBE_A02 = GPIO_NUM_14;
constexpr gpio_num_t PIN_PROBE_A03 = GPIO_NUM_16;
constexpr gpio_num_t PIN_PROBE_A04 = GPIO_NUM_17;

constexpr gpio_num_t PIN_PROBE_A05 = GPIO_NUM_25;
constexpr gpio_num_t PIN_PROBE_A06 = GPIO_NUM_26;
constexpr gpio_num_t PIN_PROBE_A07 = GPIO_NUM_27;
constexpr gpio_num_t PIN_PROBE_A08 = GPIO_NUM_32;

constexpr gpio_num_t PIN_STATUS_LED = GPIO_NUM_4;
constexpr gpio_num_t PIN_SETUP_BUTTON = GPIO_NUM_33;
```

---

# Sensor Module

The sensor module manages DS18B20 probes.

Responsibilities:

- Initialize 8 independent 1-Wire buses
- Detect connected sensors
- Read sensor ROM IDs
- Read temperatures
- Detect disconnections
- Detect sensor replacements
- Validate readings
- Maintain latest sensor state

---

# Probe Port Model

Each port should have a fixed logical identifier.

Conceptual structure:

```cpp
struct ProbePort {
    const char* id;
    gpio_num_t gpio;
    bool connected;
    std::string sensor_id;
    float temperature_c;
    SensorStatus status;
};
```

---

# Probe Port IDs

```text
A01
A02
A03
A04
A05
A06
A07
A08
```

These IDs are permanent for Thermone V1 hardware.

---

# Sensor States

Possible sensor states:

```text
UNKNOWN
ONLINE
DISCONNECTED
READ_ERROR
INVALID
CHANGED
```

---

# Sensor Read Interval

Initial target:

```text
5 seconds
```

This should be configurable later.

Cloud telemetry does not need to be uploaded every time the sensor is read.

---

# Sensor Task

Recommended architecture:

```text
sensor_task
```

Responsibilities:

```text
Every 5 seconds
    │
    ▼
Read A01-A08
    │
    ▼
Update Local Sensor State
    │
    ▼
Evaluate Changes
    │
    ▼
Publish Latest Readings
```

The sensor task should run independently of cloud requests.

---

# Probe Replacement Detection

Example:

```text
A04

Previous:
28-FF-AAAA

Current:
28-FF-BBBB
```

Sensor module emits:

```text
PROBE_CHANGED
```

with:

```text
port
old sensor ID
new sensor ID
```

---

# Temperature Validation

Firmware should reject obvious read failures while preserving legitimate unusual temperatures.

Example invalid sensor values:

```text
disconnected sentinel
CRC failure
impossible protocol result
```

A dangerous aquarium temperature is still valid telemetry.

---

# Network Module

The network component manages:

- Wi-Fi station mode
- Setup access point
- Ethernet if installed
- Connection state
- Reconnection
- Interface preference
- DNS readiness
- Internet readiness

---

# Network States

Recommended states:

```text
UNCONFIGURED
SETUP_AP
CONNECTING
LOCAL_CONNECTED
INTERNET_AVAILABLE
CLOUD_AVAILABLE
RECONNECTING
ERROR
```

---

# Network Interface Preference

Initial policy:

```text
Ethernet
   │
   ├── available → use
   │
   └── unavailable
          │
          ▼
        Wi-Fi
```

---

# Network Manager

The network manager exposes high-level events.

Examples:

```text
NETWORK_CONNECTED
NETWORK_DISCONNECTED
INTERNET_AVAILABLE
INTERNET_LOST
CLOUD_AVAILABLE
CLOUD_LOST
```

Other modules should react to these events instead of directly controlling Wi-Fi.

---

# Wi-Fi Credentials

Wi-Fi credentials are managed by the provisioning and storage components.

The network component receives credentials but should not log passwords.

---

# Provisioning Module

The provisioning component manages first-time local setup.

Responsibilities:

- Start setup AP
- Start local DNS responder
- Host local setup page
- Scan Wi-Fi networks
- Accept Wi-Fi credentials
- Validate new configuration
- Persist valid credentials
- Exit setup mode

---

# Provisioning Web Server

Local-only routes may include:

```text
GET  /api/status
GET  /api/wifi/scan
POST /api/wifi/connect
POST /api/restart
```

These routes should only be available while setup mode is active.

---

# Captive Portal Assets

The provisioning portal should be embedded locally.

Recommended:

```text
HTML
CSS
Minimal JavaScript
```

Avoid external dependencies.

---

# Identity Module

The identity component manages device identity.

It handles:

- Factory serial
- Model
- Hardware revision
- Hardware-derived ID
- Internal runtime device ID
- Factory credential
- Runtime device credential

---

# Identity Categories

## Permanent Factory Identity

```text
serial_number
model
hardware_revision
factory credential
```

## Runtime Identity

```text
device_id
runtime device token
```

---

# Identity Validation

At boot, the identity module verifies that required factory fields exist.

If factory identity is missing or corrupt:

```text
Enter Recovery Mode
```

Normal runtime operation should not continue with an unknown identity.

---

# Cloud Module

The cloud component handles communication with Thermone APIs.

Responsibilities:

- Registration
- Authentication
- HTTP client
- Telemetry uploads
- Heartbeats
- Configuration
- Command polling
- Event reporting
- Firmware checks

---

# Cloud API Client

Recommended internal interface:

```text
CloudClient
```

Conceptual methods:

```cpp
register_device();
send_heartbeat();
send_telemetry();
fetch_configuration();
fetch_commands();
check_firmware();
send_event();
```

---

# Cloud Requests Must Be Non-Blocking

Network requests must not permanently block:

```text
sensor_task
```

or:

```text
device health monitoring
```

Use:

- Dedicated FreeRTOS task
- Timeouts
- Queues
- Event groups

---

# Registration Module

On an unregistered controller:

```text
Network Available
       │
       ▼
Factory Identity Loaded
       │
       ▼
POST /register
       │
       ▼
Receive Device ID
       │
       ▼
Receive Runtime Token
       │
       ▼
Store Securely
```

---

# Registration Retry

Registration should use retry/backoff.

Example:

```text
5 sec
10 sec
30 sec
60 sec
```

Do not create duplicate device records if the response is lost.

---

# Telemetry Module

The telemetry component converts latest sensor state into upload frames.

Responsibilities:

- Capture sensor state
- Add timestamps
- Add device metadata
- Batch all 8 ports
- Queue uploads
- Buffer failed uploads
- Retry historical data

---

# Telemetry Frame

Conceptual:

```cpp
struct TelemetryFrame {
    std::string frame_id;
    int64_t recorded_at;
    ProbeReading ports[8];
};
```

---

# Telemetry Upload Interval

Initial target:

```text
30 seconds
```

Sensor sampling remains more frequent.

---

# Telemetry Queue

Architecture:

```text
sensor_task
    │
    ▼
latest readings
    │
    ▼
telemetry_task
    │
    ▼
telemetry queue
    │
    ├── cloud available → upload
    │
    └── cloud unavailable → persist
```

---

# Offline Storage

The storage component should persist telemetry when Internet access is unavailable.

Possible implementation:

```text
LittleFS
```

or another suitable ESP-IDF filesystem.

NVS should be used primarily for configuration and small key-value state, not large telemetry history.

---

# Offline Buffer Format

Possible implementation:

```text
ring buffer
```

Properties:

- Fixed maximum size
- Oldest records removed when full
- Corruption-resistant
- Each record timestamped
- Upload resumes after reconnect

---

# Offline Buffer Limits

A maximum storage size must be defined.

Example conceptual target:

```text
24–72 hours
```

depending on telemetry frequency and flash availability.

Exact capacity should be calculated after firmware size is known.

---

# Storage Component

The storage module manages separate namespaces.

Possible structure:

```text
Factory Storage
Runtime Storage
Network Storage
Telemetry Storage
```

---

# Factory Storage

Contains:

```text
serial_number
model
hardware_revision
factory credential
provisioning version
```

---

# Runtime Storage

Contains:

```text
device_id
runtime device token
firmware channel
config version
```

---

# Network Storage

Contains:

```text
SSID
Wi-Fi credential
network preferences
```

---

# Configuration Manager

Cloud configuration should be cached locally.

Possible configuration:

```text
telemetry interval
heartbeat interval
temperature thresholds
enabled ports
firmware channel
```

---

# Config Version

Thermone should track:

```text
config_version
```

Example:

```text
local = 11
cloud = 12
```

Then:

```text
download and apply version 12
```

---

# Configuration Validation

Cloud configuration must be validated before applying.

Example:

```text
telemetry_interval = 0
```

should not accidentally create a tight request loop.

Firmware should enforce safe min/max limits.

---

# Heartbeat Module

The health component gathers:

- Uptime
- Free heap
- Reset reason
- Firmware version
- Network state
- RSSI
- Connected probe count
- Offline queue size

The cloud module sends this periodically.

---

# Heartbeat Interval

Initial:

```text
60 seconds
```

---

# Command Module

The command component handles remote commands.

Initial commands may include:

```text
restart
identify
sync_config
check_firmware
enter_setup_mode
```

---

# Command Processing Flow

```text
Cloud Command
     │
     ▼
Validate
     │
     ▼
Already Processed?
     │
     ├── YES → ACK existing result
     │
     └── NO
           │
           ▼
        Execute
           │
           ▼
        Record Result
           │
           ▼
           ACK
```

---

# Command Persistence

Potentially disruptive commands should be tracked.

A reboot during command processing should not cause dangerous repeated execution.

---

# Identify Command

The identify command may cause:

```text
Status LED flashes
```

for a limited time.

This helps users identify one physical controller among many.

---

# Status Module

The status component controls the RGB LED.

Other modules should not manipulate the LED directly.

Instead they publish state.

Example:

```text
status.set(DeviceStatus::CLOUD_CONNECTED);
```

---

# LED Priority

Some states should override others.

Example priority:

```text
Hardware Error
      ↓
Factory Reset Warning
      ↓
OTA Updating
      ↓
Provisioning
      ↓
Cloud Error
      ↓
Normal
```

---

# Button Module

The button module handles GPIO 33.

Responsibilities:

- Debouncing
- Press duration
- Short press
- Setup-mode hold
- Factory-reset hold

---

# Button Events

Possible events:

```text
BUTTON_SHORT_PRESS
BUTTON_SETUP_HOLD
BUTTON_FACTORY_RESET_HOLD
```

---

# Button Timing

Initial conceptual values:

```text
Short:
< 2 seconds

Setup:
~5 seconds

Factory reset:
~10 seconds
```

Exact values require physical testing.

---

# Time Module

The time component manages UTC time.

Responsibilities:

- NTP synchronization
- Clock validity
- Timestamp generation
- Time sync state
- Drift detection

---

# NTP Flow

```text
Internet Connected
      │
      ▼
NTP Sync
      │
      ▼
UTC Valid
```

---

# Unsynchronized Time

If UTC is unavailable:

- Monitoring continues
- Telemetry can be buffered
- Time validity is recorded
- NTP retries continue

Firmware should not stop functioning solely because NTP is unavailable.

---

# OTA Module

The OTA component handles firmware updates.

Responsibilities:

- Firmware checks
- Manifest validation
- Download
- SHA-256 verification
- Signature verification when enabled
- Partition write
- Boot selection
- Rollback
- Result reporting

Detailed behavior is documented in:

```text
firmware/ota-updates.md
```

---

# OTA Must Not Block Sensor Safety

During OTA:

- Sensor monitoring should continue where practical
- Existing firmware remains bootable
- Power loss must not destroy both OTA slots

---

# Health Module

The health module gathers operational information.

Potential fields:

```text
uptime
free heap
minimum free heap
reset reason
network reconnect count
cloud failures
sensor failures
offline queue depth
```

---

# Watchdog

Thermone should use watchdog protection.

The watchdog protects against:

- Deadlocks
- Hung network operations
- Broken tasks
- Unexpected firmware stalls

Critical tasks must regularly demonstrate liveness.

---

# Recovery Module

Recovery mode provides minimal functionality when normal firmware cannot safely operate.

Possible triggers:

- Repeated boot failures
- Corrupt runtime config
- Failed OTA
- Manual recovery trigger
- Missing application image

---

# Recovery Responsibilities

Recovery mode should provide:

- Network setup
- Cloud access where possible
- OTA repair
- Factory identity access
- Diagnostics
- Factory reset

It should avoid unnecessary normal-product features.

---

# Factory Reset

Factory reset should clear customer/runtime state.

Possible items cleared:

```text
Wi-Fi credentials
runtime device token
cached configuration
offline telemetry
```

Permanent factory identity remains.

---

# Network Reset

Network reset is less destructive.

Clear:

```text
Wi-Fi configuration
```

Keep:

```text
device registration
runtime credential
factory identity
```

---

# FreeRTOS Task Architecture

Possible V1 tasks:

```text
sensor_task
network_task
cloud_task
telemetry_task
provisioning_task
status_task
health_task
ota_task
```

Not every module requires a separate task.

The final number should remain manageable.

---

# Suggested Task Responsibilities

## sensor_task

```text
Read 8 probe buses
Update current readings
Detect sensor changes
```

## network_task

```text
Manage Wi-Fi/Ethernet
Reconnect
Publish network state
```

## cloud_task

```text
Registration
Heartbeat
Config
Commands
Events
```

## telemetry_task

```text
Build frames
Upload
Buffer failures
Replay backlog
```

## status_task

```text
LED behavior
```

---

# Inter-Task Communication

Use standard ESP-IDF primitives such as:

```text
queues
event groups
mutexes
notifications
```

Avoid uncontrolled global mutable state.

---

# Event Bus

A lightweight internal event model may include:

```text
SENSOR_CONNECTED
SENSOR_DISCONNECTED
SENSOR_CHANGED
NETWORK_CONNECTED
NETWORK_LOST
CLOUD_CONNECTED
CLOUD_LOST
CONFIG_UPDATED
OTA_AVAILABLE
BUTTON_SETUP
FACTORY_RESET
```

---

# Logging

Use structured logging categories.

Examples:

```text
THERMONE_SENSOR
THERMONE_NETWORK
THERMONE_CLOUD
THERMONE_OTA
THERMONE_STORAGE
```

Never log secrets.

---

# Log Levels

Use:

```text
ERROR
WARN
INFO
DEBUG
VERBOSE
```

Production firmware should avoid excessive verbose logging.

---

# Secret Redaction

Never print:

```text
factory credential
device token
Wi-Fi password
claim token
private keys
```

---

# Firmware Version

Every build has a version.

Example:

```text
1.0.0
```

Firmware reports:

```text
firmware_version
```

in heartbeat and telemetry metadata.

---

# Build Metadata

Useful non-secret metadata may include:

```text
version
git commit
build time
environment
hardware target
```

Example:

```text
Version: 1.0.0
Commit: a14fd22
Environment: staging
Hardware: THV1-1.0
```

---

# Environment Configuration

The firmware build must target:

```text
development
staging
production
```

Production builds must use production API roots.

They must never automatically fall back to development.

---

# Hardware Revision

Firmware must know the hardware target.

Example:

```text
THV1
Hardware 1.0
```

OTA compatibility depends on this value.

---

# Partition Layout

Exact layout is defined later, but V1 needs space for:

```text
bootloader
partition table
NVS
factory NVS
OTA slot A
OTA slot B
offline data filesystem
```

---

# Example Conceptual Layout

```text
Flash
 │
 ├── NVS
 ├── Factory NVS
 ├── OTA Data
 ├── App OTA_0
 ├── App OTA_1
 └── LittleFS
```

Exact sizes depend on the selected ESP32 flash capacity.

---

# Startup Sequence

Recommended startup:

```text
1. Initialize logging
2. Initialize board
3. Initialize status LED
4. Load factory identity
5. Validate factory identity
6. Load runtime state
7. Initialize storage
8. Initialize button
9. Initialize sensor buses
10. Begin sensor monitoring
11. Initialize networking
12. Establish network
13. Sync time
14. Register if necessary
15. Start cloud communication
16. Start normal background tasks
17. Check firmware
```

---

# Important Startup Rule

Temperature monitoring should start before waiting indefinitely for cloud registration.

Example:

```text
Sensor system initialized
      │
      ▼
Start monitoring
      │
      ▼
Then attempt Internet/cloud
```

---

# Graceful Cloud Failure

If Thermone Cloud is unavailable at boot:

```text
Sensors:
running

Local monitoring:
running

Cloud:
retrying
```

The controller must not become useless.

---

# Graceful Network Failure

If no Wi-Fi exists:

```text
Sensors:
running

Provisioning:
available

Cloud:
offline
```

---

# Memory Management

Firmware should minimize frequent heap allocation in high-frequency loops.

Prefer:

- Fixed-size buffers where practical
- Reusable objects
- Bounded queues
- Controlled JSON sizes

This improves long-term reliability.

---

# JSON Handling

Telemetry JSON should remain compact.

Avoid unnecessary duplication.

The ESP32 should not build extremely large dynamic strings if a streaming serializer can be used.

---

# Error Handling

Errors should be classified.

Example:

```text
SENSOR_ERROR
NETWORK_ERROR
AUTH_ERROR
STORAGE_ERROR
OTA_ERROR
CONFIG_ERROR
```

Not every error should reboot the device.

---

# Restart Policy

Restart only when necessary.

Examples that may justify restart:

- Successful OTA
- User-issued restart
- unrecoverable subsystem corruption
- watchdog event

A temporary cloud failure should not cause a reboot loop.

---

# Reboot Tracking

Store enough information to detect unusual reboot frequency.

Report:

```text
reset_reason
uptime
boot_count
```

to Thermone Cloud.

---

# Safe Defaults

If cloud configuration is unavailable, use conservative local defaults.

Example:

```text
Sensor read:
5 seconds

Telemetry:
30 seconds

Heartbeat:
60 seconds
```

Never use invalid zero-duration loops.

---

# Testing Strategy

Firmware tests should cover:

- Port mapping
- DS18B20 detection
- Probe replacement
- Offline telemetry
- Wi-Fi reconnect
- API timeout
- Credential rejection
- OTA interruption
- Button timing
- Factory reset
- Corrupt configuration
- Device reboot recovery

---

# Hardware-in-the-Loop Testing

Important functions should eventually be tested using physical controllers.

Examples:

```text
8 actual probes
network disconnect
router reboot
API outage
power-cycle during OTA
sensor replacement
```

---

# Simulator / Mock Support

Where practical, modules should allow mocked dependencies.

Example:

```text
Mock CloudClient
Mock Storage
Mock Sensor Driver
```

This makes automated testing easier.

---

# Firmware Module Dependency Direction

Prefer:

```text
Application Logic
      │
      ▼
Interfaces
      │
      ▼
Hardware / Network Drivers
```

Avoid:

```text
Every module directly calling every other module
```

---

# V1 Module Summary

```text
board
    Hardware mappings

sensors
    DS18B20 monitoring

network
    Wi-Fi / Ethernet

provisioning
    Setup AP and portal

identity
    Factory and runtime identity

cloud
    Device API communication

telemetry
    Reading batches and upload queue

storage
    Persistent state and offline data

ota
    Firmware updates

commands
    Remote command execution

status
    RGB LED state

button
    Physical input

time
    UTC/NTP

health
    Device diagnostics

watchdog
    Firmware liveness

recovery
    Repair/reset operation
```

---

# V1 Firmware Success Criteria

The firmware architecture is successful when:

1. All 8 probes can be monitored independently.
2. A broken probe does not stop other ports.
3. Cloud outages do not stop local monitoring.
4. Network operations do not block sensors.
5. Wi-Fi setup can run locally.
6. Factory identity survives customer resets.
7. Runtime credentials are stored separately.
8. Telemetry can be buffered offline.
9. Device registration can recover from network loss.
10. Configuration can be cached locally.
11. Commands can be processed safely.
12. OTA updates can roll back.
13. Status LED reflects device state.
14. Setup/reset button is reliable.
15. Secrets never appear in normal logs.
16. Hardware revision changes can use separate board mappings.
17. Firmware can be tested module-by-module.

---

# Core Firmware Principle

Thermone firmware should be structured so that:

```text
Sensors keep monitoring
        │
        ▼
regardless of whether
        │
        ▼
Wi-Fi, Cloud, Dashboard,
or OTA services
are temporarily unavailable.
```

The cloud enhances the controller.

The cloud must not be required for the controller to continue performing its basic monitoring function.