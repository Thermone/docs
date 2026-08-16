# Thermone V1 Controller Hardware Specification

## Purpose

This document defines the hardware architecture for the first Thermone V1 temperature-monitoring controller.

The V1 controller is intended to monitor up to **8 aquarium temperature probes** using an ESP32-based controller.

The design goals are:

- Compact size
- Easy installation
- Replaceable external temperature probes
- Independent identification of every probe port
- Wi-Fi connectivity
- Optional Ethernet support
- USB power
- Simple field servicing
- Low manufacturing complexity
- Compatibility with future Thermone firmware and cloud services

---

# V1 Hardware Goals

Thermone V1 must provide:

- 8 physical temperature probe ports
- 1 DS18B20 probe per port
- 1 independent GPIO data line per probe
- Shared 3.3V power
- Shared ground
- External quick-disconnect probe connectors
- ESP32 Wi-Fi connectivity
- USB power input
- Status LED
- Setup/reset button
- OTA firmware support
- Local configuration storage
- Compact enclosure

---

# Controller Overview

```text
┌───────────────────────────────────────────┐
│              THERMONE V1                 │
│                                           │
│  Status LED                               │
│      ●                                    │
│                                           │
│  Setup / Reset Button                     │
│      ○                                    │
│                                           │
│  ESP32                                    │
│                                           │
│  A01  A02  A03  A04  A05  A06  A07  A08 │
│   ○    ○    ○    ○    ○    ○    ○    ○  │
│                                           │
│                  USB Power                │
└───────────────────────────────────────────┘
```

---

# ESP32 Controller

Thermone V1 uses an ESP32 development board for the first hardware revision.

The exact board must provide:

- Wi-Fi
- Sufficient GPIO pins
- USB programming capability
- Non-volatile flash storage
- OTA support
- Stable 3.3V output
- Adequate RAM and flash
- Reliable availability

---

# Initial ESP32 Candidate

For the first functional prototype, a standard ESP32 DevKit-style board may be used.

Example class:

```text
ESP32-WROOM-32 DevKit
```

The exact vendor and revision must be documented before production.

---

# Development Board vs Custom PCB

Thermone V1 prototype uses:

```text
Standard ESP32 development board
```

Thermone V1 does not initially require a custom ESP32 PCB.

This allows:

- Faster prototyping
- Easier replacement
- Easier firmware development
- Lower initial engineering cost
- USB programming without custom hardware

A custom PCB may be introduced in a later hardware revision.

---

# Probe Architecture

Thermone V1 supports 8 independent DS18B20 temperature probe ports.

Each physical port has its own dedicated 1-Wire bus.

This means:

```text
A01 → Independent GPIO
A02 → Independent GPIO
A03 → Independent GPIO
A04 → Independent GPIO
A05 → Independent GPIO
A06 → Independent GPIO
A07 → Independent GPIO
A08 → Independent GPIO
```

This differs from the common DS18B20 shared-bus configuration.

---

# Why Independent Probe Buses

Although DS18B20 supports multiple sensors on one shared data line, Thermone V1 intentionally uses separate data lines.

Benefits include:

- Physical port directly identifies the tank connection
- Easier installation
- Easier troubleshooting
- Easier probe replacement
- Failure isolation
- Cleaner device software
- Cleaner dashboard representation
- No ambiguity during setup

---

# Physical Port Naming

Ports are permanently identified as:

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

These names are used consistently across:

- Enclosure labels
- Firmware
- API
- Database
- Dashboard
- Documentation

---

# Provisional GPIO Mapping

Initial proposed GPIO assignments:

| Port | GPIO |
|---|---:|
| A01 | GPIO 4 |
| A02 | GPIO 16 |
| A03 | GPIO 17 |
| A04 | GPIO 18 |
| A05 | GPIO 19 |
| A06 | GPIO 21 |
| A07 | GPIO 22 |
| A08 | GPIO 23 |

This mapping is provisional.

The final mapping must be validated against the exact ESP32 board before hardware is finalized.

---

# Probe Electrical Interface

Each DS18B20 probe requires:

```text
3.3V
DATA
GND
```

Each port receives:

- Shared 3.3V supply
- Shared ground
- Independent data line
- Independent 4.7kΩ pull-up resistor

---

# Per-Port Circuit

