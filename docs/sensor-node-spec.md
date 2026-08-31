# Hall Sensor Node — Build Spec

Part 2 of 3. Board selection, bill of materials, firmware, and a test plan for
the wireless temperature nodes. Sized to prototype **three** nodes, not twelve.

See also: [Design Considerations](design-considerations.md) (part 1).

---

## 1. On AI hardware-design tools

Schematik and Blueprint are AI hardware-design tools that generate wiring
diagrams, BOMs, assembly guides, and firmware from a natural-language prompt,
with Fritzing and CAD export. They are reasonable ways to get a pictorial
diagram to hand to a helper — treat the output as a draft to check, not a spec.

Bear in mind this node is **four wires**. The hard parts of this project are
sensor siting, cross-calibration, RF coverage, and the mains interface, none of
which a schematic generator has visibility into. Don't let an easy-looking
schematic step absorb effort that belongs in the test plan (§8).

---

## 2. Board shortlist

One question decides this, and it can only be answered inside the building:
does the radio hold a solid link from all twelve positions, through whatever
the hall is built from?

| Board | ≈ Price | Antenna | I²C connector | Why it's on the list |
|---|---|---|---|---|
| **Seeed XIAO ESP32C3** ← lead | $5 | **u.FL + external antenna included** | solder 4 wires | Best RF per dollar. The external antenna is the whole reason it leads — the one variable you cannot fix in software later. |
| **Adafruit QT Py ESP32-S3** | $13 | PCB trace | **STEMMA QT — zero solder** | Fastest to assemble twelve times. Plug board → cable → sensor. Weaker antenna is the trade. |
| **Seeed XIAO ESP32C6** | $8 | u.FL + external | solder 4 wires | Hedge. Adds an 802.15.4 radio, so if WiFi disappoints there's a Thread/Zigbee path without rebuying boards. |
| ESP32-C3 "SuperMini" clones | $3 | ceramic chip, no u.FL | solder | **Listed to warn you off.** Widely reported poor antenna performance — the ground plane crowds the ceramic. Wrong part for a large hall. |

**Recommendation:** buy one of each of the top three and run the RF survey (§8,
test 2) before committing to twelve of anything. About $26 of boards to de-risk
a $300 order and a rollout. If the survey is comfortable everywhere, take the
QT Py for the build and never solder. If it's marginal anywhere, take the
XIAO C3 for its external antenna.

---

## 3. Sensor choice

Two families, for different jobs. Don't pick one for everything.

| Part | Accuracy | Bus | Use it for |
|---|---|---|---|
| **SHT45** breakout (Qwiic/STEMMA QT) | ±0.1 °C | I²C | The occupied-zone grid. Best accuracy at this price; humidity comes free. |
| SHT41 / SHT31-D breakout | ±0.2 °C | I²C | Same job, cheaper. Fine — well inside what cross-calibration cleans up. |
| **DS18B20** stainless probe, 1–3 m lead | ±0.5 °C | 1-Wire | AC supply-air probes, and the ceiling leg of a column node. The cable length is the point. |
| BME280 | ±0.5 °C | I²C | Skip. Worse temperature accuracy than SHT4x; its pressure channel is useless here. |

### Why DS18B20 for the long runs

I²C is designed for a few tens of centimetres on a PCB. Pushing it 2–3 m to the
ceiling works *sometimes* — at 100 kHz with tuned pull-ups — and fails
intermittently in ways that cost you a weekend. **1-Wire is built for long cable
runs**: tens of metres, several devices per bus.

So a *column node* — the thing that measures your stratification ΔT — is one
ESP32 with an SHT45 on a short cable at head height and a DS18B20 on a 2–3 m
cable run up to the ceiling. Both sensors share one node, one clock, and one
calibration session, which is exactly what you want when the quantity of
interest is the difference between them.

---

## 4. Node layout — get the sensor off the board

The single physical decision that determines whether your data is trustworthy.
It costs one cable.

```
  A — SENSOR ON THE BOARD              B — SENSOR ON A CABLE
  ┌───────────────────────┐            ┌───────────────────────┐
  │  ┌────────┐ +1-3°C    │            │      ┌─────────┐      │
  │  │ ESP32  │──────▶┌───┴──┐         │      │  ESP32  │      │
  │  └────────┘       │SHT4x │         │      └─────────┘      │
  │   sealed enclosure └──────┘        └───────────┬───────────┘
  └───────────────────────┘                        │ I²C, 150 mm
                                                   ▼
          reads 26.8 °C                     ┌═════════════┐
                                            │    SHT4x    │  vented shield
   error shrinks when the fan runs          └═════════════┘
   — so it tracks your actuator                reads 25.1 °C
                                            airflow does not move this
```

