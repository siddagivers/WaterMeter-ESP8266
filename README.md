# WaterMeter-ESP8266

ESP8266 / Wemos D1 mini water-meter reader for a **Sensus 420 / 420PC** fitted with a **Sensus HRI-A4 D1** pulse module.

The firmware counts HRI pulses, calculates water flow, stores the total locally in LittleFS, reports total consumption and flow to Domoticz, provides a live web dashboard, supports dual Wi-Fi profiles, and includes ElegantOTA updates.

## Current firmware

**WM-2.8-HRI-D1MINI-DASH-STATIC**

Main sketch:

`WaterMeter_D1Mini_HRI_WM-2.8_Dashboard_StaticIP.ino`

## Hardware

- Wemos / LOLIN D1 mini or compatible ESP8266 clone
- Sensus 420 / 420PC water meter
- Sensus HRI-A4 D1 pulse module
- 1 kΩ series resistor
- 4.7 kΩ pull-up resistor
- Existing 6–8 m LAN cable extension is suitable for the low-frequency pulse signal

## HRI wiring

The currently identified extension-cable wires are:

```text
3.3 V
  |
  4.7 kΩ
  |
  +---------------- D5 / GPIO14
  |
  1 kΩ
  |
BLUE ---------------- HRI pulse

BROWN --------------- D1 mini GND

BLUE/WHITE ----------- leave disconnected / insulated
```

Important: **Do not connect the old 12 V inductive-sensor supply to the HRI.** The HRI-A4 has its own internal lithium battery and provides a solid-state pulse switch output.

The ESP8266 input is configured as active LOW using `INPUT_PULLUP` and a `FALLING` interrupt.

## HRI D1 behavior

For Sensus HRI-prepared domestic meters, the HRI-A4 D1 output is **1 litre per output pulse** with a nominal **124 ms pulse length**. The Sensus HRI uses inductive scanning of the prepared meter pointer. HRI-A4 also compensates reverse flow before continuing forward output pulses, so the remote pulse count follows the mechanical meter total.

Official documentation:

- Sensus HRI sensor data sheet: https://www.xylem.com/siteassets/brand/sensus/resources/data-sheets/hri-sensor-data-sheet.pdf
- Sensus 420/420PC data sheet: https://www.xylem.com/siteassets/brand/sensus/resources/data-sheets/420-420pc-data-sheet.pdf

## Pulse filtering

The installation has a hard physical sanity limit of **45 L/min**.

With 1 L per pulse:

```text
45 L/min = 0.75 L/s = 1 pulse every 1333.33 ms
```

The firmware uses a minimum valid raw pulse interval of **1334 ms**.

Any pulse arriving faster than this is rejected completely:

- no litre is added
- Domoticz total is not incremented
- flow is not updated from that pulse
- the pulse LED does not indicate an accepted litre

The filter compares each new pulse with the previous **raw** HRI pulse. This prevents a continuous noisy stream from eventually becoming accepted just because enough time has elapsed since the last accepted pulse.

## Domoticz

```text
Domoticz host: 192.168.1.16
Port:          8080
Total litres:  IDX 6805
Flow L/min:    IDX 6804
```

The total counter is sent when accepted HRI pulses arrive. Flow is reported approximately every second.

## Network

```text
Static IP: 192.168.1.119
Gateway:   192.168.1.1
Subnet:    255.255.255.0
DNS:       192.168.1.1
Hostname:  WaterMeter-ESP8266
mDNS:      watermeter-esp8266.local
```

The firmware stores **Primary** and **Backup** Wi-Fi profiles and can roam to the stronger configured network when RSSI remains poor.

When normal Wi-Fi is unavailable, the ESP starts a rescue/setup AP:

```text
SSID: WM-SETUP-xxxxxx
IP:   192.168.4.1
```

## Dashboard

Open:

```text
http://192.168.1.119/
```

The live dashboard refreshes once per second and displays:

- total water consumption
- current smoothed flow in L/min
- instantaneous flow from the latest accepted pulse interval
- flow gauge, 0–45 L/min
- Wi-Fi RSSI gauge
- accepted HRI pulses since boot
- litres counted since boot
- rejected >45 L/min pulses
- last pulse interval
- time since the last accepted pulse
- current D5 input state
- uptime and firmware version
- active Wi-Fi profile and IP address
- Domoticz IDX information

The dashboard also provides Wi-Fi profile configuration and manual local-counter adjustment.

## OTA firmware update

ElegantOTA is enabled.

```text
http://192.168.1.119/update
http://watermeter-esp8266.local/update
```

When running from the rescue AP:

```text
http://192.168.4.1/update
```

The first firmware installation normally needs USB; subsequent versions can be uploaded over OTA.

## Arduino IDE

For a typical Chinese D1 mini clone, start with:

```text
Board: LOLIN(WEMOS) D1 mini (clone)
CPU frequency: 80 MHz
Flash: 4 MB
Upload speed: 460800
```

If the clone profile behaves incorrectly, `LOLIN(WEMOS) D1 R2 & mini` is also commonly compatible.

Required libraries/components:

- ESP8266 Arduino core
- LittleFS
- ElegantOTA

## More documentation

See [DOCUMENTATION.md](DOCUMENTATION.md) for firmware architecture, filtering logic, web endpoints, persistence, Wi-Fi behavior, troubleshooting and maintenance notes.
