# WaterMeter-ESP8266 Technical Documentation

## 1. Purpose

This project reads a Sensus HRI-A4 D1 pulse output with a Wemos/LOLIN D1 mini ESP8266 and integrates the water meter with Domoticz.

The design goals are:

- reliable 1 L pulse counting
- rejection of physically impossible pulse bursts
- persistent total-water counter
- continuously updated flow rate
- simple local diagnostics through a browser
- primary/backup Wi-Fi support
- fixed LAN address
- OTA firmware updates

Current firmware: **WM-2.8-HRI-D1MINI-DASH-STATIC**.

## 2. Meter and HRI interface

### Sensus 420 / 420PC

The Sensus 420/420PC has a register prepared for the Sensus HRI inductive interface. The HRI is retrofittable without modifying the meter's water path.

The HRI does not use a normal mechanical reed switch. It inductively scans the prepared pointer and presents an electronic solid-state output.

### HRI-A4 D1

For the domestic-meter D1 variant used here:

- HRI input resolution: 1 litre
- output pulse weight: 1 litre
- nominal D1 output pulse length: 124 ms
- brown: ground/common
- white: pulse output
- yellow: cable-cut contact
- output: solid-state switch
- maximum output rating: 24 V DC / 20 mA
- internal battery: 3 V lithium
- enclosure: IP68
- vendor states cable can be extended up to 30 m, with transient protection recommended for outdoor runs

HRI-A4 is a balanced-pulse version. Reverse volume is accumulated internally and must be compensated by subsequent forward movement before normal forward output pulses continue. This is intended to keep a remote counter aligned with the mechanical meter reading.

Vendor references:

- https://www.xylem.com/siteassets/brand/sensus/resources/data-sheets/hri-sensor-data-sheet.pdf
- https://www.xylem.com/siteassets/brand/sensus/resources/data-sheets/420-420pc-data-sheet.pdf

## 3. Electrical connection

The sealed cable extension was identified electrically. The extension colors used in this installation are:

```text
LAN BLUE       = HRI pulse
LAN BROWN      = HRI common / ground
LAN BLUE/WHITE = unused cable-cut lead
```

Recommended D1 mini wiring:

```text
                         +3.3 V
                            |
                          4.7 kΩ
                            |
HRI BLUE ---- 1 kΩ --------+-------- D5 / GPIO14

HRI BROWN -------------------------- GND

HRI BLUE/WHITE --------------------- insulated, unused
```

The 4.7 kΩ resistor is the external pull-up. The 1 kΩ series resistor provides additional GPIO protection. Firmware also enables the ESP8266 internal pull-up.

Do not apply the old 12 V sensor supply to the HRI.

## 4. Pulse acquisition

GPIO configuration:

```cpp
const uint8_t PULSE_PIN = D5; // GPIO14
pinMode(PULSE_PIN, INPUT_PULLUP);
attachInterrupt(digitalPinToInterrupt(PULSE_PIN), onPulse, FALLING);
```

Idle state is HIGH. An HRI pulse pulls the input LOW, causing a FALLING interrupt.

A short 50 ms ISR deglitch filter ignores electrical edges that are much shorter than a genuine HRI D1 pulse.

## 5. Hard 45 L/min plausibility filter

The maximum expected installation flow is below 45 L/min. Therefore the firmware treats any faster pulse sequence as invalid.

For a 1 L pulse weight:

```text
pulse_interval_ms = 60000 / flow_L_per_min

60000 / 45 = 1333.33 ms
```

The implementation uses:

```cpp
const float MAX_LPM_ACCEPT = 45.0f;
const uint32_t MIN_VALID_PULSE_INTERVAL_MS = 1334;
```

A raw HRI edge with an interval below 1334 ms is rejected entirely.

### Important implementation detail

The comparison is against the previous **raw** pulse, not only the previous accepted pulse.

Example of false pulses every 1000 ms:

```text
raw pulse #1 -> accepted as first reference
raw pulse #2 -> 1000 ms -> rejected
raw pulse #3 -> 1000 ms -> rejected
raw pulse #4 -> 1000 ms -> rejected
...
```

This avoids an error where a noisy stream could periodically become accepted when measuring only from the previous accepted pulse.

Rejected fast pulses do not:

- increase the local litre counter
- update Domoticz total litres
- feed the flow EMA
- trigger an accepted-pulse indication

The number of rejected pulses remains visible on the dashboard for diagnostics.

## 6. Flow calculation

For a valid 1 L pulse interval:

```text
instantaneous_L_per_min = 60000 / interval_ms
```

The accepted instantaneous flow is smoothed using an exponential moving average:

```cpp
LPM_ALPHA = 0.25
```

The flow value is sent to Domoticz approximately once per second.

The output is returned to zero after an idle period. The zeroing threshold is adaptive using the previous valid pulse spacing, with a minimum idle threshold of 30 seconds and a hard maximum of 5 minutes.

## 7. Counter handling

Every accepted pulse adds exactly:

```text
1.0 litre
```

The total is stored in:

```text
/water_counter.txt
```

using LittleFS.

Normal saves occur after a meaningful change and periodically, reducing flash writes while still protecting the total across reboots.

The dashboard also allows the local total to be set manually. A manually-set total is saved and sent to Domoticz.

## 8. Domoticz integration

Current values:

```text
Host:        192.168.1.16
Port:        8080
Counter IDX: 6805
Flow IDX:    6804
```

Counter request format:

```text
/json.htm?type=command&param=udevice&idx=6805&nvalue=0&svalue=<litres>
```

