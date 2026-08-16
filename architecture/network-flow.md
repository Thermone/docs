# Thermone V1 Network Flow

## Purpose

This document defines the network communication model for Thermone V1.

The design goals are:

- No direct public exposure of the ESP32
- No customer router port forwarding
- Secure outbound HTTPS communication
- Reliable Wi-Fi provisioning
- Optional Ethernet support
- Clear device-to-cloud responsibilities
- Reliable operation during temporary Internet outages
- Safe OTA firmware delivery
- Remote dashboard access from anywhere

---

# High-Level Network Architecture

```text
                         INTERNET
                            │
                            │
                ┌───────────┴───────────┐
                │                       │
                ▼                       ▼
        Thermone Device API      Thermone Dashboard
                │                       │
                │                       │
                ▼                       ▼
         Thermone Database        User Browser
                ▲
                │
                │ HTTPS
                │
         ┌──────┴──────┐
         │             │
         │ Thermone V1 │
         │ Controller  │
         │   ESP32     │
         └──────┬──────┘
                │
        ┌───────┴────────┐
        │                │
        ▼                ▼
      Wi-Fi           Ethernet
        │                │
        └───────┬────────┘
                │
                ▼
          Customer Router
```

---

# Core Rule

Thermone controllers initiate outbound connections.

Users do not directly connect to deployed ESP32 controllers over the public Internet.

```text
ESP32
  │
  │ Outbound HTTPS
  ▼
Thermone Cloud
```

Not:

```text
Internet
   │
   ▼
Customer Router
   │
 Port Forward
   │
   ▼
ESP32
```

Direct public exposure of the controller is prohibited.

---

# Why Outbound-Only Communication

Outbound-only communication provides several advantages:

- No port forwarding
- No static public IP requirement
- No dynamic DNS requirement
- No public ESP32 web server
- Reduced attack surface
- Easier customer installation
- Works behind typical home routers
- Works behind NAT
- Works on most residential networks

---

# Network Interfaces

Thermone V1 may support two network interfaces.

## Wi-Fi

Wi-Fi is the primary network interface.

Requirements:

- 2.4 GHz Wi-Fi support
- WPA2 support
- DHCP
- DNS
- HTTPS/TLS

Future hardware may support additional Wi-Fi standards.

---

## Ethernet

Ethernet may be supported depending on final V1 hardware.

Ethernet provides:

- More stable connectivity
- Reduced Wi-Fi dependence
- Easier rack installations
- Better performance in high-interference environments

Ethernet should use DHCP by default.

Static IP configuration may be considered later.

---

# Preferred Network Selection

Recommended network priority:

```text
Ethernet available?
       │
       ├── YES
       │     │
       │     ▼
       │   Use Ethernet
       │
       └── NO
             │
             ▼
           Wi-Fi
```

If Ethernet fails, the controller may attempt Wi-Fi as a fallback if Wi-Fi credentials exist.

---

# First Boot Network Flow

A brand-new controller starts with no customer Wi-Fi credentials.

```text
Power On
   │
   ▼
Load Configuration
   │
   ▼
Ethernet Connected?
   │
   ├── YES
   │     │
   │     ▼
   │   DHCP
   │     │
   │     ▼
   │ Internet Available?
   │
   └── NO
         │
         ▼
Stored Wi-Fi Credentials?
         │
         ├── YES
         │     │
         │     ▼
         │ Attempt Wi-Fi
         │
         └── NO
               │
               ▼
         Start Setup AP
```

---

# Setup Access Point

When no usable network connection exists, the controller creates a temporary Wi-Fi access point.

Example SSID:

```text
Thermone-A482
```

The suffix should use a non-secret identifier.

Possible source:

```text
Last 4 characters of serial number
```

Example:

```text
Serial:
THV1-000482

Setup SSID:
Thermone-0482
```

---

# Setup AP Security

The setup network should not be permanently open.

Possible V1 strategies:

- Random setup password printed on device
- Password derived from factory provisioning
- Temporary onboarding password
- Button-triggered setup network

