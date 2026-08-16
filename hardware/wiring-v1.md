# Thermone V1 Wiring Guide

## Purpose

This document defines the physical wiring for the Thermone V1 prototype.

The prototype uses:

- 1 ESP32 DevKit-style board
- 8 independent DS18B20 temperature probes
- 8 waterproof 3-pin external connectors
- 8 individual 4.7kΩ pull-up resistors
- 1 RGB status LED
- 1 setup/reset button
- USB power
- Shared 3.3V and ground buses

The main design rule is:

```text
Every temperature probe has its own DATA GPIO.

All probes share 3.3V and GND.
```

---

# Complete V1 Connection Summary

## Temperature Ports

| Port | ESP32 GPIO | Connector Pin 1 | Connector Pin 2 | Connector Pin 3 |
|---|---:|---|---|---|
| A01 | GPIO 13 | 3.3V | DATA | GND |
| A02 | GPIO 14 | 3.3V | DATA | GND |
| A03 | GPIO 16 | 3.3V | DATA | GND |
| A04 | GPIO 17 | 3.3V | DATA | GND |
| A05 | GPIO 25 | 3.3V | DATA | GND |
| A06 | GPIO 26 | 3.3V | DATA | GND |
| A07 | GPIO 27 | 3.3V | DATA | GND |
| A08 | GPIO 32 | 3.3V | DATA | GND |

---

# Connector Standard

Every Thermone temperature port uses the same pin arrangement.

```text
Pin 1 = 3.3V
Pin 2 = DATA
Pin 3 = GND
```

This standard must remain identical across all eight ports.

---

# Recommended Internal Wire Colors

Use consistent wire colors inside every controller.

```text
RED     = 3.3V
YELLOW  = DATA
BLACK   = GND
```

If other colors are used, document them before assembly.

---

# High-Level Wiring

```text
                         ESP32
                 ┌─────────────────┐
                 │                 │
3.3V BUS ◄───────┤ 3V3             │
                 │                 │
GND BUS ◄────────┤ GND             │
                 │                 │
A01 DATA ◄───────┤ GPIO 13         │
A02 DATA ◄───────┤ GPIO 14         │
A03 DATA ◄───────┤ GPIO 16         │
A04 DATA ◄───────┤ GPIO 17         │
A05 DATA ◄───────┤ GPIO 25         │
A06 DATA ◄───────┤ GPIO 26         │
A07 DATA ◄───────┤ GPIO 27         │
A08 DATA ◄───────┤ GPIO 32         │
                 │                 │
STATUS LED ◄─────┤ GPIO 4          │
BUTTON ◄─────────┤ GPIO 33         │
                 └─────────────────┘
```

---

# Shared Power Bus

All probe VCC connections share one 3.3V bus.

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

Use a clean distribution point such as:

- Perfboard
- Soldered bus wire
- Small terminal strip

Do not twist eight wires directly onto the ESP32 header pin.

---

# Shared Ground Bus

All probe grounds share the ESP32 ground.

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

The RGB LED and setup button also use the common ground.

---

# Independent Data Wiring

Each temperature connector has its own DATA wire.

```text
GPIO 13 ───────── A01 Pin 2
GPIO 14 ───────── A02 Pin 2
GPIO 16 ───────── A03 Pin 2
GPIO 17 ───────── A04 Pin 2
GPIO 25 ───────── A05 Pin 2
GPIO 26 ───────── A06 Pin 2
GPIO 27 ───────── A07 Pin 2
GPIO 32 ───────── A08 Pin 2
```

No DATA lines are connected together.

---

# Pull-Up Resistors

Each DATA line gets one 4.7kΩ resistor to 3.3V.

## A01

```text
3.3V
  │
  │
4.7kΩ
  │
  ├──────── A01 DATA
  │
GPIO 13
```

## A02

```text
3.3V
  │
  │
4.7kΩ
  │
  ├──────── A02 DATA
  │
GPIO 14
```

Repeat the same pattern through A08.

---

# Complete Pull-Up Layout

