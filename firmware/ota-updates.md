# Thermone V1 OTA Firmware Updates

## Purpose

This document defines the Over-the-Air firmware update process for Thermone V1 controllers.

OTA updates allow Thermone devices to receive:

- Bug fixes
- Security updates
- Sensor improvements
- Network improvements
- New features
- Recovery fixes

without requiring physical access to the controller.

The OTA system must prioritize:

- Reliability
- Firmware authenticity
- Compatibility
- Recoverability
- Rollback
- Controlled rollout

---

# Core OTA Principle

Thermone must never overwrite the only known-good application image.

The device should use an A/B style OTA layout:

```text
Current Firmware
      │
      ▼
Download New Firmware
      │
      ▼
Write Inactive Slot
      │
      ▼
Verify
      │
      ▼
Boot New Slot
      │
      ▼
Health Check
      │
      ├── PASS → Confirm
      │
      └── FAIL → Roll Back
```

---

# OTA Architecture

```text
Thermone ESP32
      │
      ▼
Thermone Device API
      │
      ▼
Firmware Manifest
      │
      ▼
Signed Download URL
      │
      ▼
Firmware Storage
      │
      ▼
Firmware Binary
      │
      ▼
ESP32 OTA Partition
```

---

# Firmware Components

The OTA system includes:

```text
Firmware Binary
Firmware Manifest
SHA-256 Hash
Firmware Signature
Hardware Compatibility Metadata
Release Channel
Version Number
```

---

# Firmware Versioning

Thermone firmware uses semantic versioning.

Example:

```text
1.0.0
```

Meaning:

```text
MAJOR.MINOR.PATCH
```

Examples:

```text
1.0.0
1.0.1
1.1.0
2.0.0
```

---

# Version Rules

## Patch

```text
1.0.0 → 1.0.1
```

Used for:

- Bug fixes
- Minor security fixes
- Small reliability improvements

---

## Minor

```text
1.0.0 → 1.1.0
```

Used for:

- New backward-compatible features
- New diagnostics
- New cloud functionality

---

## Major

```text
1.0.0 → 2.0.0
```

Used for:

- Breaking firmware behavior
- Major architecture changes
- Incompatible platform changes

---

# Build Metadata

A firmware build should also record:

```text
Version
Git commit
Build environment
Hardware target
Build timestamp
```

Example:

```text
Version:
1.1.0

Commit:
f3a92bc

Environment:
production

Hardware:
THV1-1.0
```

---

# Release Channels

Thermone supports separate firmware channels.

Initial channels:

```text
development
staging
production
```

Future channels may include:

```text
alpha
beta
canary
stable
```

---

# Development Channel

Used for active development.

Characteristics:

- Frequent updates
- Experimental
- May be unstable
- Development devices only

---

# Staging Channel

Used for release candidates.

Characteristics:

- Near-production quality
- Hardware tested
- OTA tested
- Used before production rollout

---

# Production Channel

Used for customer controllers.

Characteristics:

- Stable
- Approved
- Signed
- Tested
- Rollback-capable

---

# Firmware Promotion Flow

Recommended release process:

```text
Code Change
    │
    ▼
Build Development
    │
    ▼
Automated Tests
    │
    ▼
Development Hardware Test
    │
    ▼
Promote to Staging
    │
    ▼
Staging OTA Test
    │
    ▼
Production Approval
    │
    ▼
Production Rollout
```

---

# Hardware Compatibility

Every firmware release must declare compatible hardware.

Example:

```text
Model:
THV1

Hardware Revision:
1.0
```

A device must never install firmware intended for incompatible hardware.

---

# Firmware Manifest

Example:

```json
{
  "version": "1.1.0",
  "channel": "production",
  "model": "THV1",
  "hardware_revisions": [
    "1.0"
  ],
  "size_bytes": 1245184,
  "sha256": "abc123...",
  "mandatory": false,
  "release_id": "fwrel_019c...",
  "download_url": "https://firmware.thermone.com/signed/..."
}
```

---

# Firmware Check

The device periodically checks for updates.

Endpoint:

```http
GET /v1/device/firmware/check
```

Possible request parameters:

```text
current_version
hardware_revision
model
channel
```

---

# Example Check

```http
GET /v1/device/firmware/check?current_version=1.0.0&hardware_revision=1.0&model=THV1&channel=production
```

---

# No Update Response

