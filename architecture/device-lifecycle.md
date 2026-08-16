# Thermone V1 Device Lifecycle

## Purpose

This document defines the complete lifecycle of a Thermone V1 controller.

The lifecycle begins when a controller is prepared for manufacturing and ends when the device is retired, replaced, or permanently decommissioned.

The lifecycle is designed to provide:

- Secure factory identity
- Predictable first-time setup
- Reliable user claiming
- Secure device authentication
- Safe OTA firmware updates
- Recoverable networking
- Ownership transfer support
- Factory reset support
- Device retirement support

---

# Lifecycle Overview

```text
Manufacturing
     │
     ▼
Factory Provisioned
     │
     ▼
Unclaimed
     │
     ▼
First Power-On
     │
     ▼
Network Setup
     │
     ▼
Cloud Registration
     │
     ▼
Claimed
     │
     ▼
Activated
     │
     ▼
Normal Operation
     │
     ├───────────────┐
     │               │
     ▼               ▼
Firmware Update   Network Recovery
     │               │
     └───────┬───────┘
             │
             ▼
       Normal Operation
             │
       ┌─────┴─────┐
       │           │
       ▼           ▼
Ownership      Factory
Transfer       Reset
       │           │
       └─────┬─────┘
             │
             ▼
        Re-Provision
             │
             ▼
        Normal Operation
             │
             ▼
        Decommissioned
```

---

# Device States

Thermone uses explicit device states.

Initial recommended states are:

```text
manufactured
factory_provisioned
unclaimed
provisioning
registered
claimed
active
offline
updating
recovery
reset_pending
transfer_pending
disabled
decommissioned
```

These states may be represented internally using more detailed backend state fields.

---

# 1. Manufacturing

The manufacturing phase begins when a physical Thermone controller is assembled.

The controller may include:

- ESP32 development board
- 8 probe connectors
- Status LED
- USB power input
- Optional Ethernet hardware
- Waterproof enclosure

At this stage, the controller has not yet been assigned to a customer.

---

# 2. Factory Device Record Creation

Before flashing the controller, the provisioning system creates a factory device record.

Example:

```json
{
  "serial_number": "THV1-000001",
  "model": "THV1",
  "hardware_revision": "1.0",
  "status": "manufactured"
}
```

The provisioning service generates:

- Factory serial number
- Random factory credential
- Random claim token
- QR code
- Device label information

The claim token and device credential must be different values.

---

# 3. Factory Identity

Each controller receives a permanent factory identity.

Example:

```text
Serial:
THV1-000001

Hardware Revision:
1.0

Model:
THV1
```

The serial number is intended for:

- Manufacturing
- Inventory
- Support
- QR association
- Customer identification

The serial number is not an authentication secret.

---

# 4. Claim Token Generation

Each device receives a cryptographically random claim token.

Example conceptual value:

```text
b5f30e8e5cf94e1097bcb91681c46f2a...
```

The token is used to generate the device QR code.

Example:

```text
https://app.thermone.com/claim/<claim-token>
```

The backend should store a secure representation of the claim token.

The claim token must not be reused as:

- Device API authentication
- Wi-Fi credentials
- Firmware authentication
- Admin authentication

---

# 5. Factory Credential

The controller also receives a factory credential.

The factory credential allows the bootstrap firmware to securely identify itself to Thermone Cloud before a permanent runtime device credential has been issued.

The factory credential is:

- Unique per controller
- Not printed on the enclosure
- Not included visibly in the QR code
- Revocable
- Stored securely where practical

---

# 6. Bootstrap Firmware Installation

Before shipping, every controller receives Thermone bootstrap firmware.

Bootstrap firmware responsibilities include:

- Reading factory identity
- Loading stored network configuration
- Starting first-time setup mode
- Connecting to Wi-Fi
- Connecting to Ethernet when available
- Contacting Thermone Cloud
- Authenticating using factory credentials
- Registering the controller
- Obtaining runtime credentials
- Checking for production firmware
- Installing OTA firmware
- Supporting recovery
- Supporting factory reset

A controller must never ship completely blank.

---

# 7. Factory Validation

After flashing, the manufacturing process performs a validation test.

Recommended checks:

- ESP32 boots correctly
- Factory serial is readable
- Factory credential exists
- Device can start setup mode
- Status LED works
- Probe ports are detectable
- Wi-Fi hardware works
- Ethernet works if installed
- OTA partition layout is valid
- Firmware version is reported
- QR code matches factory record

If the unit fails validation, it must not be marked ready for shipment.

---

# 8. Factory Provisioned State

After successful testing:

```text
status = factory_provisioned
```

