# Project Context

Working constraints and phase plan agreed with the project owner. Read this
before proposing anything — it changes what's "sensible" to recommend.

## Builder profile

- **Comfortable soldering.** Don't default to no-solder/plug-together parts
  to save assembly time — solder-based modules (better antennas, more
  connector choices, cheaper) are equally on the table.
- **Has a 3D printer.** Custom enclosures are in scope, not just off-the-shelf
  project boxes. Notably: sensor housings designed to integrate into the
  hall's chairs, rather than sit on walls/columns as separate visible units.
  **Constraint this creates (settled 2026-09):** a chair-mounted enclosure can
  only be a self-contained, single-sensor, single-MCU occupied-zone node — it
  cannot also carry the ceiling/stratification sensor, because the chairs get
  stacked and moved and a cable tethering a chair to a fixed ceiling point
  doesn't survive that. Stratification is measured by a *second, independent*
  node (identical hardware, no DS18B20 needed) mounted separately and
  permanently near the ceiling, never on a chair. See
  `sensor-node-spec.md` §3.
- **Location: India.** Source parts locally where possible (Robu, Robocraze,
  Amazon.in, etc.) rather than assuming US suppliers/prices. Component
  availability and pricing should be re-checked against Indian sourcing, not
  carried over from earlier USD estimates in `sensor-node-spec.md`.
- **Cost-sensitive — optimize for cheap.** *(Corrected 2026-09; earlier
  guidance here said "no fixed budget," which was wrong.)* Default to the
  cheapest part that still does the job, not the highest-spec one. Where a
  cheaper part carries real risk (e.g. an unverified antenna in a large hall),
  don't just downgrade silently — test the cheap option against a known-good
  one first (the RF survey in `sensor-node-spec.md` §8 already does this),
  then commit to whichever passes at the lower price.

## Working style

- **Fail fast, iterate.** Don't over-invest in upfront planning once there's
  enough clarity to start building. Planning phase should converge, not
  perfect itself.
- **Precise, step-by-step communication preferred over long documents in
  chat.** Long reference material still belongs in the repo (docs/), but
  chat replies should stay short and action-oriented unless a fuller writeup
  is asked for.

## Phase plan

1. **Planning (current).** Hall drawings/measurements, AC model specs, fan
   layout and wiring — gathered from the project owner — folded into the
   existing design docs. Goal is clarity to start Phase 2, not a finished
   spec.
2. **Sensor build-up.** Build and test one sensor node, confirm it works,
   then scale up node-by-node while it reports to a laptop. Only once that's
   solid, move the receiving end from laptop to Raspberry Pi.
3. **Heat map + control logic.** Combine multiple live sensors into a heat
   map of the hall. Use it to write the fan decision logic. Initially the
   owner manually toggles fans per the system's output and observes the
   temperature response — validating the logic before any automation.
4. **Relay / automation.** Wiring the Pi's decisions to actual fan control.
   Deferred — details to be worked out when this phase starts, contingent on
   the fan-wiring survey (see `design-considerations.md` §2).

## Tool usage note

`schematik.io` and `blueprint.io` (AI hardware-design tools) are to be used
**once**, later — not now. When the design is settled enough to generate a
schematic/BOM/firmware from a prompt, produce that prompt for the owner to
run through the tool themselves (this session can't access either domain or
drive a logged-in web app directly). Don't reach for this until Phase 2
hardware decisions are locked in.
