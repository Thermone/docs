# Thermone Documentation

Thermone is a connected aquarium monitoring platform built around network-connected ESP32 controllers.

The initial Thermone V1 controller monitors up to eight independent DS18B20 temperature probes and reports temperature, device health, alerts, and firmware status to Thermone Cloud.

This repository contains the technical documentation for:

- Hardware
- Firmware
- APIs
- Provisioning
- Device authentication
- Security
- Network architecture
- Recovery
- OTA updates

---

# Thermone V1 Overview

The V1 architecture is:

```text
Temperature Probes
        │
        ▼
Thermone ESP32 Controller
        │
        ▼
Wi-Fi / Ethernet
        │
        ▼
Thermone Cloud
        │
        ├── Device API
        ├── Dashboard API
        ├── Database
        ├── Firmware Service
        └── Alert Services
                │
                ▼
        Thermone Dashboard
                │
                ▼
              User
```

The controller communicates outbound to Thermone Cloud.

Customers do not need:

- Port forwarding
- Static public IP addresses
- Direct ESP32 Internet access
- Router configuration

---

# Repository Structure

```text
docs/
├── README.md
│
├── architecture/
│   ├── system-overview.md
│   ├── device-lifecycle.md
│   ├── network-flow.md
│   └── environments.md
│
├── hardware/
│   ├── controller-v1.md
│   ├── gpio-map-v1.md
│   ├── wiring-v1.md
│   └── parts-v1.md
│
├── api/
│   ├── device-api.md
│   ├── dashboard-api.md
│   └── authentication.md
│
├── provisioning/
│   ├── factory-provisioning.md
│   ├── wifi-setup.md
│   └── device-claiming.md
│
├── firmware/
│   ├── firmware-architecture.md
│   ├── ota-updates.md
│   └── recovery-mode.md
│
└── security/
    ├── threat-model.md
    ├── secrets.md
    └── device-authentication.md
```

---

# Architecture

## System Overview

File:

```text
architecture/system-overview.md
```

Defines:

- Overall Thermone system
- Cloud architecture
- Controller responsibilities
- Device identity
- Physical probe model
- Dashboard relationship
- Offline operation
- Firmware model

---

## Device Lifecycle

File:

```text
architecture/device-lifecycle.md
```

Defines the complete device lifecycle:

```text
Manufacturing
    │
    ▼
Factory Provisioning
    │
    ▼
Network Setup
    │
    ▼
Registration
    │
    ▼
Claiming
    │
    ▼
Normal Operation
    │
    ▼
Updates / Recovery
    │
    ▼
Transfer / Decommission
```

---

## Network Flow

File:

```text
architecture/network-flow.md
```

Defines:

- Wi-Fi
- Ethernet
- Setup access point
- Device-to-cloud connections
- Dashboard remote access
- Telemetry flow
- Heartbeats
- Offline buffering
- OTA download flow

---

## Environments

File:

```text
architecture/environments.md
```

Defines the isolated environments:

```text
development
staging
production
```

Production devices must never automatically fall back to development services.

---

# Hardware

## Controller V1

File:

```text
hardware/controller-v1.md
```

Defines the physical Thermone V1 controller.

Initial V1 hardware includes:

```text
ESP32
8 temperature probe ports
DS18B20 probes
USB power
Status LED
Setup/reset button
Water-resistant enclosure
```

---

## GPIO Map

File:

```text
hardware/gpio-map-v1.md
```

Current Thermone V1 mapping:

| Port | GPIO |
|---|---:|
| A01 | 13 |
| A02 | 14 |
| A03 | 16 |
| A04 | 17 |
| A05 | 25 |
| A06 | 26 |
| A07 | 27 |
| A08 | 32 |

Additional assignments:

```text
Status LED → GPIO 4
Setup Button → GPIO 33
```

---

## Wiring

File:

```text
hardware/wiring-v1.md
```

Each probe has:

```text
3.3V
DATA
GND
```

The probes share:

```text
3.3V
GND
```

but each probe uses its own data GPIO.

Each DATA line receives:

```text
4.7kΩ pull-up to 3.3V
```

---

## Parts

File:

```text
hardware/parts-v1.md
```

Defines the prototype bill of materials.

Major components:

- ESP32 DevKit
- 8 DS18B20 probes
- 8 waterproof connectors
- 8 4.7kΩ resistors
- RGB LED
- Setup/reset button
- Perfboard
- Enclosure
- USB power supply
- Wiring and assembly hardware

---

# APIs

## Device API

File:

```text
api/device-api.md
```

The Device API is used by physical Thermone controllers.

Initial endpoints include:

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

## Dashboard API

File:

```text
api/dashboard-api.md
```

The Dashboard API supports:

- Users
- Locations
- Controllers
- Tanks
- Temperature history
- Alerts
- Device health
- Device claiming
- Device commands

---

## Authentication

File:

```text
api/authentication.md
```

Thermone separates:

```text
User JWT
Factory Credential
Runtime Device Token
Claim Token
Service Credentials
```

These credentials must never be reused for another purpose.

---

# Provisioning

## Factory Provisioning

File:

```text
provisioning/factory-provisioning.md
```

Factory provisioning creates:

- Device serial
- Factory credential
- Claim token
- QR code
- Hardware identity binding
- Bootstrap firmware
- Manufacturing test record

Example serial:

```text
THV1-000482
```

---

## Wi-Fi Setup

File:

```text
provisioning/wifi-setup.md
```

First-time Wi-Fi provisioning:

```text
Power On
   │
   ▼
Thermone-0482
   │
   ▼
Phone Connects
   │
   ▼
Setup Portal
   │
   ▼
Select Wi-Fi
   │
   ▼
Enter Password
   │
   ▼
Thermone Goes Online
```

