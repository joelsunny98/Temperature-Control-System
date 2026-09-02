# Church Hall Thermal Control — Design Considerations

Scope: ~2,400–2,500 sq ft rectangular hall, 4 air conditioners (two different
types, two per side), 12 ceiling fans. Goal: detect hot/cold pockets
automatically and drive the fans to even out comfort.

This document is the "what do I need to think about before I buy anything"
pass. It is deliberately opinionated, and it flags the two or three places
where the obvious design fails.

---

## 1. Reframe: what you are actually controlling

The instinctive design is: measure temperature → if a zone is hot, turn its fan
on → if it is cold, turn it off. This does not work as stated, for three
reasons. Understanding them shapes everything else.

**(a) A ceiling fan does not cool air. It cools people.** The fan moves no heat
out of the room; it adds a little (motor losses). What it does is raise air
speed over skin, which increases convective and evaporative heat loss. At
roughly 0.8 m/s, occupants feel about 2–3 °C cooler than the air actually is.
So the quantity you care about is not air temperature — it is something closer
to *operative temperature adjusted for air speed*, which is what ASHRAE 55
calls the elevated-air-speed comfort zone.

**(b) Your actuator moves your sensor the wrong way.** In a hall with high
ceilings the air is stratified — warm air sits at the top. When a fan switches
on it destratifies its column, dragging warm ceiling air down past the sensor.
A sensor mounted under a fan will frequently read *warmer* after the fan starts,
even though the people underneath feel cooler. A naive controller reacts by
running the fan harder, or by concluding the fan is not working and oscillating.
This is the single biggest trap in the project.

**(c) The lag is minutes, not seconds.** The hall has enormous thermal mass
(slab, walls, furniture, bodies). Any control loop must be slow, hysteretic, and
dwell-limited, or it will hunt.

The practical consequence: **the control variable should be differential, not
absolute.** Two signals matter far more than "is zone 7 at 25.1 °C":

- **Horizontal spread** — this zone versus the coolest zones in the hall. That
  is literally the complaint people are making ("it's hot over here").
- **Vertical stratification** — ceiling temperature minus occupied-zone
  temperature in the same column. A large ΔT means there is cool air being
  wasted at head height and warm air pooling above, which is exactly the
  condition a fan fixes. ASHRAE 55 flags head-to-ankle differences above ~3 °C
  as a discomfort source in its own right.

If you only take one thing from this document: instrument for those two
differentials, and control on them.

---

## 2. Phase 0 — the survey you must do before buying hardware

Every decision below depends on facts you do not have yet. Budget a couple of
weekends for this.

**Map the room.** Dimensions, ceiling height (and whether it is flat, vaulted,
or has a stage), window positions and orientation, door positions, the exact
positions of the 4 AC indoor units and their throw direction, and the positions
of the 12 fans. Draw it to scale — you will use this grid for the whole project.

**Find out how the fans are actually wired.** This is the highest-risk unknown
in the entire project and it can invalidate the plan. Twelve fans in a hall of
this size are very often *not* twelve independent circuits — they are commonly
ganged into three or four banks on three or four wall switches, or on a single
fan-speed controller per bank. If that is the case you have four controllable
zones, not twelve, and no amount of software fixes it without rewiring. Open the
switch plates (power off) or get an electrician to trace it. Also note: are
there neutrals in the switch boxes? Almost every smart switch needs one.

**Characterise the two AC types.** Model numbers, capacity, where each unit's
return-air thermostat senses, whether they have a wired controller, IR remote,
or any BMS/Modbus interface. Two different types on opposite sides is a strong
suspect for the root cause: different throw lengths, different setpoint
calibration, and each unit cycling off based on *its own* local return air while
the far side of the hall bakes.

**Take manual readings during a full, occupied service.** A cheap thermocouple
or a NIST-ish handheld probe, a 20-point walk of the hall at three heights
(ankle ~0.1 m, seated head ~1.1 m, standing head ~1.7 m), repeated at the start,
middle, and end. Log it. This tells you the magnitude of the problem — whether
you are fighting 2 °C or 6 °C — and whether the pattern is stable.