The final strategy will be defined in the security documentation.

---

# Local Setup Portal

The user connects directly to the Thermone setup Wi-Fi.

```text
Phone
  │
  ▼
Thermone-0482
  │
  ▼
ESP32 Local Web Server
```

The setup portal is local.

Example address:

```text
http://192.168.4.1
```

A captive portal may automatically redirect the user.

---

# Wi-Fi Discovery

The setup portal may scan nearby networks.

Example:

```text
Available Networks

FishRoomWiFi
HomeNetwork
OfficeWiFi
```

The user selects a network and enters its password.

---

# Wi-Fi Credential Flow

```text
User Phone
    │
    ▼
Local Setup Portal
    │
    ▼
ESP32
    │
    ▼
Store Credentials Locally
```

Wi-Fi credentials should not be sent to Thermone Cloud.

The cloud does not need the customer's Wi-Fi password.

---

# Wi-Fi Connection Test

After receiving credentials:

```text
Save Credentials
      │
      ▼
Disable / Pause Setup AP
      │
      ▼
Join Customer Wi-Fi
      │
      ▼
DHCP
      │
      ▼
DNS Test
      │
      ▼
Internet Test
      │
      ▼
Thermone API Test
```

---

# Failed Wi-Fi Setup

If connection fails:

```text
Join Wi-Fi
    │
    X
Failure
    │
    ▼
Restart Setup AP
    │
    ▼
Show Error
```

Possible error categories:

- Incorrect password
- SSID unavailable
- Unsupported network
- DHCP failure
- DNS failure
- Internet unavailable
- Thermone API unavailable

---

# Local Network Without Internet

A controller may successfully connect to Wi-Fi but have no Internet access.

```text
Wi-Fi Connected
      │
      ▼
Internet Unavailable
```

The controller should:

- Continue local monitoring
- Keep network credentials
- Retry Internet access
- Avoid repeatedly entering setup mode
- Report connectivity once the cloud becomes available

---

# Cloud Registration Flow

Once Internet access is available:

```text
ESP32
  │
  ▼
DNS Lookup
  │
  ▼
api.thermone.com
  │
  ▼
TLS Connection
  │
  ▼
POST /v1/device/register
```

The registration request uses factory authentication.

---

# Registration Request

Conceptual request:

```http
POST /v1/device/register
Content-Type: application/json
Authorization: Factory <credential>
```

Example body:

```json
{
  "serial_number": "THV1-000001",
  "hardware_revision": "1.0",
  "bootstrap_version": "1.0.0",
  "hardware_id": "ESP32-HARDWARE-ID"
}
```

---

# Registration Response

Example response:

```json
{
  "device_id": "019c3e92-...",
  "device_token": "runtime-device-credential",
  "firmware_channel": "production"
}
```

The controller securely stores the runtime credential.

---

# Runtime Device Communication

After registration, the controller uses runtime authentication.

Example:

```http
Authorization: Bearer <device-token>
```

The factory credential should not be used for routine telemetry.

---

# Main Runtime Connections

The controller may communicate with several logical services.

```text
ESP32
 │
 ├── Device API
 │
 ├── Telemetry API
 │
 ├── Firmware Service
 │
 └── Configuration API
```

These services may initially be hosted behind a single API deployment.

---

# Telemetry Flow

Temperature telemetry follows:

```text
DS18B20 Probes
      │
      ▼
ESP32
      │
      ▼
Read + Validate
      │
      ▼
Create Batch
      │
      ▼
HTTPS POST
      │
      ▼
Thermone API
      │
      ▼
Database
```

---

# Telemetry Endpoint

Example:

```text
POST /v1/device/telemetry
```

Example payload:

```json
{
  "device_id": "019c3e92-...",
  "firmware_version": "1.0.0",
  "ports": [
    {
      "port": "A01",
      "sensor_id": "28-FF-64-1D-92-16-03-8C",
      "temperature_c": 26.72,
      "temperature_f": 80.10,
      "status": "online"
    }
  ]
}
```