```text
                    3.3V
                      │
                      │
                    4.7kΩ
                      │
                      ├──────── DATA
                      │
ESP32 GPIO ───────────┘
                      │
                      ▼
                  M8 Connector
                      │
                      ▼
                   DS18B20
```

---

# Shared Power Architecture

The eight probes may share power and ground.

```text
ESP32 3.3V
    │
    ├── A01 VCC
    ├── A02 VCC
    ├── A03 VCC
    ├── A04 VCC
    ├── A05 VCC
    ├── A06 VCC
    ├── A07 VCC
    └── A08 VCC
```

Ground follows the same arrangement.

---

# Pull-Up Resistors

Each data line requires its own pull-up resistor.

Recommended initial value:

```text
4.7kΩ
```

Example:

```text
A01 DATA → 4.7kΩ → 3.3V
A02 DATA → 4.7kΩ → 3.3V
A03 DATA → 4.7kΩ → 3.3V
...
A08 DATA → 4.7kΩ → 3.3V
```

This means Thermone V1 requires:

```text
8 × 4.7kΩ resistors
```

---

# Temperature Probes

Initial sensor type:

```text
DS18B20
```

Requirements:

- Waterproof probe
- Stainless steel probe housing
- 3-wire configuration
- Unique 64-bit device address
- Suitable cable length
- Aquarium-safe external construction

The exact probe model will be selected and validated during prototype testing.

---

# Probe Hardware ID

Every DS18B20 contains a unique hardware address.

Example:

```text
28-FF-64-1D-92-16-03-8C
```

Thermone reports both:

```text
Port: A01
Sensor ID: 28-FF-64-1D-92-16-03-8C
```

This allows the system to detect sensor replacement.

---

# Probe Connector

Thermone V1 should use external connectors so the enclosure does not need to be opened when a probe is installed or replaced.

Initial preferred connector:

```text
3-pin M8 waterproof circular connector
```

Required conductors:

```text
VCC
DATA
GND
```

---

# Connector Pin Standard

Thermone should define one permanent connector pin standard.

Proposed mapping:

| Connector Pin | Function |
|---|---|
| Pin 1 | 3.3V |
| Pin 2 | DATA |
| Pin 3 | GND |

This standard must remain consistent across all Thermone probes and controllers.

---

# Connector Polarity Protection

Connector orientation should prevent accidental reverse insertion.

A keyed connector is strongly preferred.

If possible, future revisions should include electrical protection against:

- Incorrect probe wiring
- Static discharge
- Short circuits
- Accidental voltage injection

---

# Probe Port Layout

Recommended physical layout:

```text
A01  A02  A03  A04
 ○    ○    ○    ○

A05  A06  A07  A08
 ○    ○    ○    ○
```

or:

```text
A01 A02 A03 A04 A05 A06 A07 A08
 ○   ○   ○   ○   ○   ○   ○   ○
```

The final arrangement depends on enclosure size.

---

# USB Power

Thermone V1 is powered using USB.

Preferred input:

```text
USB-C
```

Nominal supply:

```text
5V
```

Recommended adapter:

```text
5V / 2A
```

The final required current will be measured during prototype testing.

---

# Power Flow

```text
USB-C
  │
  ▼
5V Input
  │
  ▼
ESP32 Development Board
  │
  ▼
Onboard 3.3V Regulation
  │
  ├── ESP32
  └── DS18B20 Probes
```

---

# Power Budget

DS18B20 sensors consume very little current.

The primary power load is the ESP32.

The design should still include adequate margin for:

- Wi-Fi transmission peaks
- Status LED
- Ethernet module if installed
- Future peripherals

---

# Ethernet

Ethernet is optional for Thermone V1.

If included, it may use:

- ESP32 Ethernet-capable development board
- External SPI Ethernet module
- External PHY supported by the ESP32

Ethernet support must not prevent Wi-Fi operation.

---

# Network Priority

If both Ethernet and Wi-Fi are available:

```text
Ethernet
   │
   ├── Available → Prefer Ethernet
   │
   └── Unavailable → Use Wi-Fi
```

Wi-Fi credentials may remain stored as a fallback.

---

# Status LED

Thermone V1 includes one externally visible status indicator.

Preferred implementation:

```text
RGB LED
```