**Expect the pattern to be static.** The hot and cold pockets are created by
fixed AC positions and fixed room geometry. There is a real chance the survey
shows the same map every single time, in which case a *fixed* fan configuration
(some fans always on at low speed during services) captures most of the benefit
for near-zero cost. Find that out before spending money. It is also the strongest
argument for building the sensing side first and the control side second.

**Get an HVAC tech to look at air balance.** If the two AC types are simply
mismatched or their vanes are throwing badly, a few hundred dollars of balancing
and vane adjustment may remove half the problem. That is cheaper than anything
in this document.

---

## 3. Sensing

### Sensor choice

- **SHT31 over BME280** for this application. SHT31 is ±0.2 °C typical, ±2 % RH.
  BME280 is ±0.5 °C typical (worse across the full range) and its humidity
  channel is ±3 %. You are trying to resolve gradients that may only be 2–3 °C,
  so half a degree of sensor error is a large fraction of your signal.
  SHT4x is also good and cheap. BME280's pressure channel is useless to you.
- **Absolute accuracy matters less than inter-sensor agreement.** You are
  comparing sensors to each other. A sensor that reads 0.8 °C high creates a
  permanent phantom hot pocket and a fan that runs forever.

### Calibration (do not skip this)

Before installation, put all 12 sensors in one place — same shelf, same
airflow, no direct sun — and log them together for several hours across a
temperature swing. Compute a per-sensor offset (and ideally a two-point gain
correction if you can get them to two different stable temperatures). Store
those offsets in the sensor's config or in the Pi's device table. Repeat
annually. This one afternoon of work is what makes the heat map trustworthy.

### Self-heating — a subtle confound

An ESP32 with the WiFi radio active dissipates enough power to warm its own PCB
by 1–3 °C, and it will warm a sensor mounted on the same board. Worse: that
offset *changes with air speed*, so it shrinks when the fan below it turns on —
which means your sensor error is correlated with your actuator. That is exactly
the kind of coupling that makes a control loop misbehave in ways that are very
hard to debug.

Mitigations, in order of preference:
1. Put the sensor on a short cable (10–20 cm I²C is fine) so it sits outside and
   below the electronics enclosure, in free air.
2. Duty-cycle the ESP32 — sleep between readings so the board is cool when it
   samples (sample *first*, then bring up the radio).
3. Characterise and subtract the offset. Least reliable, because of the airflow
   dependence above.

### Enclosure and mounting

- **Vented, not sealed.** A sealed plastic box is a greenhouse with a several-
  minute time constant. You want a louvred/perforated housing, ideally a small
  radiation shield, with the sensing element in moving air.
- **Avoid radiant error.** No direct sun from windows or skylights, no line of
  sight to hot light fixtures, not against an exterior wall that bakes.
- **Not directly in an AC discharge jet** — those sensors will read the supply
  air temperature, not the room, and will lie to your controller. (Unless you
  put one there deliberately; see below.)

### Layout — reconsider "12, one under each fan"

Sensors mounted under fans measure the air the fan is already conditioning. You
want to measure the air in the *occupied zone* — where people actually are.
Suggested layout:

- **8–10 sensors at occupied-zone height (~1.5–1.8 m)** on walls, columns, or
  short standoffs, arranged on a grid that aligns with fan zones. At ~200 sq ft
  per sensor that is roughly 14 ft spacing — coarse, but adequate, because you
  need zone-level decisions, not a true continuous field.
- **2–3 sensors at ceiling level** in the same columns as occupied-zone
  sensors. These give you the stratification ΔT, which is your best "should I
  destratify?" signal, and they are cheap.
- **4 supply-air probes**, one at each AC's discharge. These are pure diagnostic
  gold: they tell you instantly whether a unit is actually cooling, cycling off,
  iced up, or in defrost. A large fraction of "the hall is hot" complaints will
  turn out to be "AC #3 has been off for 20 minutes."
