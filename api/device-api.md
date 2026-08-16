# Thermone V1 Device API

## Purpose

This document defines the HTTP API used by Thermone hardware devices to communicate with Thermone Cloud.

The Device API is intended for machine-to-machine communication between:

```text
Thermone ESP32 Controller
        │
        ▼
Thermone Cloud API
```

The Device API is separate from normal user-facing dashboard authentication.

---

# Base URLs

## Development

```text
https://api.dev.thermone.com/v1/device
```

## Staging

```text
https://api.staging.thermone.com/v1/device
```

## Production

```text
https://api.thermone.com/v1/device
```

---

# Transport Security

All production Device API communication must use:

```text
HTTPS
```

Plain HTTP must not be used for production device traffic.

The device must validate the Thermone server certificate.

---

# Content Type

Unless otherwise specified, API requests use:

```http
Content-Type: application/json
```

Responses also use JSON.

---

# API Responsibilities

The Device API handles:

- Factory registration
- Runtime authentication
- Telemetry ingestion
- Device heartbeat
- Configuration synchronization
- Command polling
- Firmware update checks
- Device health
- Credential management
- Device status reporting

---

# Authentication Phases

Thermone devices use two major authentication phases.

## Phase 1: Factory Authentication

Used only before normal runtime credentials have been issued.

Example:

```text
Factory Credential
```

Used for:

```text
POST /register
```

The factory credential must not be used for routine telemetry.

---

## Phase 2: Runtime Authentication

After registration, the API issues a dedicated runtime credential.

The device uses this credential for:

- Telemetry
- Heartbeats
- Configuration
- Commands
- Firmware checks

Example header:

```http
Authorization: Bearer <device-token>
```

---

# Device Identifiers

The API distinguishes several identifiers.

## Serial Number

Human-readable.

Example:

```text
THV1-000001
```

---

## Internal Device ID

Server-generated.

Example:

```text
019c3e92-0000-7000-8000-000000000001
```

---

## Hardware ID

Derived from the physical controller.

Example:

```text
ESP32-A4CF12B78E31
```

The hardware ID is not an authentication secret.

---

# Common Request Headers

After registration, device requests should include:

```http
Authorization: Bearer <device-token>
Content-Type: application/json
User-Agent: Thermone-Firmware/1.0.0
```

Thermone may also use:

```http
X-Thermone-Device-ID: <device-id>
X-Thermone-Hardware: THV1
X-Thermone-Hardware-Revision: 1.0
X-Thermone-Firmware: 1.0.0
```

The exact final header set may be simplified during implementation.

---

# API Versioning

The initial device API uses:

```text
/v1/device
```

Breaking changes should not silently modify V1 behavior.

Future incompatible versions may use:

```text
/v2/device
```

---

# 1. Register Device

## Endpoint

```http
POST /v1/device/register
```

## Purpose

Registers a factory-provisioned Thermone controller with Thermone Cloud.

This endpoint is normally called by bootstrap firmware.

---

## Authentication

Factory credential required.

Conceptual header:

```http
Authorization: Factory <factory-credential>
```

---

## Example Request

```json
{
  "serial_number": "THV1-000001",
  "model": "THV1",
  "hardware_revision": "1.0",
  "hardware_id": "ESP32-A4CF12B78E31",
  "bootstrap_version": "1.0.0"
}
```

---

## Request Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `serial_number` | string | yes | Factory serial |
| `model` | string | yes | Device model |
| `hardware_revision` | string | yes | Hardware revision |
| `hardware_id` | string | yes | Physical MCU identity |
| `bootstrap_version` | string | yes | Bootstrap firmware version |

---

## Successful Response

```json
{
  "success": true,
  "device": {
    "device_id": "019c3e92-0000-7000-8000-000000000001",
    "serial_number": "THV1-000001",
    "status": "registered"
  },
  "credentials": {
    "device_token": "runtime-device-token"
  },
  "configuration": {
    "firmware_channel": "production"
  }
}
```

---

## Status Codes

```text
200 OK
201 Created
400 Bad Request
401 Unauthorized
403 Forbidden
409 Conflict
500 Internal Server Error
```

---