```json
{
  "update_available": false,
  "current_version": "1.0.0",
  "latest_version": "1.0.0"
}
```

---

# Update Available Response

```json
{
  "update_available": true,
  "current_version": "1.0.0",
  "latest_version": "1.1.0",
  "mandatory": false,
  "release_id": "fwrel_019c...",
  "download_url": "https://firmware.thermone.com/signed/...",
  "sha256": "abc123...",
  "size_bytes": 1245184
}
```

---

# Firmware Check Frequency

Initial target:

```text
Every 6 hours
```

The server may remotely configure this interval within safe limits.

The device may also check:

- After first registration
- After manual dashboard command
- After recovery mode
- After a failed update cooldown

---

# Download Source

Large firmware binaries should not necessarily be streamed through the Device API.

Recommended architecture:

```text
Device API
   │
   ▼
Signed Temporary URL
   │
   ▼
Firmware Storage / CDN
```

---

# Signed Download URLs

Firmware URLs should preferably be:

- HTTPS
- Temporary
- Time-limited
- Bound to a specific firmware object

Example expiration:

```text
10–30 minutes
```

---

# Firmware Integrity

The device must verify firmware integrity before activation.

Minimum V1 validation:

```text
SHA-256
```

Recommended production validation:

```text
SHA-256
+
Cryptographic Signature
```

---

# Why Hash Verification Matters

A SHA-256 hash helps detect:

- Corrupted downloads
- Partial downloads
- Unexpected file modification

But a hash alone is not sufficient to prove who created the firmware unless the manifest itself is authenticated.

---

# Firmware Signing

Production firmware should eventually be cryptographically signed.

The device contains or trusts a public verification key.

The private signing key remains on secure Thermone infrastructure.

---

# Signing Model

```text
Thermone Signing Service
        │
        ▼
Private Signing Key
        │
        ▼
Firmware Signature
        │
        ▼
Firmware Release

ESP32
  │
  ▼
Public Verification Key
  │
  ▼
Verify Signature
```

---

# Private Key Security

The firmware signing private key must never be:

- Stored in firmware repo
- Embedded in ESP32
- Printed in CI logs
- Shared casually
- Included in dashboard code

---

# Public Verification Key

A public key may be embedded in firmware or protected boot configuration because it is not secret.

Its job is to verify signed firmware.

---

# OTA Partition Strategy

Thermone V1 should use two application slots.

Conceptually:

```text
OTA_0
OTA_1
```

Example:

```text
Current:
OTA_0

Download target:
OTA_1
```

After update:

```text
Boot:
OTA_1
```

If OTA_1 fails:

```text
Rollback:
OTA_0
```

---

# Conceptual Flash Layout

```text
┌────────────────────────────┐
│ Bootloader                 │
├────────────────────────────┤
│ Partition Table            │
├────────────────────────────┤
│ NVS                        │
├────────────────────────────┤
│ Factory Data               │
├────────────────────────────┤
│ OTA Metadata               │
├────────────────────────────┤
│ OTA_0 Application          │
├────────────────────────────┤
│ OTA_1 Application          │
├────────────────────────────┤
│ Offline Telemetry Storage  │
└────────────────────────────┘
```

Exact partition sizes will be defined after firmware-size testing.

---

# OTA Start Conditions

The device should not begin OTA blindly.

Before update, validate:

- Cloud connected
- Manifest valid
- Hardware compatible
- New version allowed
- Enough storage available
- Device not already updating
- Device not in unsafe recovery state

---

# Power Conditions

Thermone is normally USB powered, so battery level is not a concern.

However, firmware should still minimize update risk if power appears unstable.

Future hardware with battery support may add stricter checks.

---

# OTA State Machine

Recommended states:

```text
IDLE
CHECKING
UPDATE_AVAILABLE
DOWNLOADING
VERIFYING
INSTALLING
REBOOT_PENDING
VALIDATING
CONFIRMED
FAILED
ROLLBACK
```

---

# OTA Flow

```text
IDLE
 │
 ▼
CHECKING
 │
 ▼
Update Available?
 │
 ├── NO → IDLE
 │
 └── YES
       │
       ▼
DOWNLOADING
       │
       ▼
VERIFYING
       │
       ├── FAIL → FAILED
       │
       └── PASS
             │
             ▼
         INSTALLING
             │
             ▼
       REBOOT_PENDING
             │
             ▼
          Reboot
             │
             ▼
        VALIDATING
             │
        ┌────┴────┐
        │         │
       PASS      FAIL
        │         │
        ▼         ▼
   CONFIRMED   ROLLBACK
```