The device is now ready to be packaged or placed into inventory.

---

# 9. Unclaimed State

A factory-provisioned device initially has no customer owner.

Example:

```text
serial:
THV1-000001

owner:
NULL

claim_status:
available

device_status:
unclaimed
```

The QR code may be scanned before the controller is powered on.

The backend should still recognize the device.

---

# 10. First Power-On

When the customer powers on the controller, bootstrap firmware starts.

The first decision is whether valid network configuration already exists.

```text
Power On
   │
   ▼
Stored Network Configuration?
   │
   ├── YES
   │     │
   │     ▼
   │ Connect to Network
   │
   └── NO
         │
         ▼
    Enter Setup Mode
```

---

# 11. Ethernet Detection

If Thermone V1 includes Ethernet support, the controller checks Ethernet first.

```text
Ethernet Link?
    │
    ├── YES
    │     │
    │     ▼
    │ Request DHCP Address
    │     │
    │     ▼
    │ Internet Available?
    │
    └── NO
          │
          ▼
       Wi-Fi
```

Ethernet does not necessarily eliminate setup mode.

The customer may still need setup mode for:

- Claiming
- Wi-Fi backup configuration
- Device naming
- Local diagnostics

---

# 12. Wi-Fi Setup Mode

If no usable network connection exists, Thermone enters provisioning mode.

The controller creates a temporary Wi-Fi network.

Example:

```text
Thermone-A482
```

The suffix may be derived from a non-secret portion of the device identity.

The setup network should not expose secret credentials in its SSID.

---

# 13. Local Setup Portal

The user connects a phone, tablet, or computer to the temporary Thermone network.

The controller presents a local setup page.

Example:

```text
Thermone Setup

Device:
THV1-000001

Available Wi-Fi Networks:

HomeWiFi
OfficeWiFi
FishRoomNetwork

Wi-Fi Password:
[ ************* ]

[ Connect ]
```

The controller stores the supplied network credentials locally.

The user's Wi-Fi password should not be transmitted to Thermone Cloud.

---

# 14. Wi-Fi Connection Attempt

After credentials are provided:

```text
Provisioning Portal
       │
       ▼
Save Credentials
       │
       ▼
Attempt Wi-Fi Connection
       │
       ├── SUCCESS
       │      │
       │      ▼
       │ Internet Test
       │
       └── FAILURE
              │
              ▼
       Return to Setup Mode
```

The device should clearly distinguish between:

- Invalid Wi-Fi password
- Network unavailable
- DHCP failure
- Internet unavailable
- Thermone API unavailable

---

# 15. Internet Connectivity Test

A successful Wi-Fi connection does not guarantee Internet access.

The device checks whether it can reach Thermone services.

Example:

```text
Wi-Fi Connected
      │
      ▼
DNS Working?
      │
      ▼
HTTPS Available?
      │
      ▼
Thermone API Reachable?
```

If the local network is available but the Internet is not, the controller remains operational locally and retries cloud connectivity.

---

# 16. Device Registration

Once Internet access is available, the bootstrap firmware contacts the Thermone Device API.

Example endpoint:

```text
POST /v1/device/register
```

Example request:

```json
{
  "serial_number": "THV1-000001",
  "hardware_revision": "1.0",
  "bootstrap_version": "1.0.0",
  "hardware_id": "ESP32-HARDWARE-ID"
}
```

Authentication is performed using the factory credential.

---

# 17. Server Registration Validation

The API validates:

- Serial exists
- Factory credential is valid
- Device is not disabled
- Hardware revision is supported
- Registration attempt is valid
- Hardware identity does not conflict with another device

If validation fails, the controller enters a recoverable registration error state.

---

# 18. Internal Device Record Creation

The API creates or activates the permanent device record.

Example:

```json
{
  "device_id": "019c3e92-0000-7000-8000-000000000001",
  "serial_number": "THV1-000001",
  "status": "registered"
}
```

The internal device ID is different from the human-readable serial number.

---

# 19. Runtime Credential Issuance

After registration, Thermone issues a runtime device credential.

Example conceptual response:

```json
{
  "device_id": "019c3e92-0000-7000-8000-000000000001",
  "device_token": "generated-device-credential"
}
```

The runtime credential is used for normal API communication.

The factory credential should no longer be used for routine telemetry.

---

# 20. Runtime Credential Storage

The controller stores:

- Internal device ID
- Runtime device credential
- Registration state
- Network configuration
- Firmware channel
- Relevant configuration

Sensitive values should be stored securely where practical.

---

# 21. Registered State

After registration:

```text
status = registered
```

The device may still be unclaimed by a user.

