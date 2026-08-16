# Thermone V1 Factory Provisioning

## Purpose

This document defines the factory provisioning process for Thermone V1 controllers.

Factory provisioning is the process that converts a generic assembled ESP32 controller into a uniquely identifiable Thermone product.

The process creates and links:

- Factory device record
- Thermone serial number
- Hardware revision
- Factory credential
- Claim token
- QR code
- Bootstrap configuration
- ESP32 hardware identity
- Manufacturing test status

The core goal is:

```text
Every physical Thermone controller must have its own unique identity before it leaves manufacturing.
```

---

# High-Level Provisioning Flow

```text
Assembled Controller
       │
       ▼
Connect to Factory Computer
       │
       ▼
Provisioning Tool
       │
       ├── Create Factory Record
       ├── Generate Serial
       ├── Generate Factory Credential
       ├── Generate Claim Token
       ├── Read ESP32 Hardware ID
       ├── Generate Bootstrap Config
       ├── Flash Bootstrap Firmware
       ├── Generate QR Code
       ├── Generate Device Label
       └── Run Validation Tests
               │
               ▼
         Ready for Shipment
```

---

# Factory Provisioning Responsibilities

The Thermone provisioning system is responsible for:

- Creating a unique device record
- Generating human-readable serial numbers
- Generating secure factory credentials
- Generating secure claim tokens
- Associating physical hardware with its record
- Flashing bootstrap firmware
- Writing device-specific configuration
- Generating QR labels
- Performing manufacturing tests
- Recording test results
- Marking units ready for shipment

---

# Provisioning Components

The factory provisioning system consists of:

```text
Thermone Provisioning CLI
        │
        ├── Thermone API
        ├── Serial Generator
        ├── Secret Generator
        ├── QR Generator
        ├── Label Generator
        ├── ESP32 Flash Tool
        └── Hardware Test Runner
```

---

# Provisioning Repository

Factory provisioning code lives in:

```text
Thermone/provisioning
```

Possible structure:

```text
provisioning/
├── factory/
│   ├── create-device.ts
│   ├── generate-serial.ts
│   ├── generate-secret.ts
│   ├── generate-claim-token.ts
│   ├── generate-qr.ts
│   ├── generate-label.ts
│   ├── read-hardware-id.ts
│   ├── flash-device.ts
│   ├── verify-device.ts
│   └── manufacture.ts
│
├── config/
├── templates/
├── tests/
└── README.md
```

---

# Factory Device Record

Before the physical controller is provisioned, Thermone creates a factory record.

Conceptual record:

```json
{
  "serial_number": "THV1-000001",
  "model": "THV1",
  "hardware_revision": "1.0",
  "status": "manufactured"
}
```

The exact database structure will be defined separately.

---

# Serial Number Format

Recommended V1 serial format:

```text
THV1-000001
```

Meaning:

```text
TH
│
└── Thermone

V1
│
└── Hardware/Product Generation

000001
│
└── Sequential Human-Readable Unit Number
```

Examples:

```text
THV1-000001
THV1-000002
THV1-000003
THV1-000482
```

---

# Serial Number Requirements

Serial numbers must be:

- Unique
- Immutable
- Human-readable
- Printable
- Easy to type
- Easy for support staff to read

Serial numbers are not secrets.

---

# Serial Number Allocation

Serial numbers should be allocated by the Thermone backend.

Do not allow factory workstations to independently guess the next serial.

Bad:

```text
Last device was 482.
I'll manually make this one 483.
```

Good:

```text
Provisioning CLI
      │
      ▼
Thermone Factory API
      │
      ▼
Reserve Next Serial
      │
      ▼
THV1-000483
```

This prevents duplicates when multiple provisioning stations exist.

---

# Internal Factory Device ID

The backend should also create an internal unique identifier.

Example:

```text
019c3e92-0000-7000-8000-000000000001
```

This internal ID is different from:

```text
THV1-000001
```

The internal ID is used for database relationships.

The serial number is used by humans.

---

# ESP32 Hardware Identity

During provisioning, the tool reads a stable hardware-derived ESP32 identifier.

