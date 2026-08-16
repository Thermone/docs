# Thermone V1 GPIO Map

## Purpose

This document defines the GPIO assignments for the Thermone V1 controller.

The V1 prototype uses an ESP32 development board based on the:

```text
ESP32-WROOM-32 / ESP32-WROOM-32E family
```

Recommended development-board form factor:

```text
ESP32-DevKitC-style board
```

The GPIO map is designed to support:

- 8 independent DS18B20 temperature probe ports
- One physical setup/reset button
- One RGB status LED
- USB serial programming
- Future Ethernet expansion
- Future hardware expansion

---

# Design Principles

Thermone V1 follows these GPIO rules:

1. Every probe receives its own GPIO.
2. Probe power and ground are shared.
3. Every probe data line receives its own 4.7kΩ pull-up resistor.
4. ESP32 boot-strapping pins should be avoided where practical.
5. Flash-connected pins must never be used.
6. USB/UART programming pins remain reserved.
7. Common SPI pins are reserved for possible Ethernet support.
8. Future expansion pins should remain available where practical.

---

# Final V1 Probe GPIO Map

| Thermone Port | ESP32 GPIO | Function |
|---|---:|---|
| A01 | GPIO 13 | DS18B20 Data |
| A02 | GPIO 14 | DS18B20 Data |
| A03 | GPIO 16 | DS18B20 Data |
| A04 | GPIO 17 | DS18B20 Data |
| A05 | GPIO 25 | DS18B20 Data |
| A06 | GPIO 26 | DS18B20 Data |
| A07 | GPIO 27 | DS18B20 Data |
| A08 | GPIO 32 | DS18B20 Data |

This map replaces earlier provisional mappings.

---

# Physical Port Relationship

Each GPIO maps permanently to one physical enclosure connector.

```text
GPIO 13 → A01
GPIO 14 → A02
GPIO 16 → A03
GPIO 17 → A04
GPIO 25 → A05
GPIO 26 → A06
GPIO 27 → A07
GPIO 32 → A08
```

The firmware must never reorder these mappings dynamically.

---

# Why Dedicated GPIOs Are Used

DS18B20 sensors normally support multiple sensors on one 1-Wire bus.

Thermone intentionally does not use that architecture for V1.

Instead:

```text
A01 ─ GPIO 13
A02 ─ GPIO 14
A03 ─ GPIO 16
A04 ─ GPIO 17
A05 ─ GPIO 25
A06 ─ GPIO 26
A07 ─ GPIO 27
A08 ─ GPIO 32
```

This provides direct physical-port identification.

If a sensor is connected to A04, firmware automatically knows:

```text
Physical Port:
A04

GPIO:
17
```

The DS18B20 hardware address then provides a second identity.

Example:

```text
Port:
A04

GPIO:
17

Sensor ID:
28-FF-19-83-C2-16-04-27
```

---

# Probe Electrical Wiring

Every probe uses three conductors:

```text
3.3V
DATA
GND
```

Power and ground are shared.

Data is independent.

---

# A01 Circuit

```text
                    3.3V
                      │
                    4.7kΩ
                      │
                      ├──────────── DATA
                      │
GPIO 13 ──────────────┘
                      │
                      ▼
                A01 Connector
                      │
                      ▼
                  DS18B20
```

---

# A02 Circuit

```text
                    3.3V
                      │
                    4.7kΩ
                      │
                      ├──────────── DATA
                      │
GPIO 14 ──────────────┘
                      │
                      ▼
                A02 Connector
                      │
                      ▼
                  DS18B20
```

The same circuit is repeated for A03 through A08.

---

# Pull-Up Resistors

Every probe data bus receives its own resistor.

| Port | GPIO | Pull-Up |
|---|---:|---:|
| A01 | 13 | 4.7kΩ |
| A02 | 14 | 4.7kΩ |
| A03 | 16 | 4.7kΩ |
| A04 | 17 | 4.7kΩ |
| A05 | 25 | 4.7kΩ |
| A06 | 26 | 4.7kΩ |
| A07 | 27 | 4.7kΩ |
| A08 | 32 | 4.7kΩ |

All resistors connect:

```text
DATA → 4.7kΩ → 3.3V
```

---

# Shared 3.3V Bus

All temperature probes share the ESP32 3.3V supply.

