# Thermone V1 Device Authentication

## Purpose

This document defines how a Thermone V1 controller proves its identity to Thermone Cloud.

Device authentication must support:

- Factory registration
- Runtime API access
- Credential rotation
- Credential revocation
- Hardware identity binding
- Recovery
- Environment isolation
- Protection against device cloning
- Protection against one compromised controller affecting the entire fleet

The core rule is:

```text
Every Thermone controller has its own unique authentication identity.
```

Thermone must never rely on a single shared fleet-wide device secret.

---

# Authentication Lifecycle

A Thermone controller moves through two primary authentication stages.

```text
FACTORY AUTHENTICATION
        │
        ▼
Initial Device Registration
        │
        ▼
Runtime Credential Issued
        │
        ▼
RUNTIME AUTHENTICATION
```

---

# Device Identity Components

Every controller has several identifiers.

```text
Serial Number
Hardware ID
Factory Credential
Internal Device ID
Runtime Device Credential
```

Each has a different purpose.

---

# Serial Number

Example:

```text
THV1-000482
```

Purpose:

- Human-readable identification
- Manufacturing
- Support
- Inventory

The serial number is not secret.

It must never authenticate a controller by itself.

---

# Hardware ID

The ESP32 has a hardware-derived identity.

Conceptual example:

```text
ESP32-A4CF12B78E31
```

Purpose:

- Bind a factory record to a physical ESP32
- Detect unexpected hardware replacement
- Detect duplicate provisioning
- Improve clone detection

The hardware ID is also not secret.

---

# Internal Device ID

Thermone Cloud creates an internal runtime device ID.

Example:

```text
019c3e92-0000-7000-8000-000000000001
```

Purpose:

- Database relationships
- API identity
- Internal device references

The internal device ID is not an authentication secret.

---

# Factory Credential

Each controller receives a unique factory credential during manufacturing.

Example conceptual form:

```text
fc_prod_<random-secret>
```

Purpose:

```text
Initial registration
Approved recovery
```

The factory credential must not be used for routine telemetry after runtime registration.

---

# Runtime Device Credential

After successful registration, the backend issues a runtime device credential.

Example conceptual form:

```text
dt_prod_<random-secret>
```

Purpose:

- Telemetry
- Heartbeats
- Config sync
- Command polling
- Firmware checks
- Device events

---

# Authentication Separation

Never use:

```text
Serial Number
```

as:

```text
Device Password
```

Never use:

```text
Hardware ID
```

as:

```text
Device Password
```

Never use:

```text
Claim Token
```

as:

```text
Runtime Device Token
```

---

# Factory Authentication Flow

Before runtime registration:

```text
ESP32
 │
 ├── Serial
 ├── Hardware ID
 └── Factory Credential
        │
        ▼
Thermone Device API
        │
        ▼
Factory Record Validation
        │
        ▼
Runtime Device Registration
```

---

# Registration Endpoint

Conceptually:

```http
POST /v1/device/register
```

Authentication:

```http
Authorization: Factory <factory-credential>
```

---

# Registration Request

Example:

```json
{
  "serial_number": "THV1-000482",
  "model": "THV1",
  "hardware_revision": "1.0",
  "hardware_id": "ESP32-A4CF12B78E31",
  "bootstrap_version": "1.0.0"
}
```

---

# Factory Credential Lookup

The backend finds the factory record using a safe lookup field such as:

```text
serial number
```

Then validates the supplied secret against:

```text
factory_credential_hash
```

---

# Factory Credential Comparison

Conceptually:

```text
Provided Factory Credential
          │
          ▼
       Hash
          │
          ▼
Compare With Stored Verifier
          │
     ┌────┴────┐
     │         │
    Match    No Match
     │         │
     ▼         ▼
 Continue    Reject
```

---

# Registration Validation

The server must validate:

1. Factory serial exists.
2. Factory credential is valid.
3. Factory record belongs to current environment.
4. Device is not disabled.
5. Hardware revision is supported.
6. Hardware ID is acceptable.
7. Device registration state permits registration.
8. No conflicting device registration exists.

