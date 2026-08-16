# Thermone V1 System Overview

## Purpose

Thermone V1 is a network-connected aquarium temperature monitoring system built around an ESP32 controller.

Each Thermone V1 controller supports up to **8 independent DS18B20 temperature probes**.

Each probe has:

- A dedicated physical port
- A dedicated ESP32 GPIO data line
- A unique DS18B20 hardware ID
- A logical tank assignment within the Thermone platform

Thermone provides:

- Real-time temperature monitoring
- Temperature history
- High and low temperature alerts
- Remote dashboard access
- Device provisioning
- Secure device registration
- Device claiming
- OTA firmware updates
- Device health monitoring
- Offline monitoring and temporary data storage

---

# High-Level Architecture

```text
DS18B20 Temperature Probes
            │
            ▼
┌───────────────────────────┐
│ Thermone V1 Controller    │
│                           │
│ ESP32                     │
│                           │
│ A01 ─ Temperature Probe   │
│ A02 ─ Temperature Probe   │
│ A03 ─ Temperature Probe   │
│ A04 ─ Temperature Probe   │
│ A05 ─ Temperature Probe   │
│ A06 ─ Temperature Probe   │
│ A07 ─ Temperature Probe   │
│ A08 ─ Temperature Probe   │
└─────────────┬─────────────┘
              │
              │ Wi-Fi / Ethernet
              │
              ▼
        Public Internet
              │
              │ HTTPS
              ▼
┌───────────────────────────┐
│ Thermone Cloud            │
│                           │
│ Device API                │
│ Database                  │
│ Firmware Service          │
│ Alert Service             │
│ Authentication            │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│ Thermone Dashboard        │
│                           │
│ Web                       │
│ Mobile / PWA              │
└─────────────┬─────────────┘
              │
              ▼
             User
```

---

# Main Components

## Thermone V1 Controller

The Thermone V1 Controller is the physical monitoring device installed near the aquariums.

### Hardware

The initial V1 hardware consists of:

- ESP32 development board
- 8 independent DS18B20 probe ports
- 8 waterproof temperature probes
- Individual GPIO data line for each probe
- Shared 3.3V power bus
- Shared ground bus
- One 4.7kΩ pull-up resistor per data line
- Waterproof external probe connectors
- USB power
- Status LED
- Waterproof enclosure

Ethernet may be added depending on the final V1 hardware configuration.

### Controller Responsibilities

The controller is responsible for:

- Reading all connected temperature probes
- Detecting connected and disconnected probes
- Reading DS18B20 hardware IDs
- Associating readings with physical ports
- Connecting to Wi-Fi
- Connecting to Ethernet when supported
- Providing first-time network setup
- Authenticating with the Thermone API
- Uploading temperature telemetry
- Sending device heartbeats
- Receiving device configuration
- Checking for firmware updates
- Installing OTA firmware updates
- Storing temporary readings during Internet outages
- Recovering from network failures
- Reporting device health

---

# Physical Probe Ports

Thermone V1 provides eight physical temperature ports.

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

Each physical port has its own independent 1-Wire data line.

Example GPIO mapping:

| Port | ESP32 GPIO |
|---|---:|
| A01 | GPIO 4 |
| A02 | GPIO 16 |
| A03 | GPIO 17 |
| A04 | GPIO 18 |
| A05 | GPIO 19 |
| A06 | GPIO 21 |
| A07 | GPIO 22 |
| A08 | GPIO 23 |

> **Important:** This GPIO map is provisional. The final GPIO assignments must be validated against the exact ESP32 development board selected for Thermone V1 before hardware production.

Power and ground may be shared between all eight ports.

Each data line receives its own 4.7kΩ pull-up resistor.

---

# Sensor Identification

Each DS18B20 temperature sensor contains a unique 64-bit hardware address.

Example:

```text
28-FF-19-83-C2-16-04-27
```

Thermone records both the physical controller port and the sensor hardware ID.

Example:

```text
Controller: THV1-000001
Port: A03
Sensor ID: 28-FF-19-83-C2-16-04-27
```

This provides two levels of identification.

### Physical Identity

```text
Controller → A03
```

The physical connector identifies where the probe is connected.