# Registration Rules

The API must verify:

- Serial number exists
- Factory credential matches device
- Hardware revision is supported
- Device is not disabled
- Hardware identity is valid
- Device is not impersonating another serial
- Registration request belongs to correct environment

---

# Idempotent Registration

A device may retry registration because of network failures.

The registration endpoint should be designed to safely handle retries.

Example:

```text
Device registers
      │
      X
Response lost
      │
      ▼
Device retries
```

The server should not create a second device record.

---

# 2. Device Heartbeat

## Endpoint

```http
POST /v1/device/heartbeat
```

## Purpose

Reports that a controller is alive and provides operational health data.

---

## Authentication

Runtime device token required.

---

## Example Request

```json
{
  "device_id": "019c3e92-0000-7000-8000-000000000001",
  "firmware_version": "1.0.0",
  "uptime_seconds": 86400,
  "network": {
    "type": "wifi",
    "rssi": -54
  },
  "memory": {
    "free_heap": 185420
  },
  "ports": {
    "connected": 7,
    "total": 8
  }
}
```

---

## Example Response

```json
{
  "success": true,
  "server_time": "2026-08-16T23:00:00Z",
  "next_heartbeat_seconds": 60
}
```

---

# Heartbeat Frequency

Initial target:

```text
60 seconds
```

The server may return a recommended interval.

The firmware should enforce reasonable lower and upper limits.

---

# Heartbeat Use

The backend uses heartbeats for:

- Online/offline status
- Firmware inventory
- Wi-Fi quality
- Device uptime
- Memory monitoring
- Connected probe count
- Restart detection

---

# 3. Upload Telemetry

## Endpoint

```http
POST /v1/device/telemetry
```

## Purpose

Uploads temperature data from one Thermone controller.

---

## Authentication

Runtime device token required.

---

## Batch Design

All probe readings from one controller should be sent in a single request.

Preferred:

```text
1 request
8 probe readings
```

Avoid:

```text
8 separate requests
```

---

## Example Request

```json
{
  "device_id": "019c3e92-0000-7000-8000-000000000001",
  "recorded_at": "2026-08-16T23:00:30Z",
  "firmware_version": "1.0.0",
  "ports": [
    {
      "port": "A01",
      "gpio": 13,
      "sensor_id": "28-FF-64-1D-92-16-03-8C",
      "temperature_c": 26.72,
      "temperature_f": 80.10,
      "status": "online"
    },
    {
      "port": "A02",
      "gpio": 14,
      "sensor_id": "28-FF-73-20-A4-16-05-B1",
      "temperature_c": 25.94,
      "temperature_f": 78.69,
      "status": "online"
    },
    {
      "port": "A03",
      "gpio": 16,
      "sensor_id": null,
      "temperature_c": null,
      "temperature_f": null,
      "status": "disconnected"
    }
  ]
}
```

---

# Telemetry Port Status Values

Initial allowed values:

```text
online
disconnected
read_error
sensor_changed
invalid
```

These may later become an enum in the database and API schema.

---

# Telemetry Response

```json
{
  "success": true,
  "accepted": 8,
  "server_time": "2026-08-16T23:00:31Z",
  "next_upload_seconds": 30
}
```

---

# Telemetry Frequency

Initial target:

```text
30 seconds
```

The controller may read sensors more frequently locally.

Example:

```text
Local sensor read:
5 seconds

Cloud upload:
30 seconds
```

---

# Telemetry Timestamp

The controller should use UTC timestamps.

Example:

```text
2026-08-16T23:00:30Z
```

If the controller does not yet have reliable time, it may mark the payload accordingly.

---

# Offline Telemetry

Telemetry may be delayed because of Internet loss.

Therefore the server must support readings recorded before upload time.

Example:

```json
{
  "recorded_at": "2026-08-16T22:52:00Z",
  "uploaded_at": "2026-08-16T23:04:00Z"
}
```

The server determines `uploaded_at`.

---

# Buffered Batch Upload

Future or V1 implementation may allow several telemetry frames in one request.

Example endpoint:

```http
POST /v1/device/telemetry/batch
```

Example:

```json
{
  "device_id": "019c3e92-...",
  "frames": [
    {
      "recorded_at": "2026-08-16T23:00:00Z",
      "ports": []
    },
    {
      "recorded_at": "2026-08-16T23:00:30Z",
      "ports": []
    }
  ]
}
```

This is useful after an outage.

---

# 4. Get Device Configuration

## Endpoint

```http
GET /v1/device/config
```

## Purpose

Returns current cloud-managed device settings.

---

## Authentication

Runtime device token required.

---

## Example Response

```json
{
  "success": true,
  "config_version": 12,
  "telemetry_interval_seconds": 30,
  "heartbeat_interval_seconds": 60,
  "firmware_check_interval_seconds": 21600,
  "temperature_unit": "F",
  "ports": [
    {
      "port": "A01",
      "enabled": true,
      "tank_id": "tank_123",
      "low_warning_f": 78.0,
      "high_warning_f": 82.0
    },
    {
      "port": "A02",
      "enabled": true,
      "tank_id": "tank_124",
      "low_warning_f": 76.0,
      "high_warning_f": 80.0
    }
  ]
}
```

---

# Configuration Versioning

Every device configuration should have a version.

Example:

```text
config_version = 12
```

The device stores its last applied version.

If the server still reports version 12, the device does not need to rewrite the same configuration repeatedly.

---

# 5. Check Configuration Version

Optional lightweight endpoint:

```http
GET /v1/device/config/version
```

Example response:

```json
{
  "config_version": 12
}
```

This can reduce network usage.

---

# 6. Device Commands

## Endpoint

```http
GET /v1/device/commands
```

## Purpose

Allows the device to retrieve pending cloud commands.

V1 may use polling rather than persistent MQTT or WebSocket connections.

---

## Example Response

```json
{
  "commands": [
    {
      "command_id": "cmd_123",
      "type": "restart",
      "created_at": "2026-08-16T23:04:00Z"
    }
  ]
}
```

---

# Initial Command Types

Potential V1 command types:

```text
restart
sync_config
check_firmware
enter_setup_mode
identify
```

Factory reset should require stricter authorization and may not initially be available as a remote command.

---

# Command Acknowledgement

After executing a command:

```http
POST /v1/device/commands/{command_id}/ack
```

Example:

```json
{
  "status": "completed",
  "completed_at": "2026-08-16T23:04:08Z"
}
```

---

# Command Failure

Example:

```json
{
  "status": "failed",
  "error_code": "RESTART_REJECTED",
  "message": "Device currently updating firmware"
}
```

---

# Command Idempotency

Commands must have unique IDs.

If the ESP32 receives the same command twice, it should not execute dangerous operations repeatedly.

---

# 7. Firmware Check

## Endpoint

```http
GET /v1/device/firmware/check
```

## Purpose

Checks whether a compatible firmware version is available.

---

## Request Parameters

Conceptually:

```text
current_version
hardware_revision
firmware_channel
```

Example:

```http
GET /v1/device/firmware/check?current_version=1.0.0&hardware_revision=1.0&channel=production
```

---

## No Update Response

```json
{
  "update_available": false,
  "current_version": "1.0.0",
  "latest_version": "1.0.0"
}
```

---

## Update Available Response

```json
{
  "update_available": true,
  "current_version": "1.0.0",
  "latest_version": "1.1.0",
  "hardware_revision": "1.0",
  "channel": "production",
  "mandatory": false,
  "download_url": "https://firmware.thermone.com/signed/...",
  "sha256": "abc123...",
  "size_bytes": 1245184
}
```

---

# Firmware Compatibility

The API must not offer incompatible firmware.

Example:

```text
Hardware V1.0
```

must not accidentally receive firmware intended only for:

```text
Hardware V2.0
```

---

# Firmware Download URLs

Download URLs should preferably be:

- HTTPS
- Temporary
- Signed
- Time limited

The API itself does not need to stream large firmware files.

---

# 8. Report OTA Result

## Endpoint

```http
POST /v1/device/firmware/result
```

## Purpose

Reports the outcome of an OTA operation.

---

## Successful Update

```json
{
  "previous_version": "1.0.0",
  "new_version": "1.1.0",
  "status": "success"
}
```