---

# Hardware Binding

On first valid registration, the backend binds:

```text
Serial Number
     │
     ▼
Hardware ID
```

Example:

```text
THV1-000482
     │
     ▼
ESP32-A4CF12B78E31
```

---

# First Hardware Binding

If the factory record does not yet have a hardware ID:

```text
hardware_id = incoming validated hardware ID
```

This should normally happen during factory provisioning, so production registration should primarily confirm the existing value.

---

# Hardware Mismatch

Example:

Factory record:

```text
ESP32-A4CF12B78E31
```

Incoming device:

```text
ESP32-111111111111
```

Default behavior:

```text
Reject registration
```

Error:

```text
HARDWARE_MISMATCH
```

---

# Legitimate Hardware Replacement

Hardware replacement must use an explicit authorized workflow.

Do not automatically overwrite the existing hardware ID.

A support or factory tool may perform:

```text
hardware replacement authorization
```

before the new ESP32 can bind to the serial.

---

# Clone Detection

A copied flash image may contain:

```text
same serial
same factory credential
same runtime credential
```

but run on another ESP32.

Hardware binding allows Thermone to detect:

```text
credential says THV1-000482

hardware ID says unexpected ESP32
```

and reject or quarantine the request.

---

# Hardware ID Limitation

Hardware identity helps detect cloning, but should not be considered cryptographic proof by itself.

The security model remains:

```text
Secret Credential
+
Expected Hardware Identity
+
Server Device State
```

---

# Runtime Registration

After successful factory authentication, the backend creates or activates the runtime device record.

Example:

```json
{
  "device_id": "019c3e92-...",
  "serial_number": "THV1-000482",
  "status": "registered"
}
```

---

# Runtime Credential Generation

The backend generates a cryptographically secure runtime credential.

Recommended:

```text
256 bits of randomness
```

Conceptual:

```text
dt_prod_Z8P...
```

---

# Runtime Credential Issuance

The raw runtime token should normally be returned only during credential issuance.

Example:

```json
{
  "success": true,
  "device": {
    "device_id": "019c3e92-..."
  },
  "credentials": {
    "device_token": "dt_prod_example-secret"
  }
}
```

The device must store it successfully.

---

# Runtime Token Server Storage

Preferred server model:

```text
device_token_hash
```

rather than plaintext token storage.

---

# Token Record

Conceptual:

```text
device_credentials

id
device_id
token_hash
status
created_at
last_used_at
expires_at
revoked_at
```

---

# Token Status

Possible values:

```text
active
rotation_pending
revoked
expired
compromised
```

---

# Runtime Authentication Request

After registration:

```http
Authorization: Bearer <device-token>
```

---

# Runtime Authentication Flow

```text
Incoming Request
      │
      ▼
Extract Bearer Token
      │
      ▼
Find Token Verifier
      │
      ▼
Validate Token
      │
      ▼
Token Active?
      │
      ├── NO → Reject
      │
      └── YES
            │
            ▼
Authenticated Device
```

---

# Device Principal

Once authentication succeeds, the server should produce an authenticated device principal.

Conceptually:

```text
AuthenticatedDevice {
    device_id
    serial_number
    environment
    model
    hardware_revision
}
```

Route handlers should use this trusted principal.

---

# Do Not Trust Payload Device ID

Bad:

```text
POST telemetry

{
  "device_id": "dev_someone_else"
}
```

and API blindly trusts it.

Good:

```text
Authorization token
      │
      ▼
Authenticated device = dev_123
```

Then:

```text
payload device_id missing
```

or:

```text
payload device_id must equal dev_123
```

---

# Preferred V1 Approach

For authenticated runtime routes, the payload may omit `device_id` entirely.

Example:

```json
{
  "recorded_at": "2026-08-16T23:00:30Z",
  "ports": []
}
```

The API already knows the device from authentication.

