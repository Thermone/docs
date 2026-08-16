# Thermone V1 Dashboard API

## Purpose

This document defines the API used by the Thermone dashboard and future mobile applications.

The Dashboard API is separate from the Device API.

The Device API authenticates physical controllers.

The Dashboard API authenticates users.

```text
Thermone Controller
        │
        ▼
    Device API


Thermone User
        │
        ▼
   Dashboard API
```

The Dashboard API is responsible for:

- User account access
- Device claiming
- Controller management
- Locations
- Tanks
- Probe-port assignments
- Current temperatures
- Historical temperatures
- Alert configuration
- Device health
- Firmware information
- Ownership and permissions
- Account settings

---

# Base URLs

## Development

```text
https://api.dev.thermone.com/v1
```

## Staging

```text
https://api.staging.thermone.com/v1
```

## Production

```text
https://api.thermone.com/v1
```

---

# Authentication

Dashboard requests require an authenticated Thermone user.

Initial implementation may use:

```text
Supabase Auth
```

Supported login methods may include:

- Email and password
- Magic link
- Google
- Apple

The client receives a user session token.

Example:

```http
Authorization: Bearer <user-jwt>
```

The user token must never be used as an ESP32 device credential.

---

# Authentication Separation

Thermone has separate authentication domains.

```text
USER AUTHENTICATION

Dashboard
   │
   ▼
User JWT


DEVICE AUTHENTICATION

ESP32
   │
   ▼
Device Credential
```

These credentials must never be interchangeable.

---

# User Identity

Each Thermone user receives an internal user ID.

Example:

```text
usr_019c3e92...
```

or the underlying Supabase Auth UUID.

User IDs are not secrets.

---

# Authorization Model

Authentication answers:

```text
Who is this user?
```

Authorization answers:

```text
What is this user allowed to access?
```

Every Dashboard API request must enforce authorization on the server.

The frontend must never be trusted to enforce ownership by itself.

---

# Ownership Model

Initial V1 relationship:

```text
User
 │
 ├── Locations
 │
 └── Devices
      │
      └── Ports
           │
           └── Tanks
```

Future versions may support organizations and multiple users.

---

# Organization Support

Even if V1 initially focuses on individual users, the database should allow future support for:

```text
Organization
     │
     ├── Owner
     ├── Admin
     ├── Member
     └── Viewer
```

Potential examples:

- Fish store
- Breeder
- Aquarium facility
- Research facility
- Public aquarium

---

# Common Response Format

Successful responses should generally follow:

```json
{
  "success": true,
  "data": {}
}
```

Errors:

```json
{
  "success": false,
  "error": {
    "code": "DEVICE_NOT_FOUND",
    "message": "Device was not found"
  }
}
```

---

# Pagination

Endpoints returning large collections should support pagination.

Example:

```http
GET /v1/temperature-history?limit=100&cursor=abc
```

Response:

```json
{
  "success": true,
  "data": [],
  "pagination": {
    "next_cursor": "xyz",
    "has_more": true
  }
}
```

---

# 1. Get Current User

## Endpoint

```http
GET /v1/me
```

## Purpose

Returns the currently authenticated Thermone user.

---

## Example Response

```json
{
  "success": true,
  "data": {
    "id": "usr_019c...",
    "email": "user@example.com",
    "full_name": "Example User",
    "temperature_unit": "F",
    "timezone": "America/New_York"
  }
}
```

---

# 2. Update Current User

## Endpoint

```http
PATCH /v1/me
```

Example request:

```json
{
  "full_name": "Example User",
  "temperature_unit": "F",
  "timezone": "America/New_York"
}
```

Allowed temperature units:

```text
F
C
```

---

# 3. List Locations

## Endpoint

```http
GET /v1/locations
```

A location represents a physical place containing Thermone devices.

Examples:

```text
Home Fish Room
Breeding Room
Fish Store
Garage Rack
Basement
```

---

## Example Response

```json
{
  "success": true,
  "data": [
    {
      "id": "loc_001",
      "name": "Main Fish Room",
      "device_count": 3
    }
  ]
}
```

---

# 4. Create Location