---

## Failed Update

```json
{
  "previous_version": "1.0.0",
  "attempted_version": "1.1.0",
  "status": "failed",
  "error_code": "HASH_MISMATCH"
}
```

---

# 9. Device Events

## Endpoint

```http
POST /v1/device/events
```

## Purpose

Reports important events that are not routine telemetry.

Examples:

- Probe inserted
- Probe removed
- Probe changed
- Wi-Fi reconnected
- Controller rebooted
- Setup mode entered
- OTA failed
- Recovery mode entered

---

## Example Probe Event

```json
{
  "type": "probe_changed",
  "recorded_at": "2026-08-16T23:15:00Z",
  "data": {
    "port": "A04",
    "previous_sensor_id": "28-FF-AAAA...",
    "new_sensor_id": "28-FF-BBBB..."
  }
}
```

---

# 10. Time Synchronization

Thermone should primarily use NTP for controller time.

The API response may still provide server time.

Example:

```json
{
  "server_time": "2026-08-16T23:16:00Z"
}
```

This may help detect large clock drift.

---

# Standard API Response Format

Successful responses should generally follow:

```json
{
  "success": true,
  "data": {}
}
```

or an endpoint-specific response where simplicity is preferable.

Errors should follow a consistent format.

---

# Standard Error Response

Example:

```json
{
  "success": false,
  "error": {
    "code": "INVALID_DEVICE_TOKEN",
    "message": "Device authentication failed"
  }
}
```

---

# Error Codes

Initial examples:

```text
INVALID_REQUEST
INVALID_DEVICE_TOKEN
INVALID_FACTORY_CREDENTIAL
DEVICE_DISABLED
DEVICE_NOT_FOUND
HARDWARE_MISMATCH
FIRMWARE_NOT_COMPATIBLE
RATE_LIMITED
CONFIG_NOT_FOUND
COMMAND_NOT_FOUND
INTERNAL_ERROR
```

---

# HTTP Status Code Strategy

## 200

```text
Successful request
```

## 201

```text
New device/resource created
```

## 400

```text
Malformed or invalid request
```

## 401

```text
Authentication required or invalid
```

## 403

```text
Authenticated but forbidden
```

## 404

```text
Resource not found
```

## 409

```text
Conflict
```

## 422

```text
Semantically invalid request
```

## 429

```text
Rate limited
```

## 500

```text
Unexpected server error
```

## 503

```text
Service temporarily unavailable
```

---

# Device Retry Behavior

The ESP32 must not treat every HTTP failure identically.

Example:

```text
200
→ success

400
→ do not continuously retry identical bad request

401
→ attempt credential recovery

429
→ wait according to Retry-After

500
→ retry with backoff

503
→ retry with backoff
```

---

# Retry-After

Where appropriate, the API should return:

```http
Retry-After: 60
```

The firmware should respect this value.

---

# Request Timeouts

The firmware should use finite network timeouts.

It must never freeze sensor monitoring indefinitely because an API request hangs.

Networking and sensor monitoring should be architecturally separated.

---

# Idempotency

Certain write endpoints should support safe retry behavior.

Registration is the clearest example.

Telemetry can include a unique frame identifier if duplicate prevention becomes necessary.

Example:

```json
{
  "frame_id": "019c3e92-...",
  "recorded_at": "..."
}
```

The server can reject or ignore duplicates.

---

# Rate Limits

Different endpoints should have different rate limits.

Example conceptual limits:

```text
Registration:
very strict

Telemetry:
based on expected device interval

Heartbeat:
based on expected heartbeat interval

Commands:
moderate

Firmware checks:
low frequency
```

The exact values belong in infrastructure configuration.

---

# Device Ownership

The ESP32 API does not decide user ownership.

Ownership belongs to Thermone Cloud.

Example:

```text
Device ID
   │
   ▼
Device Record
   │
   ▼
Owner / Organization
```

A device token only authenticates the physical controller.

It does not grant user dashboard privileges.

---

# Environment Isolation

Development device credentials must not work in production.

Example:

```text
DEV token
   │
   ▼
api.thermone.com
   │
   ▼
401 / 403
```

