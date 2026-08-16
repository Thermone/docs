# Thermone V1 Authentication and Authorization

## Purpose

This document defines authentication and authorization for Thermone V1.

Thermone has several different actors:

- Human users
- Factory provisioning tools
- Physical Thermone controllers
- Backend services
- Administrators

Each actor must use the correct credential type.

The core rule is:

```text
User credentials
Device credentials
Factory credentials
Claim tokens
Service credentials
```

must remain separate.

No credential type should be reused for another purpose.

---

# Authentication Domains

Thermone V1 uses four primary authentication domains.

```text
1. User Authentication
2. Device Runtime Authentication
3. Factory Authentication
4. Device Claiming
```

These systems may interact, but they do not share credentials.

---

# High-Level Authentication Model

```text
                     THERMONE CLOUD
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
     USER               DEVICE              FACTORY
       │                   │                   │
       ▼                   ▼                   ▼
 User JWT            Device Token      Factory Credential
       │                   │                   │
       ▼                   ▼                   ▼
Dashboard API         Device API       Registration API


                  DEVICE CLAIMING

QR Claim Token
      │
      ▼
Authenticated User
      │
      ▼
Claim Endpoint
      │
      ▼
Device Ownership
```

---

# 1. User Authentication

Human users authenticate through Thermone's dashboard authentication system.

Initial implementation may use:

```text
Supabase Auth
```

Possible login methods:

- Email and password
- Magic link
- Google
- Apple

---

# User Session Token

After authentication, the user receives a session token.

Conceptually:

```text
JWT
```

The dashboard sends the token with requests:

```http
Authorization: Bearer <user-jwt>
```

---

# User JWT Purpose

A user JWT identifies:

```text
Who is making this dashboard request?
```

It may be used for:

- Dashboard API access
- Locations
- Devices
- Tanks
- Alerts
- Account preferences
- Device claiming
- Device management

It must never be used as an ESP32 device credential.

---

# User Authentication Flow

```text
User
 │
 ▼
Thermone Dashboard
 │
 ▼
Supabase Auth
 │
 ▼
Credentials Valid?
 │
 ├── NO → Reject
 │
 └── YES
       │
       ▼
     JWT
       │
       ▼
Dashboard API
```

---

# User Password Storage

Thermone must never store plaintext user passwords.

If Supabase Auth is used, password storage and hashing are handled by the authentication provider.

Thermone application code should never log:

- Passwords
- Password reset tokens
- Magic-link tokens
- User JWTs unless securely redacted

---

# User Session Expiration

User sessions should expire according to the authentication provider's session policy.

Refresh tokens may be used to maintain a session.

The dashboard should automatically refresh valid sessions when appropriate.

---

# Recent Authentication

Sensitive account actions may require recent authentication.

Examples:

- Removing a controller
- Transferring ownership
- Changing email
- Changing password
- Deleting an account
- Performing a high-risk device action

---

# 2. Authorization

Authentication establishes identity.

Authorization determines access.

Example:

```text
Authenticated user:
user_123
```

attempts:

```text
GET /v1/devices/dev_999
```

Thermone must verify:

```text
Does user_123 have access to dev_999?
```

before returning device information.

---

# Authorization Rule

Never assume that knowing a resource ID grants permission.

Example:

```text
tank_123
```

is not secret.

The API must verify ownership or membership for every protected resource.

---

# Resource Ownership

Initial V1 ownership may be:

```text
User
  │
  └── Device
       │
       └── Tank
```

Future architecture may use:

```text
Organization
     │
     ├── Owner
     ├── Admin
     ├── Member
     └── Viewer
```

---

# Server-Side Authorization

Authorization must happen on the server.

The frontend hiding a button is not security.

Bad:

```text
Frontend hides "Delete Device"
```

Good:

```text
API checks permission before deletion
```

---

# Row Level Security

If Supabase is used, Row Level Security should reinforce API authorization.

Users should only be able to access authorized rows.

Examples include:

- Their locations
- Their devices
- Their tanks
- Their alerts
- Their preferences

---

# 3. Factory Authentication

Factory authentication is used before a controller has received its permanent runtime device credential.

Each manufactured controller receives a unique factory credential.

---

