# Thermone V1 Wi-Fi Setup

## Purpose

This document defines the first-time network provisioning flow for Thermone V1 controllers.

The goal is to let a customer connect a Thermone controller to their home or facility Wi-Fi without:

- Opening the enclosure
- Connecting the controller to a computer
- Installing special desktop software
- Exposing the controller to the public Internet
- Sending the customer's Wi-Fi password to Thermone Cloud

The setup experience should be simple enough to complete from a phone.

---

# High-Level Setup Flow

```text
Power On Thermone
        │
        ▼
Does Device Have
Valid Network Credentials?
        │
   ┌────┴────┐
   │         │
  YES        NO
   │         │
   ▼         ▼
Connect    Start Setup AP
   │         │
   ▼         ▼
Internet  User Connects
   │         │
   ▼         ▼
Thermone  Setup Portal
 Cloud        │
              ▼
        User Selects Wi-Fi
              │
              ▼
        User Enters Password
              │
              ▼
        Device Joins Wi-Fi
              │
              ▼
        Internet Available
              │
              ▼
        Register With Cloud
```

---

# First Power-On

When a new Thermone controller powers on, bootstrap firmware checks local persistent storage.

It checks for:

```text
Valid Wi-Fi credentials
```

and, if supported:

```text
Ethernet connectivity
```

---

# Network Selection Order

Recommended V1 priority:

```text
1. Ethernet
2. Stored Wi-Fi
3. Setup Access Point
```

Conceptually:

```text
Power On
   │
   ▼
Ethernet Available?
   │
   ├── YES → Use Ethernet
   │
   └── NO
         │
         ▼
Stored Wi-Fi?
   │
   ├── YES → Try Wi-Fi
   │
   └── NO
         │
         ▼
Start Setup Mode
```

---

# Setup Mode

Thermone enters setup mode when:

- The device has never been configured
- Stored Wi-Fi credentials are missing
- The user manually requests setup mode
- Recovery firmware requests network reconfiguration
- A factory reset has cleared Wi-Fi configuration

A temporary Wi-Fi access point is created.

---

# Setup Wi-Fi SSID

Recommended format:

```text
Thermone-XXXX
```

Example:

```text
Thermone-0482
```

The suffix should be derived from a non-secret portion of the serial number.

For example:

```text
Serial:
THV1-000482

SSID:
Thermone-0482
```

The SSID must not expose:

- Factory credential
- Claim token
- Device runtime credential
- MAC address if avoidable
- User identity

---

# Setup Access Point Security

The temporary setup network should not remain permanently open.

Possible V1 approaches include:

```text
Random setup password printed on label
```

or:

```text
Setup password derived from a dedicated factory setup secret
```

The final production approach should use a unique per-device setup password where practical.

---

# Example Device Label

```text
THERMONE

Serial:
THV1-000482

Setup Wi-Fi:
Thermone-0482

Setup Password:
K7XP92QF

[ QR CODE ]
```

The setup Wi-Fi password is separate from:

- Claim token
- Factory credential
- Runtime device credential

---

# Setup Network Lifetime

The setup network should only remain active while needed.

Recommended behavior:

```text
No stored network
→ setup AP stays active

Valid Wi-Fi connected
→ setup AP shuts down
```

The setup AP may also time out after an extended period when manually triggered.

---

# Captive Portal

When a user joins the Thermone setup network, the controller should provide a local setup portal.

Possible local IP:

```text
192.168.4.1
```

The device may use captive-portal behavior to encourage the phone to open the setup page automatically.

---

# Manual Setup URL

If the captive portal does not automatically appear, the user can navigate to:

```text
http://192.168.4.1
```

A future friendly local name may also be considered.

---

# Setup Portal Home Screen

Example:

```text
Thermone

Set Up Your Controller

Device:
THV1-000482

Let's connect Thermone to your Wi-Fi.

[ Continue ]
```

---

# Wi-Fi Scan

The ESP32 scans nearby Wi-Fi networks.

Example UI:

```text
Choose Wi-Fi

○ FishRoomWiFi
○ HomeNetwork
○ Verizon-4821
○ Other Network

[ Continue ]
```

The list should ideally display:

- SSID
- Signal strength
- Security type

---

# Hidden Networks

The setup portal should support entering a hidden SSID manually.

Example:

```text
Network Name
[ __________________ ]

Password
[ __________________ ]

[ Connect ]
```

---

# Wi-Fi Band

Thermone V1 using standard ESP32-WROOM hardware should be documented as supporting:

```text
2.4 GHz Wi-Fi
```

The setup portal should make this clear if the customer attempts to use an unsupported configuration.

---

# Wi-Fi Credential Entry

The customer enters:

```text
SSID
Password
```

The password field should be hidden by default.

Example:

```text
Wi-Fi Password
[ ••••••••••••• ]

[ Show Password ]
```

---

# Wi-Fi Credential Handling

Credentials flow:

```text
Phone
  │
  ▼
Local Thermone Setup Portal
  │
  ▼
ESP32
  │
  ▼
Local Persistent Storage
```

The Wi-Fi password should not be uploaded to Thermone Cloud.

---

# Credential Storage

The ESP32 stores customer network credentials locally.

Initial implementation may use:

```text
NVS
```

Future production revisions should consider:

```text
Encrypted NVS
Flash encryption
```

---

# Do Not Log Credentials

Firmware must never print the customer's Wi-Fi password to the normal serial log.

Bad:

```text
Connecting to FishRoomWiFi with password MySecretPassword
```

Good:

```text
Connecting to configured Wi-Fi network
SSID: FishRoomWiFi
Password: [REDACTED]
```

---

# Connection Attempt

After the customer submits credentials:

```text
Save Wi-Fi Configuration
         │
         ▼
Attempt Connection
         │
         ▼
Success?
```

---

# Successful Wi-Fi Connection

If successful:

```text
Wi-Fi Connected
      │
      ▼
DHCP Address Received
      │
      ▼
DNS Available
      │
      ▼
Internet Check
      │
      ▼
Thermone Cloud Check
```

The local setup page may show:

```text
Connected!

Thermone has joined:
FishRoomWiFi

Checking Internet connection...
```

---

# Successful Setup Screen

Example:

```text
Thermone is Connected

✓ Wi-Fi connected
✓ Internet available
✓ Thermone Cloud reachable

Device:
THV1-000482

[ Continue Setup ]
```

If device claiming is already complete:

```text
[ Open Dashboard ]
```

---

# Setup AP Shutdown

After successful Wi-Fi configuration, Thermone should:

1. Confirm network connectivity
2. Save configuration
3. Shut down the temporary setup AP
4. Continue cloud registration

---

# Incorrect Wi-Fi Password

If authentication fails:

```text
Could not connect to Wi-Fi.

The password may be incorrect.

[ Try Again ]
```

The setup AP should become available again.

---

# Wi-Fi Network Not Found

Example:

```text
Network not found.

Make sure your Wi-Fi is:
- Powered on
- Within range
- Using 2.4 GHz

[ Scan Again ]
```

---

# DHCP Failure

If Thermone joins the Wi-Fi network but cannot obtain an IP address:

```text
Connected to Wi-Fi, but your router did not provide a network address.

[ Retry ]
```

Firmware should not falsely report this as a bad password.

---

# No Internet

A customer network may provide Wi-Fi but no Internet.

State:

```text
Wi-Fi:
Connected

Internet:
Unavailable
```

The setup page may display:

```text
Connected to Wi-Fi

Thermone cannot currently reach the Internet.

You can leave the controller powered on.
It will keep trying automatically.
```

---

# Thermone Cloud Unavailable

The Internet may work while Thermone Cloud is temporarily unavailable.

Example:

```text
Wi-Fi:
Connected

Internet:
Connected

Thermone Cloud:
Unavailable
```

This should not cause the device to forget valid Wi-Fi credentials.

---

# Internet Retry