Conceptual example:

```text
ESP32-A4CF12B78E31
```

The actual format will depend on the ESP32 identity source chosen during firmware implementation.

---

# Hardware Identity Purpose

The hardware ID helps Thermone:

- Associate the physical ESP32 with its factory record
- Detect duplicate provisioning
- Detect hardware replacement
- Troubleshoot manufacturing issues

The hardware ID is not a password.

---

# Duplicate Hardware Protection

Before provisioning completes, the backend should check whether the detected ESP32 hardware ID already exists.

Example:

```text
Hardware ID:
ESP32-A4CF12B78E31
```

Backend:

```text
Already assigned?
```

If yes:

```text
STOP provisioning
```

unless an authorized reprovisioning workflow is being used.

---

# Factory Credential Generation

Each controller receives a unique factory credential.

The credential is generated using a cryptographically secure random number generator.

Recommended entropy:

```text
256 bits
```

Conceptual example:

```text
fc_prod_wD2v7K...
```

The exact token format may change.

---

# Factory Credential Requirements

The factory credential must be:

- Unique per controller
- Random
- High entropy
- Environment-specific
- Revocable
- Never printed on the enclosure
- Never included in the customer QR code
- Never committed to Git

---

# Factory Credential Purpose

The bootstrap firmware uses the factory credential for first registration.

Conceptually:

```http
POST /v1/device/register
Authorization: Factory <factory-credential>
```

The credential proves that the device was provisioned by Thermone.

---

# Factory Credential Storage

The raw factory credential is required during device flashing.

The provisioning tool may receive it once from the backend.

The backend should store a secure hash or verifier where practical.

Conceptual database field:

```text
factory_credential_hash
```

The plaintext credential should not be available through normal administrative interfaces.

---

# Factory Credential on Device

The provisioning tool writes the credential into persistent ESP32 storage.

Potential storage:

```text
NVS
```

Future production revisions may use:

```text
Encrypted NVS
Secure Boot
Flash Encryption
Hardware-backed key storage
```

The exact security configuration is defined separately.

---

# Claim Token Generation

Every unit also receives a claim token.

The claim token is independent from the factory credential.

Recommended entropy:

```text
256 bits
```

Conceptual value:

```text
MWWXv5IKG2b3a...
```

---

# Claim Token Purpose

The claim token allows the customer to associate the device with their Thermone account.

It is used in:

```text
https://app.thermone.com/claim/<claim-token>
```

---

# Claim Token Rules

The claim token must:

- Be random
- Be difficult to guess
- Be different from the factory credential
- Be different from the runtime device credential
- Be revocable
- Become invalid after successful claim or transfer policy
- Be environment-specific

---

# Claim Token Server Storage

The backend should preferably store:

```text
claim_token_hash
```

rather than retaining plaintext indefinitely.

---

# QR Code Generation

The provisioning process generates a QR code using the claim URL.

Example:

```text
https://app.thermone.com/claim/<claim-token>
```

The QR code should not include:

- Factory credential
- Runtime device token
- Wi-Fi credentials
- Private keys
- Service API keys

---

# QR Code Workflow

```text
Generate Claim Token
       │
       ▼
Build Claim URL
       │
       ▼
Generate QR Image
       │
       ▼
Print Device Label
```

---

# Device Label

Recommended label content:

```text
THERMONE

Model:
THV1

Serial:
THV1-000482

Hardware:
V1.0

[ QR CODE ]

Scan to set up
```

Future labels may also include:

- Power rating
- Regulatory markings
- Manufacturing date
- Batch number
- Country of origin

---

# Bootstrap Configuration

Every controller receives device-specific bootstrap configuration.

Conceptual configuration:

```json
{
  "serial_number": "THV1-000482",
  "model": "THV1",
  "hardware_revision": "1.0",
  "environment": "production"
}
```

Secrets are stored separately from public device metadata where practical.

---

# Public Factory Configuration

Non-secret values may include:

```text
serial_number
model
hardware_revision
environment
manufacturing_batch
```

---

# Secret Factory Configuration

Secret values may include:

```text
factory_credential
```

These must not appear in ordinary debug output.

---

# Bootstrap Firmware

The same base bootstrap firmware binary may be used across many devices.

Device-specific identity is preferably injected separately.

Conceptually:

```text
bootstrap.bin
+
device-specific NVS/config
```

instead of recompiling the entire firmware for every serial number.

---

# Why Avoid Per-Device Firmware Compilation

Bad production model:

```text
Compile firmware for THV1-000001
Compile firmware for THV1-000002
Compile firmware for THV1-000003
...
```

Preferred:

```text
One tested bootstrap firmware
          │
          +
Unique device provisioning data
```

Benefits:

- Faster manufacturing
- Reproducible firmware
- Easier version control
- Easier debugging
- Fewer build variations

---

# Factory Firmware Version

The provisioning record stores the bootstrap firmware version.

Example:

```text
bootstrap_version:
1.0.0
```

This allows Thermone to know what was installed during manufacturing.

---

# Provisioning CLI

The factory process should eventually be performed with one command.

Example:

```bash
npm run manufacture
```

or:

```bash
thermone manufacture
```

---

# Example Manufacturing CLI Flow

```text
Thermone Factory Provisioning
=============================

Searching for ESP32...

Device found:
/dev/tty.usbserial-0001

Reading hardware identity...

ESP32-A4CF12B78E31

Requesting factory record...

Serial:
THV1-000482

Generating factory credential...
✓

Generating claim token...
✓

Creating device record...
✓

Generating bootstrap configuration...
✓

Flashing bootstrap firmware...
✓

Writing device identity...
✓

Restarting controller...
✓

Running hardware test...
✓

Generating QR code...
✓

Generating label...
✓

Device ready.

Serial:
THV1-000482
```

---

# Provisioning Operator

The provisioning operator should not need access to production database credentials.

The CLI should authenticate with a limited factory service credential.

That service credential should only be permitted to perform required provisioning operations.

---

# Factory API

A dedicated internal factory API may be used.

Conceptual endpoints:

```text
POST /internal/factory/devices
POST /internal/factory/devices/{id}/hardware
POST /internal/factory/devices/{id}/test-results
POST /internal/factory/devices/{id}/complete
```

These endpoints must not be publicly available without authentication.

---

# Manufacturing Environment

Development units should use development provisioning.

Example serials:

```text
DEV-THV1-0001
```

Staging:

```text
STG-THV1-0001
```

Production:

```text
THV1-000001
```

Development factory credentials must not work against production APIs.

---

# Production Provisioning Protection

Production provisioning actions should require:

- Authenticated factory tooling
- Correct environment
- Restricted API permissions
- Audit logging
- Secure secret handling

---

# Device Connection

The provisioning computer connects to the ESP32 over:

```text
USB
```

The factory tooling should detect the correct serial port automatically where practical.

---

# ESP32 Flash Process

Conceptual flow:

```text
Erase / Prepare Device
        │
        ▼
Flash Bootloader
        │
        ▼
Flash Partition Table
        │
        ▼
Flash Bootstrap Firmware
        │
        ▼
Write Factory Configuration
        │
        ▼
Write Factory Credential
        │
        ▼
Restart
```

Exact commands are defined during firmware implementation.

---

# Factory Data Partition

Thermone may use a dedicated NVS partition for factory configuration.

Conceptually:

```text
factory
```

This may contain:

```text
serial_number
model
hardware_revision
factory_credential
provisioning_version
```

---

# Customer Data Partition

Customer-generated configuration should be separate where practical.

Example:

```text
runtime
```

may contain:

```text
Wi-Fi credentials
runtime device token
configuration cache
```

This allows factory identity to survive certain reset operations.

---

# Factory Identity Persistence

A normal customer factory reset should not erase:

```text
serial_number
factory identity
hardware revision
```

It may erase:

```text
Wi-Fi credentials
runtime configuration
cached readings
runtime credential
```

depending on the final recovery design.

---

# Permanent vs Resettable Data

## Permanent

```text
Factory Serial
Model
Hardware Revision
Factory Provisioning Metadata
```