---

# Download Progress

The device may expose progress internally.

Example:

```text
OTA:
42%
```

The status LED may indicate update state.

The dashboard may also show:

```text
Firmware update in progress
```

---

# Sensor Monitoring During Download

Where practical, temperature monitoring should continue during firmware download.

The OTA task must not monopolize the CPU.

At minimum:

```text
Sensor task remains alive
```

during network transfer.

---

# Telemetry During OTA

Telemetry may be reduced or temporarily paused during the final installation/reboot stage.

The cloud should not treat a brief OTA reboot as a fault immediately.

---

# OTA Status LED

Suggested:

```text
Purple pulse
```

during OTA.

Possible states:

```text
Purple slow blink
→ downloading

Purple fast blink
→ installing

Green
→ update successful

Red
→ update failed
```

---

# Download Failure

Possible causes:

- Internet lost
- DNS failure
- Storage unavailable
- Server unavailable
- Signed URL expired
- Connection timeout

Expected behavior:

```text
Abort update
Keep current firmware
Retry later
```

---

# Partial Download

A partial firmware file must never be activated.

If download fails:

```text
Discard incomplete update
```

or safely overwrite it during a later retry.

---

# Hash Mismatch

If SHA-256 does not match:

```text
Do not install
```

Report:

```text
HASH_MISMATCH
```

to Thermone Cloud.

---

# Signature Failure

If signature verification fails:

```text
Do not install
```

Report:

```text
SIGNATURE_INVALID
```

This event should be treated as security-sensitive.

---

# Hardware Mismatch

Example:

```text
Device:
THV1 HW 1.0

Firmware:
THV2 HW 2.0
```

Result:

```text
Reject update
```

---

# Downgrade Protection

Production firmware should not normally install an older version automatically.

Example:

```text
Current:
1.2.0

Offered:
1.1.0
```

Result:

```text
Reject
```

unless the manifest explicitly authorizes rollback/downgrade.

---

# Emergency Downgrade

Thermone may need to roll back a bad production release.

The OTA system should support an explicitly authorized rollback version.

Example:

```json
{
  "version": "1.1.5",
  "allow_downgrade": true
}
```

Only trusted server policy should enable this.

---

# Mandatory Updates

Some releases may be marked:

```text
mandatory = true
```

Examples:

- Critical security fix
- Broken cloud compatibility
- Dangerous firmware bug

The final user-facing behavior depends on product policy.

For a headless monitoring device, mandatory updates may be installed automatically within a safe maintenance window.

---

# Optional Updates

Normal feature updates may use:

```text
mandatory = false
```

The device may still auto-install based on Thermone policy.

---

# V1 Update Policy

Recommended initial V1 behavior:

```text
Production firmware updates automatically
when approved by Thermone.
```

The customer does not need to manually approve routine device firmware updates.

---

# Release Rollout

Do not immediately update every production device.

Use staged rollout.

Example:

```text
1%
 ↓
5%
 ↓
25%
 ↓
50%
 ↓
100%
```

Pause between stages and evaluate health.

---

# Canary Devices

Thermone should maintain internal production-like canary hardware.

Example:

```text
Production Channel
Canary Group
```

These units receive the release before general customer devices.

---

# Rollout Targeting

Possible targeting dimensions:

- Serial range
- Device percentage
- Hardware revision
- Geography
- Internal test group
- Specific device IDs

---

# Rollout Health Metrics

Monitor:

- OTA success rate
- Reboot rate
- Rollback rate
- API errors
- Device offline rate
- Sensor read failures
- Memory usage
- Watchdog resets

---

# Automatic Rollout Pause

If failure rate exceeds a threshold:

```text
Pause firmware rollout
```

Example:

```text
OTA rollback rate > 2%
```

may trigger review.

Exact thresholds are determined later.

---

# New Firmware Boot Validation

After rebooting into new firmware, the device should validate basic health before confirming the image.

Checks may include:

- App started successfully
- Storage initialized
- Factory identity readable
- Runtime identity readable
- Sensor subsystem initialized
- Network stack initialized
- Watchdog operating

---

# Validation Window

The new image should not be considered fully successful immediately.

Example conceptual window:

```text
30–120 seconds
```

During this period, firmware proves basic stability.

---

# Confirming Firmware

After successful validation:

```text
Mark OTA image valid
```

This prevents bootloader rollback.

---

# Failed Validation

If the firmware:

- Crashes
- Fails startup
- Resets repeatedly
- Cannot initialize essential components

then it should not be confirmed.

The bootloader can roll back to the previous slot.

---

# Rollback Flow

```text
Install 1.1.0
     │
     ▼
Boot 1.1.0
     │
     X
Health Check Failure
     │
     ▼
Reboot
     │
     ▼
Boot Previous 1.0.0
```

---

# Rollback Reporting

After restoring connectivity, the device reports:

```json
{
  "previous_version": "1.0.0",
  "attempted_version": "1.1.0",
  "status": "rolled_back",
  "reason": "startup_validation_failed"
}
```

---

# OTA Result Endpoint

Example:

```http
POST /v1/device/firmware/result
```

---

# Successful OTA Report

```json
{
  "release_id": "fwrel_019c...",
  "previous_version": "1.0.0",
  "new_version": "1.1.0",
  "status": "success"
}
```

---

# Failed OTA Report

```json
{
  "release_id": "fwrel_019c...",
  "previous_version": "1.0.0",
  "attempted_version": "1.1.0",
  "status": "failed",
  "error_code": "DOWNLOAD_TIMEOUT"
}
```

---

# OTA Error Codes

Initial examples:

```text
MANIFEST_INVALID
HARDWARE_MISMATCH
DOWNLOAD_TIMEOUT
DOWNLOAD_FAILED
URL_EXPIRED
HASH_MISMATCH
SIGNATURE_INVALID
INSUFFICIENT_SPACE
FLASH_WRITE_FAILED
BOOT_FAILED
VALIDATION_FAILED
ROLLED_BACK
UNKNOWN_ERROR
```

---

# Retry Strategy

OTA failures should not cause aggressive retry loops.

Example:

```text
First failure
→ retry after 15 minutes

Repeated failure
→ retry after several hours
```

Server configuration may override retry timing.

---

# Failed Firmware Blacklisting

If a device repeatedly fails a particular release:

```text
Do not continuously retry same release
```

The backend may mark:

```text
release incompatible with device
```

or pause updates for that unit.

---

# Firmware Manifest Caching

The device may cache the last checked version and release ID.

This prevents unnecessary repeated downloads after transient reboot conditions.

---

# Firmware Download Resume

V1 does not need download resume if implementation complexity is high.

A failed download may restart from the beginning.

Future versions may support partial resume.

---

# Recovery Firmware

Thermone should retain a recovery path independent of the currently broken application.

Recovery should support:

- Network setup
- Firmware retrieval
- Reinstallation
- Factory reset
- Diagnostics

Detailed behavior is defined in:

```text
firmware/recovery-mode.md
```

---

# Bootstrap Firmware and OTA

The initial bootstrap firmware may install production firmware through OTA.

Flow:

```text
Factory Bootstrap
      │
      ▼
Device Registers
      │
      ▼
Get Production Firmware
      │
      ▼
Install OTA Slot
      │
      ▼
Boot Production Firmware
```

---

# Bootstrap Compatibility

Bootstrap firmware must only install firmware compatible with:

```text
Model
Hardware Revision
Flash Layout
```

---

# OTA and Device Identity

Firmware updates must not erase:

```text
Factory serial
Factory identity
Runtime device credential
Wi-Fi configuration
```

unless an explicit migration requires it.

---

# Configuration Migration

A new firmware release may require new configuration format.

Migrations must be:

- Versioned
- Backward-aware
- Recoverable where possible

Example:

```text
config schema 2
→ config schema 3
```

---

# Migration Failure

If required configuration migration fails:

```text
Do not silently destroy config
```

Potential responses:

- Restore backup
- Enter recovery
- Roll back firmware

---

# OTA During Internet Outage

No update is attempted while cloud access is unavailable.

Normal sensor monitoring continues.

---

# OTA During Sensor Alert

A critical temperature alert may be a reason to postpone a non-critical OTA update.

Recommended future logic:

```text
Critical active condition?
      │
      ├── YES → defer optional OTA
      │
      └── NO → continue
```

This prevents avoidable monitoring interruptions during an active aquarium problem.

---

# Scheduled Updates

