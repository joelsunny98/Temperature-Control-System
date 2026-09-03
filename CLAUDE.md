# Temperature Control System — orientation for a new session

Read `docs/project-context.md` in full before doing anything else in this
repo. It holds the builder's working constraints (cost-sensitive, solder-
capable, India-based sourcing, fail-fast pace) and the phase plan — the
things that make an otherwise-reasonable suggestion wrong for this project.

Chat history is not the source of truth here. **The `docs/` folder is.**
Every real decision — what to build, why, and what was rejected and why —
is committed there as it's made, specifically so a session with no memory of
prior conversations can pick this project up cold. If you're starting fresh,
this file and `docs/project-context.md` are enough to get oriented; you do
not need prior chat transcripts.

## Reading order for a new session

1. `docs/project-context.md` — constraints, working style, and a running
   "Decisions" log with dates. Start here always.
2. `docs/design-considerations.md` — the original problem analysis: why the
   hall has hot/cold pockets, sensing/networking/control theory, general
   principles. Mostly stable background, not the current build state.
3. `docs/hall-survey.md` — the actual hall: layout, fan switchboard wiring,
   AC specs, mounting heights. Ground truth about the physical space.
4. `docs/sensor-node-spec.md` — sensor node design reasoning: board/sensor
   choice, why particular parts were picked or rejected, test plan.
5. `docs/chair-node-build-guide.md` — the current procedural build reference:
   BOM, wiring, firmware, assembly steps for one node. This is what to
   follow when actually building something.

When these disagree, the more specific/recent doc wins — e.g.
`chair-node-build-guide.md`'s wiring beats an older diagram in
`sensor-node-spec.md` if they ever conflict. `project-context.md`'s
Decisions log has dates for exactly this reason.

## Current state (update this section as the project moves)

- **Phase:** Planning is essentially done; Phase 2 (sensor build-up) in
  progress — building node #1.
- **Node #1 design is settled:** ESP32-C3 SuperMini + SHT31, chair-mounted,
  battery-powered (LiPo + charge/boost module + manual SPST switch — switch
  gates the MCU only, charging stays independent). No DS18B20, no
  stratification sensing — see `project-context.md` Decisions for why.
- **Not yet done:** node #1 hasn't been physically built/tested yet. AC
  brand for the ceiling-mounted units is owner-confirmed (Mitsubishi) but
  exact model numbers are unconfirmed and not currently blocking anything.
  Fan-to-switch mapping is confirmed (`hall-survey.md` §3).
- **GitHub:** work happens on `claude/church-hall-thermal-system-mhr5b5`,
  push access requires the Claude GitHub App installed on this repo (already
  done once — if a session ever reports push 403s again, that's the thing to
  check first, not a re-litigation of whether GitHub is reachable).

Keep this section current — a stale "current state" is worse than none,
since it actively misleads the next session instead of just being absent.