## Resettable

```text
Wi-Fi Credentials
Runtime Device Token
Tank Configuration Cache
Offline Telemetry
User Settings
```

---

# Manufacturing Test

Every controller must pass a factory validation test.

Recommended tests:

1. ESP32 boots
2. Serial number can be read
3. Hardware ID matches record
4. Factory credential exists
5. Setup button works
6. Status LED works
7. Wi-Fi radio works
8. USB power is stable
9. All eight probe GPIOs initialize
10. Flash layout is valid
11. Bootstrap version matches expected build

---

# Probe Port Factory Test

The production line should eventually validate every port.

Example test fixture:

```text
Known DS18B20 Sensor
        │
        ▼
A01
```

Verify:

```text
Sensor detected
ROM ID read
Temperature valid
```

Repeat for:

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

---

# Faster Production Testing

Future production fixtures may connect test sensors to all eight ports simultaneously.

```text
Factory Test Fixture

A01 ─ Test Sensor
A02 ─ Test Sensor
A03 ─ Test Sensor
A04 ─ Test Sensor
A05 ─ Test Sensor
A06 ─ Test Sensor
A07 ─ Test Sensor
A08 ─ Test Sensor
```

The provisioning tool verifies every channel automatically.

---

# Test Result Record

Example conceptual result:

```json
{
  "device_id": "factory_123",
  "tests": {
    "boot": "pass",
    "wifi": "pass",
    "status_led": "pass",
    "setup_button": "pass",
    "a01": "pass",
    "a02": "pass",
    "a03": "pass",
    "a04": "pass",
    "a05": "pass",
    "a06": "pass",
    "a07": "pass",
    "a08": "pass"
  }
}
```

---

# Test Failure

If any required test fails:

```text
status = factory_test_failed
```

The device must not be marked ready for shipment.

---

# Rework Flow

A failed device may enter:

```text
rework
```

After repair:

```text
Run Factory Test Again
```

A full audit trail should remain.

---

# Factory Completion

After successful provisioning and validation:

```text
status = factory_provisioned
```

Example backend state:

```json
{
  "serial_number": "THV1-000482",
  "status": "factory_provisioned",
  "claim_status": "available",
  "owner_id": null
}
```

---

# Inventory State

A provisioned controller can exist in inventory before shipment.

Example:

```text
THV1-000482

Provisioned:
Yes

Claimed:
No

Registered:
No

Owner:
None
```

This is a valid state.

---

# QR Scanned Before First Boot

The QR code works because the factory record already exists.

Example:

```text
User scans THV1-000482
```

even though:

```text
Device has never connected to the Internet
```

The dashboard can show:

```text
Thermone THV1-000482

Waiting for controller to connect.
```

---

# First Cloud Registration

When the customer eventually powers the controller:

```text
Bootstrap Firmware
       │
       ▼
Internet
       │
       ▼
Factory Credential
       │
       ▼
Thermone Device API
       │
       ▼
Factory Record Found
       │
       ▼
Permanent Runtime Device Record Activated
```

---

# Registration Linkage

The backend links:

```text
Factory Device
      │
      ▼
Runtime Device
```

Example:

```text
Serial:
THV1-000482

Internal Factory ID:
factory_123

Runtime Device ID:
dev_456
```

---

# Runtime Credential

The Device API then issues a runtime credential.

This credential is not generated by the factory QR system.

It is issued during secure cloud registration.

---

# Factory Credential After Registration

After successful runtime registration:

```text
Factory credential
```

should become restricted.

It may be retained only for:

- Approved recovery workflows
- Manufacturing diagnostics
- Secure reprovisioning

It should not continue authenticating normal telemetry.

---

# Label Printing

The provisioning tool should generate a print-ready label automatically.

Inputs include:

```text
Brand
Model
Serial
Hardware revision
QR
```

Example:

```text
┌──────────────────────────┐
│        THERMONE          │
│                          │
│ Model THV1               │
│ HW V1.0                  │
│ S/N THV1-000482          │
│                          │
│      [ QR CODE ]         │
│                          │
│ Scan to set up           │
└──────────────────────────┘
```