Example:

```text
Device:
registered

Owner:
NULL
```

---

# 22. Firmware Check

After registration, the bootstrap firmware checks the firmware service.

Example:

```text
GET /v1/device/firmware/check
```

The request includes:

- Device model
- Hardware revision
- Current firmware version
- Update channel

---

# 23. Production Firmware Download

If a newer compatible version is available:

```text
Bootstrap Firmware
       │
       ▼
Firmware Manifest
       │
       ▼
Download Firmware
       │
       ▼
Verify Firmware
       │
       ▼
Write OTA Partition
```

The firmware must be verified before booting it.

---

# 24. First OTA Boot

After firmware installation:

```text
Install OTA
    │
    ▼
Restart
    │
    ▼
Boot Production Firmware
    │
    ▼
Self-Test
```

The firmware performs a startup health check.

Example checks:

- Storage accessible
- Device credential accessible
- Sensor subsystem initialized
- Network stack initialized
- Watchdog operational

If startup validation fails, rollback should occur where supported.

---

# 25. Device Claiming

Device claiming is separate from network provisioning.

The customer scans the QR code.

Example:

```text
https://app.thermone.com/claim/<claim-token>
```

The dashboard validates the token.

---

# 26. Claim Before Device Registration

A customer may scan the QR before the controller has connected to Thermone Cloud.

The dashboard should support this case.

Example:

```text
Thermone Controller
THV1-000001

Waiting for this controller to connect.

1. Power on the device.
2. Complete network setup.
3. Keep this page open.
```

Once the controller registers, the claim process can continue.

---

# 27. User Authentication

Before completing a claim, the user must authenticate.

Supported methods may include:

- Email and password
- Magic link
- Google
- Apple

Authentication implementation is defined separately.

---

# 28. Claim Validation

The backend checks:

- Claim token exists
- Claim token is valid
- Device is eligible for claiming
- Device is not already owned
- Device is not disabled
- User is authenticated

---

# 29. Ownership Creation

After successful claim:

```text
Device:
THV1-000001

Owner:
user_xxxxx

Claim Status:
claimed
```

The controller becomes visible in the user's dashboard.

---

# 30. Claimed State

After successful ownership assignment:

```text
status = claimed
```

The device may transition immediately to:

```text
status = active
```

after its next successful heartbeat.

---

# 31. Initial Controller Setup

After claiming, the dashboard guides the user through initial configuration.

Example:

```text
Name Your Controller

[ Main Fish Room ]

Location

[ Home ]

Temperature Unit

[ °F ]

[ Continue ]
```

---

# 32. Probe Discovery

The controller reads each physical probe port.

Example:

```text
A01 → Sensor detected
A02 → Sensor detected
A03 → No sensor
A04 → Sensor detected
```

Because each probe port has its own GPIO, the controller knows the physical location directly.

---

# 33. Probe Identity

When a probe is detected:

```text
Port:
A01

Sensor ID:
28-FF-64-1D-92-16-03-8C
```

The controller reports both values to Thermone Cloud.

---

# 34. Tank Assignment

The dashboard allows the user to assign a port to a tank.

Example:

```text
Port A01

Current Temperature:
80.2°F

Assign To:

[ Betta Breeding Tank 01 ]

[ Save ]
```

The relationship becomes:

```text
Controller
THV1-000001
      │
      ▼
Port A01
      │
      ▼
Sensor
28-FF-...
      │
      ▼
Tank
Betta Breeding Tank 01
```

---

# 35. Active State

A properly configured controller enters normal active operation.

```text
status = active
```

Normal operation includes:

- Sensor reads
- Telemetry uploads
- Heartbeats
- Configuration checks
- Alert evaluation
- Firmware checks
- Offline buffering
- Connectivity recovery

---

# 36. Normal Telemetry Cycle

Example:

```text
Read Sensors
     │
     ▼
Validate Readings
     │
     ▼
Create Telemetry Batch
     │
     ▼
Internet Available?
     │
     ├── YES → Upload
     │
     └── NO  → Buffer Locally
```

---

# 37. Heartbeat Cycle

Heartbeats inform Thermone Cloud that the controller is healthy.

Example heartbeat information:

```json
{
  "firmware_version": "1.0.0",
  "uptime_seconds": 86400,
  "wifi_rssi": -54,
  "free_heap": 185420,
  "connected_ports": 7,
  "network": "wifi"
}
```

---

# 38. Offline State

If the device stops communicating with Thermone Cloud for a configured period:

```text
status = offline
```

This cloud-side state does not necessarily mean the controller itself has stopped operating.

The controller continues local temperature monitoring.

