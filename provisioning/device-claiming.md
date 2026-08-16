# Thermone V1 Device Claiming

## Purpose

This document defines how a Thermone controller becomes associated with a customer account.

Device claiming is separate from:

- Factory provisioning
- Wi-Fi setup
- Device registration
- Runtime device authentication

The claim process uses:

```text
User Authentication
+
Claim Token
```

to establish ownership.

---

# Core Principle

A Thermone device may exist in several states:

```text
Manufactured
Registered
Unclaimed
Claimed
Transferred
Disabled
```

The claim process only manages who owns the device.

It does not provide the ESP32 with its runtime API credential.

---

# High-Level Claim Flow

```text
Factory
   │
   ▼
Generate Claim Token
   │
   ▼
Generate QR Code
   │
   ▼
Print QR on Device
   │
   ▼
Customer Scans QR
   │
   ▼
Thermone Dashboard
   │
   ▼
Login / Create Account
   │
   ▼
Validate Claim Token
   │
   ▼
Device Available?
   │
   ├── NO → Show Appropriate Status
   │
   └── YES
          │
          ▼
      Assign Ownership
          │
          ▼
     Device Appears
     in Dashboard
```

---

# Claim Token

Every Thermone device receives a unique claim token during factory provisioning.

Example conceptual token:

```text
tv3ET5yP6J8l2u7G...
```

The token must be:

- Cryptographically random
- Unique
- High entropy
- Difficult to guess
- Environment-specific
- Revocable
- Single-purpose

---

# Claim URL

The QR code contains a claim URL.

Production:

```text
https://app.thermone.com/claim/<claim-token>
```

Development:

```text
https://app.dev.thermone.com/claim/<claim-token>
```

Staging:

```text
https://app.staging.thermone.com/claim/<claim-token>
```

---

# QR Contents

The QR must not contain:

- Factory credential
- Runtime device credential
- Wi-Fi password
- Private key
- Supabase service-role key
- Firmware signing key

The QR should contain only the claim URL.

---

# Human-Readable Label

The device label should also include:

```text
THERMONE

Model:
THV1

Serial:
THV1-000482

[ QR CODE ]

Scan to set up
```

The serial number is public.

The claim token remains encoded in the QR.

---

# Claim Token Server Storage

The backend should preferably store:

```text
claim_token_hash
```

rather than plaintext claim tokens.

The raw token may exist briefly during:

- Generation
- QR creation
- Label printing

After provisioning, normal internal tools should not need to expose it.

---

# Claim States

Recommended claim states:

```text
available
pending
claimed
transfer_pending
expired
revoked
disabled
```

---

# Factory State

A newly manufactured controller may be:

```text
device_status:
factory_provisioned

claim_status:
available

owner_id:
NULL
```

This means the QR can already be scanned even if the ESP32 has never connected to Thermone Cloud.

---

# Scanning Before First Power-On

The user may scan the QR before plugging the controller in.

Flow:

```text
Scan QR
   │
   ▼
Claim Token Found
   │
   ▼
Factory Device Exists
   │
   ▼
Device Registered?
   │
   └── NO
         │
         ▼
Create Pending Claim Session
```

---

# Pending Claim Screen

Example:

```text
Thermone Controller Found

Serial:
THV1-000482

This controller has not connected yet.

1. Plug in the controller.
2. Connect it to Wi-Fi.
3. Keep this page open.

Waiting for Thermone...
```

---

# Pending Claim Session

The backend may create:

```text
claim_session_id
```

Example:

```text
claim_019c3e92...
```

The session stores:

- User ID
- Factory device ID
- Claim status
- Created time
- Expiration time

---

# Claim Session Endpoint

Conceptual:

```http
GET /v1/devices/claim/{claim_session_id}
```

Possible responses:

```text
waiting_for_device
ready_to_claim
claimed
expired
invalid
```

---

# Device Registers While User Waits

