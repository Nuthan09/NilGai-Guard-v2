# NilGai Guard v2 — field system (ESP8266 edition)

IoT Lab, CSE Department, IIT (BHU)

This version matches the hardware you have now: **NodeMCU ESP-12E (ESP8266) boards**, sensor nodes at corners **A** and **C**, **transmitter-only** corners **B** and **D** (IR transmitters + 12 V battery, no controller), and a **portable master** carried by a person.

```
        A ●──────── side AB ────────○ B     ● sensor node: NodeMCU + 4 IR receivers + horn
        │                           │       ○ transmitter-only corner: 4 IR transmitters + 12 V battery
     side DA                     side BC
        │                           │
        D ○──────── side CD ────────● C

  A watches AB (channel 1) + DA (channel 2)      C watches BC (channel 1) + CD (channel 2)

  A, C ──ESP-NOW──►  portable master  ──phone hotspot──►  MQTT ──► GitHub dashboard
                          │                                    └─► Google Sheets (history + labels)
                          └── own Wi-Fi "NilGaiGuard-Master" → http://192.168.4.1 (offline page)
```

How it behaves
- **A and C decide everything themselves**: beam timing, person / NilGai / small animal, entering / exiting / turned back, learning — and they **sound their own horn** at once. The horn never waits for the master.
- Every crossing is written to the sensor node's **own flash first** (about 500 crossings). When the master is in radio range it is sent over ESP-NOW and only deleted after the master confirms it. When the person carrying the master walks away, nothing is lost; the crossings arrive when the master comes back (marked *delivered late* on the pages).
- The master stores every crossing in its flash, sends it to the dashboard (MQTT) and Google Sheets, and sends Telegram alerts. A small buzzer on the master (optional) warns the person carrying it.
- Every crossing is kept for training, including wrong and uncertain ones; you label them on the dashboard or the offline page.

Folder contents

| Path | What it is |
|---|---|
| `firmware/ESP8266/NilGaiGuard_v2_Slave_8266/` | **sensor node sketch** (flash on A and C) |
| `firmware/ESP8266/NilGaiGuard_v2_Master_8266/` | **portable master sketch** |
| `web/index.html` | dashboard for GitHub Pages (repo `NilGai-Guard-v2`) |
| `cloud/GoogleSheets_Code.gs` | Google Apps Script that stores the data in your Sheet |
| `tools/train_from_labels.py` | trains a better model from your labelled crossings |
| `firmware/ESP32_optional/` | the same system for ESP32 boards (Bluetooth or ESP-NOW), if you buy ESP32s later |

---

## 1. Hardware and wiring

### Sensor node (A and C) — NodeMCU ESP-12E

| NodeMCU pin | GPIO | Connect to |
|---|---|---|
| **D7** | 13 | channel 1 **profile** receiver (upper, OUTER beam) — relay NC |
| **D6** | 12 | channel 1 **gait** receiver (lower, INNER / field-side beam) — relay NC |
| **D1** | 5 | channel 2 **profile** receiver — relay NC |
| **D2** | 4 | channel 2 **gait** receiver — relay NC |
| **D5** | 14 | horn amplifier (CA-3110) input: D5 → 1 kΩ → 1 µF → amplifier IN, GND → amplifier GND |
| **D8** | 15 | strobe LED (via 220 Ω) or transistor base (optional) |
| D4 | 2 | on-board blue LED (link status, nothing to connect) |
| A0 | — | optional battery sense: +12 V → **1.2 MΩ** → A0, then set `VBAT_FULLSCALE = 15.2` |
| GND | — | every receiver **relay COM**, amplifier GND, LM7805/buck GND, battery − |
| VIN | — | +5 V from the LM7805 (or a 12 V→5 V buck module) |