Flow request format:

```text
/json.htm?type=command&param=udevice&idx=6804&nvalue=0&svalue=<L/min>
```

The firmware uses a raw `WiFiClient` HTTP request rather than `ESP8266HTTPClient`.

## 9. Wi-Fi operation

Two profiles are stored in LittleFS:

- Primary
- Backup

Configuration file:

```text
/wifi_profiles.json
```

The ESP attempts the last-used network first, then the other configured profile.

RSSI roaming parameters:

```text
Weak threshold:        -80 dBm
Weak-RSSI hold time:    15 s
Minimum dwell:           5 min
Target improvement:     10 dB
```

When the current link remains poor, both configured SSIDs are scanned and the alternate network is used only if it is substantially stronger.

## 10. Static network configuration

The firmware currently uses:

```text
IP:       192.168.1.119
Gateway:  192.168.1.1
Subnet:   255.255.255.0
DNS:      192.168.1.1
Hostname: WaterMeter-ESP8266
mDNS:     watermeter-esp8266.local
```

Both Primary and Backup SSIDs are assumed to be on a LAN where these settings are valid.

If the backup Wi-Fi belongs to another subnet, replace static configuration with DHCP or separate static settings per profile.

## 11. Setup / rescue AP

During startup or when station Wi-Fi is unavailable, the device exposes an access point:

```text
SSID: WM-SETUP-<chip-id>
Address: 192.168.4.1
```

The AP allows access to the same configuration/dashboard server while normal station connectivity is unavailable.

The AP is stopped after a successful station connection and restarted if Wi-Fi is subsequently lost.

## 12. Web interface

### `/`

Main dashboard.

The dashboard displays:

- total litres
- smoothed L/min
- last-interval L/min
- 0–45 L/min flow gauge
- Wi-Fi RSSI gauge
- current SSID and active profile
- static IP
- accepted pulses since boot
- litres accumulated since boot
- rejected high-rate pulses
- last accepted pulse interval
- time since last accepted pulse
- current D5 HIGH/LOW state
- uptime
- firmware version
- Domoticz IDs

### `/api/status`

JSON endpoint used by the live dashboard. The browser polls it every second without reloading the complete page.

### `/wifi/save`

POST endpoint used to save Primary/Backup profiles.

### `/wifi/switch?to=primary`

Switch to Primary Wi-Fi.

### `/wifi/switch?to=backup`

Switch to Backup Wi-Fi.

### `/counter/set`

POST endpoint to set the local total-water counter.

### `/update`

ElegantOTA firmware upload page.

## 13. OTA

ElegantOTA is initialized with the same ESP8266 web server.

Normal LAN:

```text
http://192.168.1.119/update
```

mDNS:

```text
http://watermeter-esp8266.local/update
```

Rescue AP:

```text
http://192.168.4.1/update
```

The initial flash should normally be performed over USB. OTA can be used afterward.

## 14. LED behavior

The built-in LED is active LOW on typical D1 mini boards.

- rapid blinking during initial Wi-Fi setup/connection
- three long flashes after first successful Wi-Fi connection
- brief flash for an accepted HRI pulse

Rejected >45 L/min pulses are not treated as accepted litre pulses.

## 15. Arduino IDE setup

Recommended starting board profile for the current Chinese D1 mini clone:

```text
LOLIN(WEMOS) D1 mini (clone)
```

Suggested options:

```text
CPU:          80 MHz
Flash:        4 MB
Upload speed: 460800
```

If necessary, test the standard `LOLIN(WEMOS) D1 R2 & mini` profile.

The project requires an ESP8266 core with LittleFS support and the ElegantOTA library.

## 16. Troubleshooting

### Counter does not increase

Check the dashboard `Input D5` state. Idle should be HIGH. During an HRI pulse it should briefly become LOW.

Verify:

```text
BLUE -> 1 kΩ -> D5
D5 -> 4.7 kΩ -> 3.3 V
BROWN -> GND
```

Use continuity or voltage mode on the HRI output if necessary. HRI D1 pulses are approximately 124 ms, so some multimeter frequency modes may be unreliable at the very low pulse rate.

### Every pulse is rejected

Look at `Last pulse interval` and the rejected counter. If valid water flow produces intervals below 1334 ms, the actual installation exceeds the configured 45 L/min ceiling and the limit needs to be reconsidered.

### Counter has an old incorrect large value

Use the dashboard counter adjustment form and set the correct litre total. It is then saved to LittleFS and pushed to Domoticz.

### Dashboard works but Domoticz does not update

Confirm that `192.168.1.16:8080` is reachable from the water-meter network and that IDX 6805 and 6804 still correspond to the intended Domoticz devices.

### ESP is not reachable at 192.168.1.119

Check for an IP conflict. The address must not be assigned to another device. If the LAN gateway is not `192.168.1.1`, change the static settings in the firmware.

### Wi-Fi lost for a long time

The setup AP is brought back while disconnected. The firmware also retries Primary/Backup Wi-Fi and eventually reboots after the configured long offline interval.

## 17. Modification checklist

When changing firmware, keep these installation-specific values consistent unless intentionally changing them:

```text
Pulse input:           D5 / GPIO14
HRI pulse weight:      1 L
Maximum accepted flow: 45 L/min
Domoticz total IDX:    6805
Domoticz flow IDX:     6804
Domoticz IP:           192.168.1.16
Static ESP IP:         192.168.1.119
Hostname:              WaterMeter-ESP8266
```

Increment `FW_VERSION` for every published firmware revision so the dashboard clearly shows which build is running.