When the physical controller connects:

```text
ESP32
 │
 ▼
Register With Thermone
 │
 ▼
Factory Record
 │
 ▼
Runtime Device Record
```

The claim session can then transition:

```text
waiting_for_device
        │
        ▼
ready_to_claim
```

The dashboard may update automatically.

---

# Scanning After Registration

The simpler case is:

```text
Device Already Online
       │
       ▼
User Scans QR
       │
       ▼
Claim Token Valid
       │
       ▼
Runtime Device Found
       │
       ▼
Ownership Can Be Assigned
```

---

# User Authentication

A claim requires an authenticated Thermone user.

If the customer is not logged in:

```text
Scan QR
   │
   ▼
Login / Create Account
   │
   ▼
Return to Claim Flow
```

The claim token should be preserved safely through authentication.

---

# Claim Endpoint

Conceptual endpoint:

```http
POST /v1/devices/claim
```

Requires:

```text
User JWT
+
Claim Token
```

Example request:

```json
{
  "claim_token": "raw-token-from-qr"
}
```

---

# Claim Validation

The backend must verify:

1. User is authenticated.
2. Claim token exists.
3. Claim token has not expired.
4. Claim token has not been revoked.
5. Device is not disabled.
6. Device is eligible for claiming.
7. Device is not owned by another account.
8. Environment matches.
9. Claim attempt is rate-limited appropriately.

---

# Successful Claim

After validation:

```text
owner_id = authenticated_user
claim_status = claimed
```

Example:

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

# Claim Token Invalidation

After a successful claim, the original token should no longer be usable.

Recommended:

```text
claim_status = claimed
claim_token_used_at = timestamp
```

A second attempt should fail.

---

# Duplicate Claim Attempt

If another account scans the same QR after the device has already been claimed:

```text
This Thermone controller is already associated with an account.
```

The API should not reveal:

- Owner email
- Owner name
- Organization name

unless policy explicitly allows it.

---

# Same User Scans Again

If the current owner scans the QR again, Thermone may show:

```text
This controller is already in your account.

[ Open Controller ]
```

This is better than returning a generic failure.

---

# Claiming and Network Setup

Claiming and network setup are separate.

Valid sequence A:

```text
Scan QR
   │
   ▼
Pending Claim
   │
   ▼
Set Up Wi-Fi
   │
   ▼
Device Registers
   │
   ▼
Claim Completes
```

Valid sequence B:

```text
Set Up Wi-Fi
   │
   ▼
Device Registers
   │
   ▼
Scan QR
   │
   ▼
Claim Completes
```

Both must be supported.

---

# Claiming and Firmware Download

The customer does not need to wait for the latest production firmware before establishing ownership.

Possible flow:

```text
Device Registers
      │
      ├── Claim Available
      │
      └── Firmware Update Available
```

The two processes can continue independently.

The dashboard may show:

```text
Device claimed
Firmware update in progress
```

---

# Claiming Unregistered Inventory

A device may be in inventory:

```text
factory_provisioned
registered = false
claimed = false
```

Its QR still works because the factory record exists.

---

# Device Ownership Record

Conceptual record:

```text
device_ownership

id
device_id
user_id
role
created_at
ended_at
```

For simple V1 ownership:

```text
role = owner
```

---

# Future Organization Ownership

Future Thermone versions may assign controllers to organizations rather than directly to a single user.

Example:

```text
Organization:
City Water Aquatics

Device:
THV1-000482

Members:
Owner
Admin
Staff
Viewer
```

The claim system should not prevent this future structure.

---

# Initial Setup After Claim

After successful claiming, the user proceeds to controller configuration.

Example:

```text
Controller added!

Name your Thermone:

[ Main Betta Rack ]

Location:

[ Fish Room ]

[ Continue ]
```

---

# Location Assignment

The device can be assigned to:

```text
Main Fish Room
Breeding Room
Grow-Out Room
Store
Office
```