Customer Wi-Fi passwords remain on the device.

They are not sent to Thermone Cloud.

---

## Device Claiming

File:

```text
provisioning/device-claiming.md
```

Claim flow:

```text
QR Claim Token
       +
Authenticated User
       │
       ▼
Device Ownership
```

The claim token is not a device API credential.

---

# Firmware

## Firmware Architecture

File:

```text
firmware/firmware-architecture.md
```

Recommended modules:

```text
board
sensors
network
provisioning
identity
cloud
telemetry
storage
ota
commands
status
button
time
health
watchdog
recovery
```

---

## OTA Updates

File:

```text
firmware/ota-updates.md
```

OTA follows:

```text
Download
   │
   ▼
Verify
   │
   ▼
Install Inactive Slot
   │
   ▼
Reboot
   │
   ▼
Validate
   │
   ├── Success → Confirm
   └── Failure → Rollback
```

---

## Recovery Mode

File:

```text
firmware/recovery-mode.md
```

Recovery handles:

- Failed firmware
- Boot loops
- Broken Wi-Fi configuration
- Runtime credential recovery
- Corrupt configuration
- Network reset
- Factory reset
- Known-good firmware reinstall

---

# Security

## Threat Model

File:

```text
security/threat-model.md
```

Threats considered include:

- QR theft
- Token theft
- Device cloning
- API abuse
- Cross-user access
- Firmware tampering
- OTA bricking
- Setup network abuse
- Cloud outages
- Physical access

---

## Secrets

File:

```text
security/secrets.md
```

Defines how Thermone protects:

- Device credentials
- Factory credentials
- Claim tokens
- Supabase keys
- Database credentials
- Firmware signing keys
- GitHub secrets
- Wi-Fi passwords

---

## Device Authentication

File:

```text
security/device-authentication.md
```

Device authentication is based on:

```text
Unique Device Credential
        +
Expected Device Record
        +
Expected Hardware Context
```

Public identifiers alone never authenticate a device.

---

# Thermone Repositories

The Thermone GitHub organization is divided into:

```text
firmware
api
dashboard
provisioning
infrastructure
docs
```

---

# Repository Responsibilities

## firmware

ESP32 software.

Includes:

- Sensors
- Networking
- Provisioning
- OTA
- Recovery
- Telemetry
- Device health

---

## api

Cloud backend.

Includes:

- Device API
- Dashboard API
- Device registration
- Authentication
- Telemetry ingestion
- Alerts
- Firmware manifests

---

## dashboard

Customer-facing web application.

Includes:

- Login
- Device setup
- Tanks
- Temperatures
- Charts
- Alerts
- Controller settings

---

## provisioning

Manufacturing and device onboarding tooling.

Includes:

- Serial generation
- Factory credentials
- Claim tokens
- QR generation
- Label generation
- ESP32 flashing
- Manufacturing tests

---

## infrastructure

Cloud infrastructure and deployment.

May include:

- Supabase
- Hosting
- DNS
- Firmware storage
- Monitoring
- CI/CD
- Infrastructure as code

---

## docs

Technical specifications and design decisions.

---

# Core Thermone Principles

## Independent Probe Ports

Each temperature connector has its own GPIO.

```text
A01 → independent bus
A02 → independent bus
...
A08 → independent bus
```

This makes physical identification and troubleshooting simple.

---

## Cloud Is Not Required for Local Monitoring

If Thermone Cloud becomes unavailable:

```text
Sensor monitoring continues.
```

The controller buffers telemetry and reconnects later.

---

## Device Credentials Are Unique

Every controller has its own authentication boundary.

```text
One device
=
One credential
```

A compromised device must not compromise the entire Thermone fleet.

---

## Ownership Is Cloud Managed

Physical reset does not automatically remove cloud ownership.

```text
Factory Reset
≠
Remove From Account
```

---

## Firmware Must Be Recoverable

A failed update must not permanently disable a controller.

```text
A/B OTA
+
Rollback
+
Recovery
```

---

## Secrets Stay Where Needed

A component should receive a secret only if it genuinely needs that secret.

---

# V1 Product Goal

Thermone V1 is successful when a customer can:

1. Receive a controller.
2. Plug it in.
3. Connect it to Wi-Fi.
4. Register it with Thermone Cloud.
5. Scan the QR code.
6. Claim the controller.
7. Connect up to 8 probes.
8. Assign probes to tanks.
9. View live temperature remotely.
10. View temperature history.
11. Receive alerts.
12. See controller health.
13. Replace probes without losing tank history.
14. Continue monitoring during Internet outages.
15. Receive firmware updates remotely.
16. Recover from failed updates or network configuration problems.

---

# Current Project Phase

The initial Thermone architecture and V1 design documentation are being defined before implementation.

The planned implementation sequence is:

```text
Documentation
     │
     ▼
Infrastructure
     │
     ▼
Database
     │
     ▼
API
     │
     ▼
Provisioning
     │
     ▼
Firmware
     │
     ▼
Dashboard
     │
     ▼
End-to-End Testing
```

---

# Documentation Change Policy

When implementation changes an architectural decision, update the corresponding documentation.

Examples:

```text
GPIO changed
→ update hardware docs

API endpoint changed
→ update API docs

Authentication flow changed
→ update security docs

Provisioning flow changed
→ update provisioning docs
```

The documentation should describe the system that actually exists, not an outdated earlier design.

---

# Status

Thermone V1:

```text
Architecture: In Design
Hardware: Prototype Planning
API: Specification
Firmware: Specification
Dashboard: Not Implemented
Production: Not Released
```