---

# Telemetry Timing

Initial V1 target:

```text
Sensor read:
every 5-10 seconds

Cloud telemetry:
every 30 seconds
```

These values are provisional and should be tuned during testing.

Thermone should avoid unnecessary cloud requests while still providing useful live monitoring.

---

# Heartbeat Flow

Heartbeats provide controller health information.

```text
ESP32
  │
  ▼
Collect Health Data
  │
  ▼
POST Heartbeat
  │
  ▼
Thermone API
```

Example endpoint:

```text
POST /v1/device/heartbeat
```

Example heartbeat:

```json
{
  "firmware_version": "1.0.0",
  "uptime_seconds": 86400,
  "free_heap": 185420,
  "wifi_rssi": -54,
  "connected_ports": 7,
  "network": "wifi"
}
```

---

# Heartbeat Interval

Initial recommendation:

```text
Every 60 seconds
```

The final interval may change based on production load and monitoring needs.

---

# Device Online Status

The cloud determines device status based on recent heartbeats.

Example:

```text
Last heartbeat:
20 seconds ago

Status:
Online
```

Possible threshold:

```text
No heartbeat for 3 minutes
        │
        ▼
Mark device offline
```

Exact thresholds are configurable.

---

# Dashboard Network Flow

The dashboard never reads directly from the ESP32.

```text
User Browser
     │
     ▼
app.thermone.com
     │
     ▼
Thermone API / Supabase
     │
     ▼
Database
```

The dashboard displays data stored in or streamed through Thermone Cloud.

---

# Remote Dashboard Access

Because the dashboard is cloud-hosted, a user may access Thermone remotely.

Example:

```text
Fish Room
New York
   │
   ▼
Thermone Controller
   │
   ▼
Thermone Cloud
   │
   ▼
User Traveling
Florida
```

The user only needs:

- Internet access
- Thermone account
- Authorized access to the device

---

# Real-Time Dashboard Updates

Thermone may use:

- Supabase Realtime
- WebSockets
- Server-Sent Events
- Short polling

for dashboard updates.

The controller itself does not need to maintain a direct socket connection to every dashboard user.

---

# Recommended V1 Flow

```text
ESP32
  │
  │ HTTPS POST
  ▼
API
  │
  ▼
PostgreSQL
  │
  ▼
Realtime
  │
  ▼
Dashboard
```

---

# Command Flow

Some dashboard actions may eventually send commands to controllers.

Example:

```text
Restart Controller
```

The user does not directly contact the controller.

Instead:

```text
Dashboard
   │
   ▼
Thermone Cloud
   │
   ▼
Command Stored
   │
   ▼
ESP32 Fetches Command
   │
   ▼
Execute
```

---

# V1 Command Delivery

For V1, command delivery may use periodic polling.

Example:

```text
GET /v1/device/commands
```

every:

```text
15-30 seconds
```

This avoids requiring persistent MQTT or WebSocket infrastructure initially.

---

# Future Command Delivery

Future versions may use:

- MQTT
- Persistent WebSockets
- Push messaging
- Device message broker

These are not required for the first functional V1.

---

# Configuration Flow

Dashboard configuration changes are stored in Thermone Cloud.

Example:

```text
Tank A01
Low warning = 78°F
High warning = 82°F
```

Flow:

```text
Dashboard
   │
   ▼
API
   │
   ▼
Database
   │
   ▼
Device Polls Configuration
   │
   ▼
ESP32 Stores Local Copy
```

---

# Local Configuration Cache

Important device configuration should be cached locally.

Examples:

- Temperature thresholds
- Port configuration
- Tank identifiers
- Reporting interval
- Firmware channel

This allows basic operation during Internet outages.

---

# OTA Network Flow

Firmware updates use HTTPS.

```text
ESP32
  │
  ▼
Firmware Check
  │
  ▼
Thermone API
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
ESP32 Downloads Binary
```

---

# Firmware Check Endpoint

Example:

```text
GET /v1/device/firmware/check
```