# Factory Credential Purpose

The factory credential allows bootstrap firmware to prove:

```text
I am the factory-provisioned unit associated with this serial number.
```

It is used for:

```http
POST /v1/device/register
```

It should not normally be used after registration.

---

# Factory Credential Properties

A factory credential must be:

- Unique per device
- Cryptographically random
- High entropy
- Non-guessable
- Revocable
- Environment-specific
- Hidden from customers
- Hidden from QR codes
- Stored securely where practical

---

# Example Factory Identity

```text
Serial:
THV1-000482

Factory Credential:
random secret
```

The serial number is public.

The factory credential is secret.

---

# Factory Credential Storage

The controller stores the factory credential during manufacturing.

Possible future secure storage methods include:

- ESP32 NVS
- Encrypted NVS
- Secure flash
- Hardware-backed key storage on future hardware

The exact mechanism is defined in firmware security documentation.

---

# Factory Credential Server Storage

The server should avoid storing raw credentials when not necessary.

Preferred conceptual model:

```text
factory_credential_hash
```

rather than:

```text
factory_credential_plaintext
```

The provisioning system may briefly handle the raw credential while flashing the device.

---

# Factory Registration Request

Conceptual example:

```http
POST /v1/device/register
Authorization: Factory <factory-credential>
Content-Type: application/json
```

Body:

```json
{
  "serial_number": "THV1-000482",
  "hardware_id": "ESP32-A4CF12B78E31",
  "hardware_revision": "1.0",
  "bootstrap_version": "1.0.0"
}
```

---

# Factory Credential Validation

The server validates:

- Serial exists
- Credential matches serial
- Device has not been disabled
- Environment matches
- Hardware registration is valid
- Device state allows registration

---

# Factory Credential Lifecycle

Recommended lifecycle:

```text
Generated
   │
   ▼
Flashed to Device
   │
   ▼
Used for Registration
   │
   ▼
Runtime Credential Issued
   │
   ▼
Factory Credential Restricted
```

The factory credential may remain available only for specific recovery workflows.

Routine telemetry must not use it.

---

# 4. Runtime Device Authentication

After successful registration, Thermone issues a runtime device credential.

This credential authenticates a physical controller during normal operation.

---

# Runtime Device Credential Purpose

Used for:

- Telemetry uploads
- Heartbeats
- Configuration retrieval
- Command polling
- Firmware checks
- Event reporting
- Credential rotation

---

# Runtime Device Request

Example:

```http
Authorization: Bearer <device-token>
```

---

# Runtime Device Token Properties

The device token must be:

- Unique per device
- High entropy
- Revocable
- Rotatable
- Environment-specific
- Never shared between controllers

---

# Device Token Example

Conceptual only:

```text
dt_prod_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

The actual token format may use a random opaque value.

---

# Device Token Storage on Server

The preferred model is to store only a secure verifier or hash of the raw device token where possible.

Example:

```text
device_token_hash
```

The raw token is issued to the controller once.

---

# Device Token Storage on Device

The controller stores:

```text
device_id
device_token
```

in persistent storage.

The token must not appear in:

- Serial logs
- Normal telemetry
- User-facing dashboard responses
- QR codes

---

# Device Token Validation

Each runtime request should validate:

1. Token exists
2. Token is valid
3. Token is active
4. Token belongs to the requesting device
5. Device is not disabled
6. Token belongs to current environment

---

# Device Identity and Token Binding

A device request may include:

```text
device_id
```

but the API should derive or validate device identity from authentication.

The server must not trust:

```json
{
  "device_id": "someone-elses-device"
}
```

simply because the payload says so.

---

# Recommended Rule

The device token determines the authenticated device.

Then the API verifies that payload identifiers match.

Conceptually:

```text
Device token
    │
    ▼
Authenticated Device
    │
    ▼
dev_123

Payload claims:
dev_123

Match?
 YES
