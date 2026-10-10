# hackeron-esphome

> 🇫🇷 **Version française plus bas : [aller à la version française](#version-francaise).**

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
- **Not fully decoded yet**: the exact meaning of E.12 on this generation, chlorinator alarm 5, the mapping of the `Alarm Rdx` field, and a few unused bytes. The date frame `J` (setting the clock) and the `B` frame are not used.

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

---

<a id="version-francaise"></a>

# hackeron-esphome (version française)

**Passerelle Bluetooth entre un régulateur de piscine Corelec / CCEI _Akeron_ et Home Assistant, sur un ESP32 avec ESPHome.**

Les appareils Akeron (électrolyseurs au sel, avec régulation pH et Redox en option) ne sont accessibles que par l'application mobile Corelec, en Bluetooth Low Energy. Ce firmware transforme un ESP32 bon marché, placé près de l'Akeron, en passerelle permanente. Il se connecte en BLE, lit toutes les mesures et tous les réglages environ toutes les 30 secondes, et les expose à Home Assistant sous forme d'entités natives. Vous pouvez alors tracer l'historique de l'eau, être prévenu des alarmes et modifier les consignes depuis Home Assistant, sans l'application.

```
 ┌──────────────┐   BLE (GATT)    ┌──────────────┐  Wi-Fi / API ESPHome  ┌────────────────┐
 │    Akeron    │ ◄─────────────► │    ESP32     │ ◄───────────────────► │ Home Assistant │
 │  régulateur  │  portée ~5-10 m │ ce firmware  │   (chiffrée)          │    entités     │
 └──────────────┘                 └──────────────┘                       └────────────────┘
```

## Sommaire

- [Ce que vous obtenez](#fr-fonctions)
- [Compatibilité](#fr-compatibilite)
- [Fonctionnement](#fr-fonctionnement)
- [Installation](#fr-installation)
- [Liste des entités](#fr-entites)
- [Codes d'alarme et d'alerte](#fr-alarmes)
- [Bon à savoir / limites connues](#fr-bon-a-savoir)
- [Dépannage](#fr-depannage)
- [Notes sur le protocole](#fr-protocole)
- [Crédits](#fr-credits)

<a id="fr-fonctions"></a>

## Ce que vous obtenez

- **Mesures** : pH, Redox (mV), température de l'eau, sel (g/L).
- **Consignes, en lecture et en écriture** : consigne pH, consigne Redox, production de chlore (ELX %), production en mode volet, plage de salinité.
- **États** : production de chlore en cours, pompes pH en marche, flow switch, mode volet, boost, modes Sleep/Timer, modèle de l'appareil.
- **Alarmes et alertes** avec des textes lisibles (régulateur, électrolyseur, champ d'alertes), ainsi que les seuils bas de température et de sel réglés dans l'appareil.
- **Actions** : boost de 2 h (marche/arrêt), reset des alarmes, marche forcée de la pompe pH-, étalonnage des sondes (pH, Redox, sel, température), configuration des capteurs et des pompes.
- **Robustesse** : recherche et reconnexion BLE automatiques, contrôle du CRC de chaque trame reçue (avec un compteur d'erreurs), entités remises à « inconnu » quand la liaison tombe, pour ne jamais afficher de valeurs périmées.

<a id="fr-compatibilite"></a>

## Compatibilité

| | |
|---|---|
| **Appareil de piscine** | Gamme Corelec / CCEI Akeron : le firmware reconnaît les modèles _Regul 4Rx, Regul, Regul 3, Duo, Duo Regul 3, Duo Regul 4Rx, Akeron, Duo Regul 4Amp_. Développé et testé sur un **Duo Regul 4Rx** (électrolyseur + pH + Redox). Les entités des fonctions absentes de votre modèle (par exemple le Redox sur un appareil pH seul) restent simplement vides. |
| **Passerelle** | Toute carte **ESP32** classique avec Bluetooth (le YAML vise `esp32dev`, framework ESP-IDF). Les autres variantes d'ESP32 avec BLE (C3, S3…) n'ont pas été testées et demanderaient un autre bloc `esp32:`. L'ESP8266 n'est **pas** pris en charge (pas de Bluetooth). |
| **ESPHome** | 2025.5.0 ou plus récent (`min_version`) ; utilisé actuellement avec ESPHome 2026.9. |
| **Home Assistant** | Toute version compatible avec votre ESPHome, via l'intégration ESPHome native. Pas besoin de MQTT. |

> Certains Akeron envoient leurs trames autrement en BLE (une seule notification de 17 octets au lieu de `*` puis 16 octets). Ces appareils ne sont pas encore décodés. Si vous voyez `Connected` mais que toutes les mesures restent `unknown`, voir le [Dépannage](#fr-depannage).

<a id="fr-fonctionnement"></a>

## Fonctionnement

1. **Découverte et connexion.** 30 s après le démarrage, l'ESP32 lance une recherche BLE passive. Quand il voit l'Akeron dont l'adresse MAC est configurée (`Device Present`), il s'y connecte (`ble_client`) et arrête la recherche. En cas de déconnexion, il vide les entités et relance la recherche.
2. **Interrogation.** Toutes les 30 s, tant qu'il est connecté, l'ESP32 envoie cinq demandes de lecture espacées de 5 s : `M` (mesures et états), `S` (réglages pH), `A` (électrolyseur), `E` (Redox), `D` (seuils). L'Akeron répond à chacune par une notification, que le firmware vérifie (longueur + CRC) puis décode en entités.
3. **Écriture.** Quand vous modifiez une consigne ou appuyez sur un bouton dans Home Assistant, l'ESP32 envoie une trame d'écriture où tous les octets valent `0xFF` (« ne pas modifier »), sauf le champ concerné. Trois secondes plus tard, il relit la trame : Home Assistant affiche donc ce que l'Akeron a réellement appliqué.
4. **La relecture n'écrit jamais.** Les valeurs lues dans l'Akeron sont seulement *affichées* (`publish_state`), jamais renvoyées à l'appareil. Les anciennes versions les renvoyaient, ce qui pouvait remplacer en silence votre consigne de chlore par une valeur passagère (voir l'historique git, commit `dcb9ecf`).

L'Akeron est souvent alimenté en même temps que la pompe de filtration. Dans ce cas, il disparaît chaque nuit et les entités affichent `unknown` jusqu'à la reprise de la filtration. C'est normal : l'ESP32 continue de chercher et se reconnecte tout seul en une minute environ.

<a id="fr-installation"></a>

## Installation

### 1. Matériel

- Une carte de développement ESP32 (ESP32-WROOM / « ESP32 DevKit ») et une alimentation USB 5 V.
- Placez-la **à portée Bluetooth de l'Akeron**, idéalement dans le local technique, à quelques mètres de l'appareil. Les murs et les coffrets métalliques réduisent beaucoup la portée BLE ; vérifiez aussi que le Wi-Fi passe à cet endroit.

### 2. Trouver l'adresse MAC Bluetooth de l'Akeron

Utilisez une application de scan BLE sur un téléphone (par exemple _nRF Connect_ ou _LightBlue_), filtration en marche. L'Akeron s'annonce sous le nom **`CORELEC Regulateur`** ou **`REGUL.`**. Notez son adresse, par exemple `B4:E3:F9:65:71:74`.

Fermez d'abord l'application Corelec : tant qu'un téléphone est connecté, l'Akeron peut cesser de s'annoncer.

Autre méthode : flashez le firmware avec n'importe quelle adresse, décommentez le bloc `on_ble_advertise` sous `esp32_ble_tracker:` et lisez les logs. Chaque appareil BLE détecté y apparaît avec son nom et son adresse.

### 3. Créer l'appareil dans ESPHome

1. Dans l'**ESPHome Device Builder** (module complémentaire Home Assistant ou `esphome dashboard`), créez un nouvel appareil, choisissez ESP32, puis **éditez**-le et remplacez son contenu par [`hackeron-esphome.yaml`](hackeron-esphome.yaml).
2. Renseignez l'adresse de votre Akeron dans le bloc `substitutions:` :
   ```yaml
   substitutions:
     mac_akeron: "B4:E3:F9:65:71:74"   # adresse MAC BLE de votre Akeron
   ```
3. Si vous le souhaitez, changez `esphome: name:` (par défaut `hackeron-esp`). Ce nom détermine les identifiants des entités dans Home Assistant.
4. Vérifiez que votre `secrets.yaml` (bouton **Secrets** d'ESPHome) définit :
   ```yaml
   wifi_ssid: "votre-ssid"
   wifi_password: "votre-mot-de-passe-wifi"
   api_encryption_key: "clé base64 de 32 octets"   # à générer sur https://esphome.io/components/api.html
   ota_password: "un-mot-de-passe-pour-les-mises-a-jour-ota"
   ```

### 4. Flasher

- **Premier flash en USB** : branchez l'ESP32 sur l'ordinateur qui fait tourner le tableau de bord (ou utilisez **Install → Manual download** puis flashez avec [web.esphome.io](https://web.esphome.io)). Sur certaines cartes, il faut maintenir le bouton **BOOT** au début du flash.
- **Mises à jour suivantes** : **Install → Wirelessly** (OTA, sans fil).

Si le Wi-Fi est injoignable, l'ESP32 ouvre un point d'accès de secours `Esp-Akeron Fallback Hotspot` (mot de passe = celui de votre Wi-Fi), avec un portail captif pour saisir de nouveaux identifiants Wi-Fi.

### 5. L'ajouter à Home Assistant

Home Assistant découvre l'appareil automatiquement (**Paramètres → Appareils et services → Découvert → ESPHome**). Validez et saisissez la clé `api_encryption_key`. Filtration en marche, **Connection Status** doit passer à `Connected` en moins d'une minute, et les mesures doivent apparaître à l'interrogation suivante (30 s au plus).

<a id="fr-entites"></a>

## Liste des entités

Les identifiants ci-dessous supposent le nom d'appareil `hackeron-esp` (par exemple `sensor.hackeron_esp_ph`). Les noms d'entités sont ceux du firmware, en anglais ou en français selon les cas.

### Mesures et états

| Entité | Type | Description |
|---|---|---|
| PH | capteur (pH) | pH mesuré |
| Redox | capteur (mV) | Potentiel Redox mesuré (plage acceptée 350-1000 mV, les autres lectures sont ignorées) |
| Water Temperature | capteur (°C) | Température de l'eau |
| Salt | capteur (g/L) | Taux de sel (0-40 g/L) |
| PH Setpoint / Redox Setpoint | capteur | Consignes actuelles, lues dans l'appareil |
| PH Threshold Min / Max | capteur (pH) | Seuils d'alarme pH |
| Elx | capteur (%) | Production de chlore **appliquée à l'instant** (voir [Bon à savoir](#fr-bon-a-savoir)) |
| Boost Time | capteur (min) | Temps de boost restant |
| ELX Pump | binaire | Production de chlore active |
| PH Pump / PH Minus Pump | binaire | Pompe doseuse pH+ / pH- en marche |
| Forced Pump | binaire | Sorties forcées (marche manuelle) |
| Flow Switch Active | binaire | Entrée flow switch (détecteur de débit) |
| Cover Active | binaire | Mode volet actif |
| Boost 2h | binaire | Boost en cours |
| Mode Sleep / Mode Timer | binaire | Mode Sleep de l'Akeron (arrêt après N heures de filtration) / mode Timer (« 24h/24 », N/24 de chaque heure) |
| Durée Sleep Timer | capteur (h) | N pour le mode Sleep/Timer (unité déduite de la notice, non confirmée) |
| Model | texte | Modèle annoncé par l'appareil |

### Commandes

| Entité | Type | Description |
|---|---|---|
| Akeron PH Set | nombre | Consigne pH, 6,5-7,8, pas de 0,05 |
| Akeron Redox Set | nombre | Consigne Redox, 350-900 mV, pas de 10 |
| Akeron Elx Set | nombre | Consigne de production de chlore, 5-100 %, pas de 5 |
| Cover Production | nombre | Production en mode volet, en proportion de la production normale, 5-50 % |
| Cover Force | interrupteur | Forcer le mode volet |
| Salinité | sélection | Plage de sel : 4-8, 8-15 ou > 15 g/L |
| Boost Start 2h / Boost Stop | bouton | Lancer un boost de 2 heures / l'arrêter |
| Reset Alarmes | bouton | Acquitter et réinitialiser les alarmes de l'appareil |
| Force pH- | bouton | Faire tourner la pompe pH- en marche forcée (fonction « forçage des sorties pendant 1 minute » de l'Akeron) |

### Configuration et étalonnage (catégorie _config_)

| Entité | Type | Description |
|---|---|---|
| Config Pompe PH+ / Config Pompe PH- | interrupteur | Déclarer les pompes pH installées. Le firmware refuse de désactiver la dernière (l'appareil n'accepte pas « aucune pompe »). |
| Config Capteur Temp / Config Capteur Sel / Config Flow Switch | interrupteur | Déclarer les capteurs installés |
| Value Calibrate PH / Redox / Salt / Temp + Send Calibrate … | nombre + bouton | Étalonnage : mesurez l'eau avec un **instrument de référence**, saisissez cette valeur, puis appuyez sur le bouton _Send_ correspondant. Les plages suivent l'application officielle (pH 6,5-8,5, Redox 200-650 mV, sel 3-35 g/L, température 8-40 °C). |
| Contrôle CRC trames | interrupteur | Active le contrôle du CRC des trames reçues (activé par défaut ; ne le couper que pour du débogage). |

### Diagnostic

| Entité | Description |
|---|---|
| Connection Status | `Scanning...`, `Found - Connecting...`, `Connected` ou `Idle` |
| Device Present | Akeron vu dans les annonces BLE ; c'est ce qui déclenche la connexion. Passe normalement à « off » quelques minutes après la connexion, car un Akeron connecté cesse de s'annoncer. |
| Connect to Akeron Device | À couper pour libérer la liaison BLE (par exemple pour utiliser l'application Corelec), à rallumer pour se reconnecter |
| BLE Scanner | Lance / arrête la recherche BLE |
| Alarm, Alarm Text | Alarme du régulateur (code et texte) |
| Warning, Warning Text | Alertes (champ de bits, les textes peuvent se cumuler : `E2 Sel ; E4 Température`) |
| Elx Alarm, Alarm Elx Text | Alarme de l'électrolyseur |
| Alarm Rdx | Champ brut d'alarme Redox |
| Seuil alarme / alerte température basse, Seuil alerte / alarme sel bas | Seuils réglés dans l'appareil |
| Erreurs CRC | Nombre de trames reçues rejetées pour CRC incorrect |
| hackeron restart | Redémarrer l'ESP32 |

<a id="fr-alarmes"></a>

## Codes d'alarme et d'alerte

Sources : les notices officielles Corelec ([2021 _SALT DUO / SALT REGUL pH / REGUL3 / REGUL4 Rx_](https://www.easy-blue.fr/uploads/pdf/2021-akeron-duo-notice.pdf), section 6, qui correspond à la génération Bluetooth gérée ici ; les notices plus récentes _SALT DUO V2_ et _REGUL REDOX 1.4_ sur [akeron.fr](https://www.akeron.fr/nos-supports-techniques/documentation)), ainsi que l'application Corelec _Regul'App_ décompilée pour l'emplacement de chaque code dans les trames. Les seuils indiqués sont les valeurs d'usine (les entités de diagnostic `Seuil …` montrent ceux réellement réglés).

**Alarmes du régulateur** (`Alarm` / `Alarm Text`)

| Code | Signification | Effet sur l'appareil | Que faire |
|---|---|---|---|
| E.10 | Erreur de lecture de la sonde pH : lecture < 5,2 ou > 9,5 (5,5 sur les appareils V2) | Régulation pH inhibée, production de chlore maintenue | Contrôler le pH par un autre moyen, rééquilibrer l'eau, vérifier ou changer la sonde |
| E.11 | pH stagnant : pas de variation significative malgré plusieurs injections | Régulation pH inhibée, production de chlore maintenue | Bidon vide, pompe défectueuse, tube péristaltique percé, crépine bouchée, tuyau pincé ou obstrué |
| E.12 | Absente de la notice 2021. Sur cette génération, l'application Corelec affiche l'icône **flow switch** (très probablement absence de débit côté régulateur). Les appareils Wi-Fi plus récents _DUO+ V2_ réutilisent E.12 pour « eau sous 15 °C » (simple alerte) | ? | Vérifier le débit ; vérifier la température de l'eau |
| E.13 | pH sous le seuil d'alarme (6 par défaut) | Régulation pH inhibée, production de chlore maintenue (appareils V2 : simple alerte) | En général bidon de correcteur vide et dérive naturelle du pH : rééquilibrer l'eau, remplacer le bidon |
| E.14 | pH au-dessus du seuil d'alarme (9 par défaut) | Comme E.13 | Comme E.13 |
| E.15 | Correction inversée : le pH évolue dans le mauvais sens (de 3 % dans les 10 min suivant une injection) | Injection bloquée jusqu'à la prochaine mise en marche. À la 3e fois, bloquée jusqu'à un reset des alarmes. Production de chlore maintenue | Mauvais produit sur la pompe : mettre le bon bidon sur la bonne pompe, rééquilibrer l'eau, puis **Reset Alarmes** |
| E.18 | Eau trop froide : sous 12 °C | Production de chlore arrêtée (l'appareil affiche `!!!` à la place de la température). Sous 15 °C, simple alerte (voir `E4`) | Hiverner la piscine |
| E.19 | Sel trop bas : sous 2,0 g/L | Production de chlore arrêtée (« Sécurité salinité trop faible ») | Trop de remplissages, fuite, ou sel insuffisant en début de saison : faire l'appoint jusqu'à 5 g/L |
| E.20 | Redox trop fort : au-dessus de 950 mV | Production de chlore arrêtée | Ajout de chlore manuel, bassin couvert ou sonde incohérente : découvrir le bassin, attendre que le taux redescende, contrôler TAC / pH / TH / stabilisant / sel |
| E.21 | Redox faible : sous 350 mV | Simple alerte, production maintenue | Sel trop bas, temps de filtration trop court, stabilisant hors plage, étalonnage de la sonde, eau déséquilibrée ou électrolyseur défectueux |
| E.22 | Redox trop faible : sous 250 mV (sonde défectueuse ou débranchée, ou chlore très bas) | Production de chlore arrêtée | Fréquent à la mise en service : chlore choc ou relance de la production par **Boost Start 2h**. Contrôler TH / TAC / stabilisant (au-delà de 30 ppm, c'est trop), vérifier la connexion de la sonde, la tester dans des solutions à 450 / 650 mV |

E.10 à E.22 sont lus dans l'octet d'alarme principal. Les alarmes Redox pourraient aussi arriver par le champ séparé `Alarm Rdx`, dont la numérotation n'est pas encore connue (il n'a jamais été vu différent de zéro sur l'appareil de test).

**Alertes** (`Warning` / `Warning Text`, champ de bits : plusieurs peuvent s'afficher en même temps)

| Texte | Signification (affichage de l'appareil) |
|---|---|
| `E2 Sel` | Alerte `!.!` : sel sous 3,0 g/L (production maintenue jusqu'à 2,0 g/L), ou eau au-dessus de 35 °C ou sous 15 °C (la mesure de sel ne peut plus être corrigée en température) |
| `E4 Température` | Eau sous 15 °C (`!!!` en alternance avec la température), production maintenue |
| `E8 Redox` | Probablement l'alerte E.21 « Redox faible » (déduit, non confirmé) |

L'écran connaît aussi l'alerte `?.?` : la sonde de sel n'est pas étalonnée ou doit l'être à nouveau. Elle utilise probablement le bit d'alerte restant, encore jamais observé.

**Alarmes de l'électrolyseur** (`Elx Alarm` / `Alarm Elx Text`)

| Code | Signification | Que faire |
|---|---|---|
| 1 | Électrode en court-circuit ou **entartrée**, ou taux de sel supérieur à la plage sélectionnée | Contrôler les plaques. Nettoyer l'électrode dans une solution de nettoyage. Vérifier la plage **Salinité** |
| 2 | Alerte (pas une alarme) : manque de sel, eau trop froide, ou électrode en fin de vie | Faire l'appoint de sel jusqu'à 5 g/L. Sous 15 °C, éteindre l'électrolyseur. Changer l'électrode au-delà d'environ 15 000 h |
| 3 | Électrode usée, absente ou mal connectée, pas de sel dans l'eau, ou **manque d'eau / air dans le vase** | Vérifier la connectique et le taux de sel, éliminer les prises d'air du circuit hydraulique |
| 4 | Court-circuit électrique de l'appareil (plaques qui se touchent, tartre) | Débrancher l'électrode : si l'alarme reste, l'appareil est en cause. Sinon, replacer ou nettoyer l'électrode |
| 5 | Non documentée | — |
| 6 | Surchauffe de l'appareil (local à plus de 50 °C et fonctionnement à pleine puissance) | Arrêter l'appareil, ventiler le local technique, redémarrer |
| 7 | **Pas de débit** dans le vase : détecteur de débit hors service ou mal placé, vanne fermée, pompe de filtration arrêtée, ou appareil non asservi à la pompe | Rétablir le débit, vérifier ou changer le détecteur, éliminer les prises d'air |

Des alarmes 1 ou 3 brèves, d'environ une minute, au démarrage de la filtration sont courantes (air dans le vase). N'intervenez que si elles durent.

Les électrolyseurs plus anciens pilotés par un _Akeron Regul Redox_ externe via leur entrée flow switch affichent aussi l'alarme 7 dès que le Redox dépasse sa consigne. C'est ainsi que ces installations suspendent normalement la production.

<a id="fr-bon-a-savoir"></a>

## Bon à savoir / limites connues

- **`Elx` est la production appliquée à l'instant, pas une consigne figée.** L'Akeron inverse régulièrement la polarité de l'électrode (toutes les 4 h par défaut). Il suspend alors la production pendant une minute environ et annonce **10 %**. Il peut aussi annoncer **100 %** pendant quelques secondes. Ces pics sont normaux et la valeur revient seule à votre consigne. Le curseur **Akeron Elx Set** suit le même octet, car l'appareil utilise un seul champ pour les deux.
- **Fréquence de mise à jour** : chaque valeur est rafraîchie environ toutes les 30 s. Après une écriture, la valeur est relue 3 s plus tard.
- **Certaines trames sont perdues** : l'Akeron envoie de temps en temps des trames tronquées (octets perdus à l'intérieur même de l'appareil). Le firmware les écarte (longueur ou CRC incorrects). Les valeurs se mettent simplement à jour à l'interrogation suivante, il n'y a rien à faire.
- **Un seul client BLE à la fois** : si l'application Corelec ne parvient pas à se connecter pendant que l'ESP32 est connecté, coupez **Connect to Akeron Device**, utilisez l'application, puis rallumez-le.
- **Pas de code PIN** : le code PIN n'est vérifié que par l'application mobile, pas par le protocole BLE.
- **Les fonctions dangereuses ne sont volontairement pas exposées** (réinitialisation usine, changement de modèle, changement de PIN).
- **Pas encore entièrement décodé** : le sens exact d'E.12 sur cette génération, l'alarme 5 de l'électrolyseur, la correspondance du champ `Alarm Rdx` et quelques octets inutilisés. La trame de date `J` (mise à l'heure) et la trame `B` ne sont pas utilisées.

<a id="fr-depannage"></a>

## Dépannage

| Symptôme | À vérifier |
|---|---|
| `Connection Status` reste sur `Scanning...` | L'Akeron est-il alimenté (filtration en marche) ? L'adresse MAC est-elle correcte ? L'ESP32 est-il à portée ? Un téléphone y est-il connecté avec l'application Corelec ? |
| `Connected` mais toutes les valeurs restent `unknown` | Consultez les logs ESPHome. Si `Erreurs CRC` augmente, coupez **Contrôle CRC trames** et transmettez les logs. Si rien n'est décodé du tout, votre appareil envoie peut-être des trames de 17 octets : décommentez la ligne de log `raw_hex` du capteur `akeron_data`, reflashez et ouvrez une issue avec la sortie. |
| `Error reading char at handle …` dans d'anciens logs | Corrigé : le capteur interne `akeron_data` n'est plus interrogé (notifications seulement). |
| Une consigne change toute seule | Vérifiez que votre version inclut le commit `dcb9ecf` (la relecture n'est plus écrite dans l'appareil). De courts pics de `Elx` sont normaux (voir plus haut). |
| Wi-Fi perdu | Connectez-vous au point d'accès `Esp-Akeron Fallback Hotspot` pour reconfigurer le Wi-Fi. |

Étiquettes de log utiles (réglées dans `logger:`, niveau `debug` par défaut) : `akeron_data` (trames décodées), `akeron send` (chaque écriture envoyée à l'appareil), `ble_client`, `ble_scanner`.

<a id="fr-protocole"></a>

## Notes sur le protocole

Le protocole a été vérifié par rapport à l'application Android officielle de Corelec et à la notice Akeron.

- **GATT** : service `0bd51666-e7cb-469b-8e4d-2742f1ba77cc`, une seule caractéristique `e7add780-b042-4876-aae1-112855353cc1` pour les écritures et les notifications.
- **Demande de lecture** (6 octets) : `2A 52 3F <mnémonique> CRC 2A`, soit `* R ? X crc *`.
- **Réponse / écriture** (17 octets) : `2A <mnémonique> d2 … d14 CRC 2A`. **CRC = XOR** des octets 0 à 14. Dans une écriture, `0xFF` signifie « ne pas modifier ».
- **Mnémoniques** : `M` mesures et états, `S` pH, `E` Redox, `A` électrolyseur, `D` seuils et sorties forcées, `B` (inconnue), `J` date.
- Sur cet appareil, chaque réponse arrive en deux notifications : `*` seul, puis 16 octets commençant par la mnémonique. Dans le YAML, `x[i]` correspond donc à l'octet `i+1` de la trame de 17 octets.

| Trame | Champs principaux (position dans la trame de 17 octets) |
|---|---|
| M | 2-3 pH ×100 · 4-5 Redox mV · 6-7 température ×10 · 8-9 sel ×10 · 10 alarme · 11 alertes (bits 0-3) + alarme Redox (bits 4-7) · 12 modèle (bits 0-3) + sorties (filtration, chlore, pH-, pH+) · 13 bits de configuration |
| S | 2-3 consigne pH ×100 · 10-11 / 12-13 seuils d'alarme pH max / min ×100 |
| E | 2-3 consigne Redox mV |
| A | 2 production % · 3-4 boost en minutes · 9 production volet % · 10 plage de sel + flow switch + bits volet · 12 alarme électrolyseur · 13 bits Sleep/Timer + durée |
| D | 4-7 seuils de température · 8 / 9 alerte / alarme sel ×10 · 10 sorties forcées (écriture) |

<a id="fr-credits"></a>

## Crédits

- [Hackeron](https://github.com/sylvaing/Hackeron) : la passerelle Akeron ⇄ MQTT d'origine et la rétro-ingénierie du protocole.
- Portage ESPHome par **emoulin**.
- Discussion : [fil du forum HACF](https://forum.hacf.fr/t/hackeron-gateway-mqtt-electrolyseur-piscine/11947).

Les contributions et les issues sont les bienvenues, en particulier les logs d'autres modèles Akeron.
