# Thermone V1 Recovery Mode

## Purpose

This document defines the recovery behavior for Thermone V1 controllers.

Recovery mode exists to prevent a software or configuration failure from permanently disabling a controller.

Thermone must remain recoverable from:

- Failed OTA updates
- Repeated boot failures
- Corrupted runtime configuration
- Corrupted Wi-Fi configuration
- Broken device credentials
- Application crashes
- Invalid firmware state
- Manual recovery requests

The core recovery principle is:

```text
Permanent factory identity survives.

Recoverable customer/runtime state may be reset.
```

---

# Recovery Goals

Recovery mode should allow a Thermone controller to:

1. Boot into a known-safe environment.
2. Preserve factory identity.
3. Restore network access.
4. Re-establish cloud communication.
5. Repair or reinstall production firmware.
6. Clear corrupt runtime configuration.
7. Reset Wi-Fi when necessary.
8. Perform factory reset when explicitly requested.
9. Provide basic diagnostics.
10. Avoid becoming permanently bricked.

---

# Recovery Architecture

```text
Normal Firmware
     │
     ▼
Serious Failure Detected
     │
     ▼
Recovery Required?
     │
     ├── NO → Continue Normal Recovery Logic
     │
     └── YES
            │
            ▼
       Recovery Mode
            │
       ┌────┼────┐
       │    │    │
       ▼    ▼    ▼
    Network OTA  Reset
    Repair Repair Tools
```

---

# Recovery Mode Is Separate From Normal Operation

Normal application firmware contains the full Thermone feature set.

Recovery mode should remain intentionally minimal.

Recovery mode should not attempt to provide:

- Full telemetry history
- Tank management
- Complex cloud commands
- Advanced UI
- Non-essential features

Its job is to restore the controller to a working state.

---

# Recovery Entry Conditions

Thermone may enter recovery mode when:

```text
Repeated boot failures
OTA validation failure
Runtime configuration corruption
Missing required application image
Manual recovery trigger
Critical storage corruption
Authentication recovery required
```

---

# 1. Boot Failure Detection

Thermone should track repeated unsuccessful boots.

Example:

```text
Boot attempt 1
   ↓
Crash

Boot attempt 2
   ↓
Crash

Boot attempt 3
   ↓
Crash

Recovery mode
```

The exact threshold will be defined during testing.

Recommended initial threshold:

```text
3 consecutive failed boots
```

---

# Successful Boot Reset

When normal firmware remains healthy for a defined period:

```text
boot_failure_count = 0
```

Example validation period:

```text
60–120 seconds
```

---

# Boot Failure Counter

The boot counter must be stored somewhere that survives resets.

Possible storage:

```text
RTC memory
NVS
OTA boot state
```

Implementation must avoid excessive flash writes during crash loops.

---

# 2. OTA Failure Recovery

The preferred first response to a broken OTA firmware image is:

```text
Bootloader Rollback
```

Example:

```text
Firmware 1.0.0
     │
     ▼
Install 1.1.0
     │
     ▼
1.1.0 fails
     │
     ▼
Rollback to 1.0.0
```

Recovery mode is only required if normal rollback cannot restore a healthy application.

---

# OTA Recovery Hierarchy

Recommended order:

```text
1. Validate new OTA image
2. Roll back to previous known-good image
3. If rollback fails → Recovery Mode
4. Recovery downloads known-good firmware
```

---

# 3. Recovery Firmware

Thermone should maintain a minimal recovery-capable image or boot path.

Recovery firmware responsibilities:

- Read factory identity
- Initialize networking
- Start recovery Wi-Fi
- Contact recovery API
- Download known-good firmware
- Perform network reset
- Perform factory reset
- Show recovery status
- Report diagnostics

---

# Recovery Firmware Size

Recovery firmware should be substantially simpler than normal production firmware.

Prefer minimal dependencies.

The fewer features it contains, the lower the chance that normal application bugs also break recovery.

---

# Recovery State

Internal state:

```text
RECOVERY
```

Possible recovery sub-states:

```text
RECOVERY_BOOT
RECOVERY_NETWORK
RECOVERY_CLOUD
RECOVERY_FIRMWARE
RECOVERY_RESET
RECOVERY_ERROR
```