```

---

# 5. Device Claim Token

A claim token is completely separate from both factory and runtime device authentication.

The claim token exists only to link a physical product with a user account.

---

# Claim Token Example

QR contains:

```text
https://app.thermone.com/claim/<claim-token>
```

Example conceptual token:

```text
c5f9c9f0c06d4efebceab2b3a4...
```

---

# Claim Token Properties

Claim tokens should be:

- Cryptographically random
- High entropy
- Hard to guess
- Environment-specific
- Revocable
- Single-purpose
- Invalidated or rotated after successful claim

---

# Claim Token Is Not Device Authentication

Never use:

```text
claim token
```

to authenticate:

```text
telemetry
heartbeats
firmware
configuration
```

The claim token exists only for account ownership workflows.

---

# Claim Flow

```text
User Scans QR
      │
      ▼
Thermone Dashboard
      │
      ▼
User Authenticates
      │
      ▼
POST /devices/claim
      │
      ▼
Validate Claim Token
      │
      ▼
Validate Device State
      │
      ▼
Assign Ownership
```

---

# Claim Token Storage

The backend should store a secure hash or equivalent verifier rather than the raw claim secret whenever practical.

Example:

```text
claim_token_hash
```

---

# Claim Token After Successful Claim

Recommended behavior:

```text
claim_status = used
```

or:

```text
claim_token revoked
```

A used claim token should not allow another account to claim the device.

---

# Ownership Transfer

If a device is later transferred:

```text
Old claim token
    │
    ▼
Remain invalid
```

Thermone generates:

```text
New claim token
```

for the transfer process.

---

# QR Code Security

The QR code itself may be physically visible.

Therefore it must never contain:

- Factory credential
- Device runtime token
- Wi-Fi password
- Private key
- Admin token

It should contain only the claim URL/token needed for the ownership workflow.

---

# 6. Service-to-Service Authentication

Thermone backend services may need to communicate with:

- Supabase
- Firmware storage
- Email provider
- Alert service
- Internal administrative services

These use service credentials.

Service credentials are separate from user and device credentials.

---

# Service Credentials

Examples:

```text
SUPABASE_SERVICE_ROLE_KEY
FIRMWARE_STORAGE_SECRET
EMAIL_PROVIDER_API_KEY
```

These must only exist server-side.

They must never be:

- Embedded in dashboard JavaScript
- Embedded in ESP32 firmware unless specifically safe for that use
- Printed in logs
- Sent to users

---

# GitHub Secrets

Deployment credentials should be stored using:

```text
GitHub Actions Secrets
```

or an appropriate secret manager.

Never commit:

```text
.env
service keys
database passwords
signing private keys
device secrets
```

---

# 7. Administrator Authentication

Thermone administrator access is distinct from normal user access.

Possible roles:

```text
support
operations
administrator
security
```

Administrator permissions should follow least privilege.

---

# Admin Actions

Examples:

- View device health
- Disable compromised device
- Review registration problems
- Investigate OTA failure
- Assist ownership transfer
- Revoke credentials

Sensitive actions must be audit logged.

---

# Support Access

Support staff should not automatically receive unrestricted access to:

- User credentials
- Device tokens
- Wi-Fi passwords
- Private keys

Support tools should expose only what is required.

---

# 8. Authentication by Endpoint

## Device Registration

```http
POST /v1/device/register
```

Credential:

```text
Factory Credential
```

---

## Device Telemetry

```http
POST /v1/device/telemetry
```

Credential:

```text
Runtime Device Token
```

---

## Device Heartbeat

```http
POST /v1/device/heartbeat
```

Credential:

```text
Runtime Device Token
```

---

## Device Configuration

```http
GET /v1/device/config
```

Credential:

```text
Runtime Device Token
```

---

## Device Commands

```http
GET /v1/device/commands
```

Credential:

```text
Runtime Device Token
```

---

## Firmware Check

```http
GET /v1/device/firmware/check
```

Credential:

```text
Runtime Device Token
```

---

## Dashboard APIs

Example:

```http
GET /v1/devices
```

Credential:

```text
User JWT
```

---

## Device Claiming

```http
POST /v1/devices/claim
```

Requires:

```text
User JWT
+
Claim Token
```

---

# Credential Matrix

| Action | User JWT | Claim Token | Factory Credential | Device Token |
|---|---:|---:|---:|---:|
| Login | — | — | — | — |
| View dashboard | Yes | No | No | No |
| Claim device | Yes | Yes | No | No |
| Register controller | No | No | Yes | No |
| Upload telemetry | No | No | No | Yes |
| Send heartbeat | No | No | No | Yes |
| Check firmware | No | No | No | Yes |
| Fetch device config | No | No | No | Yes |
| View temperature history | Yes | No | No | No |
| Rename controller | Yes | No | No | No |

---

# 9. Credential Rotation

Runtime device credentials must support rotation.

Recommended flow:

```text
Current Device Token
       │
       ▼
