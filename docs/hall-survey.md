# Hall Survey — First Pass

Based on the hand-drawn layout, hall photos, switchboard photos, and walkthrough
video supplied 2026-09-02. This replaces guesswork in
[Design Considerations §2](design-considerations.md#2-phase-0--the-survey-you-must-do-before-buying-hardware)
with what's now actually known, and narrows what's still open.

## 1. Layout

![Hall layout: 12 fans in a 4×3 grid labeled by their switchboard number, 2 ceiling-mounted ACs on the left wall, 2 floor-mounted tower ACs on the right wall, a pulpit at the front, doors and the fan switchboard at the bottom, with seating turning to face the pulpit](images/hall-layout.svg)

- Rectangular hall, 2,400–2,500 sq ft, entrance at one short end, the pulpit at
  the opposite ("front") end.
- **12 fans in a 4×3 grid** — 4 fans deep (front to back), 3 columns per side
  plus a denser center block. **Fan numbers on the diagram are the
  switchboard's own numbering** (see §3) — there's now one canonical ID per
  fan, not a separate spatial label to translate.
- **Seating is not uniform.** A large block seats consistently under the
  center 6 fans (4, 5, 6, 7, 8, 9). Two corner pockets (1, 10) are also
  consistently seated. The two pockets nearest the doors (3, 12) taper off —
  fewer people sit that close to the entrance. This matters directly for
  sensor priority (§4) and eventually for which fans matter most to control
  tightly.
- **Seating faces the pulpit, not just "forward."** Confirmed by the updated
  sketch: chairs away from the center column turn inward toward the pulpit
  rather than sitting in uniform forward-facing rows — shallowest angle at the
  front corners (fans 1, 10), steepest at the back corners (fans 3, 12), with
  the center column (4–9) facing straight ahead since it's already on-axis.
  Overall the seating reads as a C/horseshoe curving around three sides of the
  room, opening toward the pulpit. **This is a Phase 2 enclosure detail worth
  remembering:** a chair-integrated sensor housing designed for one chair
  orientation won't drop in cleanly on a chair angled 30–50° differently in
  another zone — the print design needs to account for varying chair angle by
  zone, not assume one orientation hall-wide.
- **4 ACs, 2 types, one type per side** — confirms the original problem
  statement. 2 ceiling-mounted units on the left wall, 2 floor-mounted tower
  units on the right wall.
- Fan switchboard and a power point are at the entrance end, left side.

## 2. Ceiling: false ceiling, perforated metal tile

The hall has a suspended/false ceiling in perforated metal tiles, standard
grid system. This matters for Phase 2 in two ways:

- **Ceiling-level sensors (the DS18B20 stratification leg, sensor-node-spec.md
  §3) can potentially clip into the ceiling grid** rather than needing a
  separate bracket — worth checking the tile/grid spec when you're there next.
- Pendant-mount fans hang below the ceiling on a drop rod — sensor housings at
  fan height need to clear the swept blade radius, not just sit near the fan.

## 3. Fan switchboard — the wiring question is resolved, favorably

This was the single highest-risk unknown in the whole project (design-
considerations.md §2: *"twelve fans are very often not twelve independent
circuits"*). The photo answers it:

**Each of the 12 fans has its own individual ON/OFF switch and its own
individual speed-regulator knob, all landing at one switchboard location**
near the entrance. Two panels hold 6 switch+regulator pairs each — laid out as
`knob(reg) — switch — switch — knob(reg)` per plate, each plate covering 2
fans, labeled 1–12 on tape.

This is the best possible outcome:

- **No electrical ganging to work around.** Every fan is independently
  switchable already — nothing described in design-considerations.md's "fans
  wired in banks" risk scenario applies here.
- **Installation work concentrates in one place.** An electrician (or you,
  bench-testing first per sensor-node-spec.md §9) doesn't need to open 12
  separate ceiling boxes — every fan's line is accessible at this one
  switchboard.
- **The existing regulators solve speed control for free.** sensor-node-
  spec.md §9 already recommended on/off-only automation with speed fixed
  manually — that's exactly this hardware. Leave the regulator knobs as they
  are; automate only the ON/OFF switch leg, in parallel with or replacing the
  existing rocker.

The separate panel with 10 plain rocker switches and 2 power sockets is
**confirmed to be the hall lighting circuits**, not fans. That power socket
location is a convenient spot to bench-power a Raspberry Pi or relay
controller later, since it's already at the same wall as every fan's wiring.

### Switchboard number ↔ hall position

Confirmed by manually testing each switch. This is now each fan's one
canonical ID — used on the diagram in §1 and in every doc from here on.

| Fan (switch #) | Position |
|---|---|
| 1 | Right wall, front |
| 2 | Right wall, middle |
| 3 | Right wall, back (near doors — tapering seating) |
| 4 | Center-right column, front |
| 5 | Center-right column, middle |
| 6 | Center-right column, back |
| 7 | Center-left column, front |
| 8 | Center-left column, middle |
| 9 | Center-left column, back |
| 10 | Left wall, front |
| 11 | Left wall, middle |
| 12 | Left wall, back (near doors — tapering seating) |

The numbering runs in clean column order — right wall (1–3), center-right
column (4–6), center-left column (7–9), left wall (10–12), each top-to-bottom
— which is a good sign the switchboard was wired methodically and this mapping
is reliable, not something stitched together ad hoc.

## 4. AC identification — inconclusive from photos, here's what to get

I looked at the AC photos and pulled frames from the walkthrough video. Neither
gives a reliable read on make/model:

- The **floor-mounted tower units** (right wall) are visible closely enough to
  guess at a manufacturer, but a guess is exactly the wrong thing to hand you
  for something you'll use to order an IR blaster or look up a datasheet — a
  misread brand sends you down the wrong path entirely.
- The **ceiling-mounted units** (left wall) are large, elongated, mounted
  tight against the ceiling — consistent with a "ceiling suspended" ductless
  indoor unit rather than a standard high-wall split (which is smaller/more
  square) or a cassette (which sits flush inside the ceiling grid, not hanging
  below it). That's a shape observation, not a brand identification.

**What actually resolves this:** every AC indoor and outdoor unit carries a
**rating label / nameplate** — a sticker, usually on the side or back panel
(sometimes behind a small flap), listing brand, exact model number, capacity
(tons or BTU/hr), refrigerant type, and often a QR code. That's the
authoritative source design-considerations.md §2 originally asked for
("Model numbers, capacity... whether they have a wired controller, IR remote,
or any BMS/Modbus interface").

**Action step:** next visit, photograph the nameplate on one unit of each type
(one ceiling-mounted, one floor-mounted tower) — close enough to read the
model number — plus whichever remote control or wall controller is used for
each. That's a 5-minute job and it's the one piece of data that unblocks
Phase 4 AC-side integration (design-considerations.md §7, "Do not forget the
ACs themselves").

## 5. Updated open questions

Cross-referencing design-considerations.md §13:

| # | Question | Status |
|---|---|---|
| 1 | Ceiling height, flat or vaulted | Flat, false ceiling — height still not measured |
| 2 | How are the 12 fans switched | **Resolved — individually, see §3** |
| 3 | Neutrals in switch boxes | Still open — check when at the switchboard |
| 4 | AC make/model, IR or wired/BMS | In progress — nameplate photo needed, §4 |
| 5 | Other uses / schedule for the hall | Still open |
| 6 | Typical/peak occupancy | Still open — seating layout (§1) confirms it varies sharply by zone, not just by day |
| 7 | Solar gain / window orientation | Windows confirmed present (curtained) on at least one wall — orientation not yet known |
| 8 | Heated in winter? | Still open |
| 9 | Who else can maintain this | Still open |
| 10 | Budget / approval | Superseded — see project-context.md (no fixed budget) |