Locations are user-facing organization tools.

They do not change device authentication.

---

# First Probe Setup

After claim:

```text
A01  Sensor Connected
A02  Sensor Connected
A03  No Sensor
...
```

The user can assign a port to a tank.

Example:

```text
A01
80.2°F

Assign To:
Betta Pair 01
```

---

# Claim Token Expiration

Claim tokens should support expiration or policy-controlled validity.

Possible strategies:

### Strategy A

```text
Valid until first successful claim
```

### Strategy B

```text
Valid for fixed period after manufacturing
```

### Strategy C

```text
Valid until explicitly revoked
```

For V1, the simplest model is:

```text
Valid until claimed or revoked
```

unless future security review requires shorter expiry.

---

# Claim Token Rotation

A token may need replacement before a device is claimed.

Reasons:

- QR label exposed
- Packaging compromised
- Token accidentally published
- Incorrect label printed

Flow:

```text
Invalidate Old Token
      │
      ▼
Generate New Token
      │
      ▼
Generate New QR
      │
      ▼
Replace Label
```

---

# Lost QR Code

If a user loses or damages the QR code, support should provide a secure recovery path.

Do not allow claiming based solely on:

```text
serial number
```

because serial numbers are public identifiers.

---

# Recovery Options

Possible recovery methods:

- Authenticated support workflow
- Proof of purchase
- Physical access to device
- Setup button plus local challenge
- New claim token issued by authorized support

The final recovery policy will be defined later.

---

# Serial Number Claiming

Never allow:

```text
POST /claim
{
  "serial": "THV1-000482"
}
```

to claim a controller by itself.

Serial number alone is insufficient proof of possession.

---

# Physical Claim Confirmation

A future stronger security model may require physical confirmation.

Example:

```text
Dashboard says:

Press the setup button on THV1-000482
within 60 seconds.
```

Then:

```text
User presses device button
       │
       ▼
ESP32 sends proof
       │
       ▼
Claim completes
```

This is not strictly required for initial V1 but is a strong future improvement.

---

# Ownership Transfer

A claimed controller may later be transferred.

Example:

```text
Current Owner
      │
      ▼
Transfer Device
      │
      ▼
Confirm Action
      │
      ▼
Existing Ownership Ends
      │
      ▼
Generate New Claim Token
      │
      ▼
New Owner Scans Token
```

---

# Original Claim Token During Transfer

The original factory claim token should remain invalid.

Do not reactivate it.

Generate a new transfer token instead.

---

# Transfer Token

A transfer token should be:

- Random
- Single-use
- Time-limited
- Separate from factory claim token
- Separate from device credentials

Example:

```text
https://app.thermone.com/transfer/<token>
```

---

# Transfer Confirmation

Sensitive transfer actions should require recent authentication.

Example:

```text
Confirm your password or login again.
```

---

# Ownership Transfer Data

Transferring the physical controller does not necessarily transfer historical tanks and temperature history.

Possible policy:

```text
Old owner's historical data remains with old account.

Physical controller moves to new owner.
```

This protects privacy.

---

# Removing a Device

The current owner may remove a controller.

Example:

```text
Remove Thermone THV1-000482?
```

The dashboard must explain what will happen.

---

# Recommended Removal Behavior

On removal:

```text
Ownership ends
Device becomes unclaimed
New claim token is generated
Historical data remains with previous account
```

The controller may be instructed to clear user-specific local configuration.

---

# Device Reset vs Ownership Removal

These are separate.

```text
Factory Reset
```

changes physical device configuration.

```text
Remove From Account
```

changes cloud ownership.

Doing one should not silently perform the other unless explicitly designed.

---

# Device Reset While Still Claimed

If a customer factory-resets their own controller:

```text
Cloud ownership may remain.
```

After reconfiguration, the device can authenticate/recover and reconnect to the same account.

This provides a better customer experience than forcing a new claim after every reset.