True air temperature is 25.0 °C in both cases. A sealed enclosure traps the
ESP32's radio self-heat and biases a co-located sensor by 1–3 °C — and because
that bias falls when air moves over the box, **the error is correlated with the
fan you are trying to control.** A 150 mm cable and a vented shield remove both
problems for the price of the cable.

The shield can be a louvred junction box, a stack of plastic plant-pot saucers,
or a printed miniature Stevenson screen — anything that blocks radiant heat and
line-of-sight sun while letting air through. Sensor *below* the electronics,
always, since the enclosure's own plume rises.

---

## 5. Wiring

Four wires for the I²C sensor, three for the DS18B20. On a QT Py with STEMMA QT
the I²C side is a plug and the first table doesn't apply.

| SHT45 breakout | XIAO ESP32C3 pin | Qwiic cable colour |
|---|---|---|
| VIN | 3V3 | Red |
| GND | GND | Black |
| SDA | D4 / GPIO6 | Blue |
| SCL | D5 / GPIO7 | Yellow |

| DS18B20 probe | XIAO ESP32C3 pin | Note |
|---|---|---|
| Red | 3V3 | — |
| Black | GND | — |
| Yellow (data) | D2 / GPIO4 | **4.7 kΩ pull-up from data to 3V3.** Required — it will not work without it. |

Breakouts with Qwiic/STEMMA QT connectors carry their own I²C pull-ups, so don't
add more. Daisy-chaining several devices puts pull-ups in parallel and can leave
the bus too stiff to pull low — one more reason one sensor per bus is the easy
life.

---

## 6. Prototype bill of materials

Three nodes, three board variants, enough to run every test in §8. Prices are
indicative (mid-2026, before shipping and duty) — order of magnitude, not quotes.

| Qty | Item | Purpose | ≈ Cost |
|---|---|---|---|
| 1 | Seeed XIAO ESP32C3 (with external antenna) | RF benchmark — the one to beat | $5 |
| 1 | Adafruit QT Py ESP32-S3 | Assembly-ergonomics candidate | $13 |
| 1 | Seeed XIAO ESP32C6 | 802.15.4 hedge | $8 |
| 3 | SHT45 breakout, Qwiic/STEMMA QT | One per node | $24 |
| 4 | Qwiic/STEMMA QT cable, 100–200 mm | Gets the sensor off the board | $7 |
| 2 | Qwiic-to-male-header pigtail | The XIAOs have no QT socket | $4 |
| 3 | DS18B20 stainless probe, 1–3 m lead | Supply-air + ceiling leg | $8 |
| 1 | 4.7 kΩ resistors (pack) | 1-Wire pull-up | $2 |
| 1 | Reference thermometer, ±0.1 °C | **Calibration ground truth** | $25 |
| 3 | USB-C supply + cable | Mains power | $18 |
| 3 | Vented enclosure / louvred box | Radiation shield | $12 |
| 1 | Breadboard + jumper set | Bench work | $8 |
| | **Prototype stage total** | | **≈ $134** |

You don't need the Pi to start — ESPHome will log to a laptop running Mosquitto.
When you do buy it: Raspberry Pi 5 (4 GB), active cooler, official 27 W supply,
and **an SSD rather than an SD card**. Roughly $150–200 all in.

---

## 7. Firmware

ESPHome, not Arduino. You get OTA updates to all twelve nodes, MQTT with
last-will built in, and filters and calibration offsets as config rather than
code.

