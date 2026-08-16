# Thermone V1 Parts List

## Purpose

This document defines the bill of materials for the first Thermone V1 hardware prototype.

The prototype target is:

```text
1 controller
8 independent temperature probe ports
8 DS18B20 probes
Wi-Fi
USB power
Status LED
Setup/reset button
Water-resistant enclosure
```

The first prototype should prioritize:

- Reliable parts
- Easy sourcing
- Easy assembly
- Replaceable components
- Low complexity
- Enough room for debugging

Cost optimization comes later.

---

# Prototype Quantity

This BOM covers:

```text
1 × Thermone V1 prototype controller
```

with:

```text
8 × temperature probe channels
```

---

# Main Controller

## ESP32 Development Board

Quantity:

```text
1
```

Recommended type:

```text
ESP32-DevKitC style development board
ESP32-WROOM-32E module
```

Required features:

- 2.4 GHz Wi-Fi
- USB programming
- At least 8 usable GPIOs for probe channels
- Additional GPIO for status LED
- Additional GPIO for setup button
- OTA-capable flash layout
- 3.3V regulated output
- Stable USB power input

Preferred characteristics:

- Genuine or reputable ESP32 module
- Clearly labeled GPIO headers
- USB-C preferred if available
- 4 MB flash minimum
- Replaceable development board form factor

Quantity:

```text
1
```

Recommended spare quantity:

```text
1
```

---

# Temperature Sensors

## Waterproof DS18B20 Probe

Quantity:

```text
8
```

Recommended spare quantity:

```text
2
```

Total recommended order:

```text
10
```

Required characteristics:

- DS18B20 digital temperature sensor
- Waterproof stainless steel probe
- 3-wire connection
- Unique 64-bit ROM address
- Cable length suitable for aquarium racks
- Operating voltage compatible with 3.3V
- Fully sealed sensor end

Recommended cable length for first build:

```text
1–2 meters
```

Longer probes may be tested later.

---

# Probe Connectors

## 3-Pin Waterproof Panel-Mount Connector

Preferred initial type:

```text
M8 A-coded
3-pin
Panel mount
```

Quantity:

```text
8
```

Recommended spare quantity:

```text
2
```

Total recommended order:

```text
10
```

Requirements:

- 3 conductors
- Keyed connection
- Panel mount
- Water-resistant
- Locking connection
- Suitable for low-voltage signal use

Recommended environmental target:

```text
IP67 when connected
```

---

## Matching Probe-Side Connector

Quantity:

```text
8
```

Recommended spare quantity:

```text
2
```

Total recommended order:

```text
10
```

Preferred:

```text
M8
3-pin
A-coded
Field-wireable
Matching gender to panel connector
```

Field-wireable connectors are preferred for the first prototype because the DS18B20 probe cables can be terminated manually.

---

# Connector Pin Standard

All Thermone temperature connectors use:

| Pin | Function |
|---|---|
| 1 | 3.3V |
| 2 | DATA |
| 3 | GND |

This mapping must remain identical across all ports.

---

# Pull-Up Resistors

## 4.7kΩ Resistor

Quantity required:

```text
8
```

Recommended order:

```text
20+
```

Specification:

```text
4.7kΩ
1/4 watt
±5% or better
Through-hole
```

Each temperature port requires one resistor.

Example:

```text
A01 DATA → 4.7kΩ → 3.3V
```

Repeat for all eight ports.

---

# Status LED

## Addressable RGB LED

Quantity:

```text
1
```

Recommended spare:

```text
2
```

Preferred type:

```text
WS2812B / NeoPixel-compatible
single-pixel module
```

Requirements:

- RGB
- Single GPIO control
- Small physical size
- Visible through enclosure
- Low power

Recommended signal GPIO:

```text
GPIO 4
```

---

# LED Data Resistor

Quantity:

```text
1
```

Recommended value:

```text
330Ω
```

Suggested order:

```text
10
```

Connection:

```text
GPIO 4
  │
330Ω
  │
LED DATA
```

---

# LED Decoupling Capacitor

Recommended:

```text
100 µF to 470 µF
```

Quantity:

```text
1
```

Placed close to the RGB LED supply.

This is optional for early bench testing but recommended for the enclosed prototype.

---

# Setup / Reset Button

Quantity:

```text
1
```

Recommended spare:

```text
2
```

Preferred type:

```text
Momentary normally-open pushbutton
```

Possible final format:

```text
Sealed panel-mount button
```

or:

```text
Recessed tactile button
```

Recommended GPIO:

```text
GPIO 33
```

Wiring:

```text
GPIO 33
   │
 Button
   │
  GND
```

Firmware uses:

```text
INPUT_PULLUP
```

---

# Perfboard

Quantity:

```text
1
```

Recommended type:

```text
2.54 mm pitch solderable prototype board
```