This reduces spoofing opportunities and duplicated data.

---

# Endpoint Authorization

Runtime credential may access:

```text
POST /device/telemetry
POST /device/heartbeat
GET  /device/config
GET  /device/commands
POST /device/commands/{id}/ack
GET  /device/firmware/check
POST /device/firmware/result
POST /device/events
```

It must not access:

```text
Admin APIs
Other devices
Customer account APIs
Factory provisioning APIs
```

---

# Per-Device Isolation

A token for:

```text
THV1-000482
```

must not authenticate as:

```text
THV1-000483
```

Every request remains bound to one device.

---

# Environment Binding

Every credential belongs to one environment.

Possible token prefixes:

```text
dt_dev_
dt_stg_
dt_prod_
```

These prefixes are not security controls by themselves, but help avoid mistakes.

---

# Environment Validation

Credential record should contain:

```text
environment
```

Example:

```text
production
```

The production API accepts only:

```text
production credentials
```

---

# Cross-Environment Attempt

Example:

```text
development token
      │
      ▼
production API
```

Expected:

```text
401 Unauthorized
```

or:

```text
403 Forbidden
```

according to final API policy.

---

# Credential Last-Used Tracking

On successful authentication, the backend may update:

```text
last_used_at
```

and optionally:

```text
last_ip
last_firmware_version
```

Do not write so aggressively that every telemetry request causes unnecessary database load.

This metadata may be asynchronously updated.

---

# Authentication Audit Events

Important events:

```text
device_auth_success
device_auth_failure
device_token_revoked
device_token_rotated
hardware_mismatch
factory_auth_failure
credential_recovery_started
credential_recovery_completed
```

---

# Failed Authentication Logging

Safe:

```text
serial=THV1-000482
result=invalid_token
request_id=req_123
```

Do not log:

```text
token=dt_prod_secret
```

---

# Rate Limiting

Runtime authentication should support per-device rate limits.

Examples:

```text
telemetry rate
heartbeat rate
firmware check rate
command polling rate
```

Repeated invalid authentication should also be rate limited.

---

# Credential Rotation

Runtime credentials must be rotatable without physically accessing the controller.

---

# Rotation Flow

```text
Current Token
    │
    ▼
Request Rotation
    │
    ▼
Server Authenticates Current Device
    │
    ▼
Generate Replacement Token
    │
    ▼
Return Replacement
    │
    ▼
Device Stores Replacement
    │
    ▼
Device Confirms
    │
    ▼
Old Token Revoked
```

---

# Why Confirmation Is Required

Do not:

```text
Issue new token
immediately revoke old token
```

because a power loss before the new token is persisted could lock the device out.

---

# Rotation Record

Conceptually:

```text
old credential:
active

new credential:
pending
```

After confirmation:

```text
old:
revoked

new:
active
```

---

# Rotation Endpoint

Possible future endpoint:

```http
POST /v1/device/auth/rotate
```

Response:

```json
{
  "rotation_id": "rot_123",
  "new_device_token": "dt_prod_new-secret"
}
```

---

# Rotation Confirmation

Possible:

```http
POST /v1/device/auth/rotate/{rotation_id}/confirm
```

authenticated using the new token.

---

# Rotation Timeout

If the device never confirms:

```text
new token expires
old token remains active
```

after a defined timeout.

---

# Emergency Token Revocation

Thermone may revoke a runtime credential if:

- Device is stolen
- Token leaked
- Suspicious behavior detected
- Device decommissioned
- Hardware replaced

---

# Revocation Response

A revoked token receives:

```text
401 Unauthorized
```

Firmware should treat repeated 401 responses differently from temporary server errors.

---

# Firmware 401 Behavior

Example:

```text
API returns 401
      │
      ▼
Do not endlessly retry telemetry
      │
      ▼
Enter Authentication Recovery State
```

Local temperature monitoring continues.

---

# Authentication Recovery

A device with invalid runtime credentials may need to recover.

Recovery must not simply issue a new token to anyone who knows the serial number.