Authenticated Rotation Request
       │
       ▼
Server Generates New Token
       │
       ▼
Device Receives New Token
       │
       ▼
Device Stores New Token
       │
       ▼
Device Confirms New Token
       │
       ▼
Old Token Revoked
```

---

# Rotation Safety

The old token should not be revoked before the device has safely stored the replacement.

Otherwise a power failure during rotation could permanently lock out the controller.

---

# Possible Rotation States

```text
active
rotation_pending
replacement_confirmed
revoked
```

---

# 10. Credential Revocation

Credentials may be revoked for:

- Compromise
- Device theft
- Hardware replacement
- Ownership transfer
- Administrative disable
- Credential rotation

---

# Revoked Token Behavior

A revoked device token must receive:

```text
401 Unauthorized
```

or an appropriate authentication failure.

The controller should not continuously retry the same rejected token forever.

It should enter a credential recovery state.

---

# 11. Credential Recovery

Recovery must be designed carefully.

Possible secure recovery mechanisms include:

- Factory credential
- Physical recovery mode
- Manufacturer support process
- Secure reprovisioning

A compromised runtime token must not itself be sufficient to generate a new unrestricted token.

---

# 12. Environment Isolation

Credentials belong to exactly one environment.

Example:

```text
dt_dev_...
```

must not work against:

```text
api.thermone.com
```

Production tokens must not work against:

```text
api.dev.thermone.com
```

---

# Environment Credential Separation

Each environment has separate:

- Signing keys
- Device credentials
- Factory credentials
- User auth configuration
- Service secrets
- Databases

---

# 13. Token Logging

Raw authentication tokens must never be written to normal logs.

Bad:

```text
Authorization: Bearer dt_prod_secret...
```

Good:

```text
device_id=dev_123
auth_result=success
```

---

# Redaction

If request headers are logged during debugging, sensitive values must be redacted.

Example:

```text
Authorization: Bearer [REDACTED]
```

---

# 14. HTTPS

All production authentication occurs over TLS.

Never send:

- User JWTs
- Device tokens
- Factory credentials
- Claim tokens

over unencrypted public HTTP.

---

# Local Setup Exception

The local ESP32 provisioning portal may initially use:

```text
http://192.168.4.1
```

because communication occurs directly over the temporary local setup network.

The security model for local provisioning is documented separately.

---

# 15. Brute-Force Protection

Endpoints involving secrets should be rate limited.

Especially:

```text
/device/register
/devices/claim
/login
/password reset
```

Repeated failures should trigger:

- Increasing delays
- Rate limits
- Security logging

---

# Claim Token Guessing

Claim tokens must contain enough entropy that guessing them is computationally impractical.

Do not use predictable values such as:

```text
THV1-000482
```

as the claim secret.

---

# 16. Device Serial Numbers

Serial numbers are identifiers, not credentials.

Example:

```text
THV1-000482
```

It is safe for the serial number to appear on:

- Enclosure
- Packaging
- Dashboard
- Support records

Knowing a serial number must not grant access.

---

# 17. Hardware IDs

ESP32 hardware identifiers are also not secrets.

They may help verify hardware identity but should never be the only authentication mechanism.

Bad:

```text
MAC address = password
```

Good:

```text
hardware ID
+
factory credential
```

---

# 18. Password Reset

User password reset should be handled by the authentication provider.

Thermone should not implement its own insecure password-reset token system unless required.

---

# 19. Email Verification

Thermone may require email verification before certain actions.

Possible requirements:

- Device claiming
- Ownership transfer
- Sensitive account changes

Exact policy will be defined during dashboard implementation.

---

# 20. Multi-Factor Authentication

MFA is not required for the first V1 customer experience but the architecture should not prevent it.

Future options may include:

- TOTP
- WebAuthn
- Passkeys

Administrator accounts should eventually require stronger authentication than ordinary user accounts.

---

# 21. Session Security

Dashboard sessions should use:

- Secure token storage
- HTTPS
- Expiration
- Refresh token rotation where supported

Avoid storing sensitive auth tokens in insecure browser locations where a safer framework-managed option exists.

---

# 22. CORS

Dashboard APIs should restrict browser origins.

Production examples:

```text
https://app.thermone.com
```

Development may allow:

```text
http://localhost:3000
```

Do not use unrestricted production CORS unless intentionally required.

---

# 23. CSRF

If cookie-based authentication is used for any part of the dashboard, CSRF protections must be evaluated.

Bearer-token APIs and cookie-based sessions have different security requirements.

The final frontend authentication pattern must be documented during implementation.

---

# 24. API Keys in Firmware

Do not place global privileged API keys in every controller.

Bad:

```text
Every Thermone contains the same master API key
```

One leaked device would compromise the entire fleet.

Good:

```text
Every controller has its own unique credential
```

---

# 25. Service Role Key

If Supabase is used, the service-role key must never be embedded in:

- ESP32 firmware
- Browser bundle
- QR code
- Mobile application client

It belongs only on trusted server infrastructure.

---

# 26. OTA Authentication

Firmware download authorization is separate from firmware authenticity.

The device may receive a signed temporary download URL.

However, firmware must also be verified using:

- SHA-256
- Cryptographic signature when implemented
- Hardware revision compatibility

A valid download URL alone must not be considered proof that firmware is safe to install.

---

# 27. Firmware Signing Keys

Firmware signing private keys are highly sensitive.

They must never be:

- Stored in the firmware repository
- Embedded in ESP32 firmware
- Distributed to developers unnecessarily
- Included in CI logs

Devices only need the information required to verify firmware signatures.

---

# 28. Claim Before Registration

A user may scan the QR before the controller has registered.

The claim system may create a pending claim session.

Example:

```text
User JWT
   │
   +
