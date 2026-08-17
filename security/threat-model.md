# Thermone V1 Threat Model

## Purpose

This document defines the primary security threats, trust boundaries, protected assets, attack scenarios, and security requirements for Thermone V1.

The goal is not to make Thermone impossible to attack.

The goal is to:

- Identify realistic threats
- Reduce attack surface
- Protect device identity
- Protect user accounts
- Protect device credentials
- Protect firmware integrity
- Prevent unauthorized claiming
- Prevent unauthorized device control
- Preserve monitoring availability
- Limit damage if one device is compromised

---

# System Overview

Thermone includes several major components:

```text
Physical Thermone Controller
        │
        ▼
Customer Network
        │
        ▼
Public Internet
        │
        ▼
Thermone Cloud
        │
        ├── Device API
        ├── Dashboard API
        ├── Database
        ├── Firmware Service
        └── Provisioning Services
                │
                ▼
         Thermone Dashboard
                │
                ▼
               User
```

Additional trusted systems include:

```text
Factory Provisioning Workstation
GitHub / CI
Cloud Hosting
Supabase
Firmware Signing Infrastructure
```

---

# Security Objectives

Thermone should protect:

1. Device identity
2. User account access
3. Device ownership
4. Device runtime credentials
5. Factory credentials
6. Claim tokens
7. Wi-Fi credentials
8. Firmware authenticity
9. Temperature data integrity
10. Customer privacy
11. Device availability
12. Cloud availability

---

# Protected Assets

## Factory Credential

Purpose:

```text
Prove factory-provisioned device identity
```

Impact if compromised:

```text
Potential unauthorized registration or recovery
```

Protection level:

```text
HIGH
```

---

## Runtime Device Credential

Purpose:

```text
Authenticate normal ESP32 API requests
```

Impact if compromised:

- Fake telemetry
- Unauthorized device API access
- Device impersonation

Protection level:

```text
HIGH
```

---

## Claim Token

Purpose:

```text
Associate physical controller with user account
```

Impact if compromised before claim:

```text
Potential unauthorized device claiming
```

Protection level:

```text
HIGH
```

---

## User Authentication Tokens

Purpose:

```text
Authorize dashboard access
```

Impact if compromised:

- Account takeover
- Device access
- Tank data access
- Configuration changes

Protection level:

```text
HIGH
```

---

## Wi-Fi Password

Purpose:

```text
Connect controller to customer's network
```

Impact if compromised:

```text
Customer network compromise risk
```

Protection level:

```text
HIGH
```

Thermone Cloud should not require this password.

---

## Firmware Signing Private Key

Purpose:

```text
Authorize Thermone firmware
```

Impact if compromised:

```text
Potential fleet-wide malicious firmware deployment
```

Protection level:

```text
CRITICAL
```

---

## Supabase / Database Service Credentials

Impact if compromised:

- Database modification
- Customer data exposure
- Device ownership manipulation
- Credential theft

Protection level:

```text
CRITICAL
```

---

# Trust Boundaries

Thermone has several major trust boundaries.

```text
User Browser
     │
     │ HTTPS
     ▼
Thermone Cloud
```

```text
ESP32
  │
  │ HTTPS
  ▼
Thermone Cloud
```

```text
Phone
  │
  │ Local Setup Wi-Fi
  ▼
ESP32 Setup Portal
```

```text
Factory Workstation
      │
      ▼
Physical ESP32
```

```text
CI / Release System
      │
      ▼
Firmware Storage
```

Every trust boundary requires explicit security controls.

---

# Threat Actors

Potential attackers include:

## Remote Internet Attacker

Has:

```text
Internet access
```

Does not have physical access to Thermone.

---

## Nearby Network Attacker

Has access to:

```text
Customer LAN
```

or is physically nearby during setup.

---

## Physical Attacker

Has temporary or permanent physical access to the controller.

---

## Malicious Customer

Owns a legitimate Thermone device and attempts to attack:

- Other customers
- Thermone API
- Firmware infrastructure
- Device authentication

---

## Compromised User Account

An attacker obtains a customer's account session.

---

## Compromised Device

An individual ESP32 credential or firmware is extracted.

---

## Compromised Factory Workstation

Provisioning tooling or credentials are stolen.

---

## Compromised Cloud Service

A backend service, secret, or database is compromised.

---