Approximate size:

```text
50 × 70 mm
```

or smaller if the wiring layout fits.

The perfboard is used for:

- 3.3V bus
- Ground bus
- Eight 4.7kΩ resistors
- Eight data connections
- LED support components
- Wiring distribution

---

# Hookup Wire

Recommended:

```text
22–24 AWG stranded copper wire
```

Use at least three colors.

Suggested color standard:

```text
Red    = 3.3V
Black  = GND
Yellow = DATA
```

Recommended quantity:

```text
5–10 meters total
```

For a single prototype, short internal runs will use much less than this.

---

# Heat-Shrink Tubing

Quantity:

```text
1 assortment
```

Recommended sizes:

```text
2 mm
3 mm
4 mm
6 mm
```

Use for:

- Probe cable termination
- Internal solder joints
- LED wiring
- Button wiring
- Connector connections

Adhesive-lined heat shrink may be used where extra moisture protection is useful.

---

# USB Power

## USB Power Adapter

Quantity:

```text
1
```

Specification:

```text
5V
2A minimum
```

Use a reputable, safety-certified power adapter.

---

## USB Cable

Quantity:

```text
1
```

Connector depends on selected ESP32 board.

Preferred:

```text
USB-C
```

Recommended length:

```text
1–2 meters
```

The cable should support both:

- Power
- Data/programming

---

# Enclosure

Quantity:

```text
1
```

Preferred material:

```text
ABS
```

or:

```text
Polycarbonate
```

Target approximate dimensions:

```text
120–160 mm wide
70–100 mm tall
35–50 mm deep
```

The exact enclosure depends on:

- ESP32 size
- M8 connector dimensions
- Internal wiring clearance
- Button placement
- LED placement

---

# Enclosure Requirements

The enclosure should provide:

- Removable cover
- Screw closure
- Enough wall thickness for M8 panel connectors
- Room for ESP32
- Room for perfboard
- USB cable entry
- Status LED visibility
- Setup button access

Target environmental protection:

```text
IP54 minimum
```

Higher is preferred.

---

# M8 Protective Caps

Quantity:

```text
8
```

Recommended for unused probe ports.

These protect unused connectors from:

- Water
- Humidity
- Dust
- Corrosion
- Accidental contact

---

# USB Cable Gland

If the ESP32 USB connector remains inside the enclosure, use a cable gland.

Quantity:

```text
1
```

Typical size:

```text
PG7
```

or a size appropriate for the selected USB cable.

The gland provides:

- Strain relief
- Splash protection
- Cable retention

---

# Alternative USB Arrangement

Instead of permanently routing a cable through a gland, a future revision may use:

```text
Panel-mount USB-C extension
```

This allows power to be disconnected externally without opening the enclosure.

For the first prototype, either approach is acceptable.

---

# Internal Standoffs

Quantity:

```text
4–8
```

Recommended:

```text
M2.5 nylon standoffs
```

or the mounting size required by the selected ESP32 board.

Use standoffs to keep electronics away from the enclosure floor.

---

# Mounting Screws

Quantity:

```text
As required
```

Suggested:

```text
M2
M2.5
M3
```

depending on enclosure and board mounting holes.

---

# Cable Ties

Quantity:

```text
10+
```

Small nylon cable ties are useful for:

- Internal wire organization
- Strain relief
- Probe cable management

---

# Adhesive Cable Tie Mounts

Optional.

Quantity:

```text
4–8
```

Useful for internal cable management.

---

# Labels

The prototype should have labels for:

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

Additional labels:

```text
POWER
SETUP
STATUS
```

---

# Device Label

The final enclosure should eventually receive:

```text
THERMONE

Model:
THV1

Serial:
THV1-000001

Hardware:
V1.0

[QR CODE]
```

Prototype labels may initially be printed using a standard label maker.

---

# Multimeter

Required tool:

```text
1
```

Used for:

- Continuity
- Voltage testing
- Short detection
- Connector verification

---

# Soldering Iron

Required tool:

```text
1
```

Recommended:

```text
Temperature-controlled soldering iron
```

---

# Solder

Quantity:

```text
1 spool
```

Use electronics-grade solder.

---

# Wire Strippers

Required:

```text
1
```

Suitable for:

```text
22–24 AWG
```

---

# Flush Cutters

Required:

```text
1
```

Used for:

- Wire trimming
- Component leads
- Cable ties

---

# Heat Gun

Recommended:

```text
1
```

Used for heat-shrink tubing.

---

# Drill

Required for enclosure modification.

Used for:

- M8 connector holes
- Button opening
- LED opening
- Cable gland

---

# Step Drill Bit

Highly recommended for plastic enclosures.

Useful for cleanly enlarging panel holes.

---

# Drill Bit Sizes

Exact sizes depend on the selected components.

Measure the actual component threads before drilling.