```text
ESP32 3V3
   │
   ├── A01 Pin 1
   ├── A02 Pin 1
   ├── A03 Pin 1
   ├── A04 Pin 1
   ├── A05 Pin 1
   ├── A06 Pin 1
   ├── A07 Pin 1
   └── A08 Pin 1
```

---

# Shared Ground Bus

All probes share ground.

```text
ESP32 GND
   │
   ├── A01 Pin 3
   ├── A02 Pin 3
   ├── A03 Pin 3
   ├── A04 Pin 3
   ├── A05 Pin 3
   ├── A06 Pin 3
   ├── A07 Pin 3
   └── A08 Pin 3
```

---

# M8 Connector Pin Standard

Thermone V1 uses one consistent connector pin standard.

| M8 Pin | Function |
|---|---|
| Pin 1 | 3.3V |
| Pin 2 | DATA |
| Pin 3 | GND |

Never change this assignment between ports.

---

# Complete Probe Wiring

```text
                     ESP32

3.3V ──────────────────────────────────────────┐
                                              │
GND ────────────────────────────────────────┐ │
                                           │ │
GPIO 13 ───────── A01 DATA                 │ │
GPIO 14 ───────── A02 DATA                 │ │
GPIO 16 ───────── A03 DATA                 │ │
GPIO 17 ───────── A04 DATA                 │ │
GPIO 25 ───────── A05 DATA                 │ │
GPIO 26 ───────── A06 DATA                 │ │
GPIO 27 ───────── A07 DATA                 │ │
GPIO 32 ───────── A08 DATA                 │ │
                                           │ │
                                           │ │
               ┌───────────────────────────┘ │
               │                             │
               │ GND                         │ 3.3V
               ▼                             ▼

              A01                           A01
              A02                           A02
              A03                           A03
              A04                           A04
              A05                           A05
              A06                           A06
              A07                           A07
              A08                           A08
```

---

# Setup / Reset Button

Recommended GPIO:

```text
GPIO 33
```

Assignment:

| Component | GPIO |
|---|---:|
| Setup / Reset Button | GPIO 33 |

The button should connect between:

```text
GPIO 33
   │
   │
 Button
   │
   ▼
  GND
```

Firmware should configure GPIO 33 using an internal pull-up.

Conceptually:

```cpp
pinMode(33, INPUT_PULLUP);
```

Normal state:

```text
HIGH
```

Button pressed:

```text
LOW
```

---

# Proposed Button Actions

Firmware may interpret button duration as:

| Button Action | Result |
|---|---|
| Short press | Status / identify |
| Hold ~5 seconds | Wi-Fi setup mode |
| Hold ~10 seconds | Factory reset |

Final timing will be defined in firmware documentation.

---

# Status LED

Recommended GPIO:

```text
GPIO 4
```

Thermone should use one addressable RGB LED if possible.

Example:

```text
WS2812B / NeoPixel-style RGB LED
```

Assignment:

| Component | GPIO |
|---|---:|
| Status RGB LED | GPIO 4 |

Using an addressable LED allows:

- Red
- Green
- Blue
- Yellow
- Purple
- White
- Flashing patterns

while consuming only one ESP32 GPIO.

---

# Proposed LED States

| Device State | LED |
|---|---|
| Booting | Blue pulse |
| Provisioning | Blue blink |
| Wi-Fi connecting | Yellow blink |
| Cloud connected | Green |
| Cloud unavailable | Yellow |
| OTA update | Purple |
| Hardware error | Red |
| Factory reset warning | Fast red blink |

The final LED state machine belongs in firmware documentation.

---

# USB Serial Pins

The following GPIOs remain reserved for UART/programming behavior:

```text
GPIO 1
GPIO 3
```

Typical usage:

| GPIO | Function |
|---|---|
| GPIO 1 | UART TX |
| GPIO 3 | UART RX |

Thermone V1 must not use these for temperature probes.

---

# Flash-Connected Pins

Do not use the following GPIOs on standard ESP32-WROOM modules:

```text
GPIO 6
GPIO 7
GPIO 8
GPIO 9
GPIO 10
GPIO 11
```

These are associated with the module's SPI flash interface.

They must remain untouched.

---

# Input-Only Pins

The original ESP32 provides input-only GPIOs including:

```text
GPIO 34
GPIO 35
GPIO 36
GPIO 39
```