Likewise, production tokens must not authenticate against development services unless explicitly migrated.

---

# Secret Handling

The Device API must never log raw:

- Factory credentials
- Device tokens
- Claim tokens
- Private keys
- Wi-Fi passwords

Logs may contain:

```text
device_id
serial_number
request_id
endpoint
status_code
firmware_version
```

---

# Request IDs

Every API request should eventually receive a request ID.

Example:

```http
X-Request-ID: req_019c...
```

This helps support and debugging.

---

# Telemetry Validation

The API should validate:

- Port name exists
- Port belongs to device model
- Temperature is within technically plausible limits
- Sensor ID format is valid
- Timestamp is not wildly invalid
- Duplicate port entries do not exist

The backend should not blindly trust device payloads.

---

# Temperature Validation

DS18B20 technical limits should be handled separately from normal aquarium limits.

For example:

```text
Technically plausible sensor range
```

is different from:

```text
Safe aquarium range
```

An aquarium at 40°F may be dangerous but still represent a valid sensor reading.

Do not reject legitimate telemetry simply because it is outside a configured alert range.

---

# Sensor Change Handling

If telemetry reports:

```text
A03
Sensor ID changed
```

the API should:

1. Preserve the port
2. Record the old sensor relationship
3. Register the new sensor
4. Create a sensor replacement event
5. Preserve tank assignment
6. Notify dashboard logic where appropriate

---

# Device Online State

The API updates device presence using heartbeats.

Example conceptual state:

```text
last_seen_at
```

The dashboard may classify:

```text
Online:
heartbeat within threshold

Offline:
heartbeat older than threshold
```

The device does not need to explicitly send:

```text
"I am offline"
```

because an offline device cannot reliably communicate.

---

# Recommended Initial Endpoint List

```text
POST /v1/device/register

POST /v1/device/heartbeat

POST /v1/device/telemetry

POST /v1/device/telemetry/batch

GET  /v1/device/config

GET  /v1/device/config/version

GET  /v1/device/commands

POST /v1/device/commands/{id}/ack

GET  /v1/device/firmware/check

POST /v1/device/firmware/result

POST /v1/device/events
```

---

# V1 API Flow

```text
FIRST BOOT

ESP32
  │
  ▼
POST /register
  │
  ▼
Receive runtime credential


NORMAL OPERATION

ESP32
  │
  ├── POST /heartbeat
  │
  ├── POST /telemetry
  │
  ├── GET /config
  │
  ├── GET /commands
  │
  └── GET /firmware/check
```

---

# Recommended Timing

Initial targets:

| Operation | Interval |
|---|---|
| Sensor read | 5 seconds |
| Telemetry upload | 30 seconds |
| Heartbeat | 60 seconds |
| Config check | 60 seconds |
| Command check | 15–30 seconds |
| Firmware check | 6 hours |

These values are provisional and should be remotely configurable within safe limits.

---

# API Compatibility Rule

Firmware must not assume undocumented behavior.

The contract defined in this document should be reflected in:

- API schemas
- Firmware models
- Integration tests
- Documentation

Any breaking API change requires deliberate versioning.

---

# V1 Success Criteria

The Device API is ready for V1 firmware when:

1. A factory device can register.
2. Registration is retry-safe.
3. Runtime credentials are issued securely.
4. A registered controller can authenticate.
5. Heartbeats update device health.
6. Eight-port telemetry can be accepted.
7. Buffered historical telemetry can be accepted.
8. Device configuration can be retrieved.
9. Commands can be retrieved and acknowledged.
10. Firmware checks return compatible versions.
11. OTA results can be reported.
12. Sensor change events can be recorded.
13. Development and production credentials are isolated.
14. Invalid requests are rejected safely.
15. Secrets are not exposed through normal API responses or logs.

---

# Core Device API Principle

The Device API should remain predictable and intentionally boring.

The ESP32 should be able to:

```text
Authenticate
    │
    ▼
Send Data
    │
    ▼
Receive Configuration
    │
    ▼
Receive Commands
    │
    ▼
Update Firmware
```

without depending on dashboard-specific implementation details.