## Endpoint

```http
POST /v1/locations
```

Example:

```json
{
  "name": "Main Fish Room"
}
```

---

## Response

```json
{
  "success": true,
  "data": {
    "id": "loc_001",
    "name": "Main Fish Room"
  }
}
```

---

# 5. Update Location

## Endpoint

```http
PATCH /v1/locations/{location_id}
```

Example:

```json
{
  "name": "Breeding Room"
}
```

---

# 6. Delete Location

## Endpoint

```http
DELETE /v1/locations/{location_id}
```

Deletion rules must prevent accidental removal of active devices without explicit handling.

Possible behavior:

```text
Location has devices
       │
       ▼
Reject deletion
       │
       ▼
Require device reassignment first
```

---

# 7. List Devices

## Endpoint

```http
GET /v1/devices
```

Returns Thermone controllers accessible to the user.

---

## Example Response

```json
{
  "success": true,
  "data": [
    {
      "id": "dev_019c...",
      "serial_number": "THV1-000482",
      "name": "Main Rack",
      "location_id": "loc_001",
      "status": "online",
      "firmware_version": "1.0.0",
      "hardware_revision": "1.0",
      "last_seen_at": "2026-08-16T23:00:00Z"
    }
  ]
}
```

---

# 8. Get Device

## Endpoint

```http
GET /v1/devices/{device_id}
```

Response may include:

- Device name
- Serial
- Status
- Location
- Firmware
- Hardware revision
- Network health
- Port status
- Last heartbeat
- Current readings

---

## Example Response

```json
{
  "success": true,
  "data": {
    "id": "dev_019c...",
    "serial_number": "THV1-000482",
    "name": "Main Rack",
    "status": "online",
    "firmware_version": "1.0.0",
    "hardware_revision": "1.0",
    "last_seen_at": "2026-08-16T23:00:00Z",
    "network": {
      "type": "wifi",
      "rssi": -54
    }
  }
}
```

---

# 9. Update Device

## Endpoint

```http
PATCH /v1/devices/{device_id}
```

Example:

```json
{
  "name": "Breeding Rack",
  "location_id": "loc_002"
}
```

Users may update metadata.

Users must not be able to directly modify:

- Serial number
- Hardware ID
- Runtime device credential
- Factory identity

---

# 10. Claim Device

## Endpoint

```http
POST /v1/devices/claim
```

## Purpose

Associates an unclaimed Thermone controller with the authenticated account.

---

## Example Request

```json
{
  "claim_token": "claim-token-from-qr"
}
```

---

## Successful Response

```json
{
  "success": true,
  "data": {
    "device_id": "dev_019c...",
    "serial_number": "THV1-000482",
    "status": "claimed"
  }
}
```

---

# Claim Validation

The backend checks:

- User is authenticated
- Claim token exists
- Claim token is valid
- Device is not already owned
- Device is not disabled
- Device is eligible for claiming

---

# Claiming a Device That Has Not Connected Yet

The factory record may exist before the controller has registered.

Example response:

```json
{
  "success": true,
  "data": {
    "status": "waiting_for_device",
    "serial_number": "THV1-000482"
  }
}
```

The dashboard may display:

```text
Waiting for your Thermone controller to connect.
```

---

# 11. Get Claim Status

## Endpoint

```http
GET /v1/devices/claim/{claim_session_id}
```

Possible results:

```text
waiting_for_device
ready_to_claim
claimed
expired
invalid
```

This is useful for QR onboarding.

---

# 12. Remove Device From Account

## Endpoint

```http
POST /v1/devices/{device_id}/remove
```

This operation requires explicit confirmation.

Possible request:

```json
{
  "confirmation": true
}
```

Removing ownership should not automatically erase historical temperature data.

---

# 13. Transfer Device

Potential V1 or future endpoint:

```http
POST /v1/devices/{device_id}/transfer
```

Purpose:

- Remove existing ownership
- Generate a new claim token
- Prepare device for another owner

The original factory claim token should not necessarily become reusable.

---

# 14. List Device Ports

## Endpoint

```http
GET /v1/devices/{device_id}/ports
```

Example:

```json
{
  "success": true,
  "data": [
    {
      "port": "A01",
      "sensor_id": "28-FF-64-1D-92-16-03-8C",
      "status": "online",
      "temperature_f": 80.1,
      "tank_id": "tank_001"
    },
    {
      "port": "A02",
      "sensor_id": null,
      "status": "disconnected",
      "temperature_f": null,
      "tank_id": null
    }
  ]
}
```

---

# Port Identity

The Dashboard API treats:

```text
A01
A02
A03
...
A08
```

as stable physical controller ports.

Raw GPIO numbers are not normally needed by the customer dashboard.

---

# 15. Get Single Port

## Endpoint

```http
GET /v1/devices/{device_id}/ports/{port}
```

Example:

```text
GET /v1/devices/dev_123/ports/A03
```

---

# 16. Assign Port to Tank

## Endpoint

```http
PUT /v1/devices/{device_id}/ports/{port}/tank
```

Example request:

```json
{
  "tank_id": "tank_003"
}
```

---

# Assignment Rules

The API should validate:

- Device belongs to user
- Port exists
- Tank belongs to user or organization
- Tank is eligible for assignment

---

# 17. Unassign Port From Tank

## Endpoint

```http
DELETE /v1/devices/{device_id}/ports/{port}/tank
```

This removes the current logical assignment but does not delete:

- Port
- Sensor
- Historical readings
- Tank history

---

# 18. List Tanks

## Endpoint

```http
GET /v1/tanks
```

Optional filters:

```text
location_id
device_id
status
```

---

## Example Response

```json
{
  "success": true,
  "data": [
    {
      "id": "tank_001",
      "name": "Betta Pair 01",
      "location_id": "loc_001",
      "device_id": "dev_001",
      "port": "A01",
      "current_temperature_f": 80.1
    }
  ]
}
```

---

# 19. Create Tank

## Endpoint

```http
POST /v1/tanks
```

Example:

```json
{
  "name": "Betta Pair 01",
  "location_id": "loc_001",
  "target_low_f": 79,
  "target_high_f": 82
}
```

---

# Tank Fields

Potential initial fields:

```text
id
name
location_id
notes
target_low
target_high
temperature_unit
created_at
updated_at
```

Future fields may include:

- Species
- Fish IDs
- Volume
- Photos
- Breeding status
- Water parameters

These do not need to be part of the core temperature API initially.

---

# 20. Get Tank

## Endpoint

```http
GET /v1/tanks/{tank_id}
```

Example response:

```json
{
  "success": true,
  "data": {
    "id": "tank_001",
    "name": "Betta Pair 01",
    "target_low_f": 79,
    "target_high_f": 82,
    "current": {
      "temperature_f": 80.1,
      "recorded_at": "2026-08-16T23:00:30Z"
    },
    "assignment": {
      "device_id": "dev_001",
      "port": "A01",
      "sensor_id": "28-FF-..."
    }
  }
}
```

---

# 21. Update Tank

## Endpoint

```http
PATCH /v1/tanks/{tank_id}
```

Example:

```json
{
  "name": "Betta Spawn 2026-08-16",
  "target_low_f": 80,
  "target_high_f": 82
}
```

---

# 22. Delete Tank

## Endpoint

```http
DELETE /v1/tanks/{tank_id}
```

Deleting a tank should require explicit confirmation.

Historical data retention policy must be defined separately.

Soft deletion may be preferable.

---

# 23. Current Temperatures

## Endpoint

```http
GET /v1/temperatures/current
```

Returns the latest temperature for accessible tanks.

---

## Example Response

```json
{
  "success": true,
  "data": [
    {
      "tank_id": "tank_001",
      "tank_name": "Betta Pair 01",
      "device_id": "dev_001",
      "port": "A01",
      "temperature_f": 80.1,
      "status": "normal",
      "recorded_at": "2026-08-16T23:00:30Z"
    }
  ]
}
```

---

# Temperature Status

Dashboard-facing status may include:

```text
normal
warning_low
warning_high
critical_low
critical_high
offline
sensor_error
unknown
```

These are higher-level states than raw device sensor status.

---