```text
3.3V BUS
   │
   ├── 4.7kΩ ─── A01 DATA ─── GPIO 13
   │
   ├── 4.7kΩ ─── A02 DATA ─── GPIO 14
   │
   ├── 4.7kΩ ─── A03 DATA ─── GPIO 16
   │
   ├── 4.7kΩ ─── A04 DATA ─── GPIO 17
   │
   ├── 4.7kΩ ─── A05 DATA ─── GPIO 25
   │
   ├── 4.7kΩ ─── A06 DATA ─── GPIO 26
   │
   ├── 4.7kΩ ─── A07 DATA ─── GPIO 27
   │
   └── 4.7kΩ ─── A08 DATA ─── GPIO 32
```

---

# A01 Full Wiring

```text
ESP32 3V3
   │
   ├─────────────── A01 Connector Pin 1
   │
   └── 4.7kΩ ──┐
                │
GPIO 13 ────────┴──────── A01 Connector Pin 2

ESP32 GND ─────────────── A01 Connector Pin 3
```

---

# A02 Full Wiring

```text
ESP32 3V3
   │
   ├─────────────── A02 Connector Pin 1
   │
   └── 4.7kΩ ──┐
                │
GPIO 14 ────────┴──────── A02 Connector Pin 2

ESP32 GND ─────────────── A02 Connector Pin 3
```

---

# A03 Full Wiring

```text
ESP32 3V3
   │
   ├─────────────── A03 Connector Pin 1
   │
   └── 4.7kΩ ──┐
                │
GPIO 16 ────────┴──────── A03 Connector Pin 2

ESP32 GND ─────────────── A03 Connector Pin 3
```

---

# A04 Full Wiring

```text
ESP32 3V3
   │
   ├─────────────── A04 Connector Pin 1
   │
   └── 4.7kΩ ──┐
                │
GPIO 17 ────────┴──────── A04 Connector Pin 2

ESP32 GND ─────────────── A04 Connector Pin 3
```

---

# A05 Full Wiring

```text
ESP32 3V3
   │
   ├─────────────── A05 Connector Pin 1
   │
   └── 4.7kΩ ──┐
                │
GPIO 25 ────────┴──────── A05 Connector Pin 2

ESP32 GND ─────────────── A05 Connector Pin 3
```

---

# A06 Full Wiring

```text
ESP32 3V3
   │
   ├─────────────── A06 Connector Pin 1
   │
   └── 4.7kΩ ──┐
                │
GPIO 26 ────────┴──────── A06 Connector Pin 2

ESP32 GND ─────────────── A06 Connector Pin 3
```

---

# A07 Full Wiring

```text
ESP32 3V3
   │
   ├─────────────── A07 Connector Pin 1
   │
   └── 4.7kΩ ──┐
                │
GPIO 27 ────────┴──────── A07 Connector Pin 2

ESP32 GND ─────────────── A07 Connector Pin 3
```

---

# A08 Full Wiring

```text
ESP32 3V3
   │
   ├─────────────── A08 Connector Pin 1
   │
   └── 4.7kΩ ──┐
                │
GPIO 32 ────────┴──────── A08 Connector Pin 2

ESP32 GND ─────────────── A08 Connector Pin 3
```

---

# DS18B20 Probe Wiring

Typical waterproof DS18B20 probes often use:

```text
RED     = VCC
YELLOW  = DATA
BLACK   = GND
```

However, probe wire colors are not guaranteed.

Always verify the exact sensor documentation before connecting power.

---

# Probe-Side Connector Wiring

If using field-wireable 3-pin M8 connectors:

```text
DS18B20 VCC
    │
    ▼
M8 Pin 1

DS18B20 DATA
    │
    ▼
M8 Pin 2

DS18B20 GND
    │
    ▼
M8 Pin 3
```

---

# Panel Connector Wiring

Inside the enclosure:

```text
M8 Pin 1
   │
   ▼
3.3V Bus

M8 Pin 2
   │
   ▼
Dedicated GPIO

M8 Pin 3
   │
   ▼
Ground Bus
```

---

# Status LED

Thermone V1 should use one addressable RGB LED.

Recommended signal pin:

```text
GPIO 4
```

Typical wiring:

```text
ESP32 GPIO 4
      │
      ▼
RGB LED DATA

ESP32 5V or supported LED supply
      │
      ▼
RGB LED VCC

ESP32 GND
      │
      ▼
RGB LED GND
```

The exact supply voltage depends on the selected LED module.

---

# RGB LED Protection

For a WS2812B-style LED, consider:

```text
GPIO 4
   │
  330Ω
   │
   ▼
LED DATA
```

A small series resistor on the data line can improve signal protection.

A local decoupling capacitor may also be added near the LED.

---

# Setup / Reset Button

The setup button uses:

```text
GPIO 33
```

Wiring:

```text
GPIO 33
   │
   ▼
Button
   │
   ▼
GND
```

Firmware enables:

```text
INPUT_PULLUP
```

Therefore:

```text
Button released = HIGH
Button pressed  = LOW
```

---

# Button Diagram

```text
ESP32 GPIO 33
       │
       │
     [BUTTON]
       │
       ▼
      GND
```

No external pull-up resistor is required for the first prototype if the internal ESP32 pull-up is used.

---

# USB Power

Power enters through the ESP32 development board USB connector.

```text
5V USB Power Adapter
         │
         ▼
      USB Cable
         │
         ▼
       ESP32
         │
         ├── 3.3V Regulator
         │
         └── Controller Electronics
```

Recommended initial supply:

```text
5V / 2A
```

---

# USB Power Rule

Do not power the DS18B20 probes directly from raw USB 5V.

Use:

```text
ESP32 3.3V
```

for the probe VCC bus unless the final validated sensor design specifies otherwise.

---

# Perfboard Layout

For the prototype, use a small perfboard as the wiring distribution board.

Concept:

```text
┌─────────────────────────────────────┐
│                                     │
│  3.3V BUS ========================  │
│                                     │
│  GND BUS =========================  │
│                                     │
│  R1  A01 DATA                       │
│  R2  A02 DATA                       │
│  R3  A03 DATA                       │
│  R4  A04 DATA                       │
│  R5  A05 DATA                       │
│  R6  A06 DATA                       │
│  R7  A07 DATA                       │
│  R8  A08 DATA                       │
│                                     │
└─────────────────────────────────────┘
```

---

# Recommended Perfboard Terminal Labels

Label each connection:

```text
3V3

GND

A01
A02
A03
A04
A05
A06
A07
A08
```

This makes troubleshooting much easier.

---

# Internal Connection Example

```text
ESP32                     PERFBOARD

3V3 ───────────────────── 3V3 BUS

GND ───────────────────── GND BUS

GPIO 13 ───────────────── A01
GPIO 14 ───────────────── A02
GPIO 16 ───────────────── A03
GPIO 17 ───────────────── A04
GPIO 25 ───────────────── A05
GPIO 26 ───────────────── A06
GPIO 27 ───────────────── A07
GPIO 32 ───────────────── A08

GPIO 4 ────────────────── STATUS LED

GPIO 33 ───────────────── BUTTON
```

---

# Complete Internal Architecture

```text
                       USB POWER
                           │
                           ▼
                    ┌────────────┐
                    │   ESP32    │
                    │            │
                    │ GPIO 13 ───────── A01
                    │ GPIO 14 ───────── A02
                    │ GPIO 16 ───────── A03
                    │ GPIO 17 ───────── A04
                    │ GPIO 25 ───────── A05
                    │ GPIO 26 ───────── A06
                    │ GPIO 27 ───────── A07
                    │ GPIO 32 ───────── A08
                    │            │
                    │ GPIO 4 ────────── LED
                    │ GPIO 33 ───────── BUTTON
                    │            │
                    │ 3V3 ───────────── 3.3V BUS
                    │ GND ───────────── GND BUS
                    └────────────┘
```

---

# External Layout

Example enclosure connector layout:

```text
┌─────────────────────────────────────┐
│              THERMONE               │
│                                     │
│               ● STATUS              │
│                                     │
│            ○ SETUP                  │
│                                     │
│ A01   A02   A03   A04               │
│  ○     ○     ○     ○                │
│                                     │
│ A05   A06   A07   A08               │
│  ○     ○     ○     ○                │
│                                     │
│                    USB POWER        │
└─────────────────────────────────────┘
```

---

# Alternative Connector Layout

If enclosure width allows:

```text
A01 A02 A03 A04 A05 A06 A07 A08
 ○   ○   ○   ○   ○   ○   ○   ○
```

A two-row arrangement may allow a more compact enclosure.

---

# Wire Strain Relief

Internal and external wiring should not place mechanical stress directly on solder joints.

Use:

- Panel-mount connectors
- Cable glands where appropriate
- Heat shrink
- Cable ties
- Internal strain relief

---

# Drip Loops

All aquarium probe cables should form a drip loop before reaching the Thermone enclosure.

```text
Tank
 │
 │
 │
 ▼
 Cable
   \
    \
     \__
        \____ Thermone
```

This helps prevent water from running along the cable directly into the controller.

---

# Enclosure Placement

Install the controller:

- Above aquarium water level
- Away from splash zones
- Away from direct condensation
- Where USB power remains dry
- Where the status LED is visible
- Where the reset button remains accessible

---

# Unused Connector Protection

Unused M8 ports should use protective caps where possible.

This helps reduce:

- Dust
- Moisture
- Corrosion
- Accidental shorting

---

# Continuity Testing

Before connecting the ESP32, test every wired port with a multimeter.

For each connector verify:

```text
Pin 1 → 3.3V bus

Pin 2 → correct GPIO line

Pin 3 → GND bus
```

Also verify:

```text
Pin 1 is NOT shorted to Pin 3
```

and:

```text
DATA lines are NOT connected to each other
```

---

# Port Validation

Test one port at a time.

Example A01:

1. Connect one DS18B20 probe to A01.
2. Power the controller.
3. Confirm sensor detection.
4. Confirm correct ROM ID.
5. Confirm valid temperature.
6. Disconnect probe.
7. Confirm disconnect detection.

Repeat for all eight ports.

---

# Cross-Port Validation

After individual testing, connect eight probes simultaneously.

Confirm:

```text
A01 → Sensor 1
A02 → Sensor 2
A03 → Sensor 3
A04 → Sensor 4
A05 → Sensor 5
A06 → Sensor 6
A07 → Sensor 7
A08 → Sensor 8
```

Disconnect A03.

Expected:

```text
A03 → disconnected

A01 → still online
A02 → still online
A04 → still online
A05 → still online
A06 → still online
A07 → still online
A08 → still online
```

---

# Sensor Replacement Test

Connect Sensor A to A04.

Record:

```text
A04
Sensor ID:
28-FF-AAAA...
```

Replace it with Sensor B.

Expected:

```text
A04
Sensor ID:
28-FF-BBBB...
```

Thermone should recognize:

```text
same physical port
different physical sensor
```

---

# Power-Off Wiring Rule

Never modify internal wiring while the controller is powered.

Before opening the enclosure:

1. Disconnect USB power.
2. Wait for the ESP32 to power down.
3. Perform wiring changes.
4. Check connections.
5. Reconnect power.

---

# Prototype Safety Checks

Before first power-on:

- No exposed copper touching the enclosure
- No 3.3V/GND short
- All pull-up resistors installed correctly
- Every connector uses the correct pin order
- GPIO lines are not shorted together
- ESP32 is mounted securely
- Perfboard is insulated from the enclosure
- USB connection is dry
- Probe connectors are fully seated

---

# Final Wiring Summary

```text
POWER

ESP32 3V3
 └── Shared probe VCC bus

ESP32 GND
 └── Shared ground bus


TEMPERATURE DATA

GPIO 13 → A01 DATA
GPIO 14 → A02 DATA
GPIO 16 → A03 DATA
GPIO 17 → A04 DATA
GPIO 25 → A05 DATA
GPIO 26 → A06 DATA
GPIO 27 → A07 DATA
GPIO 32 → A08 DATA


PULL-UPS

A01 DATA → 4.7kΩ → 3.3V
A02 DATA → 4.7kΩ → 3.3V
A03 DATA → 4.7kΩ → 3.3V
A04 DATA → 4.7kΩ → 3.3V
A05 DATA → 4.7kΩ → 3.3V
A06 DATA → 4.7kΩ → 3.3V
A07 DATA → 4.7kΩ → 3.3V
A08 DATA → 4.7kΩ → 3.3V


CONTROLS

GPIO 4  → RGB Status LED
GPIO 33 → Setup / Reset Button
```

---

# V1 Wiring Principle

Thermone wiring should remain easy to understand and troubleshoot.

Every temperature channel must follow the same pattern:

```text
3.3V
  │
  ├── Probe VCC
  │
  └── 4.7kΩ
        │
GPIO ───┴── Probe DATA

GND ─────── Probe GND
```

Only the GPIO number changes between A01 and A08.