These are not selected for DS18B20 ports because 1-Wire communication requires bidirectional GPIO operation.

They may be useful for future input-only sensors.

---

# Boot-Strapping Pins

Some ESP32 GPIOs affect boot configuration.

Thermone should avoid depending on them for external devices where practical.

Pins requiring additional care include:

```text
GPIO 0
GPIO 2
GPIO 5
GPIO 12
GPIO 15
```

These are not assigned to Thermone temperature probe ports.

---

# Reserved Ethernet Pins

Thermone V1 keeps the common VSPI pins available for possible Ethernet hardware.

Reserved:

```text
GPIO 18
GPIO 19
GPIO 23
```

Potential future SPI assignments:

| Function | GPIO |
|---|---:|
| SPI SCLK | GPIO 18 |
| SPI MISO | GPIO 19 |
| SPI MOSI | GPIO 23 |

Ethernet chip-select and interrupt pins will be selected after the Ethernet module is chosen.

---

# Ethernet Reservation

Possible future architecture:

```text
ESP32
 │
 ├── GPIO 18 → SPI Clock
 ├── GPIO 19 → SPI MISO
 ├── GPIO 23 → SPI MOSI
 │
 └── Ethernet Controller
```

Possible Ethernet controllers include future supported SPI Ethernet hardware.

The exact module is not yet part of the mandatory V1 specification.

---

# Additional Reserved GPIO

GPIO 22 should remain available for future expansion.

Potential uses include:

- Ethernet control
- I2C
- Expansion sensor
- Additional status hardware

GPIO 21 may also be reserved for future I2C operation.

Recommended future I2C convention:

```text
SDA → GPIO 21
SCL → GPIO 22
```

This means Thermone retains a standard I2C expansion interface.

---

# Current GPIO Allocation

| GPIO | Assignment |
|---:|---|
| 0 | Reserved / boot |
| 1 | UART TX |
| 2 | Reserved / boot |
| 3 | UART RX |
| 4 | RGB Status LED |
| 5 | Reserved / boot / future |
| 6–11 | Flash – DO NOT USE |
| 12 | Reserved / boot |
| 13 | Probe A01 |
| 14 | Probe A02 |
| 15 | Reserved / boot |
| 16 | Probe A03 |
| 17 | Probe A04 |
| 18 | Reserved SPI |
| 19 | Reserved SPI |
| 21 | Reserved I2C |
| 22 | Reserved I2C |
| 23 | Reserved SPI |
| 25 | Probe A05 |
| 26 | Probe A06 |
| 27 | Probe A07 |
| 32 | Probe A08 |
| 33 | Setup / Reset Button |
| 34 | Future input |
| 35 | Future input |
| 36 | Future input |
| 39 | Future input |

---

# Thermone V1 GPIO Summary

```text
TEMPERATURE PORTS

A01 → GPIO 13
A02 → GPIO 14
A03 → GPIO 16
A04 → GPIO 17
A05 → GPIO 25
A06 → GPIO 26
A07 → GPIO 27
A08 → GPIO 32


CONTROL

RGB LED → GPIO 4

Setup / Reset Button → GPIO 33


RESERVED FOR SPI / ETHERNET

GPIO 18
GPIO 19
GPIO 23


RESERVED FOR I2C EXPANSION

GPIO 21
GPIO 22


UART / PROGRAMMING

GPIO 1
GPIO 3
```

---

# Firmware Constants

Firmware should centralize pin assignments rather than scattering GPIO numbers throughout the code.

Example:

```cpp
constexpr uint8_t PIN_PROBE_A01 = 13;
constexpr uint8_t PIN_PROBE_A02 = 14;
constexpr uint8_t PIN_PROBE_A03 = 16;
constexpr uint8_t PIN_PROBE_A04 = 17;

constexpr uint8_t PIN_PROBE_A05 = 25;
constexpr uint8_t PIN_PROBE_A06 = 26;
constexpr uint8_t PIN_PROBE_A07 = 27;
constexpr uint8_t PIN_PROBE_A08 = 32;

constexpr uint8_t PIN_STATUS_LED = 4;
constexpr uint8_t PIN_SETUP_BUTTON = 33;
```

Preferably these values eventually live inside a board-specific configuration file.

Example:

```text
boards/
└── thermone_v1/
    └── pins.h
```

---

# Firmware Port Structure

Firmware should represent physical ports using a fixed table.

