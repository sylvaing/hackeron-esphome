# hackeron-esphome

**Bluetooth gateway between a Corelec / CCEI _Akeron_ pool regulator and Home Assistant, running on an ESP32 with ESPHome.**

Akeron devices (salt chlorinators with optional pH and Redox regulation) can only be reached through the Corelec mobile app over Bluetooth Low Energy. This firmware turns a cheap ESP32 placed near the Akeron into a permanent bridge: it connects over BLE, reads every measurement and setting about every 30 seconds, and exposes them to Home Assistant as native entities. You can then chart the water, get notified on alarms, and change setpoints from Home Assistant, all without the phone app.

```
 ┌──────────────┐   BLE (GATT)    ┌──────────────┐  Wi-Fi / ESPHome API  ┌────────────────┐
 │    Akeron    │ ◄─────────────► │    ESP32     │ ◄───────────────────► │ Home Assistant │
 │  regulator   │  ~5-10 m range  │ this firmware│   (encrypted)         │   entities     │
 └──────────────┘                 └──────────────┘                       └────────────────┘
```

## Contents

- [What you get](#what-you-get)
- [Compatibility](#compatibility)
- [How it works](#how-it-works)
- [Installation](#installation)
- [Entities reference](#entities-reference)
- [Alarm and warning codes](#alarm-and-warning-codes)
- [Good to know / known limitations](#good-to-know--known-limitations)
- [Troubleshooting](#troubleshooting)
- [Protocol notes](#protocol-notes)
- [Credits](#credits)

## What you get

- **Measurements**: pH, Redox (mV), water temperature, salt (g/L).
- **Setpoints, readable and writable**: pH setpoint, Redox setpoint, chlorine production (ELX %), cover-mode production, salt range.
- **Status**: chlorine production running, pH pumps running, flow switch, cover mode, boost, Sleep/Timer modes, device model.
- **Alarms and warnings** with human-readable texts (regulator, chlorinator and warning bit field), plus the low temperature and low salt thresholds configured in the device.
- **Actions**: 2 h boost start/stop, alarm reset, forced pH- pump run, sensor calibration (pH, Redox, salt, temperature), sensor and pump configuration.
- **Robustness**: automatic BLE scan and reconnection, CRC check on every received frame (with an error counter), entities reset to "unknown" when the link drops so stale values are never shown.

## Compatibility

| | |
|---|---|
| **Pool device** | Corelec / CCEI Akeron family: the firmware decodes models _Regul 4Rx, Regul, Regul 3, Duo, Duo Regul 3, Duo Regul 4Rx, Akeron, Duo Regul 4Amp_. Developed and tested on a **Duo Regul 4Rx** (chlorinator + pH + Redox). Entities for features your model lacks (for example Redox on a pH-only unit) simply stay empty. |
| **Gateway** | Any classic **ESP32** board with Bluetooth (the YAML targets `esp32dev`, framework ESP-IDF). Other ESP32 variants with BLE (C3, S3…) have not been tested; they would need a different `esp32:` block. ESP8266 is **not** supported (no Bluetooth). |
| **ESPHome** | 2025.5.0 or newer (`min_version`); currently used with ESPHome 2026.9. |
| **Home Assistant** | Any version supported by your ESPHome release, through the native ESPHome integration. No MQTT needed. |

> Some Akeron units send each frame differently over BLE (as a single 17-byte notification instead of `*` + 16 bytes). Those units are not decoded yet. If you see `Connected` but all measurements stay `unknown`, see [Troubleshooting](#troubleshooting).

## How it works

1. **Discovery and connection.** 30 s after boot the ESP32 starts a passive BLE scan. When the Akeron with the configured MAC address is seen (`Device Present`), it connects (`ble_client`) and stops scanning. On disconnection it clears the entities and scans again.
2. **Polling.** Every 30 s, while connected, the ESP32 sends five read requests, 5 s apart: `M` (measurements and status), `S` (pH settings), `A` (chlorinator), `E` (Redox), `D` (thresholds). The Akeron answers each request with a notification, which the firmware checks (length + CRC) and decodes into entities.
3. **Writing.** When you change a setpoint or press a button in Home Assistant, the ESP32 sends a write frame in which every byte is `0xFF` ("leave unchanged") except the field being changed. Three seconds later it reads the frame back, so Home Assistant shows what the Akeron actually applied.
4. **Read-back never writes.** Values read from the Akeron are only *displayed* (`publish_state`). They are never sent back to the device. Earlier versions did send them back, which could silently replace your chlorine setpoint with a transient value (see the changelog in the git history, commit `dcb9ecf`).

The Akeron is often powered together with the filtration pump. In that case it disappears every night and the entities show `unknown` until filtration starts again. That is expected: the ESP32 keeps scanning and reconnects by itself within about a minute.

## Installation

### 1. Hardware

- An ESP32 development board (ESP32-WROOM / "ESP32 DevKit") and a 5 V USB power supply.
- Place it **within Bluetooth range of the Akeron**, ideally in the technical room, a few metres from the unit. Walls and metal boxes reduce BLE range a lot, so check the Wi-Fi signal there as well.

### 2. Find the Akeron Bluetooth MAC address

Use any BLE scanner app on a phone (for example _nRF Connect_ or _LightBlue_) with filtration running. The Akeron advertises as **`CORELEC Regulateur`** or **`REGUL.`**. Note its address, e.g. `B4:E3:F9:65:71:74`.

Close the Corelec app first: while a phone is connected, the Akeron may stop advertising.

Alternative: flash the firmware with any address, uncomment the `on_ble_advertise` block under `esp32_ble_tracker:`, and read the logs. Every BLE device seen is listed with its name and address.

### 3. Create the device in ESPHome

1. In the **ESPHome Device Builder** (Home Assistant add-on or `esphome dashboard`), create a new device, choose ESP32, then **edit** it and replace its content with [`hackeron-esphome.yaml`](hackeron-esphome.yaml).
2. Set your Akeron address in the `substitutions:` block:
   ```yaml
   substitutions:
     mac_akeron: "B4:E3:F9:65:71:74"   # your Akeron BLE MAC address
   ```
3. Optionally change `esphome: name:` (default `hackeron-esp`). It determines the entity ids in Home Assistant.
4. Make sure your `secrets.yaml` (ESPHome **Secrets** button) defines:
   ```yaml
   wifi_ssid: "your-ssid"
   wifi_password: "your-wifi-password"
   api_encryption_key: "32-byte base64 key"   # generate one at https://esphome.io/components/api.html
   ota_password: "a-password-for-ota-updates"
   ```

### 4. Flash

- **First flash over USB**: plug the ESP32 into the computer running the dashboard (or use **Install → Manual download** and flash with [web.esphome.io](https://web.esphome.io)). Some boards need the **BOOT** button held while flashing starts.
- **Later updates**: **Install → Wirelessly** (OTA).

If Wi-Fi is unreachable, the ESP32 opens a fallback hotspot `Esp-Akeron Fallback Hotspot` (password = your Wi-Fi password), with a captive portal for entering new Wi-Fi credentials.

### 5. Add it to Home Assistant

Home Assistant discovers the device automatically (**Settings → Devices & services → Discovered → ESPHome**). Confirm and enter the `api_encryption_key`. With filtration running, **Connection Status** should switch to `Connected` within a minute and the measurements should appear after the next poll (≤ 30 s).

## Entities reference

Entity ids below assume the device name `hackeron-esp` (e.g. `sensor.hackeron_esp_ph`).

### Measurements and status

| Entity | Type | Description |
|---|---|---|
| PH | sensor (pH) | Measured pH |
| Redox | sensor (mV) | Measured Redox potential (accepted range 350-1000 mV, other readings ignored) |
| Water Temperature | sensor (°C) | Water temperature |
| Salt | sensor (g/L) | Salt concentration (0-40 g/L) |
| PH Setpoint / Redox Setpoint | sensor | Current setpoints as read from the device |
| PH Threshold Min / Max | sensor (pH) | pH alarm thresholds |
| Elx | sensor (%) | Chlorine production **applied right now** (see [Good to know](#good-to-know--known-limitations)) |
| Boost Time | sensor (min) | Remaining boost time |
| ELX Pump | binary | Chlorine production active |
| PH Pump / PH Minus Pump | binary | pH+ / pH- dosing pump running |
| Forced Pump | binary | Outputs forced (manual run) |
| Flow Switch Active | binary | Flow switch input |
| Cover Active | binary | Cover (pool cover) mode active |
| Boost 2h | binary | Boost running |
| Mode Sleep / Mode Timer | binary | Akeron Sleep mode (stop after N hours of filtration) / Timer mode ("24h/24", N/24 of each hour) |
| Durée Sleep Timer | sensor (h) | N for the Sleep/Timer mode (unit deduced from the manual, not confirmed) |
| Model | text | Model reported by the device |

### Controls

| Entity | Type | Description |
|---|---|---|
| Akeron PH Set | number | pH setpoint, 6.5-7.8, step 0.05 |
| Akeron Redox Set | number | Redox setpoint, 350-900 mV, step 10 |
| Akeron Elx Set | number | Chlorine production setpoint, 5-100 %, step 5 |
| Cover Production | number | Production in cover mode, as a ratio of normal production, 5-50 % |
| Cover Force | switch | Force cover mode |
| Salinité | select | Salt range: 4-8, 8-15 or > 15 g/L |
| Boost Start 2h / Boost Stop | button | Start a 2-hour boost / stop it |
| Reset Alarmes | button | Acknowledge and reset the device alarms |
| Force pH- | button | Run the pH- pump in forced mode (the Akeron's "force outputs for 1 minute" function) |

### Configuration and calibration (category _config_)

| Entity | Type | Description |
|---|---|---|
| Config Pompe PH+ / Config Pompe PH- | switch | Declare which pH pumps are installed. The firmware refuses to disable the last one (the device does not accept "no pump"). |
| Config Capteur Temp / Config Capteur Sel / Config Flow Switch | switch | Declare the installed sensors |
| Value Calibrate PH / Redox / Salt / Temp + Send Calibrate … | number + button | Calibration: measure the water with a **reference instrument**, enter that value, then press the matching _Send_ button. Ranges follow the official app (pH 6.5-8.5, Redox 200-650 mV, salt 3-35 g/L, temp 8-40 °C). |
| Contrôle CRC trames | switch | Enables the CRC check of received frames (on by default; turn off only to debug). |

### Diagnostics

| Entity | Description |
|---|---|
| Connection Status | `Scanning...`, `Found - Connecting...`, `Connected` or `Idle` |
| Device Present | Akeron seen in BLE advertisements; this triggers the connection. It normally turns off a few minutes after connecting, because a connected Akeron stops advertising. |
| Connect to Akeron Device | Turn off to release the BLE link (for example to use the Corelec app), on to reconnect |
| BLE Scanner | Starts / stops scanning |
| Alarm, Alarm Text | Regulator alarm (code and text) |
| Warning, Warning Text | Warnings (bit field, texts can combine: `E2 Sel ; E4 Température`) |
| Elx Alarm, Alarm Elx Text | Chlorinator alarm |
| Alarm Rdx | Raw Redox alarm field |
| Seuil alarme / alerte température basse, Seuil alerte / alarme sel bas | Thresholds configured in the device |
| Erreurs CRC | Number of received frames rejected because of a bad CRC |
| hackeron restart | Reboot the ESP32 |

Some entity names are in French. They come from the original project, and renaming them would change existing users' entity ids.

## Alarm and warning codes

Sources: the official Corelec manuals ([2021 _SALT DUO / SALT REGUL pH / REGUL3 / REGUL4 Rx_](https://www.easy-blue.fr/uploads/pdf/2021-akeron-duo-notice.pdf), section 6, which matches the Bluetooth generation handled here; the newer _SALT DUO V2_ and _REGUL REDOX 1.4_ manuals from [akeron.fr](https://www.akeron.fr/nos-supports-techniques/documentation)), plus the decompiled Corelec _Regul'App_ for where each code sits in the frames. Thresholds are the factory defaults (the `Seuil …` diagnostic entities show the ones actually configured).

**Regulator alarms** (`Alarm` / `Alarm Text`)

| Code | Meaning | Effect on the device | What to do |
|---|---|---|---|
| E.10 | pH probe reading error: reading < 5.2 or > 9.5 (5.5 on V2 units) | pH regulation inhibited, chlorine production continues | Check the pH with another test, rebalance the water, check or replace the probe |
| E.11 | pH stagnant: no significant change despite several injections | pH regulation inhibited, chlorine production continues | Empty can, faulty pump, split peristaltic tube, clogged strainer, pinched or blocked pipe |
| E.12 | Not described in the 2021 manual. On this generation the Corelec app shows the **flow-switch** icon for it (most likely no flow on the regulator side). The newer Wi-Fi _DUO+ V2_ units reuse E.12 for "water below 15 °C" (alert only) | ? | Check the flow; check the water temperature |
| E.13 | pH below the alarm threshold (default 6) | pH regulation inhibited, chlorine production continues (V2 units: alert only) | Usually an empty corrector can and a natural pH drift: rebalance the water, replace the can |
| E.14 | pH above the alarm threshold (default 9) | Same as E.13 | Same as E.13 |
| E.15 | Reverse correction: pH moves the wrong way (by 3 % within 10 min after an injection) | Injection blocked until the next power-on. On the 3rd occurrence, blocked until an alarm reset. Chlorine production continues | Wrong product on the pump: put the right can on the right pump, rebalance the water, then **Reset Alarmes** |
| E.18 | Water too cold: below 12 °C | Chlorine production stopped (the device shows `!!!` instead of the temperature). Below 15 °C there is only an alert (see `E4`) | Winterize the pool |
| E.19 | Salt too low: below 2.0 g/L | Chlorine production stopped ("Sécurité salinité trop faible") | Too much refilling, a leak, or not enough salt at season start: add salt up to 5 g/L |
| E.20 | Redox too high: above 950 mV | Chlorine production stopped | Manual chlorine added, pool covered, or inconsistent probe: uncover the pool, wait for the level to drop, check TAC / pH / TH / stabiliser / salt |
| E.21 | Redox low: below 350 mV | Alert only, production continues | Low salt, filtration time too short, stabiliser out of range, probe calibration, unbalanced water, or a faulty chlorinator |
| E.22 | Redox too low: below 250 mV (faulty or disconnected probe, or very low chlorine) | Chlorine production stopped | Common at commissioning: shock-chlorinate or restart production with **Boost Start 2h**. Check TH / TAC / stabiliser (> 30 ppm is too much), check the probe connection, test the probe in 450 / 650 mV solutions |

E.10 to E.22 are read from the main alarm byte. The Redox alarms may instead come through the separate `Alarm Rdx` field, whose numbering is not known yet (it has never been seen non-zero on the test unit).

**Warnings** (`Warning` / `Warning Text`, a bit field, so several can show at once)

| Text | Meaning (on the device display) |
|---|---|
| `E2 Sel` | `!.!` alert: salt below 3.0 g/L (production continues down to 2.0 g/L), or water temperature above 35 °C or below 15 °C (the salt reading can no longer be temperature-compensated) |
| `E4 Température` | Water below 15 °C (`!!!` alternating with the temperature), production continues |
| `E8 Redox` | Probably the E.21 "Redox low" alert (deduced, not confirmed) |

The display also has a `?.?` alert, meaning the salt probe is not calibrated or needs recalibrating. It probably uses the remaining warning bit, which has not been observed yet.

**Chlorinator alarms** (`Elx Alarm` / `Alarm Elx Text`)

| Code | Meaning | What to do |
|---|---|---|
| 1 | Cell short-circuited or **scaled**, or salt level above the selected salt range | Check the plates. Clean the cell in a cleaning solution. Check the **Salinité** range |
| 2 | Alert (not an alarm): low salt, water too cold, or cell near end of life | Add salt up to 5 g/L. Below 15 °C, switch the chlorinator off. Replace the cell after about 15,000 h |
| 3 | Cell worn out, missing or badly connected, no salt in the water, or **no water / air in the cell housing** | Check the connection and the salt level, remove air leaks in the hydraulic circuit |
| 4 | Electrical short-circuit in the device (plates touching, scale) | Disconnect the cell: if the alarm stays, the device is at fault. Otherwise reseat or clean the cell |
| 5 | Not documented | — |
| 6 | Device over-temperature (room above 50 °C while running at full power) | Stop the device, ventilate the technical room, restart |
| 7 | **No flow** in the cell housing: flow detector faulty or badly placed, closed valve, filtration pump stopped, or device not slaved to the pump | Restore the flow, check or replace the flow detector, remove air leaks |

Short alarms 1 or 3 lasting about a minute when filtration starts are common (air in the cell housing). Act only if they persist.

Older chlorinators driven by an external _Akeron Regul Redox_ through their flow-switch input also show alarm 7 whenever the Redox is above its setpoint. That is the normal way such setups pause production.

## Good to know / known limitations

- **`Elx` is the production applied right now, not a fixed setpoint.** The Akeron reverses the cell polarity periodically (every 4 h by default). It then pauses production for about a minute and reports **10 %**. It may also report **100 %** for a few seconds. These blips are normal, and the value returns to your setpoint by itself. The **Akeron Elx Set** slider follows the same byte, because the device uses a single field for both.
- **Update rate**: every value refreshes about every 30 s. After a write, the value is read back 3 s later.
- **Some frames are lost**: the Akeron occasionally sends truncated frames (bytes missing inside the device itself). The firmware discards them (wrong length or CRC). Values simply refresh at the next poll, so nothing needs to be done.
- **One BLE client at a time**: if the Corelec app cannot connect while the ESP32 is connected, turn off **Connect to Akeron Device**, use the app, then turn it back on.
- **No PIN needed**: the PIN code is only checked by the phone app, not by the BLE protocol.
- **Dangerous functions are deliberately not exposed** (factory reset, model change, PIN change).
- **Not decoded yet**: the meaning of E.12 and chlorinator alarm 5, the mapping of the `Alarm Rdx` field, and a few unused bytes. The date frame `J` (setting the clock) and the `B` frame are not used.

## Troubleshooting

| Symptom | What to check |
|---|---|
| `Connection Status` stays `Scanning...` | Is the Akeron powered (filtration running)? Is the MAC address correct? Is the ESP32 within range? Is a phone connected to it with the Corelec app? |
| `Connected` but all values stay `unknown` | Look at the ESPHome logs. If `Erreurs CRC` grows, turn off **Contrôle CRC trames** and report the logs. If nothing is decoded at all, your unit may send 17-byte frames: uncomment the `raw_hex` log line in the `akeron_data` sensor, reflash, and open an issue with the output. |
| `Error reading char at handle …` in old logs | Fixed: the internal `akeron_data` sensor is no longer polled (notifications only). |
| A setpoint changes by itself | Check that you run a version that includes commit `dcb9ecf` (read-back no longer written to the device). Short `Elx` blips are normal (see above). |
| Wi-Fi lost | Connect to the `Esp-Akeron Fallback Hotspot` access point to reconfigure Wi-Fi. |

Useful log tags (set in `logger:`, level `debug` by default): `akeron_data` (decoded frames), `akeron send` (every write sent to the device), `ble_client`, `ble_scanner`.

## Protocol notes

The protocol was cross-checked against the official Corelec Android app and the Akeron manual.

- **GATT**: service `0bd51666-e7cb-469b-8e4d-2742f1ba77cc`, a single characteristic `e7add780-b042-4876-aae1-112855353cc1` for writes and notifications.
- **Read request** (6 bytes): `2A 52 3F <mnemonic> CRC 2A`, i.e. `* R ? X crc *`.
- **Response / write** (17 bytes): `2A <mnemonic> d2 … d14 CRC 2A`. **CRC = XOR** of bytes 0-14. In a write, `0xFF` means "do not change".
- **Mnemonics**: `M` measurements and status, `S` pH, `E` Redox, `A` chlorinator, `D` thresholds and forced outputs, `B` (unknown), `J` date.
- On this device, each answer arrives as two notifications: `*` alone, then 16 bytes starting with the mnemonic. In the YAML, `x[i]` is therefore byte `i+1` of the 17-byte frame.

| Frame | Main fields (byte index in the 17-byte frame) |
|---|---|
| M | 2-3 pH ×100 · 4-5 Redox mV · 6-7 temperature ×10 · 8-9 salt ×10 · 10 alarm · 11 warnings (bits 0-3) + Redox alarm (bits 4-7) · 12 model (bits 0-3) + outputs (filtration, chlorine, pH-, pH+) · 13 configuration bits |
| S | 2-3 pH setpoint ×100 · 10-11 / 12-13 pH alarm max / min ×100 |
| E | 2-3 Redox setpoint mV |
| A | 2 production % · 3-4 boost minutes · 9 cover production % · 10 salt range + flow switch + cover bits · 12 chlorinator alarm · 13 Sleep/Timer bits + duration |
| D | 4-7 temperature thresholds · 8 / 9 salt warning / alarm ×10 · 10 forced outputs (write) |

## Credits

- [Hackeron](https://github.com/sylvaing/Hackeron): the original Akeron ⇄ MQTT gateway and protocol reverse-engineering.
- ESPHome port by **emoulin**.
- Discussion (French): [HACF forum thread](https://forum.hacf.fr/t/hackeron-gateway-mqtt-electrolyseur-piscine/11947).

Contributions and issues are welcome, especially logs from other Akeron models.