### Sensor Identity

```text
28-FF-19-83-C2-16-04-27
```

The DS18B20 address identifies the actual probe.

This allows Thermone to detect when a probe has been replaced.

For example:

```text
A03

Previous Sensor:
28-FF-AA-AA-AA-AA-AA-AA

New Sensor:
28-FF-BB-BB-BB-BB-BB-BB
```

Thermone can recognize that the physical port remains A03 while the probe itself has changed.

---

# Tank Assignment

Physical controller ports are separate from tanks.

Example:

```text
Controller
THV1-000001
      │
      ▼
Port A03
      │
      ▼
Sensor
28-FF-19-83-C2-16-04-27
      │
      ▼
Tank
Betta Breeding Tank 03
```

This allows tanks, probes, and controllers to be managed independently.

Replacing a temperature probe should not destroy the historical temperature records associated with the tank.

---

# Thermone Device API

The Thermone Device API handles communication between physical controllers and Thermone Cloud.

Example production base URL:

```text
https://api.thermone.com/v1/device/
```

The Device API is responsible for:

- Factory device registration
- Device activation
- Device authentication
- Telemetry ingestion
- Heartbeats
- Device configuration
- Firmware update checks
- Device health
- Credential rotation
- Credential revocation

The device API must be treated separately from normal user authentication.

---

# Thermone Dashboard

The Thermone Dashboard is the customer-facing application.

Example production URL:

```text
https://app.thermone.com
```

The dashboard provides:

- Account creation
- Login
- Device claiming
- Controller management
- Tank management
- Probe assignment
- Current temperatures
- Temperature history
- Temperature graphs
- Alerts
- Alert configuration
- Device health
- Firmware information
- Controller settings
- Location management
- Account management

Because the dashboard communicates with Thermone Cloud instead of directly with the ESP32, users can access their tanks from anywhere with Internet access.

---

# Remote Access Architecture

Thermone controllers are never intended to be directly exposed to the public Internet.

The controller initiates outbound connections to Thermone Cloud.

```text
Thermone Controller
        │
        │ Outbound HTTPS
        ▼
Thermone API
        │
        ▼
Database
        │
        ▼
Thermone Dashboard
        │
        ▼
User
```

Customers do not need:

- Port forwarding
- Static public IP addresses
- Dynamic DNS
- Router configuration
- Direct ESP32 Internet exposure

---

# Database

The initial Thermone database may use Supabase/PostgreSQL.

Primary data entities include:

- Users
- Organizations
- Locations
- Devices
- Device ports
- Sensors
- Tanks
- Sensor assignments
- Temperature readings
- Alerts
- Alert rules
- Firmware versions
- Device heartbeats
- Device claims
- Factory devices
- Device credentials

The detailed schema will be documented separately.

---

# Device Identity

Thermone uses several different identifiers.

They serve different purposes and must not be treated as interchangeable.

## Factory Serial Number

Example:

```text
THV1-000001
```

Purpose:

- Human-readable identifier
- Printed on the enclosure
- Manufacturing tracking
- Customer support
- Inventory management

---

## Internal Device ID

Example:

```text
019c3e92-0000-7000-8000-000000000001
```

Purpose:

- Database primary identifier
- Internal API relationships
- Not intended to be the main customer-facing identifier

---

## Hardware Identity

The controller also records a hardware-derived ESP32 identity.

Purpose:

- Identify the physical ESP32
- Detect unexpected hardware changes
- Help prevent accidental duplicate registrations

A hardware identifier must not be treated as an authentication secret.

---

## Device Credential

Each registered controller receives its own device credential.

Purpose:

- Authenticate the controller to the Thermone Device API

The credential must:

- Be unique per device
- Be revocable
- Never be committed to Git
- Never be shared with another controller
- Be stored securely where practical

---

## Claim Token

Every manufactured controller receives a separate random claim token.

The claim token is used to associate the physical controller with a Thermone user account.

Example flow:

```text
Factory
   │
   ▼
Generate Serial
THV1-000001
   │
   ▼
Generate Random Claim Token
   │
   ▼
Generate QR Code
   │
   ▼
Print Device Label
```