---

# Status LED

Recommended recovery indication:

```text
Red + Blue alternating
```

or another unmistakable pattern.

The pattern should be visually different from:

- Setup mode
- Normal cloud failure
- OTA update

---

# 4. Manual Recovery Entry

A user or technician should be able to enter recovery mode using the physical button.

Example:

```text
Power off device

Hold Setup button

Apply power

Continue holding for 10 seconds

Release

Enter recovery mode
```

The exact sequence should be difficult to trigger accidentally.

---

# Normal Setup Button Actions

Normal runtime behavior may remain:

```text
Short press
→ identify/status

Hold ~5 seconds
→ network setup

Hold ~10 seconds
→ factory reset warning
```

Manual recovery should use a distinct power-on sequence.

---

# Recovery Button Flow

Conceptual:

```text
Button Held During Boot?
       │
       ├── NO → Normal Boot
       │
       └── YES
              │
              ▼
      Held Long Enough?
              │
              ├── NO → Normal Boot
              │
              └── YES → Recovery Mode
```

---

# 5. Recovery Network

When recovery mode cannot use stored networking, Thermone creates a temporary recovery access point.

Example:

```text
Thermone-Recovery-0482
```

This is different from normal setup:

```text
Thermone-0482
```

---

# Recovery AP

Example:

```text
SSID:
Thermone-Recovery-0482
```

The AP should use a device-specific password where possible.

---

# Recovery Portal

Local page:

```text
http://192.168.4.1
```

Example UI:

```text
THERMONE RECOVERY

Device:
THV1-000482

Status:
Recovery Mode

Choose an action:

[ Repair Firmware ]

[ Reconnect Wi-Fi ]

[ Reset Network ]

[ Factory Reset ]

[ Diagnostics ]
```

---

# Recovery Portal Restrictions

Recovery actions should require:

- Physical access
- Active recovery mode
- Local connection

Where practical, sensitive recovery operations should also require:

- Device-specific setup password
- Confirmation
- Cloud authorization

---

# 6. Network Repair

Recovery mode should allow the user to replace invalid Wi-Fi credentials.

Flow:

```text
Recovery Portal
      │
      ▼
Scan Networks
      │
      ▼
Select Wi-Fi
      │
      ▼
Enter Password
      │
      ▼
Test Connection
      │
      ▼
Store if Valid
```

This uses the same basic networking logic as normal provisioning.

---

# Recovery Network Reset

A network reset clears only:

```text
SSID
Wi-Fi password
Network preferences
```

It should preserve:

```text
Factory serial
Runtime device ID
Runtime device token
Other cloud identity
```

unless those are also corrupt.

---

# 7. Runtime Credential Recovery

The device may have valid factory identity but an invalid runtime credential.

Example:

```text
Device API returns 401 repeatedly
```

The firmware should not endlessly retry the same invalid token.

Instead:

```text
Runtime Credential Invalid
        │
        ▼
Enter Credential Recovery
```

---

# Credential Recovery

Possible flow:

```text
Device proves factory identity
        │
        ▼
Recovery API validates device
        │
        ▼
Old runtime credential revoked
        │
        ▼
New runtime credential issued
```

This flow must be protected against abuse.

---

# Recovery Credential Restrictions

Factory credentials should not automatically issue new runtime credentials without server-side checks.

The backend may verify:

- Device serial
- Hardware ID
- Device status
- Recovery eligibility
- Ownership state
- Security flags

---

# 8. Firmware Repair

Primary recovery function:

```text
Install Known-Good Firmware
```

Flow:

```text
Recovery Firmware
      │
      ▼
Connect to Internet
      │
      ▼
Authenticate Recovery
      │
      ▼
Request Recovery Manifest
      │
      ▼
Download Known-Good Firmware
      │
      ▼
Verify
      │
      ▼
Install
      │
      ▼
Reboot
```

---

# Recovery Firmware Endpoint

Conceptual:

```http
GET /v1/device/recovery/firmware
```

Exact API design may differ.

---

# Recovery Firmware Response

Example:

```json
{
  "version": "1.0.5",
  "hardware_revision": "1.0",
  "download_url": "https://firmware.thermone.com/signed/...",
  "sha256": "abc123...",
  "recovery_release": true
}
```