After valid Wi-Fi credentials are stored:

```text
Internet unavailable
       │
       ▼
Keep credentials
       │
       ▼
Retry with backoff
```

Do not repeatedly force the user through Wi-Fi setup if the local Wi-Fi connection itself is valid.

---

# Wi-Fi Reconnection

During normal operation, if Wi-Fi drops:

```text
Connection Lost
      │
      ▼
Reconnect Using Stored Credentials
```

The controller should not immediately start setup mode.

---

# Reconnection Backoff

Recommended conceptual schedule:

```text
Retry 1 → 5 sec
Retry 2 → 10 sec
Retry 3 → 30 sec
Later   → 60 sec
```

Random jitter should eventually be added.

---

# When To Re-Enter Setup Mode

The device should only re-enter user provisioning mode when necessary.

Possible conditions:

- No credentials exist
- User physically requests setup
- Stored credentials repeatedly fail for an extended period
- Factory reset occurs
- Recovery mode requires it

---

# Manual Setup Mode

The setup/reset button can manually restart Wi-Fi provisioning.

Recommended conceptual behavior:

```text
Hold Setup Button ~5 seconds
        │
        ▼
LED indicates setup mode
        │
        ▼
Thermone Setup AP starts
```

Existing credentials may remain stored until new credentials are successfully saved.

---

# Safe Wi-Fi Change

When changing Wi-Fi networks, Thermone should avoid erasing the previous working configuration too early.

Preferred flow:

```text
Existing Working Wi-Fi
        │
        ▼
User Enters New Wi-Fi
        │
        ▼
Attempt New Wi-Fi
        │
        ├── Success
        │     │
        │     ▼
        │ Save New Wi-Fi
        │
        └── Failure
              │
              ▼
        Keep Old Configuration
```

This reduces accidental lockouts.

---

# Factory Reset

A factory reset clears customer networking configuration.

Conceptually:

```text
Factory Reset
      │
      ▼
Erase Wi-Fi Credentials
      │
      ▼
Restart
      │
      ▼
Start Setup AP
```

Permanent factory identity remains.

---

# Ethernet Flow

If Ethernet hardware is supported:

```text
Power On
   │
   ▼
Ethernet Link?
   │
   ├── YES
   │     │
   │     ▼
   │   DHCP
   │     │
   │     ▼
   │ Internet
   │
   └── NO
         │
         ▼
       Wi-Fi
```

---

# Ethernet and Setup Portal

Even with active Ethernet, Thermone may temporarily provide its setup Wi-Fi for onboarding.

This allows the customer to:

- Confirm device identity
- Configure backup Wi-Fi
- View setup status
- Access local diagnostics

---

# Ethernet Preferred Mode

If both Ethernet and Wi-Fi are connected:

```text
Primary:
Ethernet

Fallback:
Wi-Fi
```

The final network-selection policy is implemented in firmware.

---

# Cloud Registration After Wi-Fi Setup

Once network access is available:

```text
ESP32
  │
  ▼
Thermone Device API
  │
  ▼
Factory Authentication
  │
  ▼
Device Registration
  │
  ▼
Runtime Credential
```

The Wi-Fi setup portal does not itself need to create the user's Thermone account.

---

# Network Setup vs Device Claiming

These are separate operations.

```text
NETWORK SETUP
Connect controller to Internet
```

versus:

```text
DEVICE CLAIM
Connect controller to user account
```

A controller can be:

```text
Network configured
Registered
Unclaimed
```

and this is valid.

---

# Claiming Before Wi-Fi Setup

The user may scan the QR before powering the controller.

Example:

```text
User scans QR
      │
      ▼
Thermone account
      │
      ▼
Pending claim
      │
      ▼
User powers controller
      │
      ▼
Wi-Fi setup
      │
      ▼
Device registers
      │
      ▼
Claim completes
```

---

# Claiming After Wi-Fi Setup

Also valid:

```text
Power controller
      │
      ▼
Wi-Fi setup
      │
      ▼
Device registers
      │
      ▼
User scans QR
      │
      ▼
Claim completes
```

---

# Recommended Customer Onboarding

A polished onboarding flow may combine the two experiences visually.

Example:

```text
1. Scan Thermone QR

2. Login or Create Account

3. Plug In Controller

4. Connect to Thermone-0482

5. Select Home Wi-Fi

6. Thermone Connects

7. Return to Dashboard

8. Device Appears Automatically
```

---

# QR-Assisted Setup

The QR code may eventually open a dashboard route such as:

```text
https://app.thermone.com/setup/<claim-token>
```

The page can guide the user through network provisioning even though the Wi-Fi password itself remains local.

---

# Setup Status Sync

After Internet connectivity is established, the device can update cloud state.

Possible states:

```text
waiting_for_network
network_connected
registering
registered
firmware_updating
ready
```

The dashboard can show progress.

---

# Example Dashboard Setup Screen

```text
Setting Up Thermone

Device:
THV1-000482

✓ Device found
✓ Wi-Fi connected
✓ Registered with Thermone
○ Checking firmware
○ Finalizing setup
```

---

# Local Setup API

The ESP32 local web server may expose internal endpoints.

Conceptual examples:

```text
GET  /api/status
GET  /api/wifi/scan
POST /api/wifi/connect
POST /api/restart
```

These endpoints exist only on the local setup network.

---

# Local Status Endpoint

Example:

```json
{
  "serial_number": "THV1-000482",
  "setup_mode": true,
  "wifi_connected": false,
  "internet_available": false,
  "cloud_connected": false
}
```

---

# Wi-Fi Scan Endpoint

Example:

```text
GET /api/wifi/scan
```

Response:

```json
{
  "networks": [
    {
      "ssid": "FishRoomWiFi",
      "rssi": -48,
      "secure": true
    },
    {
      "ssid": "HomeNetwork",
      "rssi": -70,
      "secure": true
    }
  ]
}
```

---

# Wi-Fi Connect Endpoint

Conceptual:

```text
POST /api/wifi/connect
```

Example:

```json
{
  "ssid": "FishRoomWiFi",
  "password": "user-provided-password"
}
```

This endpoint should only be available during authorized local setup mode.

---

# Password Response Rules

The ESP32 should never echo the password back.

Bad response:

```json
{
  "ssid": "FishRoomWiFi",
  "password": "MyPassword123"
}
```

Good:

```json
{
  "ssid": "FishRoomWiFi",
  "status": "connecting"
}
```

---

# Captive Portal DNS

During setup mode, the ESP32 may run a small DNS responder.

Requests to arbitrary domains may redirect to:

```text
192.168.4.1
```

This improves mobile onboarding.

---

# Setup Portal Design Requirements

The portal should be:

- Mobile-first
- Lightweight
- Fast
- Usable without Internet access
- Stored locally in firmware
- Accessible on common phones
- Clear about 2.4 GHz requirements

Avoid large remote JavaScript dependencies because the user may not have Internet access while connected to the setup AP.

---

# Setup Portal Assets

Prefer embedding local:

```text
HTML
CSS
Minimal JavaScript
```

inside firmware or the device filesystem.

Do not depend on:

```text
Google Fonts
External CDN JavaScript
Remote images
```

during provisioning.

---

# Setup LED States

Suggested behavior:

```text
Blue blinking
→ setup AP active

Yellow blinking
→ attempting Wi-Fi

Green
→ Thermone Cloud connected

Yellow solid
→ local network connected but cloud unavailable

Red
→ setup or hardware error
```

---

# Setup Timeout

For manually triggered setup mode, a timeout may be used.

Example:

```text
10 minutes
```

If no configuration changes occur:

```text
Exit setup mode
Return to previous network configuration
```

First-time unconfigured devices may remain in setup mode longer.

---

# Multiple Phones

Only one setup session should modify network credentials at a time.

The controller may implement:

```text
active_setup_session
```

to avoid conflicting submissions from multiple devices.