---

# Recovery Inputs

Potential recovery identity:

```text
Factory serial
Factory credential
Hardware ID
Recovery state
Server eligibility
```

---

# Credential Recovery Flow

```text
Runtime token rejected
       │
       ▼
Enter recovery auth flow
       │
       ▼
Prove factory credential
       │
       ▼
Verify expected hardware ID
       │
       ▼
Check cloud device status
       │
       ▼
Authorize recovery?
       │
       ├── NO → remain locked
       │
       └── YES
              │
              ▼
       Issue new runtime token
```

---

# Recovery Restrictions

Credential recovery should be rejected when:

```text
device status = stolen
device status = disabled
device status = decommissioned
```

unless an authorized administrative recovery explicitly overrides the restriction.

---

# Replay Considerations

Device bearer tokens are reusable credentials, so TLS is essential.

Future hardening may add request signing.

For V1, secure TLS + high-entropy unique tokens + server validation is acceptable.

---

# Future Request Signing

A stronger future design could use:

```text
device private key
+
request signature
```

instead of opaque bearer tokens.

Example:

```text
Ed25519 device keypair
```

This may reduce risks associated with server-side secret storage.

Not required for the first Thermone V1 prototype.

---

# Future Device Certificates

Future production hardware may use:

```text
mTLS
```

with a unique client certificate per device.

Potential advantages:

- Strong device identity
- Private keys never leave device
- Certificate revocation model

Tradeoffs:

- More provisioning complexity
- Certificate lifecycle management
- More demanding manufacturing process

V1 should not require mTLS initially.

---

# Token vs Certificate Strategy

Initial V1:

```text
Unique opaque device bearer token
```

Future hardened production:

```text
Per-device asymmetric identity
```

The API architecture should allow migration later.

---

# Runtime Credential Storage on ESP32

Initial:

```text
NVS
```

Preferred later:

```text
Encrypted NVS
+
Flash Encryption
```

---

# Token Memory Handling

Firmware should avoid unnecessary copies of the raw token.

Avoid:

```text
printing token
duplicating token across modules
storing token in multiple config files
```

Expose it only to the cloud-auth layer.

---

# Token Interface

Conceptually:

```cpp
class DeviceCredentialStore {
public:
    bool has_runtime_token();
    bool load_runtime_token(...);
    bool save_runtime_token(...);
    bool clear_runtime_token();
};
```

Other modules should not directly access NVS credential keys.

---

# Factory Credential Storage Interface

Separate:

```cpp
class FactoryIdentityStore {
public:
    bool load_serial(...);
    bool load_hardware_revision(...);
    bool load_factory_credential(...);
};
```

Factory identity should be harder to accidentally erase.

---

# Reset Behavior

## Network Reset

Preserves:

```text
runtime token
factory credential
device ID
```

Clears:

```text
Wi-Fi
```

---

## Factory Reset

May clear:

```text
runtime token
Wi-Fi
runtime config
```

Preserves:

```text
factory serial
factory credential
hardware metadata
```

depending on recovery policy.

---

# Cloud Ownership and Device Authentication

These are separate.

Example:

```text
ESP32 authentication
```

answers:

```text
Which physical Thermone controller is this?
```

User ownership answers:

```text
Which account may manage that controller?
```

---

# Ownership Change

A user transfer does not require changing the physical serial number.

Thermone may rotate the runtime token during transfer as a security improvement.

Recommended future behavior:

```text
Ownership transfer completed
      │
      ▼
Rotate runtime device credential
```

---

# Returned / Refurbished Device

For refurbished devices:

```text
Old runtime token revoked
New runtime token issued on next registration/recovery
Old claim token invalid
New claim token created
```

---

# Decommissioned Device

A decommissioned device should have:

```text
all runtime credentials revoked
```

and:

```text
normal registration denied
```

---

# Authentication Availability

If cloud authentication is unavailable:

```text
local temperature monitoring continues
```

The device buffers telemetry and retries.