The LED should be visible without opening the enclosure.

---

# Proposed LED States

| State | Indication |
|---|---|
| Booting | Blue pulse |
| Setup mode | Blue blinking |
| Connecting | Yellow blinking |
| Cloud connected | Green |
| Cloud unavailable | Yellow |
| Firmware update | Purple / blue animation |
| Hardware fault | Red |
| Factory reset warning | Red flashing |

Exact behavior will be defined in firmware documentation.

---

# Setup / Reset Button

Thermone V1 includes one physical button.

Primary functions:

- Enter network setup mode
- Reset Wi-Fi credentials
- Trigger factory reset
- Enter recovery mode if required

---

# Proposed Button Behavior

Example:

```text
Short press
→ status / setup action

Hold 5 seconds
→ network setup mode

Hold 10 seconds
→ factory reset
```

Exact timing will be validated during firmware testing.

---

# Button Placement

The button should be:

- Accessible without opening the enclosure
- Protected from accidental presses
- Recessed where practical
- Water resistant

Possible implementation:

```text
Recessed tactile button
```

or:

```text
Sealed panel-mount pushbutton
```

---

# Enclosure

Thermone V1 requires a compact protective enclosure.

Primary goals:

- Protect ESP32 electronics
- Protect wiring
- Allow external probe connection
- Allow USB power access
- Provide status LED visibility
- Allow setup/reset access
- Mount near aquarium racks

---

# Initial Enclosure Target

Prototype target:

```text
Approximately:
120–160 mm wide
70–100 mm tall
35–50 mm deep
```

The exact size depends heavily on the final M8 connector layout.

---

# Enclosure Material

Preferred:

```text
ABS
```

or:

```text
Polycarbonate
```

Requirements:

- Non-conductive
- Drillable
- Durable
- Suitable for indoor aquarium environments

---

# Water Resistance

The enclosure should protect against:

- Splashing
- Humidity
- Accidental water contact

Target:

```text
IP54 minimum
```

A higher rating is preferred if practical.

The controller is not intended for underwater use.

---

# Connector Water Resistance

Probe connectors should ideally be:

```text
IP67
```

or better when properly connected.

Unused ports may require protective caps.

---

# Internal Layout

Prototype internal layout:

```text
┌──────────────────────────────────┐
│                                  │
│        ESP32 Development Board   │
│                                  │
│        Pull-Up Resistors         │
│        / Distribution Area       │
│                                  │
│  A01 A02 A03 A04                 │
│  A05 A06 A07 A08                 │
│                                  │
└──────────────────────────────────┘
```

---

# Internal Wiring

Prototype V1 may use:

- Hookup wire
- Perfboard
- Soldered connections
- Crimp connectors
- Small terminal blocks

A custom PCB is not required for the first functional prototype.

---

# Internal Distribution Board

A small perfboard may distribute:

```text
3.3V
GND
8 independent data lines
8 pull-up resistors
```

Example:

```text
┌─────────────────────────────┐
│ 3.3V BUS                    │
│ GND BUS                     │
│                             │
│ R1 → A01 DATA               │
│ R2 → A02 DATA               │
│ R3 → A03 DATA               │
│ R4 → A04 DATA               │
│ R5 → A05 DATA               │
│ R6 → A06 DATA               │
│ R7 → A07 DATA               │
│ R8 → A08 DATA               │
└─────────────────────────────┘
```

---

# Prototype Serviceability

The V1 prototype should allow replacement of:

- ESP32 board
- Individual temperature probes
- USB cable
- Status LED
- Internal distribution board

The enclosure may need to be opened for internal electronics repair.

Probe replacement must not require opening the enclosure.

---

# Mounting

The controller should support flexible mounting.

Possible options:

- Screw slots
- Adhesive mounting
- Velcro
- Magnetic accessory plate
- Rack clip
- Wall mounting

The final enclosure may incorporate mounting ears or slots.

---

# Environmental Placement

Recommended installation:

```text
Near aquarium rack
Above expected water line
Away from direct splash
Accessible for service
```

Do not install:

- Underwater
- Inside aquarium
- Directly beneath leaking plumbing
- Where condensation can continuously enter connectors

---

# Cable Management

Probe cables should be labeled to match physical ports.

Example:

```text
A01
A02
A03
...
A08
```