Claim Token
   │
   ▼
Pending Claim
   │
   ▼
Waiting for Controller Registration
```

No device runtime credential needs to be exposed to the user.

---

# 29. Claim After Registration

If the device is already registered:

```text
User
 │
 ▼
Scan QR
 │
 ▼
Authenticate
 │
 ▼
Claim Token Valid
 │
 ▼
Registered Device Found
 │
 ▼
Ownership Assigned
```

---

# 30. User Removal From Device

Removing a controller from an account changes ownership authorization.

It does not automatically mean that the physical controller's runtime device credential must be destroyed.

However, transfer and removal policies may trigger credential rotation for additional security.

---

# 31. Device Theft

If a controller is reported stolen:

Possible actions:

```text
Mark stolen
Revoke runtime token
Disable new claims
Preserve history
Flag serial
```

The system should not delete historical data automatically.

---

# 32. Security Audit Events

Important authentication events should be recorded.

Examples:

```text
user_login_success
user_login_failure
device_registration_success
device_registration_failure
device_claimed
device_removed
device_token_rotated
device_token_revoked
factory_auth_failure
claim_token_failure
admin_device_disable
```

---

# Audit Event Fields

Example:

```json
{
  "event": "device_claimed",
  "user_id": "usr_123",
  "device_id": "dev_456",
  "timestamp": "2026-08-16T23:00:00Z",
  "request_id": "req_789"
}
```

Secrets must not be included.

---

# 33. Recommended Database Credential Records

Conceptual device credentials table:

```text
device_credentials

id
device_id
token_hash
status
created_at
expires_at
last_used_at
revoked_at
```

---

# Factory Credentials

Conceptual:

```text
factory_credentials

id
factory_device_id
credential_hash
status
created_at
used_at
revoked_at
```

---

# Claim Credentials

Conceptual:

```text
device_claim_tokens

