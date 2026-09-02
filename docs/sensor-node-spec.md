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

**"ESP32-C3" is a chip, not a brand.** Any board with those words on it or in
its listing — Seeed's XIAO, a no-name "SuperMini," anything else — has the
identical Espressif chip inside. Boards differ in antenna, size, and build
quality, not the fundamentals. Shop for the chip name, not a specific SKU.

Why this chip and not another ESP32 variant: the original ESP32 and the S3 are
dual-core with more RAM, aimed at heavier workloads (camera/AI on the S3) —
overkill and pricier for reading one sensor and publishing over WiFi. The C6
adds a Zigbee/Thread radio on top of C3 — a hedge worth paying for only if
WiFi actually fails the survey below, not by default. C3 is the cheapest
Espressif chip with solid modern WiFi, which is all this job needs.

**If your shop has ESP8266 boards instead** (Wemos D1 Mini, NodeMCU) — a
different, older Espressif family, WiFi-only, no Bluetooth — that's a
legitimate substitute if it's cheaper or easier to find locally. It works
fine for a mains-powered node reading one sensor; the only place it's weaker
is deep-sleep battery life, which doesn't apply here. Say so and the firmware
`board:` line changes by one word.

One question decides between the C3 candidates below, and it can only be
answered inside the building: does the radio hold a solid link from all
twelve positions, through whatever the hall is built from?

| Board | ≈ Price (India) | Antenna | I²C connector | Why it's on the list |
|---|---|---|---|---|
| **Seeed XIAO ESP32C3** ← known-good baseline | ₹450 | **u.FL + external antenna included** | solder 4 wires | Best RF you can be confident in. The external antenna is the one variable you cannot fix in software later — this is what test 2 (§8) checks the cheap option *against*. |
| **ESP32-C3 "SuperMini" clone** ← cost target | ₹150–200 | ceramic chip, no u.FL | solder | **Test this before buying 12.** Roughly a third of the price, but widely reported weaker antenna performance — the ground plane crowds the ceramic. May be fine in this hall, may not. Don't commit to 12 without testing one, per project-context.md's cost-sensitive guidance. |
| Adafruit QT Py ESP32-S3 | ₹1,100 | PCB trace | STEMMA QT — zero solder | Dropped — costs more than the known-good option for a weaker antenna. Only reconsider if soldering 12 boards turns out to be the real bottleneck, not cost. |
| Seeed XIAO ESP32C6 | ₹700 | u.FL + external | solder 4 wires | Dropped for now — 802.15.4 hedge isn't worth the premium unless WiFi actually fails the RF survey. |

**Recommendation:** buy **one XIAO ESP32C3 and one SuperMini clone** and run
the RF survey (§8, test 2) side by side, in the same 12 positions. If the
clone holds up, build all 12 nodes on clones — roughly ₹3,000 saved across the
full build versus the known-good board. If it drops out anywhere the XIAO
doesn't, that's the answer too, and the ₹450 for one XIAO wasn't wasted — it's
what tells you the clone isn't good enough before you've bought 12 of them.

---

## 3. Sensor choice

**Same principle as the board: buy the chip, not a specific listing.** SHT31
and SHT40 are both Sensirion (Swiss sensor maker) chips, same ±0.2 °C accuracy
tier — SHT3x (30/31/35) is the older generation, SHT4x (40/41/45) the newer
one, generally cheaper today since it's the current design. Functionally
interchangeable here. Any breakout board carrying either — Adafruit,
Sparkfun, or a generic "GY-SHT40"/"GY-SHT31" board (`GY-` is just a generic
prefix Chinese breakout makers use, not a brand — same chip, no premium
extras) — is fine. Ask your shop for "SHT31 or SHT40, whichever you have."

Two families beyond that, for different jobs. Don't pick one for everything.