---

# Known-Good Firmware

The recovery service should not simply return:

```text
latest firmware
```

It should return:

```text
known-good recovery-approved firmware
```

These may be different.

---

# Recovery Release Status

Firmware release records should support a flag such as:

```text
recovery_approved = true
```

A release becomes recovery-approved only after strong validation.

---

# Firmware Verification

Recovery mode must still verify:

- HTTPS
- Hardware compatibility
- Hash
- Signature where enabled

Recovery must not weaken firmware authenticity checks.

---

# Recovery Download Failure

If Internet access is lost:

```text
Keep recovery mode active
```

Do not erase the current bootable recovery path.

The user can retry later.

---

# 9. Corrupt Runtime Configuration

If runtime configuration cannot be parsed:

```text
Do not immediately erase everything.
```

Preferred flow:

```text
Load configuration
      │
      X
Invalid
      │
      ▼
Try backup/default
      │
      ├── Works → continue
      │
      └── Fails → recovery
```

---

# Configuration Backups

Important small configuration may keep:

```text
current
previous-known-good
```

versions.

This makes recovery easier after a bad configuration update.

---

# Safe Defaults

If cloud config is unavailable or corrupt, firmware can use safe defaults:

```text
Sensor read:
5 seconds

Telemetry:
30 seconds

Heartbeat:
60 seconds
```

Do not enter a reboot loop because of one invalid optional cloud field.

---

# 10. Storage Corruption

If offline telemetry storage is corrupt:

```text
Temperature monitoring should still continue.
```

The firmware may:

1. Quarantine or erase corrupt telemetry storage.
2. Recreate the local queue.
3. Report the event.

Do not require full factory reset for an offline-history corruption.

---

# Critical Identity Storage Corruption

If factory identity cannot be read:

```text
serial missing
factory identity corrupt
```

the device should enter:

```text
RECOVERY_ERROR
```

It should not invent a new factory identity.

---

# Factory Identity Cannot Be Regenerated Locally

Never do:

```text
Factory serial missing
      ↓
Generate random new serial
```

Factory identity must be restored through an authorized Thermone factory/support workflow.

---

# 11. Factory Reset

Factory reset is the most destructive customer-accessible recovery operation.

It should be clearly separated from:

```text
Network Reset
```

---

# Factory Reset Clears

Recommended runtime values to clear:

```text
Wi-Fi credentials
Runtime device token
Cached cloud configuration
Local tank configuration
Offline telemetry queue
Temporary command history
```

---

# Factory Reset Preserves

Must preserve:

```text
Factory serial
Model
Hardware revision
Factory provisioning metadata
Factory identity required for approved recovery
```

---

# Factory Reset Flow

```text
Factory Reset Requested
      │
      ▼
Show Warning
      │
      ▼
Confirm Physical Hold
      │
      ▼
Erase Runtime Data
      │
      ▼
Restart
      │
      ▼
Setup Mode
```

---

# Factory Reset Confirmation

Recommended physical behavior:

```text
Hold button ~10 seconds

LED begins fast red flashing

Continue holding several more seconds

Reset executes
```

This should be difficult to trigger accidentally.

---

# Factory Reset From Recovery Portal

If available:

```text
Factory Reset
```

must require a second confirmation.

Example:

```text
This removes Wi-Fi and controller configuration.

Type the device serial to continue:

THV1-000482
```

For a local embedded portal, simpler physical confirmation may be preferable.

---

# Cloud Ownership After Factory Reset

Factory reset does not necessarily remove cloud ownership.

Example:

```text
Owner:
still user_123
```

The device can later reconnect and recover into the same account.

Cloud removal and factory reset are separate operations.

---

# 12. Full Deprovisioning

Full deprovisioning is different from a customer factory reset.

It may be used by:

- Thermone factory
- Refurbishment center
- Authorized support

Full deprovisioning may rotate or remove:

- Runtime credentials
- Ownership
- Claim tokens

It must not happen accidentally from the normal user button flow.

---

# 13. Diagnostics

Recovery mode should provide limited diagnostics.

Safe diagnostic fields:

```text
Serial
Hardware revision
Recovery firmware version
Last reset reason
Boot failure count
Network state
IP address
RSSI
Flash health
OTA slot state
```

---

# Diagnostic Portal Example

```text
Thermone Recovery Diagnostics

Serial:
THV1-000482

Hardware:
V1.0

Recovery:
1.0.0

Last Reset:
Watchdog

Boot Failures:
3

Wi-Fi:
Connected

Cloud:
Unavailable
```

---

# Secrets Must Not Be Displayed

Do not display:

```text
Factory credential
Runtime token
Wi-Fi password
Claim token
Private key
```

even in recovery mode.

---

# 14. Recovery Event Reporting

When cloud connectivity becomes available, report recovery events.

Examples:

```text
recovery_mode_entered
boot_loop_detected
runtime_config_corrupt
network_reset
firmware_repair_started
firmware_repair_completed
factory_reset
recovery_failed
```

---

# Example Recovery Event

```json
{
  "type": "recovery_mode_entered",
  "reason": "repeated_boot_failure",
  "boot_failure_count": 3,
  "firmware_version": "1.1.0"
}
```

---

# 15. Boot Reasons

Firmware should record reset reasons when possible.

Examples:

```text
power_on
software_restart
watchdog
panic
brownout
ota
factory_reset
unknown
```

This is useful for diagnostics.

---

# Brownout Recovery

A brownout or unstable power condition should not immediately trigger firmware repair.

Instead:

```text
Brownout
   │
   ▼
Reboot
   │
   ▼
Normal startup
```

Repeated brownouts may generate a hardware/power warning.

---

# 16. Watchdog Recovery

If a task hangs:

```text
Watchdog Reset
```

After reboot, firmware records:

```text
last_reset_reason = watchdog
```

One watchdog event does not necessarily require recovery mode.

Repeated watchdog boots may.

---

# Repeated Watchdog Example

```text
Watchdog reset
      │
Watchdog reset
      │
Watchdog reset
      │
      ▼
Recovery Mode
```

---

# 17. Recovery Retry Policy

Recovery operations should use backoff.

Example cloud retry:

```text
5 sec
10 sec
30 sec
60 sec
5 min
```

Do not hammer the API.

---

# Recovery Mode Persistence

Once automatically entered because of repeated serious failures, recovery mode should remain active until:

- Repair succeeds
- User exits explicitly
- Factory reset succeeds
- Authorized recovery action completes

Avoid repeatedly bouncing between broken normal firmware and recovery.

---

# 18. Exit Recovery

Successful firmware repair:

```text
Recovery Firmware
      │
      ▼
Install Known-Good App
      │
      ▼
Reboot
      │
      ▼
Normal Firmware
      │
      ▼
Health Validation
      │
      ▼
ACTIVE
```

---

# Recovery Exit Validation

Before clearing recovery state:

- App boots
- Factory identity valid
- Storage available
- Sensor system initializes
- No immediate crash loop

Cloud connectivity is helpful but not strictly required to prove that the basic application runs.

---

# 19. Offline Recovery

A user may have no Internet access.

Recovery mode should still allow:

```text
Network reconfiguration
Factory reset
Diagnostics
```

Cloud firmware repair requires Internet unless future versions support local firmware upload.

---

# Local Firmware Upload

A future recovery feature may allow:

```text
Upload signed .bin through local recovery portal
```

If implemented, it must enforce the same:

- Signature
- Hardware compatibility
- Integrity checks

Do not allow arbitrary unsigned firmware uploads in production recovery.

---

# 20. Serial Console Recovery

During development, serial console diagnostics may be used.

Production users should not require a USB serial console to recover normal devices.

Serial recovery may remain an engineering/support path.

---

# 21. Recovery API Authentication

Recovery API access should use a dedicated flow.

Do not simply make normal Device API authentication optional.

Possible authentication:

```text
Factory identity
+
Recovery challenge
+
Server eligibility check
```

Exact protocol will be defined during implementation.

---

# Recovery Challenges

A future stronger workflow may use:

```text
Server sends random challenge
      │
      ▼
Device proves possession of factory secret
      │
      ▼
Recovery authorized
```

This reduces replay risk.