- **1 outdoor sensor** for context and for later feed-forward.
- Consider **1–2 CO₂ sensors** (SCD40/SCD41). CO₂ is an excellent occupancy
  proxy in an assembly space, it is the thing that actually makes a full hall
  feel stuffy, and occupancy is your single largest heat load — roughly 70 W of
  sensible heat per seated adult, so 200 people is about 14 kW, comparable to a
  significant fraction of your AC capacity.

Sample every 10–30 s. Faster buys you nothing against a thermal mass this large.

### On the "heat map"

Interpolating 12 points (inverse-distance weighting or a barycentric
interpolation over a triangulation) makes a lovely smooth picture. Do not let
that smoothness fool you into believing you have resolution you do not have.
**Interpolate for the human-facing display; make control decisions on discrete
zones.**

---

## 4. Network

### WiFi to the church network — expect friction

- Consumer/mesh APs often band-steer and can hide or deprioritise the 2.4 GHz
  band that ESP32 requires.
- Guest networks usually enable **AP/client isolation**, which silently blocks
  sensor → Pi traffic even though both are "on the WiFi."
- Captive portals are fatal to headless devices.
- The password will get rotated by someone who does not know your 12 sensors
  depend on it, and the whole system will die on a Sunday morning.

**Recommendation: run your own dedicated 2.4 GHz AP for the sensor network.** A
$30 travel router, or hostapd on the Pi with a second WiFi adapter. Static DHCP
leases. This isolates you from church IT entirely and removes a whole category
of failure. Uplink the Pi to the church network (or not at all) separately.

### Protocol

**MQTT** (Mosquitto on the Pi) is the right default:
- One topic per device, e.g. `hall/sensor/<id>/state`, JSON payload with
  temperature, humidity, RSSI, uptime, battery (if any), and a sequence counter.
- **Last Will and Testament** on each device so the broker publishes
  `offline` when a sensor drops. You need this — see the failure modes section.
- Retained messages for last-known state so the Pi recovers instantly on restart.
- Timestamp at the Pi, not the sensor. Simpler, and avoids NTP dependence on 12
  devices. Keep a sequence counter to detect gaps.

### Alternatives worth knowing

- **ESP-NOW** — connectionless, no AP required, a wake-send-sleep cycle takes
  tens of milliseconds versus 1.5–4 s for WiFi association plus DHCP. That is
  roughly a 100× battery improvement. Needs one ESP32 as a receiver, bridged to
  the Pi over USB serial. The right answer *if* you go battery-powered.
- **Zigbee** — see the buy-vs-build note below.
- **RS-485 / Modbus** — bulletproof, immune to WiFi problems, but means pulling
  a daisy-chain cable around a church hall. Probably not worth it, but if the
  electrician is already opening ceilings for the fan side, ask the question.
- **LoRa** — overkill for a single room; do not bother.

### Buy versus build — a real decision

Twelve DIY ESP32 + SHT31 nodes is twelve enclosures, twelve power supplies,
twelve firmware flashes, and twelve things to maintain. Twelve off-the-shelf
Zigbee temperature sensors (Aqara, SONOFF, and similar, roughly $12–18 each)
plus a Zigbee coordinator dongle and Zigbee2MQTT on the Pi gets you the same
data on the same MQTT bus, with 1–2 year coin-cell life and no firmware work at
all. They land in your system as ordinary MQTT topics, so nothing downstream
changes.

The honest recommendation: **use off-the-shelf Zigbee sensors for the occupied-
zone grid, and build ESP32 nodes only where you need something they cannot do**
— the AC supply-air probes, the CO₂ sensors, and anything needing mains power
and fast sampling. Build the interesting 20 %, buy the boring 80 %. If part of
the point is the learning experience, build them all — just know you are
choosing that.

---

## 5. Power for the sensors

Mains-powered is dramatically better than battery if you can get there. Perimeter
outlets in a church hall are usually plentiful; USB power supplies plus a tidy
cable run solve a permanent maintenance chore. Twelve devices needing battery
changes on an unpredictable schedule is exactly the kind of thing that quietly
kills a volunteer-maintained system.