| Part | Accuracy | Bus | ≈ Price (India) | Use it for |
|---|---|---|---|---|
| **SHT31 or SHT40 breakout** ← lead, either | ±0.2 °C | I²C | ₹200–350 | The occupied-zone grid. Whichever your shop stocks — see above, they're interchangeable here. |
| Adafruit SHT45 breakout | ±0.1 °C | I²C | ₹700–900 | Skip for the cost-sensitive build — the extra precision doesn't matter once cross-calibration (§8, test 4) is done, and it's 3× the price. |
| **DS18B20** stainless probe, 1–3 m lead | ±0.5 °C | 1-Wire | ₹80–150 | AC supply-air probes, and the ceiling leg of a column node. The cable length is the point, not the accuracy. |
| BME280 | ±0.5 °C | I²C | ₹150–200 | Skip — worse accuracy than SHT31/40, and its pressure channel is useless here. See the explanation of why this matters for a differential measurement, above in chat / project history. |
| DHT11 / DHT22 | ±0.5–2 °C | Single-wire (not I²C) | ₹80–150 | **Avoid, despite being everywhere and cheap.** Worse accuracy than even BME280, slow to respond, and not I²C — wrong tool for resolving 2–3 °C gradients between zones. |

### Why DS18B20 for the long runs

I²C is designed for a few tens of centimetres on a PCB. Pushing it 2–3 m to the
ceiling works *sometimes* — at 100 kHz with tuned pull-ups — and fails
intermittently in ways that cost you a weekend. **1-Wire is built for long cable
runs**: tens of metres, several devices per bus.

So a *column node* — the thing that measures your stratification ΔT — is one
ESP32 with a SHT31/SHT40 on a short cable at head height and a DS18B20 on a 2–3 m
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

Four wires for the I²C sensor, three for the DS18B20. Both the XIAO ESP32C3 and
the SuperMini clone need everything soldered — neither has a plug-in connector.

| SHT31/SHT40 breakout | ESP32-C3 pin | Note |
|---|---|---|
| VCC (may be labeled VIN) | 3V3 | — |
| GND | GND | — |
| SDA | GPIO6 (XIAO: pin D4) | — |
| SCL | GPIO7 (XIAO: pin D5) | SuperMini clone pin labels vary by seller — check the silkscreen. |

| DS18B20 probe | ESP32-C3 pin | Note |
|---|---|---|
| Red | 3V3 | — |
| Black | GND | — |
| Yellow (data) | GPIO4 (XIAO: pin D2) | **4.7 kΩ pull-up from data to 3V3.** Required — it will not work without it. |

Most SHT31/SHT40 breakouts already carry their own I²C pull-up resistors, so don't
add more. Daisy-chaining several I²C devices puts pull-ups in parallel and can
leave the bus too stiff to pull low — one more reason one sensor per bus is
the easy life.

### Node #1, confirmed: ESP32-C3 SuperMini + SHT31

**The DS18B20 is not part of node #1.** It's only needed for the later
*column nodes* — the ones pairing an occupied-zone sensor with a ceiling-level
one (§4) or an AC supply-air probe (hall-survey.md §5) — because I²C isn't
reliable over the 2–3 m cable those need, and 1-Wire is. The very first
build-and-test-it-works node is ESP32-C3 + SHT31 only. Skip DS18B20 until the
12-node rollout reaches those specific positions.

The pin choices above (GPIO6/GPIO7 for I²C, GPIO4 for 1-Wire) aren't arbitrary
— they avoid the ESP32-C3's strapping pins (GPIO2, 8, 9, which affect boot
mode and are best left alone) and match Espressif's own reference pinout, so
they're safe on any ESP32-C3 board, SuperMini included. On the SuperMini these
are typically silkscreened directly as `GPIO6`, `GPIO7`, `GPIO4` — no
Seeed-style `D0`–`D10` relabeling to translate, just wire straight to the
printed pin. Confirm against your specific board before soldering — clone
quality and labeling vary by seller/batch.

**Power:** the SuperMini takes 5V over its USB-C port, same as charging a
phone. During bring-up (§8 test 1), the USB cable to your laptop powers *and*
flashes it — no separate supply needed yet. Once it's a permanent install,
that becomes a USB wall adapter instead of a laptop; nothing else changes.

---

## 6. Bill of materials

Superseded by the cost-sensitive India sourcing list right below — buy for
that, not this. Two things still apply regardless of the sourcing pass:

- You don't need the Pi to start — ESPHome will log to a laptop running
  Mosquitto.
- When you do buy the Pi: Raspberry Pi 5 (4 GB), active cooler, official
  supply, and **an SSD rather than an SD card**. Check Robu.in / Robocraze /
  local distributors for current INR pricing — figure roughly ₹8,000–10,000
  all in, but verify before buying.

### India sourcing — node #1 and its RF-test partner

Per project-context.md: solder-capable, cost-sensitive, source locally.
Building **two** boards right now, not one — the known-good XIAO and the cheap
clone — because the RF survey (§8, test 2) needs both to answer the only
question that decides whether you build the remaining 10 nodes at ₹450 or
₹200 each.

| Item | Where | Notes |
|---|---|---|
| Seeed XIAO ESP32C3 ×1 | [Robu.in](https://robu.in/product/seeed-studio-xiao-esp32c3-tiny-mcu-board-with-wi-fi-and-ble-battery-charge-supported-power-efficiency-and-rich-interface/), also on [Amazon.in](https://www.amazon.in/Seeed-Studio-XIAO-ESP32C3-Microcontroller/dp/B0B94JZ2YF) | ~₹400–500. Confirm the listing shows the u.FL + external antenna variant, not a ceramic-only one. |
| ESP32-C3 SuperMini clone ×1 | [Robu.in](https://robu.in/product/esp32-c3-supermini-expansion-board/) or Amazon.in (several sellers — "ESP32-C3 Super Mini") | ~₹150–200, varies by seller. This is the one under test — don't buy 10 more until §8 test 2 passes. |
| SHT31 or SHT40 breakout ×2 | Robu.in / Robocraze, or a local electronics shop — ask for either chip name | ~₹200–350 each. One per board — either is fine, see §3. |
| DS18B20 waterproof probe, 1 m ×2–3 | [Robokits](https://robokits.co.in/sensors/temperature-humidity/ds18b20-temperature-sensor-probe-waterproof-1-meter-length) (~₹83) or [Robocraze](https://robocraze.com/products/ds18b20-waterproof-temperature-sensor-probe-1m-range-7semi) | Cheap enough to grab spares now. |
| 4.7 kΩ resistor | Any local electronics shop / Robu.in | For the DS18B20 pull-up (§5), one per board. |
| USB-C cable + 5V supply ×2 | Local | Any phone charger works for bench testing. |
| Reference thermometer, ±0.1 °C or better | Local instrument shop / Amazon.in | For §8 test 4 later — not needed for the first bring-up test. |

Total for this round: **~₹1,650–1,900** (both boards, both sensors, probes,
misc) — versus ~₹8,000+ if you'd gone straight to 12 XIAO boards and only
found out the clone was an option afterward.

Skip for now (needed later, only if the RF survey says you need the hedge):
the QT Py, the XIAO ESP32C6, enclosures, and the vented shield — get both
boards talking over MQTT first.

---

## 7. Firmware

ESPHome, not Arduino. You get OTA updates to all twelve nodes, MQTT with
last-will built in, and filters and calibration offsets as config rather than
code.

For the SuperMini clone, swap `board: seeed_xiao_esp32c3` for
`board: esp32-c3-devkitm-1` (the generic ESP32-C3 target) — everything else in
this config is identical, since both boards run the same chip.

```yaml
esphome:
  name: hall-sensor-01

esp32:
  board: seeed_xiao_esp32c3    # SuperMini clone: esp32-c3-devkitm-1
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
Put the AP where it will actually live. Carry both boards — the XIAO and the
SuperMini clone — to all twelve intended positions, sitting at each for five
minutes, logging RSSI and counting publishes for both. Do it with the hall
**full** if you can — bodies attenuate 2.4 GHz noticeably.
**Pass (clone):** RSSI ≥ −70 dBm and zero dropped publishes at every position,
matching the XIAO closely enough that the price difference isn't buying you
anything → build the remaining 10 nodes on clones.
**Fail (clone), pass (XIAO):** the antenna difference is real in this
building → build on the XIAO instead, the ₹250/node premium bought something.
**Fail (both):** move the AP. Don't paper over it in software either way.

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