```yaml
esphome:
  name: hall-sensor-01

esp32:
  board: seeed_xiao_esp32c3
  framework: { type: esp-idf }

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none        # mains-powered; keeps RSSI stable

mqtt:
  broker: 10.20.0.2
  topic_prefix: hall/sensor/01
  birth_message: { topic: hall/sensor/01/status, payload: online }
  will_message:  { topic: hall/sensor/01/status, payload: offline }

i2c:
  sda: GPIO6
  scl: GPIO7
  frequency: 100kHz            # slow = tolerant of cable length

one_wire:
  - platform: gpio
    pin: GPIO4

sensor:
  - platform: sht4x
    address: 0x44
    update_interval: 10s
    temperature:
      name: "Air temperature"
      accuracy_decimals: 2
      filters:
        - offset: 0.00          # <-- per-unit calibration lands here (test 4)
        - median: { window_size: 5, send_every: 3 }
    humidity:
      name: "Relative humidity"

  - platform: dallas_temp        # ceiling leg, column nodes only
    address: 0x000000000000
    name: "Ceiling temperature"
    update_interval: 10s
    filters:
      - offset: 0.00

  - platform: wifi_signal        # you will need this in test 2
    name: "WiFi RSSI"
    update_interval: 60s
  - platform: uptime             # and this in test 5
    name: "Uptime"

ota:
  - platform: esphome
logger:
```

**The two lines that matter most.** `will_message` is what lets the controller
know a node has died rather than acting on a frozen reading forever — the first
row of the failure-mode table in part 1. `offset` is where each node's measured
calibration correction goes, and it's why all twelve configs are identical
except for a name and two numbers.

---

## 8. Test plan

In order. Tests 2 and 3 change what you buy, so don't order twelve of anything
until both have passed.

### Test 1 — Bench bring-up (1 hour)
Flash one node, confirm readings arriving on the MQTT topic. Hold the reference
thermometer next to the sensor.
**Pass:** readings within 1 °C of reference, publishing every 10 s.

### Test 2 — RF survey (half a day, in the hall)
Put the AP where it will actually live. Carry one node of each type to all
twelve intended positions, sitting at each for five minutes, logging RSSI and
counting publishes. Do it with the hall **full** if you can — bodies attenuate
2.4 GHz noticeably.
**Pass:** RSSI ≥ −70 dBm and zero dropped publishes at every position.
**Fail:** move the AP, or commit to the external-antenna board. Don't paper over
it in software.

### Test 3 — Self-heating characterisation (overnight)
Two nodes side by side: one sensor on the board, one on a 150 mm cable in a
shield. Log both against the reference. Then put a desk fan on them and watch
what happens to the gap.
**Pass:** cabled node tracks reference within 0.3 °C with the fan both off and
on. The on-board node's error visibly moving when the fan starts is the result
you're looking for — §4 confirmed in your own data.

### Test 4 — Cross-calibration (one afternoon; repeat at full build)
Every node on one shelf, same airflow, no sun. Log four-plus hours across a real
temperature swing — an unheated garage overnight into morning works well.
Compute each node's mean deviation from the group and enter it in that node's
`offset:` filter.
**Pass:** after correction, spread across all nodes under 0.2 °C.

### Test 5 — Soak test (72 hours, unattended)
Leave everything running and walk away. Read the uptime sensor and count gaps.
**Pass:** no reboots, under 0.1 % missed publishes. A node that silently reboots
twice a day will ruin you six months from now.

---

## 9. The relay side

> **Bench work only, until an electrician is involved.** Everything above runs
> at 3.3 and 5 V and is yours to play with freely. The fan side is mains in an
> assembly occupancy and stays hands-off until the survey and the electrician
> conversation from part 1 have happened. Nothing in this section needs mains to
> develop against.

Design around a **listed smart relay module** — Shelly Gen3/Gen4, SONOFF, or
equivalent carrying the right certification for your region. Specify it on three
things:

- **Native MQTT**, so it joins the same bus as the sensors and the control
  service talks to everything one way.
- **Detached-switch mode**, so the physical wall switch keeps working locally
  and can be read back as an input. That's both the manual override people need
  and the "controller and reality disagree" failure mode closed off.
- **Inductive load rating** covering a ceiling fan motor's inrush — not just a
  resistive amp figure.

To develop control logic before any mains work: some Shelly models accept a
12–60 V DC supply (check the datasheet for the exact SKU — it varies by model),
which lets you bench the real device safely. Otherwise prove the control path
against a 5 V relay module, or even an LED on the Pi's GPIO, and swap the listed
switch in at installation. The MQTT topics don't change, so the logic you write
against the bench rig is the logic that ships.

**Order of operations:** sensors first, and alone, for several weeks. You cannot
write sensible fan logic until you've seen what the hall actually does — and per
part 1, there's a real chance the data says a fixed fan configuration solves most
of it and the relay work shrinks dramatically.