If battery is unavoidable: ESP32 deep sleep is ~10 µA, but a WiFi wake cycle
draws ~200 mA for a couple of seconds. At 30 s intervals a 3000 mAh 18650 gives
you weeks; at 5 min intervals, months. Pair deep sleep with ESP-NOW rather than
WiFi and the picture improves by roughly two orders of magnitude. Or, again,
just use coin-cell Zigbee sensors and stop thinking about it.

---

## 6. Fan control — the part that needs a licensed electrician

**Read this section before buying any relay boards.**

A church hall is, under most building codes, an *assembly occupancy*. Modifying
fixed mains wiring there is not the same as modifying it in your own house.
Realistically this means:

- The mains-side work should be done by a **licensed electrician**, and may
  require a permit and inspection depending on jurisdiction.
- The church's **insurer** is a stakeholder. DIY mains modification in a public
  building is the kind of thing that voids coverage after an incident. Get the
  building committee's sign-off in writing before anything is installed.
- Those $3 8-channel relay boards from online marketplaces have inadequate
  creepage and clearance for mains, no listing, and no enclosure. They are fine
  for switching 12 V. **Do not put them on 230/120 V in a public building.**

The design that gets past all of this: **use certified, listed smart switches**
(Shelly, SONOFF, Inovelli, Lutron, or whatever carries the right listing for
your region) as the mains-side actuator, and have the Pi command them over
WiFi/MQTT. This moves the mains-side engineering and liability onto a listed
product, and it turns the electrician's job into "install this listed device in
this switch box," which they will readily do. Your work stays on the
low-voltage, logic, and software side.

Additional electrical considerations:

- **Never use a TRIAC light dimmer on a ceiling fan.** AC induction motors on
  phase-cut dimmers hum, overheat, and fail. If you want speed control, use a
  purpose-built fan speed controller, or set each fan's own speed manually and
  control it on/off only.
- **On/off only is a legitimate design.** Set every fan to a comfortable medium
  speed at the pull chain, and let the system do binary control. Much simpler,
  and probably sufficient.
- **Inductive load ratings.** Ceiling fan motors are ~60–80 W (30 W for DC/BLDC)
  with an inrush of a few times running current. Any relay must be rated for
  inductive AC loads, not just resistive amps.
- **Fan direction is usually a manual slide switch on the motor housing** and is
  not remotely controllable. Summer is downdraft. If the hall is ever heated,
  reverse/updraft at low speed destratifies without a draft — but someone has to
  climb up and flip 12 switches seasonally. Note it in the runbook.
- **Fail-safe state.** Decide what happens when the Pi dies or the network
  drops. The controllers should have a heartbeat timeout that reverts to a
  defined default rather than freezing in whatever state they were last
  commanded into.
- **Manual override must survive.** Wire the smart switches so the physical wall
  switch still works locally. People need to be able to fix their own comfort
  without an app, and a system that cannot be overridden will be resented and
  then sabotaged.

---

## 7. Control logic

### Start with the structure, not the algorithm

You do not need PID, machine learning, or a CFD model. You need a well-damped
rule-based controller with the right inputs. What it must have:

- **Deadband / hysteresis.** Turn on above (target + 0.75 °C), off below
  (target − 0.75 °C). The deadband must be comfortably wider than your residual
  inter-sensor error after calibration, or sensors will fight each other.
- **Minimum dwell times.** Once a fan changes state it stays there for at least
  5–10 minutes. This prevents short-cycling, protects the motor, and — just as
  importantly — stops a fan chattering on and off above someone's head during a
  service.
- **Rate limiting.** Change at most one or two fans per control cycle. Changing
  six at once makes the whole room's dynamics shift and you can no longer
  attribute cause.
- **Stale data handling.** If a sensor has not reported in N intervals, mark its
  zone unknown and fall back to a default — do *not* keep acting on its last
  value. See failure modes.

### Inputs, in priority order

1. Zone temperature minus hall reference (mean, or the coolest quartile).
2. Vertical ΔT in that column (ceiling minus occupied zone).
3. Absolute comfort bounds — a hard floor and ceiling regardless of what the
   differentials say.