Request may contain:

```text
Device model
Hardware revision
Current firmware version
Firmware channel
```

---

# Firmware Manifest

Example:

```json
{
  "update_available": true,
  "current_version": "1.0.0",
  "latest_version": "1.1.0",
  "hardware_revision": "1.0",
  "download_url": "temporary-signed-url",
  "sha256": "firmware-hash",
  "mandatory": false
}
```

---

# OTA Download

The firmware binary should come from dedicated storage.

Example future architecture:

```text
ESP32
  │
  ▼
api.thermone.com
  │
  ▼
Receive Signed URL
  │
  ▼
firmware.thermone.com
  │
  ▼
Download Binary
```

---

# Firmware Download Security

OTA delivery should include:

- HTTPS
- Firmware version validation
- Hardware revision validation
- SHA-256 verification
- Cryptographic signing where supported
- Limited-time download URLs
- Rollback support

---

# Internet Failure During OTA

If connectivity is lost during firmware download:

```text
Download
   │
   X
Connection Lost
   │
   ▼
Abort Update
   │
   ▼
Keep Existing Firmware
```

The currently running firmware must remain bootable.

---

# Offline Operation

Internet connectivity is not required for local sensor monitoring.

```text
Internet Lost
    │
    ▼
Cloud Upload Stops
    │
    ▼
Local Monitoring Continues
    │
    ▼
Read Sensors
    │
    ▼
Evaluate Temperatures
    │
    ▼
Buffer Data
```

---

# Offline Buffer

The controller should store recent telemetry locally.

Possible implementation:

- Ring buffer in flash
- NVS for minimal state
- LittleFS
- SPIFFS alternative where appropriate
- External storage in future versions

The final storage strategy will be defined in firmware documentation.

---

# Buffered Reading Structure

Example:

```json
{
  "timestamp": 1786914000,
  "port": "A01",
  "sensor_id": "28-FF-...",
  "temperature_f": 80.1
}
```

---

# Reconnect Flow

```text
Internet Offline
      │
      ▼
Wait
      │
      ▼
Retry Network
      │
      ▼
Connection Restored
      │
      ▼
Authenticate
      │
      ▼
Upload Buffered Data
      │
      ▼
Resume Live Telemetry
```

---

# Retry Strategy

Use increasing retry intervals.

Example:

```text
Attempt 1:
5 seconds

Attempt 2:
10 seconds

Attempt 3:
30 seconds

Later:
60 seconds
```

Add random jitter where appropriate to prevent many devices from reconnecting simultaneously after a large outage.

---

# API Failure

The local Internet may work while the Thermone API is unavailable.

```text
Internet:
Online

Thermone API:
Unavailable
```

The device should:

- Continue monitoring
- Buffer telemetry
- Retry later
- Avoid discarding existing local configuration
- Avoid factory reset behavior

---

# DNS Failure

If DNS resolution fails:

```text
api.thermone.com
       │
       X
DNS Failure
```

The device should retry DNS and network connectivity.

Hardcoded IP addresses should generally not replace DNS for production service discovery.

---

# TLS Requirements

All device-to-cloud production traffic must use HTTPS/TLS.

Production requests should not use:

```text
http://api.thermone.com
```

Use:

```text
https://api.thermone.com
```

---

# Certificate Validation

The ESP32 must validate the server certificate.

The firmware must not disable TLS validation in production.

Development may use special certificates only when explicitly configured.

---

# Time Synchronization

TLS and accurate telemetry timestamps benefit from correct device time.

After connecting to the Internet:

```text
ESP32
  │
  ▼
NTP
  │
  ▼
UTC Time
```

The controller should operate internally using UTC.

The dashboard converts timestamps to the user's local timezone.

---

# NTP Failure

If NTP is temporarily unavailable:

- Sensor monitoring continues
- Device may use stored time estimates
- Unsynchronized readings should be marked appropriately
- Retry time synchronization later

---

# Local Time

The ESP32 should not depend on local timezone configuration for core telemetry.