Authentication failure must not stop sensor reading.

---

# Token Expiration

Initial V1 device tokens may be long-lived and rotatable rather than short-lived.

Reason:

ESP32 devices must survive:

- Internet outages
- Server outages
- Long unattended periods

Future versions may introduce:

```text
short-lived access token
+
long-lived refresh credential
```

if justified.

---

# Long-Lived Token Risk

Long-lived tokens increase exposure if stolen.

Mitigations:

- Unique per device
- Revocable
- Hardware binding
- Secure storage
- Rotation
- TLS
- Abuse monitoring

---

# Firmware Version During Authentication

The API may record:

```text
firmware_version
```

but firmware version is not an authentication factor.

Do not reject a valid token solely because the device is old unless an explicit minimum-version security policy exists.

---

# Minimum Firmware Policy

For a critical security problem, server policy may define:

```text
minimum_allowed_firmware_version
```

A device below the minimum may receive limited API access sufficient only to update itself.

Example:

```text
telemetry denied
firmware update allowed
```

This must be implemented carefully to avoid locking devices out permanently.

---

# Compromised Token Detection

Possible signals:

- Same token used from geographically impossible locations
- Same token used with multiple hardware IDs
- Abnormally high request volume
- Unsupported firmware identity
- Rapidly changing device metadata

These should trigger investigation or automatic restriction.

---

# Simultaneous Hardware IDs

A token appearing from:

```text
hardware ID A
```

and:

```text
hardware ID B
```

is suspicious.

Possible response:

```text
temporarily quarantine credential
```

and generate a security alert.

---

# Device Quarantine State

Possible cloud state:

```text
quarantined
```

A quarantined device may be allowed:

```text
heartbeat
recovery
firmware update
```

but denied:

```text
normal high-risk commands
```

until reviewed.

---

# Authentication Error Codes

Initial examples:

```text
AUTH_REQUIRED
INVALID_DEVICE_TOKEN
DEVICE_TOKEN_REVOKED
DEVICE_TOKEN_EXPIRED
INVALID_FACTORY_CREDENTIAL
HARDWARE_MISMATCH
DEVICE_DISABLED
DEVICE_DECOMMISSIONED
DEVICE_QUARANTINED
ENVIRONMENT_MISMATCH
RECOVERY_NOT_ALLOWED
```

---

# Authentication HTTP Responses

## Missing Authentication

```text
401
```

## Invalid Credential

```text
401
```

## Valid Credential but Disabled Device

```text
403
```

## Valid Credential but Wrong Environment

```text
403
```

depending on final implementation.

---

# Do Not Leak Authentication Detail

Avoid responses such as:

```text
Token was correct but hardware ID was wrong by one character
```

Prefer:

```json
{
  "success": false,
  "error": {
    "code": "DEVICE_AUTH_FAILED",
    "message": "Device authentication failed."
  }
}
```

Detailed reason may be stored internally.

---

# Database Model

Conceptual runtime credential table:

```text
device_credentials

id
device_id
token_hash
status
created_at
last_used_at
expires_at
revoked_at
rotation_parent_id
```

---

# Factory Credential Table

```text
factory_credentials

id
factory_device_id
credential_hash
status
created_at
first_used_at
last_used_at
revoked_at
```

---

# Hardware Binding Table

Could be stored directly on the device record or historically.

Conceptual:

```text
device_hardware_bindings

id
device_id
hardware_id
hardware_revision
active
bound_at
unbound_at
reason
```

This preserves replacement history.

---

# Device Auth Audit Record

Conceptual:

```text
device_auth_events

id
device_id
event_type
request_id
hardware_id
firmware_version
created_at
metadata
```

Do not store the raw credential.

---

# Factory Registration Idempotency

Registration requests may be repeated.

Example:

```text
registration succeeded
response lost
device retries
```

The backend should recognize:

```text
same factory device
same hardware
already registered
```

and safely return the existing runtime identity or initiate a secure credential recovery flow.

---

# Runtime Token Reissuance During Retry

