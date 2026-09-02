# Chair Node — Build Guide

The actual build steps for one battery-powered, chair-mounted temperature
sensor node. This is the confirmed design as of the Blueprint.io wiring
review (2026-09) — for the reasoning behind these choices (why battery, why
this sensor, why no DS18B20, why a switch instead of sleep firmware), see
`sensor-node-spec.md` and `project-context.md`. This doc is the "build it"
reference; those are the "why" reference.

---

## 1. Bill of materials

| Part | Spec | Notes |
|---|---|---|
| MCU | ESP32-C3 SuperMini (generic clone) | Confirm the listing/board silkscreen shows raw `GPIOx` labels, not Seeed-style `D0`–`D10` |
| Sensor | SHT31-D breakout, I²C | Any breakout with onboard I²C pull-ups |
| Battery | Single-cell LiPo, 3.7V, ~2000 mAh | Use a cell with a built-in protection circuit — standard on hobby-electronics LiPo pouch cells, don't buy one without |
| Charge/boost module | "TP4056 + 5V boost converter" combo | **Not** a bare TP4056 — must output a regulated 5V regardless of battery charge level (raw LiPo voltage swings 4.2V→3.0V across discharge) |
| Switch | Mini SPST slide switch | Placement matters — see §2 |
| Ground distribution bus | Small terminal strip (optional) | Tidies up multiple GND connections; not required, a direct-soldered common ground works too |
| Enclosure | 3D-printed, PLA, snap-fit chair clip | See owner's own chair-specific design; print-orientation notes in prior chat (fillet the clip's flex-arm base, keep the arm short and thick rather than long and thin, print with layer lines across the flex direction) |

Full sourcing links and price ranges: `sensor-node-spec.md` §6 (India sourcing).

---

## 2. Wiring

### 2.1 Power path — confirmed layout

**LiPo → charge/boost module (always connected) → switch → MCU 5V.** The
switch sits *after* the charge/boost module, not between the battery and the
module. This is the one detail that matters most in this whole build: get it
backwards and switching off also cuts off charging.

```
              USB-C (charging input)
                      │
                      ▼
              ┌───────────────────┐
   LiPo ──B+─►│                   │
   Battery    │  Charge + Boost   │
        ──B-─►│      Module       │
              │                   │
              │   OUT+ (5V) ──────┼──┐
              │   OUT- (GND) ─────┼─┐│
              └───────────────────┘ ││
                                     ││
                              ┌──────▼┴──┐
                              │  SPST     │   cuts power to the MCU only —
                              │  switch   │   charging circuit (above) is
                              └──────┬────┘   never affected by this switch
                                     │
                                     ▼
                          ESP32-C3 SuperMini
                             5V pin (+ GND)
```

The charge/boost module stays wired to the battery at all times, so plugging
in USB-C charges the battery whether the switch is on or off. The switch's
only job is gating power to the MCU.

| From | To | Note |
|---|---|---|
| LiPo POS | Charge/boost module `B+` | — |
| LiPo NEG | Charge/boost module `B-` | — |
| Charge/boost module `OUT+` | Switch, terminal 1 | Regulated 5V |
| Switch, terminal 2 | ESP32-C3 `5V` pin | — |
| Charge/boost module `OUT-` / GND | ESP32-C3 `GND` | Common ground — route through the ground bus if using one |

### 2.2 Sensor (I²C) wiring — unchanged from sensor-node-spec.md §5

| SHT31-D | ESP32-C3 pin | Note |
|---|---|---|
| VCC | 3V3 | Powered from the board's own regulated 3.3V output, not the raw battery rail |
| GND | GND | — |
| SDA | GPIO6 | Avoids the ESP32-C3's boot-mode strapping pins |
| SCL | GPIO7 | Avoids the ESP32-C3's boot-mode strapping pins |

No pull-up resistors to add — the SHT31-D breakout carries its own.

---

## 3. Firmware

Same ESPHome config as the USB-powered node #1, no changes needed for the
battery/switch circuit — the switch is a physical, not a software, control.
`power_save_mode: none` stays as-is; there is no sleep logic in this design.

```yaml
esphome:
  name: hall-sensor-01     # rename per node once you scale past one

esp32:
  board: esp32-c3-devkitm-1   # generic ESP32-C3 target, correct for the SuperMini
  framework: { type: esp-idf }

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none

mqtt:
  broker: 10.20.0.2            # your laptop's IP while it's running Mosquitto
  topic_prefix: hall/sensor/01
  birth_message: { topic: hall/sensor/01/status, payload: online }
  will_message:  { topic: hall/sensor/01/status, payload: offline }

i2c:
  sda: GPIO6
  scl: GPIO7
  frequency: 100kHz

sensor:
  - platform: sht4x
    address: 0x44
    update_interval: 10s
    temperature:
      name: "Air temperature"
      accuracy_decimals: 2
      filters:
        - offset: 0.00          # per-unit calibration lands here later
        - median: { window_size: 5, send_every: 3 }
    humidity:
      name: "Relative humidity"

  - platform: wifi_signal
    name: "WiFi RSSI"
    update_interval: 60s
  - platform: uptime
    name: "Uptime"

ota:
  - platform: esphome
logger:
```

---

## 4. Assembly order

1. **Wire and test on the bench before enclosing anything.** Breadboard or
   hand-solder the circuit per §2, flash the firmware per §3, confirm
   readings arrive over MQTT — all with the board powered over USB-C
   directly (bypass the battery circuit for this first check, it's one
   less variable while confirming the sensor and firmware work).
2. **Wire in the battery circuit** (LiPo → charge/boost module → switch →
   5V) once the sensor side is confirmed working.
3. **Verify the switch behavior specifically**, before closing the
   enclosure:
   - Switch on, USB-C unplugged → MCU runs on battery, readings publish.
   - Switch off → MCU stops (no more MQTT messages), regardless of USB-C.
   - Switch off, USB-C plugged in → battery charges (check the charge
     module's LED indicator) even though the MCU is off.
   - Switch on, USB-C plugged in → both charging and running at once —
     confirm the MCU doesn't reset or brown out (some boost modules dip
     briefly when a load and a charge current are both active; if it resets,
     it's the module, not your wiring).
4. **Fit the enclosure.** Mount the board, sensor, battery, module, and
   switch inside; route the sensor on its short cable per
   `sensor-node-spec.md` §4 (outside the sealed part of the enclosure, in
   free air, below the electronics) so the ESP32-C3's own heat doesn't bias
   the reading.
5. **Attach to a chair**, switch off between services.

---

## 5. Safety notes

- **Use a LiPo cell with a built-in protection circuit** (over-charge,
  over-discharge, over-current cutoff) — standard on hobby cells sold with
  bare leads or a JST connector, but confirm before buying.
- Don't compress, puncture, or tightly clamp the cell inside the enclosure —
  leave it room to sit unstressed.
- Keep it out of direct sun and away from the AC/fan airflow's warm side —
  this is a hall with known heat-distribution problems, so don't assume
  "indoors" means "cool."
- These enclosures sit on furniture in a public space, unattended, for
  extended periods — if a cell ever swells, smells odd, or the enclosure
  feels warm to the touch when it shouldn't, stop using that unit and
  replace the cell rather than investigating in place.