Recommended:

```text
ESP32 timestamps = UTC
```

Dashboard:

```text
UTC
 ↓
User timezone
 ↓
Displayed local time
```

---

# Device Discovery on Local Network

Thermone V1 does not require local LAN discovery for normal use.

Possible future technologies include:

- mDNS
- DNS-SD

Example:

```text
thermone-0482.local
```

This may be useful for diagnostics but is not required for cloud operation.

---

# Setup Portal Availability

The setup portal should normally be active only during:

- First-time setup
- Manual setup mode
- Recovery mode
- Wi-Fi reconfiguration

It should not remain permanently exposed.

---

# Setup Mode Trigger

Possible triggers:

```text
No network credentials
```

or:

```text
Physical setup button
```

or:

```text
Repeated network failure
```

The final behavior is defined in firmware documentation.

---

# Factory Reset Network Behavior

After factory reset:

```text
Erase Wi-Fi Configuration
        │
        ▼
Restart
        │
        ▼
Start Thermone Setup AP
```

The permanent factory identity remains intact.

---

# Ownership Is Not Network Configuration

Network provisioning and cloud ownership are separate.

Example:

```text
Wi-Fi configured
```

does not automatically mean:

```text
User owns device
```

Likewise:

```text
Device claimed
```

does not mean the cloud knows the user's Wi-Fi password.

---

# Security Boundaries

The network architecture has three main trust zones.

```text
┌──────────────────┐
│ Device           │
│ ESP32            │
└────────┬─────────┘
         │
         │ TLS
         ▼
┌──────────────────┐
│ Thermone Cloud   │
│ API / Database   │
└────────┬─────────┘
         │
         │ HTTPS
         ▼
┌──────────────────┐
│ User             │
│ Browser / App    │
└──────────────────┘
```

Each boundary requires independent authentication and authorization.

---

# Device Authentication

The controller authenticates using a device credential.

```text
ESP32
  │
  ▼
Device Token
  │
  ▼
Thermone API
```

Users do not receive the device token.

---

# User Authentication

Users authenticate through the dashboard authentication system.

Possible methods:

- Email/password
- Magic link
- Google
- Apple

User sessions do not contain the raw ESP32 device credential.

---

# Authorization

After user authentication, the API checks device ownership.

Example:

```text
User requests:
THV1-000482

API checks:
Does user own this device?

YES
 ↓
Allow

NO
 ↓
Deny
```

---

# Rate Limiting

Thermone APIs should enforce rate limits.

Separate limits may apply to:

- Registration
- Telemetry
- Heartbeats
- Firmware checks
- User dashboard requests
- Claim attempts

Rate limits protect infrastructure from accidental or malicious traffic.

---

# Device Request Identification

Each request should provide enough context to identify:

- Device
- Firmware version
- Hardware revision
- Request type
- Environment

Example headers may eventually include:

```text
X-Thermone-Device
X-Thermone-Firmware
X-Thermone-Hardware
```

Exact headers will be defined in the API documentation.

---

# Network Failure States

Recommended device network states:

```text
NETWORK_UNCONFIGURED

NETWORK_CONNECTING

NETWORK_CONNECTED

INTERNET_UNAVAILABLE

CLOUD_CONNECTING

CLOUD_CONNECTED

CLOUD_UNAVAILABLE

NETWORK_RECOVERY
```

These states help firmware diagnostics.

---

# Status LED Integration

Network state may be represented using the controller status LED.

Possible example:

```text
Blue blinking:
Setup mode

Blue solid:
Network connected

Green:
Thermone Cloud connected

Yellow:
Internet problem

Red:
Device fault
```

Exact LED behavior will be defined in hardware/firmware documentation.

---

# Network Health Metrics

The controller should report useful network metrics.

Wi-Fi examples:

```text
RSSI
SSID identifier if appropriate
Reconnect count
Connection uptime
```

Ethernet examples:

```text
Link status
Negotiated speed
Reconnect count
```

Sensitive network information should not be unnecessarily stored.