4. Occupancy / schedule.

### Fan-to-zone influence

A fan affects its own column and, weakly, its neighbours. Write this down as an
explicit influence matrix — start with "each fan owns its nearest sensors,
weight 1.0; adjacent sensors, weight 0.3" and refine it from logged data by
observing what actually changes when you switch each fan individually. Doing
that identification experiment once, in an empty hall, is worth a lot.

### Feed-forward is probably your biggest win

You know the service schedule. You know the hall goes from empty to 200 people
in ten minutes, adding ~14 kW of sensible heat. A purely reactive controller
will always be chasing that step change, arriving 15 minutes late, which is
exactly the complaint you have today.

**Pre-cool and pre-mix on a schedule** — start the ACs and run the fans 30–45
minutes before the service. This is simple, requires no clever logic, and likely
delivers more comfort improvement than the entire reactive loop. Build it first.

### Do not forget the ACs themselves

The fans are a downstream fix for an upstream problem. The bigger lever is the
AC setpoints and behaviour. If the units accept IR control, an IR blaster per
unit (or a Sensibo-class device) lets you offset setpoints per side, which
attacks the actual cause of the imbalance. If any unit has a wired controller
with a Modbus or BMS interface, that is better still. Worth scoping even if you
do not do it in phase one.

### Test the logic offline

Log everything from day one, then build a replay harness: feed recorded sensor
data through the controller and see what it would have done. Any change to the
control policy gets replayed against the archive before it goes live. Otherwise
each tweak costs you a week of real-hall time to evaluate, and you will only
ever get a handful of experiments per season.

---

## 8. Software architecture on the Pi

**The pragmatic stack:**
- Mosquitto (MQTT broker)
- Zigbee2MQTT and/or ESPHome for device onboarding
- Home Assistant for device management, history, dashboards, mobile access, and
  user-facing manual overrides — this is a very large amount of work you do not
  have to do
- **Your control logic as a separate, testable Python service** (or AppDaemon,
  or Node-RED if you prefer flows) that subscribes to sensor topics and
  publishes fan commands. Keep it out of YAML. It is the intellectual core of
  the project and it should be version-controlled, unit-tested, and replayable.
- InfluxDB or TimescaleDB + Grafana if you want serious long-term analysis;
  Home Assistant's built-in recorder is adequate to start.

**Operational details that bite:**
- **Do not run the Pi from an SD card.** Continuous logging wears them out and
  they fail without warning, usually at the worst time. Boot from a USB SSD.
- **UPS or clean shutdown.** Power cuts corrupt filesystems.
- **Hardware watchdog + systemd restart policies.** The system must recover
  unattended, because nobody will be there at 07:00 on Sunday.
- **NTP**, so logs from different subsystems correlate.
- **Remote access via Tailscale or similar**, never port forwarding. You will
  want to debug this from home.
- Data volume is trivial — 12 sensors × 2 channels × 2/min is a few MB a month.
  Keep everything.

---

## 9. Failure modes to design against

| Failure | Consequence if unhandled | Mitigation |
|---|---|---|
| Sensor drops off WiFi | Controller acts on a frozen stale value forever; fan pinned on | MQTT LWT + staleness timeout → zone marked unknown, fall back to default |
| Sensor reads high due to sun or self-heating | Permanent phantom hot pocket | Calibration, siting, plausibility check against neighbours |
| Pi crashes or loses power | Fans frozen in last state | Heartbeat timeout in controllers → revert to safe default |
| Church WiFi password changes | Entire system dead | Own dedicated AP |
| SD card wears out | Silent total loss of Pi | Boot from SSD, back up config to this repo |
| Someone turns a fan off at the wall | Controller and reality disagree | Read actual switch state back; treat manual action as an override with a timeout |
| AC unit fails | System runs all fans forever, fixes nothing | Supply-air probes → detect and alert, don't paper over it |
| Control loop oscillates | Fans cycling above people's heads mid-service | Deadband + minimum dwell + rate limiting |
| Nobody knows how to fix it when you're away | System gets bypassed and abandoned | Runbook, labels, AUTO/MANUAL switch, second trained person |