# Threat 1: Guessing Device Serial Numbers

Serial numbers such as:

```text
THV1-000482
```

are predictable.

Therefore they must never provide authorization.

Attack:

```text
Attacker guesses THV1-000483
```

Expected result:

```text
No access
```

Mitigation:

- Serial numbers are identifiers only
- Claiming requires claim token
- Device APIs require device credential
- Dashboard APIs require user authentication

---

# Threat 2: Guessing Claim Tokens

Attack:

```text
Attacker automatically attempts claim URLs
```

Mitigations:

- At least 128 bits of secure randomness
- Prefer 256-bit tokens
- Rate limit claim endpoint
- Store token verifier/hash
- Single-use tokens
- Revoke after successful claim
- Audit failed claim attempts

---

# Threat 3: QR Code Theft Before Claim

Scenario:

An attacker photographs a boxed Thermone QR before the owner claims it.

Potential result:

```text
Attacker could attempt first claim
```

Mitigations:

- Secure packaging
- High-entropy token
- Claim notifications
- Optional future physical confirmation
- Support recovery workflow
- Ability to revoke exposed token before shipment

Future stronger mitigation:

```text
QR token
+
physical button confirmation
```

---

# Threat 4: Reusing a Claim Token

Attack:

```text
Device already claimed
Attacker reuses QR
```

Expected:

```text
Reject
```

Mitigation:

```text
claim token becomes used/revoked after claim
```

---

# Threat 5: Claim Race Condition

Two users submit the same valid claim token simultaneously.

Mitigation:

- Database transaction
- Row locking or atomic conditional update
- Single ownership creation
- Mark claim token used in same transaction

Expected:

```text
One succeeds
One receives DEVICE_ALREADY_CLAIMED
```

---

# Threat 6: Device Impersonation

Attacker learns:

```text
device_id
serial
hardware ID
```

and attempts telemetry submission.

Mitigation:

```text
Unique runtime device token required
```

Public identifiers alone are insufficient.

---

# Threat 7: Runtime Token Theft

If one controller's token is extracted, attacker may impersonate that controller.

Mitigations:

- Unique token per controller
- Revocation
- Rotation
- Secure local storage where practical
- TLS
- No raw token logging
- Detect unusual request patterns

Critical isolation principle:

```text
Compromise of one device must not compromise every device.
```

Never use one shared fleet API key.

---

# Threat 8: Shared Master Device Key

Forbidden architecture:

```text
Every ESP32 contains:
MASTER_API_KEY
```

Impact:

```text
One extracted device compromises entire fleet.
```

Mitigation:

```text
Unique per-device credentials
```

---

# Threat 9: Factory Credential Extraction

Physical attacker extracts factory credential from flash.

Potential impact:

- Registration abuse
- Recovery abuse

Mitigations:

- Unique credential per device
- Restrict factory credential after registration
- Encrypted NVS where practical
- Flash encryption in later hardened builds
- Server-side device state validation
- Hardware ID binding
- Recovery-specific authorization

---

# Threat 10: Wi-Fi Credential Exposure

Wi-Fi password may be exposed through:

- Serial logging
- Cloud upload
- Local setup page
- Memory dumps

Mitigations:

- Never send Wi-Fi password to Thermone Cloud
- Never include password in logs
- Never echo it through setup APIs
- Store locally
- Use encrypted NVS where practical

---

# Threat 11: Setup Access Point Abuse

Attacker near the controller attempts to reconfigure Wi-Fi.

Mitigations:

- Setup AP active only when required
- Device-specific AP password
- Physical button required for manual reconfiguration
- Setup mode timeout
- Disable provisioning endpoints outside setup mode

---

# Threat 12: Open Setup AP

Leaving the Thermone AP permanently open increases risk.

Forbidden:

```text
Thermone-0482
open forever
```

Preferred:

```text
AP only active during provisioning/recovery
```

---

# Threat 13: Malicious Wi-Fi Credentials

An attacker with setup access points Thermone at a malicious network.

Mitigations:

- Require physical setup access
- TLS validation
- Device communicates only with approved Thermone domains/services
- Never disable certificate validation

Even on hostile Wi-Fi:

```text
TLS should protect cloud credentials
```

---

# Threat 14: DNS Spoofing

Attacker controls DNS and redirects:

```text
api.thermone.com
```

Mitigation:

```text
TLS certificate validation
```

The ESP32 must not accept arbitrary certificates.

---

# Threat 15: TLS Disabled

Forbidden production pattern:

```cpp
client.setInsecure();
```

or equivalent insecure certificate bypass.

Production firmware must validate TLS.

Development exceptions must never leak into production builds.

---

# Threat 16: API Replay

Attacker captures a valid telemetry request and replays it.

Potential impact:

```text
Duplicate or fake historical readings
```

Mitigations may include:

- TLS
- Frame IDs
- Timestamps
- Server-side duplicate detection
- Reasonable clock validation

High-risk commands may require stronger anti-replay behavior.

---

# Threat 17: Fake Telemetry

A compromised device submits false temperatures.

This cannot be fully prevented if the device credential and physical unit are compromised.

Mitigations:

- Device-specific credentials
- Plausibility validation
- Audit trail
- Unusual behavior detection
- Ability to revoke device

The backend should never treat a single device as globally trusted.

---

# Threat 18: Invalid Temperature Payload

Attacker or broken firmware submits:

```json
{
  "temperature_c": 999999
}
```

Mitigation:

```text
Server-side schema and range validation
```

Dangerous aquarium temperatures should still be accepted if technically plausible.

---

# Threat 19: API Flooding

A compromised ESP32 sends thousands of telemetry requests per second.

Mitigations:

- Per-device rate limiting
- Payload-size limits
- Timeouts
- Authentication
- Account/device quotas
- Abuse monitoring

---

# Threat 20: Oversized Payload

Attack:

```text
Massive JSON body
```

Mitigations:

- Maximum request body size
- Maximum number of telemetry frames
- Maximum ports per device model
- Schema validation

For THV1:

```text
Maximum physical ports = 8
```

---

# Threat 21: Device Claims Another Device ID

Compromised device token submits:

```json
{
  "device_id": "another-device"
}
```

Mitigation:

```text
Authenticated token determines device identity
```

Payload identity must match the authenticated principal.

---

# Threat 22: Dashboard IDOR

An authenticated user changes:

```text
/devices/dev_123
```

to:

```text
/devices/dev_456
```

trying to view another customer's controller.

Mitigation:

- Server-side authorization
- RLS
- Ownership verification on every request

UUIDs are not authorization.

---

# Threat 23: Cross-Tenant Tank Access

Same principle applies to:

- Tanks
- Locations
- Alerts
- Temperature history
- Device commands

Every resource requires authorization.

---

# Threat 24: Admin Privilege Abuse

Internal support accounts may have powerful access.

Mitigations:

- Least privilege
- Separate roles
- Audit logging
- Stronger admin authentication
- Restricted secret visibility
- No unnecessary access to raw credentials

---

# Threat 25: Service Role Key Leaked to Browser

Forbidden:

```text
Supabase service-role key
inside dashboard JavaScript
```

Impact:

```text
potential unrestricted database access
```

Mitigation:

```text
service-role key only on trusted server infrastructure
```

---

# Threat 26: Service Role Key Embedded in ESP32

Also forbidden.

The ESP32 should use:

```text
device-specific Thermone credential
```

not cloud administrative secrets.

---

# Threat 27: Firmware Tampering in Transit

Attacker modifies firmware download.

Mitigations:

- HTTPS
- SHA-256 verification
- Firmware signatures
- Hardware compatibility validation

---

# Threat 28: Malicious Firmware Manifest

If an attacker compromises the firmware API and changes:

```text
download_url
```

hash verification alone may not protect against an attacker who can also change the hash.

Mitigation:

```text
cryptographically signed firmware
```

The device must verify a trusted signature independently.

---

# Threat 29: Firmware Signing Key Theft

This is one of Thermone's most serious threats.

Impact:

```text
Attacker could potentially sign malicious firmware accepted by devices.
```

Mitigations:

- Highly restricted signing key
- Separate signing system
- Do not store private key in normal repo
- CI secret isolation
- Access auditing
- Consider HSM/KMS later
- Key rotation strategy

---

# Threat 30: Installing Firmware for Wrong Hardware

Example:

```text
THV1 hardware receives THV2 firmware
```

Mitigations:

- Manifest hardware model
- Hardware revision checks
- Firmware metadata validation
- Server-side compatibility rules

---

# Threat 31: OTA Bricking

