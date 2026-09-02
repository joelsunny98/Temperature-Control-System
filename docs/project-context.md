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
  Since stratification sensing is dropped (below), every node is the same
  design — self-contained, single-sensor, single-MCU, chair-mounted — so
  there's only one enclosure to design, not two.
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

## Decisions

- **Stratification sensing dropped in favor of empirical fan testing
  (2026-09).** Rather than instrumenting both ceiling and occupied-zone
  height to infer whether a fan will help (design-considerations.md §1's
  original two-differential plan), every sensor node measures only at
  chair/occupied-zone height — the height that actually matters for comfort.
  A fan's effectiveness in each zone is established directly, once enough
  nodes are up: toggle the fan, watch whether chair-height temperature in
  that zone actually drops. This is simpler to build (one node design, not
  two — see the 3D-printer bullet above) and replaces a theoretical proxy
  with a direct measurement of the outcome that matters. Traded away: if a
  fan doesn't help, chair-height-only data can't say why on its own — no
  fan-needed, poor airflow coverage, and an under-delivering AC all look the
  same (a flat reading). Cheap enough to work around by hand — a handheld
  thermometer at the AC vent settles it if a zone underperforms. `sensor-
  node-spec.md` retains the ceiling/column-node design for reference; it is
  not currently being built.
- **Chair nodes are battery-powered with a manual on/off switch (settled
  2026-09).** Follows from the chair-mount decision above — a chair that
  gets stacked and moved was never going to stay wall-plugged. Usage pattern
  turned out to make this simple: nodes only need to run during services,
  roughly 3–4 hours/week, manually switched on and off by hand. **This
  removes the deep-sleep firmware requirement entirely** — a physical SPST
  switch in series with the battery gives zero draw when off, which does the
  same job as software sleep for this usage pattern, with no firmware
  changes needed. (Superseded the original plan here, which assumed
  continuous unattended operation and would have needed a sleep-sample-wake
  firmware cycle — not needed given how this is actually used.)

  Circuit, confirmed against the actual Blueprint.io wiring output: LiPo
  cell → charge/boost module (stays connected to the battery at all times,
  so USB-C charging works regardless of switch position) → SPST switch →
  board's 5V input. The switch sits *after* the module, gating only the
  MCU — not between the battery and the module, which would have blocked
  charging while switched off. Full build steps, wiring table, firmware, and
  safety notes: `chair-node-build-guide.md`. `sensor-node-spec.md` §5 also
  carries this circuit for the "why."

  Still true: node #1 (bring-up/firmware validation) stays on USB power
  first — this battery circuit is being built as the *next* node, not
  swapped into node #1 mid-test. Operational note still applies: even with
  infrequent recharging, someone needs a habit of checking/charging before
  service, or design-considerations.md's "quietly stops happening in a
  volunteer-run building" risk still applies, just on a longer cycle.

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
3. **Heat map + empirical fan calibration.** Combine multiple live chair-height
   sensors into a heat map of the hall — horizontal spread only, per the
   Decisions section above. For each fan: toggle it on, watch which zones'
   chair-height readings actually move, record that as the fan's real-world
   effective area. This *is* the control logic's calibration data, not a
   separate validation step — write the fan decision rules from what's
   observed here, then confirm them the same way (manual toggle, watch the
   result) before any automation.
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