---

## 10. Human and organisational factors

For a volunteer-maintained system in a shared building, these matter as much as
the engineering.

- **Bus factor.** The most common way projects like this die is that the one
  person who understands it becomes unavailable. Write a runbook. Label every
  device physically. Tape a one-page "how to put everything back to manual"
  card inside the electrical cupboard. Train a second person.
- **A physical AUTO / MANUAL switch**, on the wall, that anyone can operate
  without an app or a password. Non-negotiable.
- **A feedback channel — the cheapest high-value thing in this whole project.**
  Two buttons on the wall labelled "TOO WARM" and "TOO COLD", logged with a
  timestamp against the sensor data. Comfort is subjective and you cannot
  measure it directly; those button presses are the only ground-truth labels
  you will ever get for what the setpoint should actually be in this hall, with
  these people, in these clothes. A QR code to a form works too, but buttons get
  used and forms do not.
- **Appearance and noise.** Visible boxes and cables in a worship space will
  draw comment. Agree the aesthetics with whoever cares, in advance.
- **Get formal sign-off** from the building committee before mains work starts.

---

## 11. Rough budget

Order-of-magnitude only; excludes your time.

| Item | Estimate |
|---|---|
| 12 Zigbee occupied-zone sensors + coordinator | $180–250 |
| 4–6 ESP32 nodes (supply-air probes, CO₂, ceiling) | $100–180 |
| Raspberry Pi 5 + USB SSD + PSU + case | $150–200 |
| 12 listed smart fan switches | $300–500 |
| Dedicated AP, enclosures, cable, misc | $150–250 |
| Licensed electrician | $500–2,000+ |
| **Total** | **~$1,400–3,400** |

The electrician line dominates and has the widest variance — it depends almost
entirely on what the Phase 0 wiring survey finds. If the fans turn out to be
ganged onto four switches and you want twelve-zone control, add rewiring costs
that could exceed everything else combined. Find this out first.

---

## 12. Suggested phasing

**Phase 0 — Survey.** Section 2. No hardware purchase beyond a handheld probe.
Ends with: a scale drawing, the fan wiring map, and manual readings from a real
occupied service. *Decision point: is the pattern static enough that a fixed fan
configuration solves it?*

**Phase 1 — Sense and log only.** Deploy sensors, MQTT, Pi, dashboard. No
control whatsoever. Run for 4–6 weeks including several services. Add the
TOO WARM / TOO COLD buttons now, in this phase — the labels are only useful if
they span the whole dataset. Ends with: real numbers for gradients, time
constants, and stratification.

**Phase 2 — Advisory mode.** The system computes what it *would* do and displays
it ("turn on fans 3, 4, 7") on a screen or a phone. Humans act. This validates
the entire control policy at zero risk, with no mains work done yet, and it
builds trust with the people who currently manage the fans manually. Do not skip
this. It is where you will discover the actuator/sensor coupling from §1(b) in
practice.

**Phase 3 — Scheduled feed-forward.** Automate pre-cool and pre-mix on the
service schedule. Highest benefit-to-risk ratio of anything here.

**Phase 4 — Closed-loop control of a subset.** One or two zones, full manual
override, extensive logging. Prove stability over several weeks.

**Phase 5 — Full automation**, plus optional AC setpoint control.

---

## 13. Open questions to resolve

1. Ceiling height, and is the roof flat or vaulted? Determines how much
   stratification you actually have and whether destratification is the main win.
2. How are the 12 fans switched today — individually, or in banks? *Gates the
   entire control granularity.*
3. Do the switch boxes have neutrals?
4. Make/model of both AC types, and do they accept IR or wired/BMS control?
5. Is the hall used for anything other than services, and on what schedule?
6. Typical and peak occupancy.
7. Significant solar gain — large windows, and which orientation?
8. Is the hall ever heated in winter, or is it cooling-only?
9. Who besides you can maintain this?
10. What is the actual budget and who approves it?