Faulty firmware prevents boot.

Mitigations:

- A/B OTA
- Previous known-good image retained
- New-image validation
- Automatic rollback
- Recovery mode

---

# Threat 32: Power Loss During OTA

Mitigation:

```text
write inactive partition only
```

Current firmware remains untouched until update is complete and verified.

---

# Threat 33: Repeated Malicious OTA Trigger

Attacker repeatedly tells device to update/reboot.

Mitigations:

- Authenticated commands
- Idempotent command IDs
- Rate limits
- Device state validation
- Server-side authorization

---

# Threat 34: Recovery Mode Abuse

Attacker attempts to use recovery mode to bypass normal authentication.

Mitigations:

- Physical access required
- Recovery does not ignore cloud disable state
- Recovery API has dedicated authentication
- Factory identity validation
- Sensitive operations restricted

---

# Threat 35: Factory Reset as Ownership Bypass

Attacker steals a claimed controller and factory resets it.

Factory reset must not automatically make the cloud device claimable.

Cloud ownership remains authoritative.

```text
Physical reset ≠ ownership removal
```

---

# Threat 36: Stolen Device

Recommended response:

- Owner marks device stolen
- Revoke runtime credential
- Disable transfer
- Disable new claiming
- Preserve history
- Flag serial

---

# Threat 37: Malicious Local Firmware Upload

Future recovery portal may support local firmware uploads.

Risk:

```text
attacker uploads arbitrary firmware
```

Mitigation:

```text
only signed compatible Thermone firmware accepted
```

---

# Threat 38: Debug Interfaces Left Enabled

Production hardware may expose:

- UART
- JTAG
- USB debugging

These can aid physical attacks.

V1 prototype may keep them available for development.

Before scaled production, evaluate:

- Debug-port restrictions
- Flash encryption
- Secure boot
- Physical access controls

---

# Threat 39: Flash Extraction

A determined physical attacker may read ESP32 flash.

Potential targets:

- Runtime credential
- Factory credential
- Wi-Fi password

Future mitigation:

```text
ESP32 Flash Encryption
Encrypted NVS
Secure Boot
```

These should be evaluated before commercial production.

---

# Threat 40: Device Cloning

Attacker copies flash from one controller to another ESP32.

Mitigations:

- Hardware identity binding
- Per-device credentials
- Server detects hardware mismatch
- Future flash encryption tied to device hardware

---

# Threat 41: Hardware ID Spoofing

Hardware ID alone cannot be trusted as authentication.

Use:

```text
hardware ID
+
secret credential
+
server device state
```

---

# Threat 42: Database Compromise

If database access is compromised, attacker may obtain:

- Customer data
- Device metadata
- Token hashes
- Ownership information

Mitigations:

- Least-privileged database accounts
- RLS
- Secure backups
- Encryption at rest from provider
- Secret separation
- Audit logging
- Hash stored credentials

---

# Threat 43: Raw Token Database Storage

Avoid storing secrets as plaintext when a verifier/hash is sufficient.

Preferred:

```text
claim_token_hash
device_token_hash
factory_credential_hash
```

---

# Threat 44: Log Leakage

Logs can accidentally expose secrets.

Do not log:

- Authorization headers
- Claim tokens
- Wi-Fi passwords
- Service keys
- Private signing keys

Use:

```text
[REDACTED]
```

where needed.

---

# Threat 45: Git Credential Leakage

Never commit:

```text
.env
service-role keys
database passwords
device credentials
firmware signing keys
```

Use:

- GitHub Secrets
- Environment variables
- Secret manager

---

# Threat 46: Dependency Compromise

Thermone depends on third-party code.

Potential areas:

- ESP-IDF
- Web framework
- Supabase libraries
- npm packages

Mitigations:

- Lock dependency versions
- Review updates
- Automated vulnerability scanning
- Avoid unnecessary dependencies
- Prefer maintained libraries

---

# Threat 47: CI Pipeline Compromise

CI may have access to:

- Deployment credentials
- Firmware signing processes
- Production infrastructure

Mitigations:

- Protected branches
- Restricted Actions permissions
- Environment approval
- Secret isolation
- Minimal token scopes
- Audit logs

---

# Threat 48: Malicious Pull Request

A PR could attempt to:

- Exfiltrate CI secrets
- Modify firmware signing
- Weaken TLS
- Disable authentication