---

# Label Verification

Before completing manufacturing:

1. Scan the printed QR.
2. Confirm correct serial is displayed.
3. Confirm QR points to correct environment.
4. Confirm no secret credential is visibly encoded except the intended claim token.
5. Confirm label matches physical controller.

---

# Packaging Association

The same serial may appear on:

- Device enclosure
- Product box
- Manufacturing record

This helps support identify unopened products.

---

# Manufacturing Batch

Factory records may include:

```text
batch_id
```

Example:

```text
BATCH-2026-08-001
```

Useful for identifying:

- Component problems
- Supplier issues
- Assembly defects
- Firmware versions

---

# Provisioning Version

The provisioning software should report its own version.

Example:

```text
provisioning_version:
1.2.0
```

This helps reproduce manufacturing history.

---

# Example Complete Factory Record

```json
{
  "serial_number": "THV1-000482",
  "model": "THV1",
  "hardware_revision": "1.0",
  "hardware_id": "ESP32-A4CF12B78E31",
  "bootstrap_version": "1.0.0",
  "provisioning_version": "1.2.0",
  "manufacturing_batch": "BATCH-2026-08-001",
  "status": "factory_provisioned",
  "claim_status": "available",
  "owner_id": null
}
```

Secret credential hashes would be stored separately.

---

# Audit Logging

Provisioning operations should generate audit events.

Examples:

```text
factory_device_created
serial_allocated
hardware_id_bound
bootstrap_flushed
factory_test_started
factory_test_passed
factory_test_failed
device_marked_ready
label_generated
```

---

# Provisioning Workstation Security

Factory workstations should not permanently store:

- Raw factory credentials
- Claim tokens
- Production service credentials

Temporary generated data should be deleted after successful provisioning where practical.

---

# Reprovisioning

Reprovisioning an existing production serial must not happen through the normal manufacturing command.

It should require a separate explicit workflow.

Example:

```bash
thermone reprovision THV1-000482
```

with additional authorization.

---

# Serial Reuse

A serial number must never be reassigned to a different physical unit after shipment.

Bad:

```text
Broken THV1-000482
      ↓
Reuse serial on new ESP32
```

Instead:

```text
Replacement unit:
THV1-000983
```

The backend can link it as a replacement.

---

# Hardware Replacement

If the ESP32 is replaced during factory rework before shipment, the same serial may potentially be retained if the change is documented.

Once shipped or activated, hardware replacement should follow the formal replacement workflow.

---

# Claim Token Regeneration

Before shipment, a claim token may be regenerated if:

- Label was compromised
- QR was printed incorrectly
- Token was accidentally exposed

The old token must be invalidated.

---

# Environment QR URLs

Development:

```text
https://app.dev.thermone.com/claim/<token>
```

Staging:

```text
https://app.staging.thermone.com/claim/<token>
```

Production:

```text
https://app.thermone.com/claim/<token>
```

Production QR codes must never point to staging or development.

---

# Factory Provisioning Success Criteria

A Thermone controller is considered successfully factory provisioned when:

1. A unique factory record exists.
2. A unique serial number exists.
3. ESP32 hardware identity has been recorded.
4. A unique factory credential has been generated.
5. A unique claim token has been generated.
6. Bootstrap firmware has been flashed successfully.
7. Device-specific configuration has been written.
8. Factory validation tests pass.
9. QR code has been generated.
10. QR code resolves to the correct factory record.
11. Device label matches the controller.
12. Device is marked `factory_provisioned`.
13. Device has no customer owner.
14. Secrets are not exposed in logs or labels.

---

# Core Factory Provisioning Principle

Factory provisioning binds:

```text
Physical ESP32
     │
     ▼
Hardware Identity
     │
     ▼
Thermone Serial
     │
     ▼
Factory Credential
     │
     ▼
Claim Token
     │
     ▼
Cloud Factory Record
```

before the controller reaches the customer.

This ensures every Thermone controller is uniquely identifiable, securely registerable, and ready to be claimed without requiring the final runtime device ID to exist when the QR code is printed.