V1 may update when available without a specific clock schedule.

Future versions may support:

```text
maintenance windows
```

for facilities with many controllers.

---

# Device Groups

Future rollout groups may include:

```text
internal
beta
production-canary
production-general
```

---

# GitHub Release Process

Firmware source lives in:

```text
Thermone/firmware
```

A release tag may look like:

```text
v1.1.0
```

GitHub Actions can:

```text
Tag Created
    │
    ▼
Compile Firmware
    │
    ▼
Run Tests
    │
    ▼
Calculate SHA-256
    │
    ▼
Sign Firmware
    │
    ▼
Create Artifact
    │
    ▼
Upload Firmware Storage
    │
    ▼
Create Firmware Release Record
```

---

# Release Artifact Names

Recommended convention:

```text
thermone-thv1-hw1.0-v1.1.0.bin
```

Manifest:

```text
thermone-thv1-hw1.0-v1.1.0.json
```

Hash:

```text
thermone-thv1-hw1.0-v1.1.0.sha256
```

---

# Immutable Release Artifacts

Once a production version is published:

```text
v1.1.0
```

its binary should never be silently replaced.

If code changes:

```text
create v1.1.1
```

instead.

---

# Firmware Release Record

Conceptual database record:

```text
firmware_releases

id
model
hardware_revision
version
channel
sha256
signature
storage_path
size_bytes
mandatory
rollout_percentage
status
created_at
published_at
```

---

# Release Status

Possible states:

```text
draft
testing
staging
approved
rolling_out
paused
complete
revoked
```

---

# Revoked Firmware

A known-bad release may be:

```text
revoked
```

Devices that have not installed it should no longer receive it.

Devices currently running it may be offered an emergency rollback or newer fixed version.

---

# Firmware Status in Dashboard

Users may see:

```text
Firmware:
1.1.0

Status:
Up to date
```

or:

```text
Firmware update in progress
```

The dashboard does not need to expose low-level signing metadata.

---

# Admin Firmware View

Internal Thermone tooling should show:

- Firmware version
- Release channel
- Rollout percentage
- Success rate
- Failure rate
- Rollback rate
- Devices remaining

---

# OTA Security Events

Important events should be logged:

```text
firmware_check
firmware_download_started
firmware_download_failed
firmware_hash_failed
firmware_signature_failed
firmware_installed
firmware_boot_validated
firmware_rollback
```

---

# Firmware Signature Failure Alert

A signature failure should generate a high-priority security event because it may indicate:

- Corruption
- Misconfiguration
- Unexpected firmware
- Attack attempt

---

# OTA Testing

Before production rollout, test:

1. Normal update
2. Wi-Fi loss during download
3. API loss during check
4. Power loss during download
5. Power loss before reboot
6. Invalid hash
7. Invalid signature
8. Incompatible hardware
9. Broken new firmware
10. Successful rollback
11. Repeated failed update
12. Recovery-mode reinstall

---

# Power-Loss Test

Important test:

```text
Start OTA
   │
   ▼
Interrupt power
   │
   ▼
Restore power
```

Expected:

```text
Device boots known-good firmware
```

It must not become permanently unusable.

---

# Update With 8 Active Sensors

Test OTA while:

```text
A01-A08 connected
```

Verify:

- No corrupted sensor configuration
- Correct port map after reboot
- Sensor IDs preserved
- Cloud reconnect succeeds

---

# OTA Success Criteria

Thermone OTA is ready for V1 when:

1. Device can check for updates.
2. API only offers compatible firmware.
3. Firmware is downloaded over HTTPS.
4. Download integrity is verified.
5. Production firmware authenticity can be verified.
6. Update writes to inactive partition.
7. Existing firmware remains bootable during download.
8. Device boots new firmware.
9. New firmware validates itself.
10. Failed firmware rolls back.
11. OTA result is reported.
12. Interrupted downloads do not brick device.
13. Rollout can be paused.
14. Known-bad firmware can be revoked.
15. Production releases can be staged gradually.

---

# Core OTA Principle

Thermone firmware updates should always follow:

```text
Download New
    │
    ▼
Verify New
    │
    ▼
Keep Old
    │
    ▼
Boot New
    │
    ▼
Prove New Works
    │
    ├── YES → Keep New
    │
    └── NO  → Return to Old
```

No OTA update should make a functioning Thermone controller permanently unusable because a single firmware release failed.