The dashboard should match these exact labels.

---

# Tank Identification

Thermone controller port labels identify the physical connection.

Example:

```text
A03
```

The dashboard may assign:

```text
A03 → Betta Breeding Tank 3
```

The physical controller itself does not need to know the human-readable tank name to measure temperature.

---

# QR Code Label

Each Thermone controller receives a device label.

Example:

```text
THERMONE

Model:
THV1

Serial:
THV1-000001

[ QR CODE ]

Scan to set up
```

The QR code contains the Thermone claim URL.

---

# Device Label

Recommended information:

- Thermone brand
- Model
- Serial number
- Hardware revision
- QR code
- Power rating
- Regulatory markings when applicable

Never print:

- Device API secret
- Factory credential
- Wi-Fi password
- Private key

---

# Hardware Revision

Every hardware design receives a revision.

Initial:

```text
Hardware Revision:
V1.0
```

Future examples:

```text
V1.1
V2.0
```

Firmware must report hardware revision to Thermone Cloud.

---

# Prototype Hardware ID

The ESP32 hardware identity should also be recorded.

This may include a chip-derived identifier.

It is used for:

- Registration validation
- Support
- Hardware replacement detection

It must not be treated as an authentication secret.

---

# Local Storage

The ESP32 must have sufficient flash storage for:

- Bootstrap firmware
- Production firmware
- OTA partition
- Configuration
- Device credentials
- Wi-Fi configuration
- Limited offline telemetry

Exact partition layout will be defined in firmware documentation.

---

# OTA Requirement

The selected ESP32 board must support an OTA partition arrangement.

Conceptually:

```text
Bootloader
Partition Table
NVS
OTA Slot A
OTA Slot B
Offline Data
```

Exact sizes depend on final firmware size and flash capacity.

---

# Prototype Safety

Because Thermone is installed near water:

- Use a reputable USB power adapter
- Keep AC connections away from water
- Keep enclosure above water level
- Use strain relief
- Protect exposed conductors
- Insulate solder joints
- Use waterproof probe connectors where practical
- Use drip loops on cables

---

# V1 Hardware Scope

Included in V1:

```text
ESP32
8 temperature ports
DS18B20 support
Wi-Fi
USB power
Status LED
Setup/reset button
External probe connectors
Protective enclosure
```

Optional:

```text
Ethernet
```

---

# Not Included in V1

The first Thermone controller does not require:

- pH measurement
- Salinity measurement
- Water level sensors
- Heater switching
- Pump switching
- Lighting control
- Built-in display
- Touchscreen
- Cellular connectivity
- Battery backup
- Custom ESP32 PCB

These may be added in future hardware products.

---

# Future Hardware Expansion

Future Thermone hardware may support:

- More temperature channels
- pH probes
- Leak sensors
- Water-level sensors
- Power monitoring
- Heater monitoring
- Relay outputs
- Pump monitoring
- Humidity
- Room temperature
- Conductivity
- Dissolved oxygen
- Cellular backup
- Power-loss detection

The V1 cloud architecture should remain compatible with future device types.

---

# Prototype Build Target

The first functional hardware prototype should prove:

1. ESP32 boots reliably
2. All 8 GPIO buses work independently
3. Eight DS18B20 probes can be read
4. Sensor IDs can be detected
5. Physical ports are correctly mapped
6. Wi-Fi works reliably
7. Telemetry can reach Thermone Cloud
8. Setup mode works
9. OTA works
10. Probe replacement is detected
11. Status LED works
12. Setup/reset button works

---

# V1 Prototype Definition

The first complete Thermone prototype is:

```text
1 × ESP32 development board

8 × DS18B20 temperature probes

8 × 3-pin waterproof panel connectors

8 × matching probe connectors

8 × 4.7kΩ resistors

1 × status LED

1 × setup/reset button

1 × USB power input

1 × enclosure

1 × internal distribution/perfboard
```

---

# Final Hardware Principle

Thermone V1 should remain simple.

The controller's job is to:

```text
Identify Probe
      │
      ▼
Measure Temperature
      │
      ▼
Know Physical Port
      │
      ▼
Send Reliable Data
      │
      ▼
Remain Recoverable
```

Complex business logic, account management, long-term history, and remote visualization belong in Thermone Cloud rather than the physical controller.