# 24. Temperature History

## Endpoint

```http
GET /v1/tanks/{tank_id}/temperature-history
```

Query parameters:

```text
from
to
resolution
limit
cursor
```

Example:

```http
GET /v1/tanks/tank_001/temperature-history?from=2026-08-15T00:00:00Z&to=2026-08-16T00:00:00Z
```

---

# History Response

```json
{
  "success": true,
  "data": [
    {
      "recorded_at": "2026-08-16T22:00:00Z",
      "temperature_f": 79.8
    },
    {
      "recorded_at": "2026-08-16T22:00:30Z",
      "temperature_f": 79.9
    }
  ]
}
```

---

# History Resolution

For large time ranges, the API may downsample data.

Potential values:

```text
raw
1m
5m
15m
1h
1d
```

Example:

```http
?resolution=5m
```

The server may automatically select an efficient resolution.

---

# 25. Temperature Statistics

## Endpoint

```http
GET /v1/tanks/{tank_id}/temperature-stats
```

Query range:

```text
24h
7d
30d
```

Example response:

```json
{
  "success": true,
  "data": {
    "min_f": 78.9,
    "max_f": 81.4,
    "average_f": 80.1,
    "samples": 2880
  }
}
```

---

# 26. List Alert Rules

## Endpoint

```http
GET /v1/alert-rules
```

---

# 27. Create Alert Rule

## Endpoint

```http
POST /v1/alert-rules
```

Example:

```json
{
  "tank_id": "tank_001",
  "type": "temperature_low",
  "threshold_f": 78,
  "duration_seconds": 300,
  "enabled": true
}
```

---

# Initial Alert Rule Types

```text
temperature_low
temperature_high
controller_offline
sensor_disconnected
sensor_changed
```

Future types may include:

- Rapid temperature drop
- Rapid temperature increase
- Power outage
- Leak detected
- Water level low

---

# 28. Update Alert Rule

## Endpoint

```http
PATCH /v1/alert-rules/{alert_rule_id}
```

Example:

```json
{
  "threshold_f": 79,
  "duration_seconds": 180
}
```

---

# 29. Delete Alert Rule

## Endpoint

```http
DELETE /v1/alert-rules/{alert_rule_id}
```

---

# 30. List Alerts

## Endpoint

```http
GET /v1/alerts
```

Possible filters:

```text
active
resolved
tank_id
device_id
severity
```

---

# Example Alert

```json
{
  "id": "alert_001",
  "tank_id": "tank_001",
  "type": "temperature_low",
  "severity": "critical",
  "message": "Temperature dropped below 76°F",
  "triggered_at": "2026-08-16T22:40:00Z",
  "resolved_at": null
}
```

---

# Alert Severity

Initial values:

```text
info
warning
critical
```

---

# 31. Acknowledge Alert

## Endpoint

```http
POST /v1/alerts/{alert_id}/acknowledge
```

This records that a user has seen the alert.

Acknowledgement is not the same as resolution.

---

# 32. Device Health

## Endpoint

```http
GET /v1/devices/{device_id}/health
```

Example response:

```json
{
  "success": true,
  "data": {
    "status": "online",
    "last_seen_at": "2026-08-16T23:00:00Z",
    "uptime_seconds": 86400,
    "firmware_version": "1.0.0",
    "free_heap": 185420,
    "network": {
      "type": "wifi",
      "rssi": -54
    },
    "ports": {
      "connected": 7,
      "total": 8
    }
  }
}
```

---

# 33. Firmware Information

## Endpoint

```http
GET /v1/devices/{device_id}/firmware
```

Example:

```json
{
  "success": true,
  "data": {
    "current_version": "1.0.0",
    "latest_version": "1.1.0",
    "update_available": true,
    "channel": "production"
  }
}
```

---

# 34. Request Firmware Check

Potential endpoint:

```http
POST /v1/devices/{device_id}/commands/check-firmware
```

This does not directly push firmware to the ESP32.

Instead it creates a device command.

---

# 35. Restart Device

Potential endpoint:

```http
POST /v1/devices/{device_id}/commands/restart
```

Example response:

```json
{
  "success": true,
  "data": {
    "command_id": "cmd_123",
    "status": "pending"
  }
}
```

---

# 36. Identify Device

## Endpoint

```http
POST /v1/devices/{device_id}/commands/identify
```

Possible behavior:

```text
Flash status LED
```

for a short period.

Useful when a customer owns several controllers.

---

# 37. Enter Setup Mode

Potential endpoint:

```http
POST /v1/devices/{device_id}/commands/setup-mode
```

This should only be available when technically safe.

---

# Remote Factory Reset

Remote factory reset should not be exposed casually.

If implemented, it requires:

- Recent authentication
- Explicit confirmation
- Strong authorization
- Audit log
- Device ownership validation

It may be excluded from V1.

---

# 38. Dashboard Summary

## Endpoint

```http
GET /v1/dashboard
```

Returns data needed for the main dashboard.

Example:

```json
{
  "success": true,
  "data": {
    "devices": {
      "total": 4,
      "online": 3,
      "offline": 1
    },
    "tanks": {
      "total": 28,
      "normal": 25,
      "warning": 2,
      "critical": 1
    },
    "active_alerts": 3
  }
}
```

---

# 39. Dashboard Tank Cards

A dashboard endpoint may return optimized current tank information.

Example:

```json
{
  "id": "tank_001",
  "name": "Betta Pair 01",
  "temperature_f": 80.1,
  "target_low_f": 79,
  "target_high_f": 82,
  "status": "normal",
  "controller": "Main Rack",
  "port": "A01",
  "last_updated_at": "2026-08-16T23:00:30Z"
}
```

---

# 40. Notification Preferences

## Endpoint

```http
GET /v1/notification-preferences
```

Potential options:

```text
email
push
SMS - future
```

---

# Update Notification Preferences

```http
PATCH /v1/notification-preferences
```

Example:

```json
{
  "email": true,
  "push": true
}
```

---

# 41. Audit Events

Important account actions should be logged.

Examples:

- Device claimed
- Device removed
- Ownership transferred
- Alert rule changed
- Device restart requested
- Account security change

Potential endpoint for users:

```http
GET /v1/activity
```

---

# 42. Dashboard Search

Future or V1:

```http
GET /v1/search?q=betta
```

May search:

- Tanks
- Devices
- Locations

---

# Authorization Rules

Every resource request must verify access.

Example:

```text
GET /v1/tanks/tank_001
```

Server performs:

```text
Authenticated user
       │
       ▼
Does user have access to tank_001?
       │
       ├── YES → return
       └── NO  → deny
```

Never rely on obscurity of UUIDs.

---

# Row Level Security

If Supabase is used, database Row Level Security should reinforce API authorization.

Examples:

Users should not be able to select:

```text
another user's devices
another user's tanks
another user's alert rules
```

even if direct database access is accidentally exposed through a client API.

---

# Administrator Access

Thermone staff administration should not use the same normal dashboard privileges.

Future admin roles may include:

```text
support
operations
administrator
```

Admin capabilities should have:

- Separate authorization
- Audit logging
- Least privilege

---

# Sensitive Device Fields

The Dashboard API must never return:

- Runtime device token
- Factory credential
- Raw provisioning secrets
- OTA signing private keys
- Wi-Fi password

Customer-facing responses may include:

- Serial number
- Firmware version
- Hardware revision
- Device status
- Last seen
- Sensor IDs where useful

---

# Temperature Unit Handling

Thermone should preferably store canonical temperatures in:

```text
Celsius
```

and convert for display.

For example:

```text
Database:
26.72°C

User preference:
°F

Dashboard:
80.10°F
```

This avoids storing duplicate temperature representations as separate sources of truth.

---

# Timestamp Handling

All API timestamps should use UTC.

Example:

```text
2026-08-16T23:00:30Z
```

The frontend converts them to the user's timezone.

---

# Device Offline Behavior

When a controller is offline, the API should return last-known data with clear timestamps.

Example:

```json
{
  "temperature_f": 79.8,
  "recorded_at": "2026-08-16T22:41:00Z",
  "device_status": "offline"
}
```

The dashboard must not display this as a fresh live reading.