---

# 39. Offline Monitoring

During an outage, the controller continues:

- Reading probes
- Evaluating basic temperature safety conditions
- Recording selected telemetry locally
- Attempting network recovery

Cloud-based notifications may be unavailable until connectivity returns.

---

# 40. Buffered Telemetry

The controller stores a limited number of readings locally.

Each buffered record should include:

- Timestamp or relative time information
- Port
- Sensor ID
- Temperature
- Status

When Internet access returns:

```text
Reconnect
   │
   ▼
Authenticate
   │
   ▼
Upload Buffered Telemetry
   │
   ▼
Resume Live Uploads
```

---

# 41. Network Recovery

The controller periodically attempts reconnection.

Recommended strategy:

```text
Connection Lost
     │
     ▼
Immediate Retry
     │
     ▼
Short Backoff
     │
     ▼
Increasing Backoff
     │
     ▼
Periodic Retry
```

The retry logic should avoid continuously hammering the network or API.

---

# 42. Wi-Fi Credential Failure

If stored Wi-Fi credentials become permanently unusable, the controller should eventually allow the user to re-enter setup mode.

Possible triggers:

- Physical reset/setup button
- Extended connection failure
- Dashboard command while locally reachable
- Factory reset

---

# 43. Probe Disconnection

If a probe disappears:

```text
A04

Previous:
Sensor connected

Current:
No sensor detected
```

Thermone reports:

```text
sensor_status = disconnected
```

The tank assignment should remain intact unless the user removes it.

---

# 44. Probe Replacement

If a different sensor appears on the same physical port:

```text
A04

Old Sensor:
28-FF-AAAA...

New Sensor:
28-FF-BBBB...
```

Thermone identifies this as a probe replacement.

The dashboard may ask:

```text
Replacement sensor detected on A04.

Keep this port assigned to:
Betta Tank 04?

[ Keep Assignment ]
[ Change Tank ]
```

Tank history remains associated with the tank, not the removed sensor.

---

# 45. Firmware Update State

When an OTA update begins:

```text
status = updating
```

The controller:

1. Downloads firmware
2. Verifies firmware
3. Writes inactive partition
4. Marks update pending
5. Reboots
6. Performs validation
7. Confirms successful boot

---

# 46. OTA Failure

Possible failures include:

- Download interruption
- Hash mismatch
- Signature validation failure
- Insufficient storage
- Invalid hardware target
- Boot failure

The controller must not activate firmware that fails verification.

---

# 47. Firmware Rollback

If a newly installed firmware fails startup validation:

```text
New Firmware
    │
    X
Validation Failure
    │
    ▼
Rollback
    │
    ▼
Previous Firmware
```

The failed version should be reported to Thermone Cloud once connectivity is restored.

---

# 48. Recovery Mode

Recovery mode is used when normal firmware cannot operate safely.

Possible triggers:

- Repeated boot failures
- Invalid configuration
- Failed OTA
- Corrupted application state
- Physical recovery command

Recovery mode should provide only essential functionality:

- Network setup
- Cloud registration
- Firmware repair
- Factory reset
- Diagnostics

---

# 49. User-Initiated Restart

The dashboard may eventually allow authorized users to restart a controller.

Example:

```text
Restart Controller?

This will temporarily interrupt monitoring.

[ Cancel ]
[ Restart ]
```

Restart commands must be authenticated and authorized.

---

# 50. Factory Reset

Factory reset removes customer-specific configuration from the device.

A factory reset may clear:

- Wi-Fi credentials
- Runtime device credential
- Tank-related local configuration
- Cached telemetry
- Local user settings

The permanent factory serial should remain.

---

# 51. Factory Reset Trigger

Recommended physical flow:

```text
Hold Setup Button
      │
      ▼
5 seconds
      │
      ▼
Warning LED Pattern
      │
      ▼
Continue Holding
      │
      ▼
10 seconds
      │
      ▼
Factory Reset
```

This reduces accidental resets.

Exact timing will be determined during hardware testing.

---

# 52. Server-Side Reset Handling

A factory reset does not necessarily remove cloud ownership automatically.

There are two separate concepts:

```text
Device Configuration Reset
```

and:

```text
Account Ownership Removal
```

These should not be treated as the same action.

---

# 53. Ownership Transfer

Thermone should support transferring a controller to another owner.

Recommended flow:

```text
Current Owner
     │
     ▼
Remove / Transfer Device
     │
     ▼
Confirm Identity
     │
     ▼
Ownership Cleared
     │
     ▼
New Claim Token Generated
     │
     ▼
Device Available to Claim
```

