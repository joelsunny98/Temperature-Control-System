# Temperature-Control-System

Automated thermal balancing for a church hall (~2,400–2,500 sq ft): an array of
wireless temperature sensors feeds a Raspberry Pi controller, which drives the
hall's 12 ceiling fans to even out the hot and cold pockets created by four
unevenly-matched air conditioners.

## Status

Design phase. No hardware committed yet.

## Documentation

- [Project Context](docs/project-context.md) — builder constraints, working
  style, and the phase plan. Read this first.
- [Design Considerations](docs/design-considerations.md) — part 1: sensing, calibration,
  networking, mains-side safety, control logic, failure modes, budget, phasing.
- [Sensor Node Build Spec](docs/sensor-node-spec.md) — part 2: board shortlist,
  bill of materials, ESPHome firmware, and the five-test bring-up plan.