---

# Sensor Replacement

When the hardware reports a new sensor ID on the same port:

```text
A04

Old sensor:
28-FF-AAAA

New sensor:
28-FF-BBBB
```

The dashboard should be able to show:

```text
Replacement sensor detected
```

without automatically destroying:

- Tank assignment
- Alert configuration
- Tank history

---

# Endpoint Summary

Initial Dashboard API:

```text
GET    /v1/me
PATCH  /v1/me

GET    /v1/locations
POST   /v1/locations
PATCH  /v1/locations/{id}
DELETE /v1/locations/{id}

GET    /v1/devices
GET    /v1/devices/{id}
PATCH  /v1/devices/{id}

POST   /v1/devices/claim
GET    /v1/devices/claim/{session}
POST   /v1/devices/{id}/remove
POST   /v1/devices/{id}/transfer

GET    /v1/devices/{id}/ports
GET    /v1/devices/{id}/ports/{port}
PUT    /v1/devices/{id}/ports/{port}/tank
DELETE /v1/devices/{id}/ports/{port}/tank

GET    /v1/tanks
POST   /v1/tanks
GET    /v1/tanks/{id}
PATCH  /v1/tanks/{id}
DELETE /v1/tanks/{id}

GET    /v1/temperatures/current
GET    /v1/tanks/{id}/temperature-history
GET    /v1/tanks/{id}/temperature-stats

GET    /v1/alert-rules
POST   /v1/alert-rules
PATCH  /v1/alert-rules/{id}
DELETE /v1/alert-rules/{id}

GET    /v1/alerts
POST   /v1/alerts/{id}/acknowledge

GET    /v1/devices/{id}/health
GET    /v1/devices/{id}/firmware

POST   /v1/devices/{id}/commands/restart
POST   /v1/devices/{id}/commands/identify
POST   /v1/devices/{id}/commands/check-firmware

GET    /v1/dashboard

GET    /v1/notification-preferences
PATCH  /v1/notification-preferences

GET    /v1/activity
```

---

# V1 Dashboard Flow

```text
User
 │
 ▼
Login
 │
 ▼
Dashboard
 │
 ├── Locations
 │
 ├── Devices
 │
 ├── Tanks
 │
 ├── Temperatures
 │
 ├── Alerts
 │
 └── Settings
```

---

# New User Flow

```text
Create Account
      │
      ▼
Scan Device QR
      │
      ▼
Claim Device
      │
      ▼
Create Location
      │
      ▼
Name Controller
      │
      ▼
Create Tanks
      │
      ▼
Assign A01-A08
      │
      ▼
Set Temperature Ranges
      │
      ▼
View Dashboard
```

---

# Main Dashboard Goal

A customer should be able to open the Thermone dashboard and immediately answer:

```text
Are all of my controllers online?

Are all of my tanks at the correct temperature?

Which tank needs attention?

When did the problem begin?
```

---

# API Design Principle

The Dashboard API should expose product concepts rather than low-level ESP32 implementation details.

Prefer:

```text
Tank
Controller
Port
Temperature
Alert
```

over exposing:

```text
raw GPIO
memory addresses
firmware internals
```

Low-level diagnostic data may be available separately for support and engineering.

---

# V1 Success Criteria

The Dashboard API is ready when an authenticated user can:

1. Create and manage locations.
2. Claim a Thermone controller.
3. View owned controllers.
4. Rename and organize controllers.
5. View all eight ports.
6. Create tanks.
7. Assign ports to tanks.
8. View current temperatures.
9. View historical temperatures.
10. Configure alert thresholds.
11. View active alerts.
12. Check device health.
13. See firmware information.
14. Send approved device commands.
15. Remove or transfer a controller safely.
16. Access only resources they are authorized to see.

---

# Core Dashboard API Principle

The Dashboard API connects:

```text
Thermone User
      │
      ▼
Thermone Account
      │
      ▼
Locations
      │
      ▼
Controllers
      │
      ▼
Ports
      │
      ▼
Tanks
      │
      ▼
Temperature History
      │
      ▼
Alerts
```

while keeping physical device authentication completely separate.