Example:

```cpp
struct ProbePort {
    const char* id;
    uint8_t gpio;
};

constexpr ProbePort PROBE_PORTS[] = {
    {"A01", 13},
    {"A02", 14},
    {"A03", 16},
    {"A04", 17},
    {"A05", 25},
    {"A06", 26},
    {"A07", 27},
    {"A08", 32}
};
```

This keeps the physical port identity explicit.

---

# Expected Telemetry

The GPIO itself does not need to be exposed to customers.

Internally, telemetry may contain:

```json
{
  "port": "A03",
  "gpio": 16,
  "sensor_id": "28-FF-19-83-C2-16-04-27",
  "temperature_f": 80.4
}
```

The dashboard should normally display:

```text
A03
80.4°F
```

rather than:

```text
GPIO 16
```

GPIO information is primarily useful for:

- Diagnostics
- Engineering
- Support
- Hardware debugging

---

# Probe Replacement Detection

Because each physical connector has a dedicated GPIO, replacement detection is straightforward.

Example:

```text
A05
GPIO 25

Previous Sensor:
28-FF-AAA...

New Sensor:
28-FF-BBB...
```

Firmware reports:

```text
port = A05
sensor_changed = true
```

The cloud can then preserve the tank assignment while updating the physical sensor record.

---

# Fault Isolation

Independent GPIO buses provide fault isolation.

Example:

```text
A03 shorted / disconnected
```

should not prevent:

```text
A01
A02
A04
A05
A06
A07
A08
```

from continuing to operate.

Firmware must process each port independently.

---

# Boot Behavior

During startup:

1. Configure status LED.
2. Configure setup button.
3. Initialize each probe GPIO.
4. Scan each independent 1-Wire bus.
5. Record connected sensor IDs.
6. Start networking.
7. Start cloud communication.

A sensor fault must not prevent the ESP32 from completing boot.

---

# GPIO Validation Test

Before finalizing V1 hardware, perform a physical validation test.

For every port:

```text
Connect DS18B20
      │
      ▼
Detect sensor
      │
      ▼
Read ROM ID
      │
      ▼
Read temperature
      │
      ▼
Disconnect
      │
      ▼
Detect removal
      │
      ▼
Reconnect
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

# Cross-Port Validation

After individual testing, connect eight sensors simultaneously.

Verify:

- All eight sensors are detected.
- Each port reports the correct sensor.
- Temperature readings remain stable.
- Disconnecting one sensor affects only that port.
- Wi-Fi remains stable.
- Status LED remains functional.
- Button remains functional.

---

# Pin Assignment Change Policy

Once physical V1 hardware is manufactured, GPIO assignments must not be changed silently.

If a future revision changes the pin map:

```text
Hardware V1.0
```

and:

```text
Hardware V1.1
```

must have independent board configurations.

Example:

```text
boards/
├── thermone_v1_0/
│   └── pins.h
│
└── thermone_v1_1/
    └── pins.h
```

Firmware selects the correct mapping based on hardware revision.

---

# Final V1 Pin Map

```text
┌────────────────────────────────────────────┐
│                ESP32                      │
│                                            │
│ GPIO 13 ───────── A01                     │
│ GPIO 14 ───────── A02                     │
│ GPIO 16 ───────── A03                     │
│ GPIO 17 ───────── A04                     │
│ GPIO 25 ───────── A05                     │
│ GPIO 26 ───────── A06                     │
│ GPIO 27 ───────── A07                     │
│ GPIO 32 ───────── A08                     │
│                                            │
│ GPIO 4  ───────── RGB STATUS              │
│ GPIO 33 ───────── SETUP / RESET           │
│                                            │
│ GPIO 18 ─┐                                │
│ GPIO 19 ─┼── RESERVED SPI / ETHERNET      │
│ GPIO 23 ─┘                                │
│                                            │
│ GPIO 21 ─┐                                │
│ GPIO 22 ─┴── RESERVED I2C                 │
│                                            │
│ GPIO 1/3 ─── UART                         │
└────────────────────────────────────────────┘
```

---

# V1 GPIO Rule

The Thermone V1 firmware must treat the physical port identifier as the primary logical interface.

Application code should work with:

```text
A01
A02
A03
...
A08
```

rather than directly depending on raw GPIO numbers.

The GPIO mapping belongs to the hardware abstraction layer.