Mitigations:

- Do not expose production secrets to untrusted PR jobs
- Required reviews
- Protected production environments
- Separate signing step
- Inspect workflow changes carefully

---

# Threat 49: Production Environment Accidentally Uses Development Security

Example:

```text
TLS verification disabled
debug admin route enabled
default credentials
```

Mitigations:

- Environment-specific build checks
- CI validation
- Production hardening checklist
- Fail builds containing insecure flags

---

# Threat 50: Cross-Environment Credential Use

Development token attempts production.

Expected:

```text
Reject
```

Mitigation:

- Separate databases
- Separate secrets
- Environment-specific token namespaces/issuers
- No automatic environment fallback

---

# Threat 51: Customer Account Compromise

Attacker gains dashboard access.

Potential impact:

- View temperatures
- Modify thresholds
- Rename controllers
- Send supported commands

Mitigations:

- Secure authentication
- Session expiry
- Optional MFA/passkeys later
- Recent authentication for destructive operations
- Account activity notifications

---

# Threat 52: Remote Restart Abuse

A compromised user account repeatedly restarts controller.

Mitigations:

- Rate limits
- Audit logs
- Command cooldowns
- Authorization
- Potential recent-auth requirement

---

# Threat 53: Remote Factory Reset

This is high-risk.

Recommended V1:

```text
Do not expose remote factory reset
```

or require very strong confirmation.

Physical reset is safer initially.

---

# Threat 54: Temperature Alert Manipulation

Compromised account changes alert thresholds to suppress warnings.

Mitigations:

- Authorization
- Audit log
- Optional notifications for major alert-rule changes
- Future role separation for organizations

---

# Threat 55: Availability Attack Against Cloud

If Thermone Cloud is unavailable:

```text
remote dashboard unavailable
```

but:

```text
local sensor monitoring must continue
```

This is a major architectural security/resilience requirement.

---

# Threat 56: Internet Outage

Same requirement:

```text
Cloud failure must not stop temperature reading.
```

Mitigations:

- Local monitoring
- Offline buffer
- Local thresholds
- Automatic reconnect

---

# Threat 57: Flash Wear Attack

Bad configuration or firmware causes constant writes.

Mitigations:

- Avoid writing every sensor sample to NVS
- Use bounded filesystem buffering
- Batch writes
- Wear-conscious storage design

---

# Threat 58: Memory Exhaustion

Malformed payload or long-running process exhausts ESP32 memory.

Mitigations:

- Fixed-size buffers
- Bounded queues
- Payload limits
- Avoid uncontrolled dynamic allocation
- Health reporting
- Watchdog

---

# Threat 59: Sensor Bus Fault

A shorted sensor causes hardware/software instability.

Because Thermone uses independent buses:

```text
A03 failure
```

should not prevent:

```text
A01, A02, A04-A08
```

from operating.

---

# Threat 60: Fake Sensor Replacement

A user or attacker swaps probe hardware.

Thermone detects:

```text
same port
new DS18B20 ROM ID
```

This is not necessarily malicious, but should generate an event.

---

# Threat 61: Sensor Identity Spoofing

DS18B20 ROM IDs should not be treated as strong cryptographic identity.

They are useful for:

```text
probe tracking
```

not:

```text
security authentication
```

---

# Threat 62: Unsafe Status From Stale Data

Dashboard shows old temperature as if live.

Mitigation:

- Store timestamps
- Track device online status
- Clearly show "last updated"
- Mark stale readings

This is a safety and trust issue.

---

# Threat 63: Missing Alerts During Cloud Outage

Cloud notification services may be unavailable.

Mitigation:

Thermone firmware should eventually support local threshold evaluation.

Future hardware could also support:

- Local buzzer
- External alarm
- Relay
- Local screen

---

# Threat 64: Unauthorized Firmware Downgrade

Attacker attempts older vulnerable firmware.

Mitigations:

- Version policy
- Anti-rollback
- Explicit authorized downgrade only
- Firmware signatures

---

# Threat 65: Compromised Firmware Storage

Even if storage bucket is compromised, signed firmware verification should prevent arbitrary binaries from being installed.

This is why:

```text
storage access control
```

and:

```text
firmware authenticity
```

must be independent controls.

---

# Security Zones

## Zone 1 — Physical Device

Contains:

- ESP32
- Device credentials
- Wi-Fi credentials
- Firmware
- Local telemetry

---

## Zone 2 — Customer LAN

Untrusted from Thermone's perspective.

The LAN may be hostile.

---

## Zone 3 — Public Internet

Untrusted.

All sensitive traffic requires authenticated encryption.

---

## Zone 4 — Thermone Cloud

Trusted application infrastructure.

Still designed with least privilege.

---

## Zone 5 — Factory / CI Infrastructure

Highly trusted.

Contains capabilities that may create devices or release firmware.

---

# Security Priority Levels

## Critical

Compromise could affect entire fleet.

Examples:

- Firmware signing key
- Production service-role key
- Factory provisioning master access

---

## High

Compromise affects an account or device.

Examples:

- Runtime device token
- Factory credential
- Claim token
- User session
- Wi-Fi password

---

## Medium

Useful metadata but not sufficient for access.

Examples:

- Serial number
- Device ID
- Hardware revision
- Sensor ROM ID

---

# Security Controls Summary

Thermone V1 should implement:

```text
HTTPS
Unique device credentials
Claim tokens
Server-side authorization
RLS
Rate limiting
Token hashing
Credential rotation
Credential revocation
Environment isolation
A/B OTA
Firmware hashing
Firmware signing
Rollback
Secret redaction
Protected CI
Audit logging
Offline operation
Recovery mode
```

---

# Production Hardening

Before commercial-scale production, evaluate enabling:

```text
ESP32 Secure Boot
ESP32 Flash Encryption
Encrypted NVS
Signed Application Images
Restricted Debug Interfaces
```

These may not all be required for the earliest prototype but should be considered before shipping large numbers of customer units.

---

# Security Testing

Required security tests should include:

1. Guess invalid claim tokens
2. Reuse used claim token
3. Claim device from two accounts simultaneously
4. Use wrong user's device ID
5. Use another device's runtime token
6. Use development token against production
7. Send malformed telemetry
8. Send oversized telemetry
9. Flood telemetry endpoint
10. Attempt unauthorized device command
11. Tamper with firmware hash
12. Tamper with firmware binary
13. Present firmware for wrong hardware
14. Disable Internet during OTA
15. Attempt setup portal outside setup mode
16. Attempt cloud access with revoked token
17. Verify logs contain no secrets
18. Factory reset a claimed device
19. Attempt to reclaim without ownership removal
20. Verify stale dashboard data is clearly marked

---

# Security Review Triggers

Perform a security review when Thermone adds:

- New hardware revision
- New authentication method
- New remote control capability
- New firmware-signing process
- Organization/multi-user access
- Local firmware upload
- Relay/heater control
- Payment or subscription system
- Public API
- Third-party integrations

---

# Safety-Critical Future Features

Temperature monitoring is primarily observational.

If Thermone later controls:

```text
heaters
pumps
lighting
water valves
feeders
```

the threat model must be expanded significantly.

Remote control of life-support equipment requires stronger:

- Authentication
- Local fail-safes
- Offline safety logic
- Command validation
- Rate limiting
- Hardware interlocks

---

# V1 Threat Model Success Criteria

Thermone V1 security design is acceptable when:

1. Serial numbers cannot authorize access.
2. Claim tokens are random and single-purpose.
3. Runtime device credentials are unique.
4. One compromised device cannot compromise the fleet.
5. Users cannot access another user's device by changing IDs.
6. Wi-Fi passwords never reach Thermone Cloud.
7. Production uses TLS validation.
8. Service-role credentials stay server-side.
9. Device tokens and claim tokens are not logged.
10. Firmware can be authenticated.
11. Bad OTA updates can roll back.
12. Physical factory reset does not remove cloud ownership.
13. Cloud outages do not stop local temperature monitoring.
14. Environment credentials are isolated.
15. Critical administrative actions are auditable.

---

# Core Threat Model Principle

Thermone should assume:

```text
Networks can be hostile.

Users can make mistakes.

Devices can be stolen.

Individual credentials can leak.

Cloud services can fail.

Firmware can contain bugs.
```

The architecture should therefore ensure that a failure or compromise in one layer does not automatically compromise every other layer.

The most important security boundary is:

```text
One compromised Thermone controller
must remain
one compromised Thermone controller.
```

It must not become a key to the entire Thermone fleet.