---

# 22. Disabled Devices

If cloud state is:

```text
disabled
```

recovery mode must not bypass the disable flag.

The server may allow only limited operations such as:

- Security recovery
- Support diagnostics

---

# Stolen Devices

A stolen device marked disabled must not use recovery mode to:

```text
self-reactivate
generate new ownership
```

Cloud authorization remains authoritative.

---

# 23. Environment Isolation

Development recovery firmware talks only to:

```text
development
```

Staging recovery firmware talks only to:

```text
staging
```

Production recovery firmware talks only to:

```text
production
```

Never automatically fall back across environments.

---

# 24. Recovery Partition Strategy

Conceptual flash architecture:

```text
Bootloader
   │
Partition Table
   │
Factory NVS
   │
Runtime NVS
   │
OTA Metadata
   │
OTA_0
   │
OTA_1
   │
Recovery / Rescue Capability
   │
Offline Data
```

The exact architecture depends on available flash.

---

# Dedicated Recovery App vs Integrated Recovery

There are two possible implementations.

## Option A — Dedicated Recovery Partition

```text
Separate recovery firmware image
```

Advantages:

- Strong separation
- Very small stable rescue environment

Disadvantages:

- More flash required
- More partition complexity

---

## Option B — Recovery Capability in Known-Good Bootstrap/App

Advantages:

- Simpler flash layout
- Less storage

Disadvantages:

- Less isolation

---

# V1 Recommendation

For the first prototype:

```text
Use ESP-IDF OTA rollback
+
retain recovery capability in bootstrap/known-good firmware
```

A dedicated rescue partition can be added later if flash capacity and reliability requirements justify it.

---

# 25. Recovery Testing

Required tests:

1. Broken application firmware
2. Crash during startup
3. Three repeated boot failures
4. Invalid Wi-Fi credentials
5. Corrupted runtime config
6. Invalid runtime credential
7. OTA hash failure
8. OTA boot failure
9. Lost Internet during repair
10. Factory reset
11. Network-only reset
12. Manual recovery entry
13. Power loss during recovery OTA
14. Disabled-device recovery attempt

---

# Recovery Power-Loss Test

Scenario:

```text
Recovery firmware downloading known-good app
      │
      ▼
Power disconnected
      │
      ▼
Power restored
```

Expected:

```text
Recovery remains bootable
or previous known-good app boots
```

Never:

```text
No bootable image
```

---

# 26. Recovery Success Criteria

Thermone recovery is ready for V1 when:

1. A failed OTA can roll back.
2. Repeated failed boots trigger recovery.
3. Recovery can start a local AP.
4. User can replace Wi-Fi credentials.
5. Network reset does not erase factory identity.
6. Factory reset does not erase permanent serial.
7. Known-good firmware can be downloaded and installed.
8. Recovery firmware verifies OTA integrity.
9. Runtime token failure has a recovery path.
10. Corrupt telemetry storage does not stop temperature monitoring.
11. Recovery events are reported when cloud returns.
12. Recovery does not expose secrets.
13. Disabled devices cannot bypass server policy.
14. Power failure during repair does not permanently brick the controller.
15. A repaired controller can return to normal firmware automatically.

---

# Recovery Decision Summary

```text
Minor Network Failure
       │
       ▼
Normal Retry


Wi-Fi Credentials Invalid
       │
       ▼
Network Setup


Runtime Config Invalid
       │
       ▼
Safe Default / Config Recovery


OTA New Image Fails
       │
       ▼
Rollback


Repeated Application Failure
       │
       ▼
Recovery Mode


Customer Wants Clean Setup
       │
       ▼
Factory Reset


Factory Identity Missing
       │
       ▼
Authorized Service Required
```

---

# Core Recovery Principle

Thermone recovery should preserve the most important layer:

```text
Physical Device Identity
```

while allowing damaged layers above it to be rebuilt:

```text
Factory Identity
      │
      ├── Network Configuration
      ├── Runtime Credentials
      ├── Cloud Configuration
      └── Production Firmware
```

A software failure should be repairable.

A network failure should be recoverable.

A bad firmware release should be reversible.

A customer should not need to replace functioning Thermone hardware because the application software became corrupted.