The original factory claim token should not necessarily become reusable.

A new claim token may be generated.

---

# 54. Transfer Pending State

During ownership transfer:

```text
status = transfer_pending
```

The previous user may temporarily retain access until transfer is confirmed depending on final product policy.

---

# 55. Device Removal

A user may remove a device from their account.

Removing a device should require explicit confirmation.

Example:

```text
Remove Main Fish Room Controller?

Historical tank data will remain in your account.

The controller will need to be claimed again before another account can use it.

[ Cancel ]
[ Remove ]
```

---

# 56. Disabled Device

Thermone administrators may disable a controller for reasons such as:

- Compromised credentials
- Fraud
- Security investigation
- Returned hardware
- Manufacturing defect

Example:

```text
status = disabled
```

A disabled controller should not receive normal API access.

---

# 57. Credential Rotation

Runtime device credentials should support rotation.

Example:

```text
Current Credential
       │
       ▼
Request Rotation
       │
       ▼
Server Generates New Credential
       │
       ▼
Device Stores New Credential
       │
       ▼
Confirm Successful Use
       │
       ▼
Old Credential Revoked
```

Rotation must be designed to avoid accidentally locking out the device.

---

# 58. Compromised Credential

If a device credential is suspected to be compromised:

1. Revoke the credential
2. Mark device for recovery
3. Require secure re-authentication
4. Issue a new credential
5. Audit recent activity

The recovery path must not rely solely on the compromised credential.

---

# 59. Lost or Stolen Controller

The user should be able to mark a controller as lost or stolen.

Potential actions:

- Revoke device credential
- Disable cloud access
- Preserve historical data
- Prevent new ownership claims
- Flag serial number

---

# 60. Hardware Replacement

If a customer's controller is replaced under warranty:

```text
Old Device
THV1-000001

Replacement
THV1-000982
```

Tank and historical data should be transferable to the replacement controller.

The replacement should not require deleting historical readings.

---

# 61. Decommissioning

A controller may eventually be permanently retired.

Reasons include:

- Hardware failure
- End of support
- Customer disposal
- Internal testing hardware
- Security retirement

State:

```text
status = decommissioned
```

A decommissioned device must no longer authenticate to production APIs.

---

# 62. Data Retention After Decommissioning

Decommissioning hardware should not automatically delete customer history.

Example:

```text
Controller:
Decommissioned

Historical Temperature Data:
Retained

Tank Records:
Retained
```

Data deletion is a separate account/privacy operation.

---

# 63. Lifecycle State Summary

| State | Meaning |
|---|---|
| `manufactured` | Factory record exists |
| `factory_provisioned` | Identity and bootstrap firmware installed |
| `unclaimed` | No user owns the device |
| `provisioning` | Network or initial setup underway |
| `registered` | Device registered with Thermone Cloud |
| `claimed` | Device assigned to a user |
| `active` | Device operating normally |
| `offline` | Cloud has not heard from device |
| `updating` | OTA firmware update in progress |
| `recovery` | Device running recovery functionality |
| `reset_pending` | Factory reset requested |
| `transfer_pending` | Ownership transfer underway |
| `disabled` | API access administratively blocked |
| `decommissioned` | Device permanently retired |

---

# Normal Customer Journey

```text
Customer Receives Thermone
          │
          ▼
      Plug In
          │
          ▼
Connect to Thermone Setup Wi-Fi
          │
          ▼
Select Home Wi-Fi
          │
          ▼
Device Reaches Thermone Cloud
          │
          ▼
Device Registers
          │
          ▼
Firmware Updates
          │
          ▼
Scan QR Code
          │
          ▼
Login / Create Account
          │
          ▼
Claim Device
          │
          ▼
Name Controller
          │
          ▼
Connect Probes
          │
          ▼
Assign Tanks
          │
          ▼
View Temperatures
          │
          ▼
Receive Alerts
```

---

# Device Independence

A Thermone controller must remain capable of basic monitoring when Thermone Cloud is temporarily unavailable.

The following must not require continuous Internet connectivity:

- Reading temperature probes
- Detecting probe connectivity
- Basic local temperature evaluation
- Maintaining local configuration
- Attempting network recovery

Cloud connectivity is required for:

- Remote dashboard access
- Remote notifications
- Cloud history synchronization
- Remote configuration
- OTA updates

---

# Design Principle

The lifecycle must always prioritize:

1. Reliable monitoring
2. Recoverability
3. Device security
4. User ownership integrity
5. Data continuity
6. Safe firmware updates
7. Simple customer setup

A network failure, failed firmware update, replaced sensor, or account change should not permanently brick a Thermone controller.