Be careful not to generate a new runtime token on every duplicate registration attempt.

Otherwise a lost response could invalidate the token the device already stored.

Use explicit issuance state.

---

# Registration State Machine

Possible states:

```text
factory_only
registration_pending
registered
credential_issued
active
recovery_required
```

---

# Authentication State on Device

Firmware may track:

```text
AUTH_FACTORY
AUTH_RUNTIME
AUTH_RECOVERY
AUTH_FAILED
```

---

# Device Startup Auth Flow

```text
Boot
 │
 ▼
Runtime Token Exists?
 │
 ├── YES
 │     │
 │     ▼
 │ Try Runtime Auth
 │
 └── NO
       │
       ▼
Factory Identity Available?
       │
       ├── NO → Recovery Error
       │
       └── YES
              │
              ▼
        Register / Recover
```

---

# Runtime Token Validation Test

At startup, the firmware does not necessarily need a dedicated auth endpoint.

The first heartbeat or configuration request can establish whether the token is still valid.

---

# Authentication Health

The device should distinguish:

```text
network offline
```

from:

```text
cloud offline
```

from:

```text
authentication rejected
```

These are different problems and require different recovery actions.

---

# Example Device States

```text
NETWORK_ERROR
```

→ retry network.

```text
CLOUD_ERROR
```

→ retry cloud.

```text
AUTH_ERROR
```

→ credential recovery.

---

# Security Rule: Fail Closed for Cloud Access

If authentication is uncertain:

```text
do not upload as another device
do not accept unauthorized commands
```

But continue:

```text
local temperature monitoring
```

---

# Production Hardening

Before large-scale commercial deployment, evaluate:

- ESP32 Secure Boot
- Flash Encryption
- Encrypted NVS
- Per-device asymmetric keys
- Device certificates
- Hardware-backed secret storage
- Restricted debug interfaces

---

# Migration to Asymmetric Device Authentication

The V1 database should avoid assumptions that runtime authentication will always be an opaque bearer token.

Future identity record may support:

```text
credential_type = bearer_token
```

or:

```text
credential_type = public_key
```

This makes future migration easier.

---

# Device Authentication Testing

Required tests:

1. Valid factory registration.
2. Invalid factory credential.
3. Valid factory credential with wrong hardware ID.
4. Duplicate registration retry.
5. Valid runtime token.
6. Invalid runtime token.
7. Revoked runtime token.
8. Token from another device.
9. Development token against production.
10. Runtime token with altered payload device ID.
11. Hardware clone attempt.
12. Credential rotation.
13. Power failure during credential rotation.
14. Recovery after token revocation.
15. Disabled device attempts recovery.
16. Decommissioned device attempts authentication.
17. Token does not appear in logs.
18. Factory credential does not appear in logs.

---

# Device Authentication Success Criteria

Thermone V1 device authentication is ready when:

1. Every physical controller receives a unique factory credential.
2. Every registered controller receives a unique runtime credential.
3. Serial number alone cannot authenticate.
4. Hardware ID alone cannot authenticate.
5. Claim token cannot authenticate Device API calls.
6. Runtime token cannot access another device.
7. Hardware identity can be bound to the device.
8. Unexpected hardware changes are detected.
9. Runtime credentials can be revoked.
10. Runtime credentials can be rotated without bricking the device.
11. Registration retries are safe.
12. Authentication failures do not stop local monitoring.
13. Recovery does not bypass disabled-device policy.
14. Development credentials fail in production.
15. Raw credentials are not logged.
16. One compromised device credential does not compromise the fleet.

---

# Core Device Authentication Principle

Thermone should authenticate a controller using:

```text
Unique Secret Credential
        +
Expected Device Record
        +
Expected Hardware Context
```

not:

```text
Serial number
MAC address
QR code
Sensor ID
```

alone.

The most important fleet-security rule remains:

```text
One Thermone device
=
One independent credential boundary.
```

A credential stolen from one controller must not grant access to any other controller.