Receivers: terminal 1 = +12 V, terminal 2 = GND, relay **COM → GND**, relay **NC → the pin**. The sketch uses the internal pull-ups and learns at power-up which level means “blocked” (beams must be **clear** for the first 4 s). Channel 1 is the same wiring as the v1 demo (D7 / D6 / horn D5), so the v1 board can be reused for node A.

> **Never put a beam on D3, D4 or D8**: they decide how the ESP8266 boots. A blocked beam there at power-on stops the board from starting. D0 has no interrupt.

Power per sensor node: 12 V battery → receivers (4 × ~30 mA) + amplifier; 12 V → LM7805 (with 0.33 µF in, 0.1 µF out) → NodeMCU VIN. The CA-3110 runs from 12 V directly.

### Transmitter-only corners (B and D)
Four IR transmitters (two facing A or C along each side: one upper = profile, one lower = gait) on the 12 V battery. Nothing to program. Mount the upper and lower pair the same way as at the receiving end, with the **profile beam on the outer side** and the gait beam on the field side, 30–50 cm apart.

### Portable master — NodeMCU ESP-12E
| Pin | Connect to |
|---|---|
| USB | power bank (5 V) |
| D5 | optional piezo buzzer (+), other leg to GND |
| D2 / D1 | optional SSD1306 128×64 OLED SDA / SCL (set `USE_OLED 1`) |

### Radio range (ESP-NOW, NodeMCU PCB antenna)
About 80–150 m in open field with the boards 1 m above the crop, less through dense crop. The master only needs to be in range **now and then** — the sensor nodes keep the data. If A or C sits in a metal box, put the NodeMCU near the lid or use a plastic box.

---

## 2. Arduino IDE setup (once)