Never assume a hole size from the connector family alone.

---

# Caliper

Recommended:

```text
Digital caliper
```

Useful for:

- Connector diameter
- Enclosure spacing
- Hole positioning
- Mounting dimensions

---

# Prototype BOM Summary

| Item | Required Qty | Recommended Order Qty |
|---|---:|---:|
| ESP32 DevKit | 1 | 2 |
| Waterproof DS18B20 | 8 | 10 |
| M8 panel connectors | 8 | 10 |
| M8 probe-side connectors | 8 | 10 |
| 4.7kΩ resistors | 8 | 20+ |
| RGB LED | 1 | 3 |
| 330Ω resistor | 1 | 10 |
| LED capacitor | 1 | 5 |
| Setup button | 1 | 3 |
| Perfboard | 1 | 2 |
| 22–24 AWG wire | As needed | 1 kit |
| Heat shrink | As needed | 1 kit |
| 5V/2A power adapter | 1 | 1 |
| USB cable | 1 | 2 |
| Enclosure | 1 | 2 |
| Cable gland | 1 | 2 |
| M8 protective caps | 8 | 10 |
| Standoffs | 4–8 | 1 kit |
| Cable ties | As needed | 1 pack |
| Labels | As needed | 1 set |

---

# Minimum Bench-Test Build

Before assembling the enclosure, Thermone should first be tested on the bench.

Minimum parts:

```text
1 × ESP32
1 × DS18B20
1 × 4.7kΩ resistor
Breadboard
Jumper wires
USB cable
```

Test:

```text
A01
```

first.

Then add:

```text
A02
A03
...
A08
```

one at a time.

---

# Full Bench-Test Build

Before enclosure assembly:

```text
ESP32
 │
 ├── A01 → DS18B20
 ├── A02 → DS18B20
 ├── A03 → DS18B20
 ├── A04 → DS18B20
 ├── A05 → DS18B20
 ├── A06 → DS18B20
 ├── A07 → DS18B20
 └── A08 → DS18B20
```

Confirm all eight channels operate correctly before installing waterproof connectors.

---

# Prototype Build Stages

Recommended order:

## Stage 1

```text
ESP32 + 1 sensor
```

Validate firmware and GPIO.

## Stage 2

```text
ESP32 + 8 sensors
```

Validate all independent buses.

## Stage 3

```text
Add RGB LED
Add button
```

Validate controls.

## Stage 4

```text
Build perfboard distribution
```

Validate permanent wiring.

## Stage 5

```text
Add M8 connectors
```

Validate external probe connections.

## Stage 6

```text
Install enclosure
```

Validate thermal, Wi-Fi, and mechanical performance.

---

# Spare Parts

Always keep at least:

```text
2 spare DS18B20 probes
2 spare M8 connectors
1 spare ESP32
```

during prototype development.

This prevents a single failed component from stopping development.

---

# Production Considerations

The V1 prototype BOM is not the final production BOM.

A production revision may replace:

```text
ESP32 development board
+
perfboard
+
manual wiring
```

with:

```text
custom carrier PCB
```

while still using a standard ESP32 module.

This could reduce:

- Assembly time
- Wiring mistakes
- Enclosure size
- Unit cost

---

# Cost Tracking

For each purchased component, record:

```text
Manufacturer
Manufacturer Part Number
Supplier
Supplier SKU
Unit Price
Quantity
Shipping
Purchase Date
Test Result
```

This will allow Thermone to develop a proper production BOM later.

---

# Suggested BOM Tracking Format

Example:

| Category | Manufacturer | Part Number | Supplier | Qty | Unit Cost | Status |
|---|---|---|---|---:|---:|---|
| MCU | TBD | TBD | TBD | 1 | TBD | Testing |
| Sensor | TBD | TBD | TBD | 8 | TBD | Testing |
| Connector | TBD | TBD | TBD | 8 | TBD | Testing |

Do not finalize a production supplier until the part has been physically tested.

---

# Prototype Acceptance Criteria

The V1 parts selection is acceptable when:

1. All eight probes can operate simultaneously.
2. Probe connectors remain secure.
3. A probe can be replaced without opening the enclosure.
4. Wi-Fi remains reliable.
5. The ESP32 remains stable.
6. Power supply has adequate margin.
7. Internal wiring fits the enclosure safely.
8. The status LED is clearly visible.
9. The setup button is accessible.
10. The enclosure can be mounted near aquarium racks.
11. No exposed mains voltage exists inside the controller.
12. The device can run continuously without overheating.

---

# Final Prototype BOM Principle

The first Thermone V1 should favor:

```text
Reliability
    ↓
Serviceability
    ↓
Ease of assembly
    ↓
Compactness
    ↓
Cost optimization
```

Production cost optimization should only happen after the complete controller has been tested successfully.