id
factory_device_id
token_hash
status
created_at
expires_at
used_at
```

---

# 34. Credential Expiration

Claim tokens should support expiration.

Factory and runtime credential expiration policies may differ.

Runtime tokens may initially be long-lived but rotatable.

Future production policy should evaluate periodic automatic rotation.

---

# 35. Authentication Failure Response

Example:

```json
{
  "success": false,
  "error": {
    "code": "INVALID_DEVICE_TOKEN",
    "message": "Authentication failed"
  }
}
```

Avoid returning unnecessary detail such as:

```text
Token exists but final 4 characters are wrong
```

which could aid attackers.

---

# 36. Authorization Failure

Use:

```text
403 Forbidden
```

when authentication succeeded but the actor lacks permission.

Example:

```json
{
  "success": false,
  "error": {
    "code": "ACCESS_DENIED",
    "message": "You do not have access to this resource"
  }
}
```

---

# 37. Authentication Failure

Use:

```text
401 Unauthorized
```

for invalid or missing authentication.

---

# 38. Credential Comparison

Secret comparisons should use secure server-side practices.

Do not implement naive custom password or token cryptography.

Use reputable cryptographic libraries and framework functionality.

---

# 39. Random Token Generation

All factory, claim, and runtime credentials must use a cryptographically secure random number generator.

Never use:

```text
Math.random()
timestamp
serial number
incrementing counter
```

for secret generation.

---

# 40. Minimum Secret Entropy

Claim and device tokens should contain enough random entropy to make guessing impractical.

A practical V1 target is at least:

```text
128 bits
```

of cryptographically secure randomness.

Higher values such as:

```text
256 bits
```

are acceptable.

---

# 41. Token Encoding

Random bytes may be encoded as:

```text
base64url
hex
```

Base64url is generally more compact.

The token format should avoid characters that cause problems in URLs.

---

# 42. Authentication Separation Summary

```text
USER

Credential:
User JWT

Purpose:
Dashboard access


FACTORY DEVICE

Credential:
Factory Credential

Purpose:
Initial registration / approved recovery


REGISTERED DEVICE

Credential:
Runtime Device Token

Purpose:
Telemetry and device services


DEVICE CLAIM

Credential:
Claim Token + User JWT

Purpose:
Ownership assignment


BACKEND SERVICE

Credential:
Service Secret

Purpose:
Trusted service communication
```

---

# 43. Forbidden Credential Reuse

Never do:

```text
Claim Token = Device Token
```

Never do:

```text
Serial Number = Factory Password
```

Never do:

```text
One API key for every ESP32
```

Never do:

```text
User JWT accepted as device auth
```

Never do:

```text
Service-role key stored in firmware
```

---

# 44. Initial V1 Authentication Flow

```text
MANUFACTURING

Provisioning Service
      │
      ├── Generate Serial
      ├── Generate Factory Credential
      ├── Generate Claim Token
      └── Generate QR
              │
              ▼
          ESP32 Flashed


FIRST INTERNET CONNECTION

ESP32
 │
 ▼
Factory Credential
 │
 ▼
POST /device/register
 │
 ▼
Runtime Device Token


CUSTOMER CLAIM

User
 │
 ▼
Scan QR
 │
 ▼
User Login
 │
 ▼
User JWT + Claim Token
 │
 ▼
Device Ownership


NORMAL OPERATION

ESP32
 │
 ▼
Runtime Device Token
 │
 ▼
Device API


REMOTE USER

Browser
 │
 ▼
User JWT
 │
 ▼
Dashboard API
```

---

# 45. V1 Authentication Success Criteria

Authentication is ready for V1 when:

1. Users can authenticate securely.
2. User sessions cannot be used as device credentials.
3. Each factory device has a unique credential.
4. Factory registration credentials are validated securely.
5. Registered controllers receive unique runtime credentials.
6. Runtime credentials are revocable.
7. Device credentials can be rotated safely.
8. Claim tokens are separate from API credentials.
9. Used claim tokens cannot claim another account.
10. Device serial numbers do not grant access.
11. Hardware IDs are not treated as secrets.
12. Development credentials fail in production.
13. Raw secrets do not appear in normal logs.
14. Dashboard authorization is enforced server-side.
15. Service-role credentials remain server-side.
16. OTA signing keys remain protected.

---

# Core Authentication Principle

Thermone must always know:

```text
WHO is requesting access?

WHAT credential are they using?

WHAT are they allowed to do?
```

before allowing an operation.

The system should never rely on a serial number, QR code, hidden URL, or device ID alone as proof of authorization.