---

# Local Setup Security

The setup portal should only operate while:

```text
setup_mode = active
```

Outside setup mode:

```text
Wi-Fi AP disabled
Local provisioning endpoints disabled
```

---

# Setup Button Security

A nearby person should not be able to silently alter networking without physical access.

Requiring a physical button hold before reconfiguration significantly reduces remote abuse.

---

# Wi-Fi Credential Update From Cloud

Thermone Cloud should not normally send Wi-Fi passwords to devices.

Remote Wi-Fi replacement is not part of V1.

Changing Wi-Fi should occur locally through setup mode.

---

# Network Reset vs Factory Reset

These should be separate operations.

## Network Reset

Clears:

```text
Wi-Fi credentials
```

Keeps:

```text
Device registration
Runtime credential
Factory identity
```

## Factory Reset

May clear:

```text
Wi-Fi
Runtime configuration
Runtime credential
Offline buffer
```

Keeps:

```text
Factory identity
Serial
Hardware revision
```

---

# Network Reset Flow

```text
Hold Setup Button
       │
       ▼
Network Reset
       │
       ▼
Erase Wi-Fi
       │
       ▼
Start Setup AP
```

The exact physical button timing is defined in firmware documentation.

---

# Stored Multiple Networks

V1 may initially store one Wi-Fi network.

Future versions may store:

```text
Primary Wi-Fi
Backup Wi-Fi
```

The architecture should not prohibit this.

---

# Enterprise Wi-Fi

Initial Thermone V1 may not support complex enterprise authentication such as:

```text
802.1X
WPA2-Enterprise
```

If unsupported, this should be documented clearly.

---

# Guest Networks

Some guest networks use browser-based captive portals.

Thermone may not be able to complete these automatically.

The setup portal should eventually detect:

```text
Wi-Fi connected
Internet blocked by captive portal
```

and explain the limitation.

---

# MAC Filtering

Networks requiring manual MAC address allowlisting may require displaying Thermone's network MAC address during setup.

Example:

```text
Wi-Fi MAC:
AA:BB:CC:DD:EE:FF
```

This identifier is not secret.

---

# Network Diagnostics

The local setup portal may show:

```text
Wi-Fi:
Connected

IP:
192.168.1.105

Signal:
-52 dBm

Internet:
Connected

Thermone Cloud:
Connected
```

This can reduce support issues.

---

# Diagnostic Information

Safe local diagnostics may include:

- Serial number
- Firmware version
- IP address
- RSSI
- Network status
- Cloud connectivity

Do not display:

- Wi-Fi password
- Factory credential
- Runtime token
- Claim token

---

# Setup Completion

The setup process is complete when:

```text
Network connected
+
Internet available
+
Thermone Cloud registration succeeds
```

The device may continue firmware installation after this point.

---

# Setup Success Criteria

Thermone V1 Wi-Fi provisioning is successful when:

1. A new device creates a setup network.
2. The setup SSID identifies the correct controller.
3. The user can access the setup portal from a phone.
4. Nearby networks can be scanned.
5. A 2.4 GHz Wi-Fi network can be selected.
6. Credentials are stored locally.
7. Passwords are not sent to Thermone Cloud.
8. Incorrect credentials produce a useful error.
9. Valid credentials allow Internet access.
10. The device can register with Thermone Cloud.
11. Setup AP shuts down after success.
12. Stored Wi-Fi reconnects after reboot.
13. Temporary Internet failure does not erase credentials.
14. Manual setup mode can be triggered physically.
15. A factory reset returns the device to setup mode.

---

# Core Wi-Fi Setup Principle

Thermone network provisioning should feel like:

```text
Power It
   │
   ▼
Connect To Thermone
   │
   ▼
Choose Wi-Fi
   │
   ▼
Enter Password
   │
   ▼
Thermone Goes Online
```

The customer should never need to understand:

```text
ESP32
IP networking
ports
NAT
firmware flashing
serial consoles
```

to connect a Thermone controller.