---

# Wi-Fi RSSI Example

```text
-40 dBm
Excellent

-60 dBm
Good

-75 dBm
Weak

-85 dBm
Very weak
```

The dashboard may warn users when the controller has persistently poor connectivity.

---

# Dashboard Offline Indication

If cloud heartbeats stop:

```text
Main Fish Room

Controller Offline

Last Seen:
8:42 PM
```

Individual last-known temperatures may still be displayed with a clear timestamp.

Never present stale values as live values.

---

# Last Known Temperature

Example:

```text
Tank A03

79.8°F

Last updated:
17 minutes ago

Controller offline
```

---

# Multi-Controller Support

A customer may have multiple controllers.

```text
User Account
    │
    ├── Main Rack
    │     └── THV1-000482
    │
    ├── Breeding Rack
    │     └── THV1-000483
    │
    └── Grow-Out Room
          └── THV1-000484
```

Each controller establishes its own cloud connection.

---

# Large Installations

Thermone should eventually support many controllers behind the same router.

Example:

```text
Customer Router
   │
   ├── Controller 1
   ├── Controller 2
   ├── Controller 3
   ├── Controller 4
   └── Controller 20
```

Because connections are outbound, no unique router configuration is required for each unit.

---

# Network Scalability

The API should assume that future deployments may include:

- Thousands of controllers
- Tens of thousands of probes
- Frequent telemetry uploads
- Simultaneous reconnect events

This is one reason telemetry should be batched per controller instead of sending one HTTP request per probe.

---

# Recommended Telemetry Batch

Good:

```text
1 controller
8 probes
1 request
```

Avoid:

```text
8 probes
8 independent HTTP requests
every 30 seconds
```

Batching reduces network and server overhead.

---

# Example Complete Runtime Flow

```text
Every 5 Seconds

Read A01
Read A02
Read A03
...
Read A08
      │
      ▼
Update Local State
      │
      ▼
Evaluate Safety


Every 30 Seconds

Build Telemetry Batch
      │
      ▼
Cloud Available?
      │
      ├── YES
      │     │
      │     ▼
      │ POST Telemetry
      │
      └── NO
            │
            ▼
        Buffer Data


Every 60 Seconds

Send Heartbeat


Periodically

Check Configuration
Check Commands
Check OTA Firmware
```

---

# Final Network Architecture

```text
                         THERMONE CLOUD
       ┌────────────────────────────────────────┐
       │                                        │
       │   API                                  │
       │    │                                   │
       │    ├── Registration                    │
       │    ├── Telemetry                       │
       │    ├── Heartbeats                      │
       │    ├── Configuration                   │
       │    ├── Commands                        │
       │    └── Firmware                        │
       │                                        │
       │   PostgreSQL / Supabase                │
       │                                        │
       │   Firmware Storage                     │
       │                                        │
       └───────────────────┬────────────────────┘
                           │
                        Internet
                           │
            ┌──────────────┴──────────────┐
            │                             │
            ▼                             ▼
     Customer Router                 User Device
            │                             │
       ┌────┴────┐                        │
       │         │                        │
       ▼         ▼                        ▼
     Wi-Fi    Ethernet              Web Dashboard
       │         │
       └────┬────┘
            │
            ▼
      Thermone ESP32
            │
        ┌───┼───┐
        │   │   │
        ▼   ▼   ▼
       A01 A02 ... A08
```

---

# Network Design Principles

Thermone networking follows these rules:

1. Controllers initiate outbound Internet connections.
2. ESP32 controllers are not directly exposed publicly.
3. Production device communication uses HTTPS.
4. Wi-Fi passwords remain local to the controller.
5. Cloud ownership and Wi-Fi provisioning are separate.
6. Monitoring continues when Internet access is unavailable.
7. Telemetry is buffered during temporary outages.
8. Production never automatically falls back to development services.
9. Dashboard users interact with Thermone Cloud, not directly with ESP32 hardware.
10. Network failures must not permanently disable temperature monitoring.