---

# Device Sold Without Removal

A customer may sell or give away a device without removing it first.

The new owner should not be able to simply claim it.

The dashboard should show:

```text
This controller is already associated with another account.
```

They may need:

- Previous owner transfer
- Support recovery process

---

# Stolen Device

If a user reports a controller stolen:

```text
claim_status = disabled
```

or an appropriate security state.

Potential actions:

- Disable transfer
- Disable new claims
- Revoke device runtime credentials
- Preserve customer history

---

# Device Returned to Thermone

Returned devices may enter:

```text
returned
```

or:

```text
refurbishment
```

The prior ownership must be securely cleared before resale.

A new claim token must be generated.

---

# Refurbished Devices

Refurbishment flow:

```text
Returned Device
      │
      ▼
Verify Serial
      │
      ▼
Clear Prior Runtime Data
      │
      ▼
Rotate Device Credential
      │
      ▼
Generate New Claim Token
      │
      ▼
Run Factory Tests
      │
      ▼
Ready for Resale
```

---

# Claim Audit Logging

Important claim events must be recorded.

Examples:

```text
claim_token_created
claim_token_rotated
claim_started
claim_waiting_for_device
claim_completed
claim_failed
claim_revoked
ownership_removed
ownership_transfer_started
ownership_transfer_completed
```

---

# Example Audit Event

```json
{
  "event": "device_claimed",
  "user_id": "usr_123",
  "device_id": "dev_456",
  "serial_number": "THV1-000482",
  "timestamp": "2026-08-16T23:00:00Z",
  "request_id": "req_789"
}
```

No raw claim token should be logged.

---

# Claim Attempt Rate Limiting

Claim attempts must be rate limited.

Protect against:

- Token guessing
- Automated scanning
- Credential stuffing around user login

Repeated invalid attempts should trigger increasing restrictions.

---

# Claim Error Responses

## Invalid Token

```json
{
  "success": false,
  "error": {
    "code": "INVALID_CLAIM_TOKEN",
    "message": "This Thermone claim link is invalid."
  }
}
```

---

## Already Claimed

```json
{
  "success": false,
  "error": {
    "code": "DEVICE_ALREADY_CLAIMED",
    "message": "This controller is already associated with an account."
  }
}
```

---

## Device Disabled

```json
{
  "success": false,
  "error": {
    "code": "DEVICE_DISABLED",
    "message": "This controller cannot currently be claimed."
  }
}
```

---

## Claim Expired

```json
{
  "success": false,
  "error": {
    "code": "CLAIM_TOKEN_EXPIRED",
    "message": "This claim link has expired."
  }
}
```

---

# Do Not Leak Ownership Data

For unauthorized users, do not return:

```text
Owner:
john@example.com
```

Instead:

```text
This controller is already associated with an account.
```

---

# Claim Session Expiration

Pending claim sessions should expire.

Example:

```text
30 minutes
```

or another appropriate period.

Expiration of the browser claim session does not necessarily invalidate the factory claim token.

---

# Returning to a Pending Claim

A user may close the page during setup.

Possible V1 behavior:

```text
Scan QR again
```

The backend can recognize:

```text
same authenticated user
+
same available device
```

and resume onboarding.

---

# Multiple Claim Attempts by Same User

The system should avoid creating many duplicate pending sessions.

Prefer:

```text
one active claim session
per user/device pair
```

---

# Multiple Users Scan Same Available Device

Before ownership is finalized, two users might scan the QR.

The backend must make ownership assignment atomic.

Example:

```text
User A claims
     │
     ▼
Database transaction
     │
     ▼
ownership created

User B attempts
     │
     ▼
DEVICE_ALREADY_CLAIMED
```

There must never be two owners accidentally created by a race condition.

---

# Atomic Claim Operation

The claim transaction should conceptually perform:

```text
Verify token still available
        │
        ▼
Lock device claim state
        │
        ▼
Create ownership
        │
        ▼
Mark token used
        │
        ▼
Commit
```

If any step fails, ownership should not partially complete.

---

# Claim Status in Dashboard

Possible onboarding states:

```text
Waiting for device
Connecting
Registering
Firmware updating
Ready to claim
Claiming
Configuring
Ready
```

---

# Claim Completion

Claiming is complete when:

```text
Authenticated user
        │
        ▼
owns
        │
        ▼
Thermone runtime device
```

and the dashboard can access that device using normal user authorization.

---

# Runtime Device Does Not Receive User JWT

The ESP32 does not need the customer's dashboard session token.

Never send:

```text
User JWT
```

to the device.

The controller continues authenticating with its own runtime credential.

---

# User Does Not Receive Device Token

Likewise, the user/browser does not receive:

```text
device runtime credential
```

The cloud acts as the boundary.

```text
ESP32
  │
  ▼
Device API

User
  │
  ▼
Dashboard API
```

---

# Ownership Notification

After successful claim, Thermone may send an account notification.

Example:

```text
Thermone THV1-000482 was added to your account.
```

This can help detect unauthorized claiming.

---

# Security Notification

Future security notification may include:

```text
Device claimed
Device removed
Ownership transfer requested
Ownership transfer completed
```

---

# Device Naming

After claiming, the user should name the controller.

Example:

```text
Main Betta Rack
Grow-Out Rack
Fish Room Left Wall
Rack B
```

This name does not replace the permanent serial number.

---

# Serial Number Always Preserved

Example:

```text
Display Name:
Main Betta Rack

Serial:
THV1-000482
```

The customer may change:

```text
Display Name
```

but never:

```text
Serial Number
```

---

# Location Setup

The onboarding flow may ask:

```text
Where is this controller?
```

Example options:

```text
Create New Location
Main Fish Room
Basement
Store
```

---

# QR Label Replacement

If the physical label becomes damaged after claiming, the dashboard does not need to reveal the original raw claim token.

A replacement label process should generate a new secure token or use a non-claim informational QR for already-owned devices.

---

# Future Setup QR

Future device labels may include two distinct QR purposes:

### Claim QR

```text
used for ownership
```

### Support QR

```text
contains non-secret serial/device lookup
```

V1 may use a single claim QR.

---

# Database Model

Conceptual factory device:

```text
factory_devices

id
serial_number
claim_status
owner_id
created_at
```

---

# Claim Tokens

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

# Claim Sessions

Conceptual:

```text
device_claim_sessions

id
factory_device_id
user_id
status
created_at
expires_at
completed_at
```

---

# Ownership

Conceptual:

```text
device_ownership

id
device_id
user_id
role
created_at
ended_at
```

---

# V1 Claim Success Criteria

Device claiming is ready when:

1. Every manufactured controller has a unique claim token.
2. The token is encoded in the QR.
3. Scanning the QR opens the correct Thermone environment.
4. Unauthenticated users are prompted to log in.
5. A device can be claimed before first boot.
6. A pending claim can wait for device registration.
7. A registered device can be claimed immediately.
8. A used claim token cannot claim a second account.
9. Already-owned devices cannot be stolen through the QR.
10. Ownership assignment is atomic.
11. Device runtime tokens are never exposed to users.
12. User JWTs are never exposed to ESP32 devices.
13. Claim attempts are rate limited.
14. Ownership actions are audit logged.
15. Devices can later support secure transfer.
16. Historical data can remain separate from physical ownership transfer.

---

# Core Device Claiming Principle

Thermone claiming should always maintain this separation:

```text
QR Claim Token
      │
      ▼
Proves possession of claim secret

User Login
      │
      ▼
Proves account identity

Both Valid
      │
      ▼
Ownership Created
```

The QR alone must never grant dashboard access.

The user login alone must never allow claiming an arbitrary serial number.