Example QR destination:

```text
https://app.thermone.com/claim/<claim-token>
```

The claim token must **never** be used as the device API credential.

---

# Device Claiming

A user claims a Thermone controller by scanning its QR code.

```text
Scan QR
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
Find Factory Device
   │
   ▼
Associate Device With User
   │
   ▼
Controller Appears in Dashboard
```

Claim tokens should become unusable for unauthorized additional claims after successful activation.

Ownership transfer will be designed separately.

---

# First-Time Network Setup

A new controller contains bootstrap firmware before shipping.

If the controller does not have valid Wi-Fi credentials, it enters setup mode.

Example temporary Wi-Fi network:

```text
Thermone-A482
```

The customer connects to the temporary network using a phone or computer.

The controller provides a local setup page.

Example:

```text
Thermone Setup

Select Wi-Fi Network
[ HomeWiFi ]

Password
[ ************** ]

[ Connect ]
```

The credentials are sent directly to the controller.

The customer's Wi-Fi password should never be sent to Thermone Cloud unless a future feature explicitly requires it.

After successful provisioning:

```text
ESP32
  │
  ▼
Customer Wi-Fi
  │
  ▼
Internet
  │
  ▼
Thermone API
```

The temporary setup network can then be disabled.

---

# Ethernet

Thermone may support Ethernet.

When Ethernet is available:

```text
Power On
   │
   ▼
Ethernet Connected?
   │
   ├── YES
   │     │
   │     ▼
   │    DHCP
   │     │
   │     ▼
   │   Internet
   │
   └── NO
         │
         ▼
      Wi-Fi
```

The setup access point may still be made available for device configuration even when Ethernet provides Internet connectivity.

---

# Bootstrap Firmware

Every Thermone controller must contain bootstrap/recovery-capable firmware before it is shipped.

Bootstrap responsibilities include:

- Hardware initialization
- Factory identity
- Network provisioning
- Wi-Fi setup
- Ethernet detection when supported
- Secure API connection
- Device registration
- OTA firmware installation
- Recovery behavior
- Factory reset

The device must never depend on downloading firmware before it can perform basic recovery and provisioning.

---

# Production Firmware

After registration, the controller may download the latest compatible production firmware.

Production firmware responsibilities include:

- Temperature monitoring
- Probe detection
- Sensor identification
- Telemetry
- Device heartbeats
- Device configuration
- Offline buffering
- Network recovery
- Alert-related local logic
- OTA updates
- Device health reporting

---

# OTA Firmware Updates

Thermone supports Over-the-Air firmware updates.

High-level process:

```text
Controller
    │
    ▼
Firmware Check
    │
    ▼
New Compatible Version?
    │
    ├── NO ──→ Continue Running
    │
    └── YES
          │
          ▼
     Download Firmware
          │
          ▼
     Verify Firmware
          │
          ▼
     Write OTA Partition
          │
          ▼
        Reboot
          │
          ▼
     Validate New Firmware
          │
          ├── SUCCESS → Continue
          │
          └── FAILURE → Rollback
```

Firmware should be verified before activation.

Production firmware distribution should eventually support:

- Version manifests
- Hardware compatibility
- SHA-256 verification
- Cryptographic signing
- Rollback
- Development channel
- Staging channel
- Production channel

---

# Telemetry

The controller periodically uploads temperature readings.

Example:

```json
{
  "device_id": "dev_xxxxx",
  "firmware_version": "1.0.0",
  "wifi_rssi": -54,
  "ports": [
    {
      "port": "A01",
      "sensor_id": "28-FF-64-1D-92-16-03-8C",
      "temperature_c": 26.72,
      "temperature_f": 80.10,
      "status": "online"
    },
    {
      "port": "A02",
      "sensor_id": "28-FF-73-20-A4-16-05-B1",
      "temperature_c": 25.94,
      "temperature_f": 78.69,
      "status": "online"
    }
  ]
}
```

The final telemetry frequency will be determined during firmware testing.

---

# Device Heartbeats

Controllers periodically send health information separately from or alongside telemetry.

A heartbeat may include:

```json
{
  "device_id": "dev_xxxxx",
  "firmware_version": "1.0.0",
  "uptime_seconds": 86400,
  "wifi_rssi": -54,
  "free_heap": 185420,
  "connected_ports": 8,
  "network": "wifi"
}
```

Heartbeats allow Thermone to determine whether a controller is:

- Online
- Offline
- Experiencing weak Wi-Fi
- Rebooting unexpectedly
- Running outdated firmware
- Experiencing hardware problems

---

# Offline Operation

Thermone temperature monitoring must not stop because the Internet is unavailable.

When cloud connectivity is lost, the controller should:

1. Continue reading all connected probes
2. Continue detecting unsafe temperatures
3. Store a limited amount of telemetry locally
4. Attempt network reconnection
5. Maintain device operation
6. Upload buffered readings after connectivity returns

Example:

```text
Internet Online
      │
      ▼
Normal Uploads
      │
      X
Internet Lost
      │
      ▼
Continue Monitoring
      │
      ▼
Store Readings Locally
      │
      ▼
Retry Connection
      │
      ▼
Internet Restored
      │
      ▼
Upload Buffered Data
      │
      ▼
Resume Normal Operation
```

---

# Alerts

Thermone supports configurable temperature alerts.

Example:

```text
Tank A03

Target:
79°F – 82°F

Warning:
Below 79°F for 5 minutes

Critical Low:
Below 76°F

Critical High:
Above 84°F
```

The cloud may generate user notifications.

The controller should eventually support limited local safety detection even when Internet access is unavailable.

---

# Security Principles

Thermone V1 follows these baseline principles:

- HTTPS for Internet communication
- Unique credentials per device
- Separate claim credentials and API credentials
- No production secrets committed to Git
- Device credentials must be revocable
- Server-side authorization
- User ownership validation
- Firmware integrity verification
- Rate limiting
- Secure random token generation
- Passwords and secrets should not be logged
- Sensitive device storage should be encrypted where practical
- Hardware identifiers are not authentication secrets
- Production, staging, and development environments remain separated

A dedicated Thermone security specification will define these requirements in greater detail.

---

# Environments

Thermone uses three primary environments.

## Development

Used for active development.

Example:

```text
api.dev.thermone.com
app.dev.thermone.com
```

---

## Staging

Used for release testing.

Example:

```text
api.staging.thermone.com
app.staging.thermone.com
```

---

## Production

Used by real customer devices.

Example:

```text
api.thermone.com
app.thermone.com
```

Production controllers must never communicate with development infrastructure unless explicitly configured as development hardware.

---

# Repository Architecture

Thermone is divided into six repositories.

```text
Thermone
│
├── firmware
│   └── ESP32 firmware
│
├── api
│   └── Cloud API and backend services
│
├── dashboard
│   └── Customer web application
│
├── provisioning
│   └── Factory provisioning and device claiming
│
├── infrastructure
│   └── Deployment and cloud infrastructure
│
└── docs
    └── Technical documentation
```

---

# V1 Success Criteria

Thermone V1 is considered functionally complete when a customer can:

1. Receive a Thermone controller
2. Power on the controller
3. Connect the controller to a network
4. Complete first-time provisioning
5. Register the controller with Thermone Cloud
6. Scan the controller QR code
7. Create or log into a Thermone account
8. Claim the controller
9. Plug temperature probes into A01–A08
10. Assign ports to tanks
11. View current temperatures remotely
12. View historical temperature data
13. Configure temperature thresholds
14. Receive temperature alerts
15. See controller online/offline status
16. Replace a probe without losing tank history
17. Continue local monitoring during an Internet outage
18. Automatically reconnect after an outage
19. Receive OTA firmware updates
20. Access the dashboard from anywhere with Internet access

---

# Future Expansion

Thermone should be designed so future hardware can support more than temperature monitoring.

Possible future devices and sensors include:

- pH
- Water level
- Leak detection
- Conductivity
- TDS
- Salinity
- Dissolved oxygen
- Room temperature
- Humidity
- Power monitoring
- Heater monitoring
- Pump monitoring
- Lighting control
- Automatic feeding
- Water change automation

These future capabilities should not require redesigning Thermone's core identity, authentication, provisioning, or device-management architecture.