1. **File → Preferences → Additional boards manager URLs**: `https://arduino.esp8266.com/stable/package_esp8266com_index.json`
2. **Tools → Board → Boards Manager** → install **esp8266 by ESP8266 Community** (3.1.x).
3. **Tools → Board → ESP8266 Boards → NodeMCU 1.0 (ESP-12E Module)**. Leave *Flash Size* at the default **4MB (FS:2MB OTA:~1019KB)** (the flash file system is needed).
4. **Library Manager**: install **ArduinoJson** (Benoit Blanchon, 7.x) and **PubSubClient** (Nick O'Leary). For the OLED only: **Adafruit SSD1306** + **Adafruit GFX**.

(If you see `WiFi.h: No such file or directory`, the board selected is wrong — that file is for ESP32. The ESP8266 sketches use `ESP8266WiFi.h`.)

## 3. Flash the sensor nodes (A and C)

Open `firmware/ESP8266/NilGaiGuard_v2_Slave_8266/NilGaiGuard_v2_Slave_8266.ino` and change **one line**:

```cpp
#define NODE_ID  "A"     // "C" for the other sensor node
```

Upload. Serial Monitor (115200) shows the sides, the detected polarity of all four beams, and `crossings waiting for the master`. Nothing else to set: node A watches AB + DA and node C watches BC + CD by default, and the master's configuration overrides that anyway.

Bench test commands in the Serial Monitor: `hb` (status), `horn:2000`, `mute`, `unmute`, `autodetect`, `calib`, `clear_queue`, `reboot`.

Blue LED: one short blink every 3 s = master in range; fast blink = master not heard for a minute (node still guarding).

## 4. Flash the master

Open `firmware/ESP8266/NilGaiGuard_v2_Master_8266/NilGaiGuard_v2_Master_8266.ino` and check:

```cpp
const char* WIFI_SSID = "nuthan";      // your phone hotspot
const char* WIFI_PASS = "12345667";
#define FARM_KEY      "iitbhu-nuthan09"   // must match the dashboard Settings (make it hard to guess)
#define DEFAULT_NODES "A,B,C,D"           // corners in clockwise order
#define PASSIVE_NODES "B,D"               // corners with transmitters only
#define USE_OLED 0                        // 1 if an SSD1306 is connected
```

Leave `SHEETS_URL`, `SHEETS_SECRET` and `TELEGRAM_TOKEN` empty and enter them later on the offline page (§9). Then the sketch you keep on your computer or GitHub contains no secrets.

The master always uses the fixed radio address **5E:4E:47:4D:53:01**, so the sensor nodes find it with no setup. They also find its radio channel by themselves (it follows the hotspot's channel).

Hotspot rules (the radio can only be on one channel):
- At power-up the master looks for the hotspot for 12 s.
- If it is not there, the master works **offline** and looks again for 15 s every 2 minutes. During those 15 s the sensor nodes may not reach it; they simply retry.
- Offline, the master still collects crossings, shows them on the offline page, and uploads everything once the hotspot is back. Opening the offline page also sets the master's clock from your phone, so crossings get the right time even without internet.

## 5. Wireless link — ESP-NOW

ESP8266 has no Bluetooth, so this edition uses **ESP-NOW** (Wi-Fi radio, peer-to-peer, no router). Each message is split into packets of 200 bytes; the master answers every message with a confirmation, the clock, and any commands or new settings for that node. A crossing is deleted from the node's flash only after that confirmation. Messages lost on the way are sent again; the master and Google Sheets ignore duplicates (each crossing has an id such as `A-12-37`: node, boot number, crossing number).

---

## 6. Dashboard v2 on GitHub Pages

Already set up at **https://nuthan09.github.io/NilGai-Guard-v2/** — upload the new `web/index.html` over the old one (*Add file → Upload files → Commit*). Try `?demo` at the end of the address for a simulated field with this exact layout.

New in this edition:
- B and D are drawn as **hollow corners** (transmitters only); their cards just remind you to check the battery.
- No master in the centre of the map (it is portable). A node the master cannot hear shows **out of range**, with the number of crossings it is still holding.
- Crossings that arrive late are listed with their real time, marked *delivered late*, and do not raise the red alarm banner again.
- **Configure field**: corner order, transmitter-only corners, *Auto-assign sides* (A → AB + DA, C → BC + CD), master buzzer, *also sound every other node's horn*, Telegram.
- Node settings show the ESP8266 pins of each channel (D7/D6, D1/D2).

Everything else (labels, training data, CSV export, Google Sheets history, phone notifications) is unchanged.

MQTT topics (root `nilgaiguard2/<FARM_KEY>`): `status`, `state` (every 3 s), `config` (retained), `node/<id>` (retained heartbeat), `event`, `label`, `ack`, and `cmd` (dashboard → master: `{"to":"A","c":"horn:2000"}`, `{"cfg":{…}}`, `{"label":{…}}`).

> The public broker is open: anyone who guesses the topic could read it or send commands. Use an unguessable `FARM_KEY` (e.g. `iitbhu-nuthan09-7f3k2`) and enter the same key in the dashboard Settings.

---

## 7. Saving the data — Google Sheets

Your Sheet *NilGai_Guard_v2_data* and the Apps Script are already deployed. Remaining steps:
1. Phone → Wi-Fi **NilGaiGuard-Master** → http://192.168.4.1 → **Cloud** → paste the Web app URL (`…/exec`) and the `SECRET` from the script → Save.
2. Dashboard → Settings → same URL + secret → *Reload from Sheets*.

| Tab | One row per | Columns |
|---|---|---|
| `events` | crossing (also wrong / uncertain ones) | time, node, side, prediction, direction, P(NilGai), status, notes, all features, evidence per feature, **cleaned and raw beam intervals**, learned thresholds, **label, who labelled it, correct yes/NO**, full JSON |
| `labels` | correction | time, crossing id, node, label, by |
| `status` | 10 minutes | health of every node, crossings waiting |

The master uploads 10 crossings at a time, only while the radio is quiet (an upload takes a few seconds on the ESP8266), and remembers how far it got. Its flash holds ~1,000 crossings; the full log can be downloaded from the offline page.

ThingSpeak (MATLAB): put the *Write API key* on the offline page → Cloud. Every minute: NilGai today, people today, alerts, nodes in range, last P(NilGai), Wi-Fi RSSI, turned back, free memory. OneDrive / Excel: download the Sheet as .xlsx or use *Export CSV* on the dashboard.

## 8. Farmer alerts — Telegram (optional)

1. Telegram → **@BotFather** → `/newbot` → copy the token.
2. Message your bot, open `https://api.telegram.org/bot<TOKEN>/getUpdates`, copy `"chat":{"id": …}`.
3. Offline page → Cloud → token + chat id → **Save + Telegram test**.

Alerts: NilGai entering, turned back, direction unclear (max one per side per minute). Late crossings are sent as *delayed report* with the time they happened.

## 9. Offline page on the master

Phone → Wi-Fi **NilGaiGuard-Master** (password `nilgai123`) → **http://192.168.4.1**.
Shows the field, which sensor node is in range, the crossings waiting at each node, live beams, horns, mute / snooze / arm, node settings, labelling, cloud settings and log download. When the master is on your hotspot, the same page opens at the master's IP shown on the dashboard (Master node → Hotspot IP).

---

## 10. First test — checklist

On a table first (all three boards within a few metres):
1. Flash A, C and the master. Open the master's Serial Monitor: `[event]` lines appear when you break a beam at A or C.
2. Offline page or dashboard: **2 / 2 sensor nodes in range**; B and D shown as transmitter corners.
3. Unplug the master, break some beams at A, plug the master in again: the crossings arrive (*delivered late*) — the store-and-forward works.

In the field:
4. Power A and C with **beams clear**. On each node card all four lamps must be **green**. If one stays red with nothing in the beam: re-aim the receiver (its red ALARM LED must be off), then press *Re-detect beams*.
5. Set each channel's **profile→gait gap** to the measured distance (node Settings).
6. Walk across every side: *Person … entering / exiting*. If swapped on a side, tick *swap entering / exiting* for that channel.
7. Walk the master around the field and note where A and C go *out of range* — that is your radio range.
8. Label every test crossing. These rows are your first training data.

## 11. Training a better model later

1. Label crossings during the trials — especially wrong ones.
2. Dashboard → Training data → **Export CSV** (or the `events` sheet as CSV).
3. `python3 tools/train_from_labels.py nilgai_v2_crossings.csv` — accuracy of the current model, a model trained on your labels, which features matter, and the class centres measured in your field.
4. Put the centres into `learnDefaults()` in `NGDetect.h` and re-flash — or keep labelling: each label also moves the node's thresholds 25 % towards that crossing.

## 12. Adding sensor nodes later

- To put a controller at B: flash a NodeMCU with `NODE_ID "B"`. Then Dashboard → Configure field → remove B from *transmitter-only corners* → Save → *Auto-assign sides*. (A node that turns up is also adopted automatically.)
- More corners: add the letter to the order (e.g. `A,B,E,C,D`). Every side needs a sensor node at one of its two ends.
- Up to 8 nodes on the ESP8266 master.

## 13. What was tested here and what was not

| Check | Result |
|---|---|
| ESP8266 slave and master compile | no errors, esp8266 core 3.1.2, NodeMCU 1.0 (master with and without OLED) |
| Memory | slave 36 kB / master 36 kB of 80 kB static RAM; HTTPS uploads check for ~19 kB free before starting |
| Classifier (same code as before) | 3,600 simulated crossings: 0 people called NilGai, 0 NilGai missed |
| Message chunking (2.1 kB crossing in 11 packets, lost packet, repeated message, channel probe) | passed |
| Dashboard (demo with A, C + transmitter corners B, D; auto-assign; node out of range; late crossing) and offline page (phone width) | no script errors |

**Not tested** (no hardware here): real ESP-NOW range and timing between NodeMCU boards, IR receivers, and uploads to Google / Telegram / ThingSpeak from the ESP8266. Do the table test in §10 before the field.
