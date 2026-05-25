# Mobile CBRN App — Requirements (Modules 2 & 3: CDM Parse + Chemical Plotting)

**Product:** **Mobile CBRN App** (single-page, offline, iPhone-primary; platform shell + modules — see the Module 1 requirements doc for the shell spec, §0.4 there).
**Document status:** Requirements / specification for a build. NOT a build.
**Consumer:** A future Claude Code session (and a developer new to the project).
**Source authority:** FM 3-11.3 / MCWP 3-37.2A / NTTP 3-11.25 / AFTTP(I) 3-2.56, 2 Feb 2006 ("the manual"). Public release; distribution unlimited. All geometry below is transcribed from Appendix E.
**This document covers two modules:**
- **Module 2 — Chemical Downwind Message (CDM) Parse**
- **Module 3 — Chemical Simplified Hazard Plotting** (ALL of Table E-1: Types A, B, C; Cases 1–6)

> Reuses the platform shell, training-only banner, set/field engine, and validation discipline established for Module 1. Same accuracy doctrine: faithful to the manual, proven against the manual's worked examples; anything not pinned down from the manual is tagged `UNVERIFIED`, never guessed; fail loudly over emitting plausible-but-wrong output.

---

## 0. Use Restriction (unchanged, hard requirement)
**TRAINING USE ONLY.** Not an accredited hazard-prediction system; output must never drive a real protective action, warning, or operational decision. The non-dismissible `TRAINING USE ONLY` banner appears on every screen and is burned into every exported plot image/PDF (not just metadata). Carries forward from Module 1 §0.2.

---

# MODULE 2 — Chemical Downwind Message (CDM) Parse

## M2.1 Scope
- **Parse** a Chemical Downwind Message into labeled, validated, human-readable fields. (Construct is a natural later add but is NOT required by this document; design the parser so a constructor can reuse its field definitions.)
- The CDM supplies the wind/stability data that drives Module 3 plotting: **downwind direction, downwind speed, and air stability category** (per the manual's Step-1/2/3 plotting procedure, which reads exactly those three items from the CDM).

## M2.2 What the CDM contains (verify exact format against the manual during build)
The plotting procedure (manual §E "Simplified Hazard Prediction," Steps 1–3) explicitly draws from the CDM:
1. **Wind speed** (Step 1) — selects circular vs. directional (the 10 kph threshold).
2. **Wind direction** (Step 2) — the grid downwind bearing.
3. **Air stability code** (Step 3) — adjusted via **Table D-14** before the DHD lookup in Table E-2 (Type A only).

The CDM is a multi-line downwind forecast (the chemical analog of the nuclear EDM), typically structured by line sets and/or time blocks. **Open item M2-OI-1:** the exact CDM set/line format must be transcribed from the manual (Appendix D message formats / the CDM example) during the build. Until confirmed, mark unconfirmed fields `UNVERIFIED`. Do NOT invent field names or ordering.

## M2.3 Functional requirements (Module 2)
- R-M2.1: Paste a CDM string → tokenize (reuse Module 1's tokenizer: set name, `//` terminator, `/` separators, `-` empty-field) → display labeled fields.
- R-M2.2: Extract and clearly surface the three plotting-critical values: **downwind direction (grid azimuth), downwind speed (with units), air stability code.** These are the structured outputs Module 3 consumes.
- R-M2.3: Validate field data-types and enumerations against the manual; flag unknowns as `UNVERIFIED` rather than accepting silently.
- R-M2.4: A parsed CDM produces a structured object (documented schema) that Module 3 reads directly. The two modules share this data contract.
- R-M2.5: Reuse shared services: training banner, in-app reference, scenario save/load (JSON), clipboard/export, debug log.

## M2.4 Validation (Module 2)
- Round-trip / fidelity test against the manual's CDM example(s) once the format is transcribed (M2-OI-1).
- Unit tests: the three plotting-critical values are correctly extracted from a known CDM fixture.

---

# MODULE 3 — Chemical Simplified Hazard Plotting (all of Table E-1)

## M3.1 Scope & inputs
- **Inputs:** a **parsed NBC3 CHEM** (from Module 1) + a **parsed CDM** (from Module 2).
  - From NBC3: attack/event location(s) (FOXTROT — may be multiple grids for line attacks), agent-container type (GOLF) and agent persistency (INDIA) for type/case determination, any stated PAPAA dimensions/durations, and GENTEXT type/case statement (used as a cross-check — see M3.4).
  - From CDM: downwind direction (grid), downwind speed, air stability code.
- **Output:** a labeled hazard plot showing **type, case, attack area(s), hazard area(s), and all dimensions** (M3.6).
- **Coverage:** ALL of Table E-1 — Type A (Cases 1–2), Type B (Cases 1–6), Type C. Geometry specs in M3.5.

## M3.2 Plot rendering approach — SCHEMATIC FIRST (hard decision, with rationale)
- **R-M3.2.1 (v1):** The plot is rendered as a **schematic, north-up (Grid North up) diagram on a plain scaled grid — NO map basemap.** Attack center at origin; all ranges drawn to a stated scale; all dimensions labeled. This is the doctrinal "template on grid paper" view.
- **Rationale (worst-case honesty):** Overlaying on the user's Google Maps base image requires georeferencing, which is awkward-to-impossible to perform *on an iPhone* (mobile Google Maps has no right-click-copy-coordinates). Building geometry-on-a-map first means debugging the doctrine and the georeferencing simultaneously — the failure mode where a wrong plot can't be localized. The schematic proves the geometry in isolation, is fully iPhone-native and offline, and needs no coordinate anchoring.
- **R-M3.2.2 (deferred):** Map-overlay rendering onto a georeferenced base image is a later slice (ties into the separate plotting/georeferencing requirements doc and its desktop-georeference / phone-plot workflow). The geometry engine (M3.5) MUST be written independent of the renderer so the same computed shapes can later be drawn on a map without change.
- **R-M3.2.3:** Even in schematic mode, "up" = **Grid North**, and the downwind wedge is oriented by the **grid azimuth** from the CDM/YANKEE. The schematic includes a GN arrow. This teaches the same orientation the eventual map plot uses. (Manual draws every directional case relative to a "GN line.")

## M3.3 Type & Case determination engine (Table E-1, transcribed — authoritative)
Determine **Type** then **Case** from report fields + CDM wind speed, per Table E-1:

**Type (manual §E.c):**
- **Type A** — air-contaminating, **nonpersistent** agents. *Assumed by default unless persistent liquid is confirmed.* (Persistency from INDIA field 3: `NP`→A indicator; `P`/`T`→B indicator.)
- **Type B** — ground-contaminating, **persistent** agents.
- **Type C** — attack origin unknown (detection after unobserved attack; input from NBC4 CHEM line QUEBEC, not a normal NBC3). 

**Table E-1 (Types and Cases of Chemical Attacks) — transcribe verbatim into a data file:**

| Agent container | Attack-area radius | Wind speed | Type | Case |
|---|---|---|---|---|
| BML, BOM, RKT, SHL, MNE, UNK, surface-burst MSL | 1 km | ≤10 kph | A | 1 |
| BML, BOM, RKT, SHL, MNE, UNK, surface-burst MSL | 1 km | >10 kph | A | 2 |
| BML, SHL, MNE, surface-burst RKT and MSL | 1 km | ≤10 kph | B | 1 |
| BML, SHL, MNE, surface-burst RKT and MSL | 1 km | >10 kph | B | 2 |
| BOM, UNK, air-burst RKT and MSL | 2 km | ≤10 kph | B | 3 |
| BOM, UNK, air-burst RKT and MSL | 2 km | >10 kph | B | 4 |
| SPR, GEN | 1 km | ≤10 kph | B | 5 |
| SPR, GEN | 1 km | >10 kph | B | 6 |
| Detection after unobserved attack (NBC4 CHEM) | 10 km | N/A | C | — |

> Container codes: BML=Bomblets, BOM=Bomb, MNE=Mine, MSL=Missile, RKT=Rocket, SHL=Shell, SPR=Spray, GEN=Generator, UNK=Unknown.
> **Rule (manual §E.d.2):** if agent container = UNK, use **RKT** for computations.
> **NOTE on B5/B6 vs the table:** the table lists SPR/GEN at 1 km, but the manual's Case 5/6 figures are the **line/area attacks** ("any dimension of attack area >2 km") with a 1-km circle at each end point. Treat Case 5/6 as the multi-point line geometry (M3.5) regardless; the 1 km is the per-end-point circle radius. Flag this reconciliation as `VERIFY` for the build to confirm against Table E-1 + Figures E-11/E-12.

**Determination algorithm:**
1. If input is an NBC4 CHEM (detection, no known attack) → **Type C**.
2. Else default **Type A**; if INDIA persistency indicates persistent (`P`/`T`) → **Type B**.
3. Look up the container (GOLF, or UNK→RKT) + wind-speed band (CDM) in Table E-1 to get attack-area radius and Case number.
4. For SPR/GEN or multi-grid FOXTROT (line attack) → Case 5 (≤10) or Case 6 (>10).

## M3.4 GENTEXT cross-check / conflict rule (accuracy safeguard)
- R-M3.4.1: The NBC3 GENTEXT often states the type/case (e.g., `TYPE A, CASE 1`). The tool computes type/case **independently** via M3.3 and **compares** to GENTEXT.
- R-M3.4.2: On agreement → proceed, show both ("computed and stated: Type A, Case 1").
- R-M3.4.3: On disagreement → **flag the conflict prominently and do NOT silently pick one.** Show both the computed and stated values and the inputs that drove the computation (container, persistency, wind speed). Let the user resolve. (Per the fail-loud doctrine.)

## M3.5 Geometry specifications (all transcribed verbatim from Appendix E)

> All circles are true ground circles at the stated radius. All directional cases use the **grid** downwind azimuth and a **GN** reference line. "2× attack radius upwind" rule is explicit in each wedge case. Implement the geometry engine to return shapes + the numeric parameters used + the manual paragraph cited (so the UI can show "how this was computed"). Render via the schematic renderer (M3.2).

### Shape family 1 — Concentric circles
- **Type A, Case 1** (Fig E-5, wind ≤10 kph): attack circle r=1 km; hazard circle r=10 km; concentric on attack center. No orientation.
- **Type B, Case 1** (Fig E-7, ≤10 kph): identical geometry to A1 (1 km + 10 km). *Type B differences: stability ignored; add Table E-4 mask-removal time by temperature; PAPAA durations.*
- **Type B, Case 3** (Fig E-9, attack radius >1 km but ≤2 km, <10 kph): attack circle r=2 km; hazard circle r=10 km; concentric.
- **Type C** (Fig E-13): a single circle r=10 km centered on the detection point (line QUEBEC from NBC4); this circle is **both the attack area and the hazard area**. No wind/orientation.

### Shape family 2 — Single directional wedge
General construction (manual, identical steps across A2/B2/B4):
1. Draw GN line from attack center.
2. Attack circle at radius R (R per Table E-1).
3. Downwind axis from center along grid downwind direction.
4. Plot max downwind distance D on the axis; draw perpendicular line at D, extended both sides.
5. Extend axis **upwind by 2R**; from that point draw two tangents to the attack circle, extended to the perpendicular line — forming **30° on each side** of the downwind axis.
6. Hazard area = bounded by attack-circle upwind edge, the two 30° tangents, and the perpendicular at D.

- **Type A, Case 2** (Fig E-6, >10 kph): R=1 km (upwind 2 km). **D = DHD** from the message if given, else from **Table E-2** (agent/means/wind-band/adjusted-stability via Table D-14) or **Table E-3** (Type A Case 2 simplified: shell/bomblet/mine = 10/30/50 km Unstable/Neutral/Stable; air-burst/missile/bomb/rocket/unknown = 15/30/50 km). Air stability **is** considered (Type A only).
- **Type B, Case 2** (Fig E-8, R≤1 km, >10 kph): R=1 km (upwind 2 km). **D = 10 km always** (stability ignored). Add Table E-4 time.
- **Type B, Case 4** (Fig E-10, attack radius >1 km but ≤2 km, >10 kph): R=2 km (**upwind 4 km** = 2R). **D = 10 km always.** Add Table E-4 time.

### Shape family 3 — Multi-point line/area attacks (most complex; build last)
Input: **multiple FOXTROT grids** marking the extremities of the attack area.
- **Type B, Case 5** (Fig E-11, any dimension >2 km, ≤10 kph):
  1. Plot a point at each extreme end; connect to form attack line(s).
  2. Draw a 1-km-radius circle at each end point.
  3. Connect the 1-km circles with tangents parallel to the attack line → **attack area**.
  4. Draw a 10-km-radius circle around each end point.
  5. Connect the 10-km circles with tangents parallel to the attack line → **hazard area**.
  6. Add Table E-4 time.
- **Type B, Case 6** (Fig E-12, any dimension >2 km, >10 kph):
  1. Mark extremities; connect to form attack line(s).
  2. Draw 1-km circles at each end; connect with tangents parallel to the line → **attack area**.
  3. Draw a GN line from each circle center.
  4. Treat **each end circle as a separate attack area** and run the full Shape-family-2 wedge procedure off it (downwind axis, D=10 km, perpendicular, upwind 2 km, 30° tangents).
  5. **Connect the downwind corners** of the two resulting wedges to enclose the combined hazard area.
  6. Add Table E-4 time.

### Cloud arrival time (optional annotation, all directional cases)
- Leading-edge speed = downwind speed × 1.5; earliest arrival = distance / leading-edge speed.
- Trailing-edge speed = downwind speed × 0.5; latest arrival = distance / trailing-edge speed.
- Distance measured from the upwind edge of the attack area (circle center for Case 1).

### Supporting tables to transcribe (data files, cited)
- **Table E-1** — type/case/radius determination (above).
- **Table E-2** — DHD vs. agent/means/wind-band/stability (Type A Case 2). Transcribe all sub-tables.
- **Table E-3** — Type A Case 2 simplified downwind distances (10/30/50; 15/30/50 km).
- **Table D-14** — air-stability-code adjustment feeding Table E-2.
- **Table E-4** — Type B mask-removal time by daily mean surface temperature (<10°C: 3–10 / 2–6 days; 11–20°C: 2–4 / 1–2 days; >20°C: ≤2 / ≤1 day) for attack/hazard areas respectively.

## M3.6 Plot labeling — "all dimensions" (explicit list)
The rendered plot MUST label:
- **Type and Case** (computed; and stated-from-GENTEXT if present, with conflict flag per M3.4).
- **Attack area:** radius (1 km / 2 km / 10 km per case), labeled.
- **Hazard area:** radius (10 km) or, for wedges, the **max downwind hazard distance D** (DHD value, e.g., "10 km" or the Table E-2 value), labeled on the downwind axis.
- **Downwind direction:** the grid azimuth (e.g., "Downwind Direction 120°") and a **GN arrow**.
- **Wedge half-angles:** "30°" on each side (directional cases).
- **Upwind extension:** the 2R distance (2 km or 4 km) where shown.
- **Attack center grid(s):** the MGRS from FOXTROT (and each end point for B5/B6).
- **Scale indicator** and **"Not to scale" suppressed** — the schematic IS to a stated scale (state the scale, e.g., "1 grid square = 1 km").
- **Type B only:** the mask-removal time window from Table E-4 (and PAPAA durations if present).
- **TRAINING USE ONLY** banner.

## M3.7 Functional requirements (Module 3)
- R-M3.7.1: Accept a parsed NBC3 (or NBC4 for Type C) + parsed CDM; if either is missing or incomplete for the determined case, block plotting with a specific error (fail-loud).
- R-M3.7.2: Run the M3.3 determination engine; display computed type/case and the M3.4 cross-check result before/with the plot.
- R-M3.7.3: Compute geometry in metric (km) space via a renderer-independent engine (M3.2.2); render schematic north-up to scale.
- R-M3.7.4: Label all items in M3.6.
- R-M3.7.5: Export the plot as PNG (primary on iPhone) with the banner burned in; PDF optional/secondary. Reuse shared export/share-sheet helpers.
- R-M3.7.6: "Show computation" panel: list the inputs used, the Table E-1 row matched, the DHD source (message vs E-2 vs E-3), and the manual paragraph cited.
- R-M3.7.7: Multi-grid input UI for B5/B6 (enter/confirm each FOXTROT end point).

## M3.8 iPhone/platform constraints (carry forward)
- Single self-contained `index.html`, offline, touch-first, small-screen sequential flow, ~44px targets, no reliance on browser storage (file/clipboard only). Runs as a module in the shell. (Module 1 §5/§6.)
- The plot canvas must be legible and pan/zoom-able on a phone screen; a 10 km hazard circle + labels must fit or scroll cleanly.

## M3.9 Validation & Test Plan (the heart of "accurate")
Golden-example tests, one per Table E-1 case, using the manual's sample NBC3 reports as fixtures (Figures E-5…E-13). Assert computed type/case AND geometry match the manual within tolerance.

| Case | Fixture (manual) | Assert |
|---|---|---|
| A1 | Fig E-5 (user's sample: `33UUB206300`, NERV/NP, 9 kph) | Type A, Case 1; 1 km + 10 km concentric circles |
| A2 | Fig E-6 (`32UPG560750`, 105°, 15 kph, DHD 10 km) | Type A, Case 2; 1 km circle + 30° wedge to D=10 km along grid 105° |
| B1 | Fig E-7 (`33UUB206300`, NERV/**P**, 9 kph) | Type B, Case 1; 1 km + 10 km circles; E-4 time shown |
| B2 | Fig E-8 (`32UNH250010`, NERV/P, AIR, 120°, 15 kph) | Type B, Case 2; 1 km circle + wedge to 10 km; stability ignored |
| B3 | Fig E-9 (`32UNH431562`, MSL, 8 kph) | Type B, Case 3; 2 km + 10 km circles |
| B4 | Fig E-10 (`32UNH320010`, AIR, 110°, 20 kph) | Type B, Case 4; 2 km circle + wedge to 10 km, upwind 4 km |
| B5 | Fig E-11 (two FOXTROT grids, 147°, 9 kph) | Type B, Case 5; 1 km + 10 km circles at each end, parallel tangents |
| B6 | Fig E-12 (two FOXTROT grids, 147°, 12 kph) | Type B, Case 6; per-end wedges + connected downwind corners |
| C  | Fig E-13 (NBC4 QUEBEC location) | Type C; single 10 km circle = attack+hazard |

**Tolerances:** distances ±1% or ±100 m (whichever larger); angles ±1°. Document tolerance + rationale per test.

**Determination-engine tests:** persistency `NP`→A, `P`/`T`→B; container UNK→RKT substitution; wind-band selection at the 10 kph boundary; multi-grid→Case 5/6.

**Conflict tests:** a fixture where computed case ≠ GENTEXT case → assert the conflict is surfaced, not silently resolved.

**Labeling tests:** assert each M3.6 item renders; assert `TRAINING USE ONLY` in UI and exported PNG.

**`UNVERIFIED`/`VERIFY` handling:** the SPR/GEN-vs-line-attack reconciliation (M3.3 note) and any table value not cleanly readable from the manual are surfaced, not silently assumed.

## M3.10 Build order (complexity order — each green before next)
1. Determination engine (Table E-1) + tests.
2. Shape family 1 (concentric): A1, B1, B3, C — + golden tests (Figs E-5, E-7, E-9, E-13).
3. Shape family 2 (single wedge): A2, B2, B4 — + golden tests (Figs E-6, E-8, E-10). Includes Table E-2/E-3/D-14 DHD lookup for A2.
4. Shape family 3 (line attacks): B5, B6 — + golden tests (Figs E-11, E-12). **Highest bug risk; do last.**
5. Labeling, "show computation" panel, PNG export, Table E-4 mask-time display.
6. Module 2 (CDM parse) data contract finalized and wired to Module 3.

---

## Open Items / Verify-During-Build
1. **M2-OI-1:** exact CDM line/set format — transcribe from the manual; `UNVERIFIED` until confirmed.
2. **Table E-1 SPR/GEN vs. line-attack (M3.3 note):** reconcile the 1 km table entry with the Case 5/6 line geometry against Figures E-11/E-12.
3. **DHD source precedence for A2:** confirm when to use message-provided DHD vs. Table E-2 vs. Table E-3 (E-3 is the simplified fallback; E-2 the detailed). Document the rule.
4. **PAPAA field meanings** (durations/dimensions) — finalize from Table III-2 (also a Module 1 open item).
5. **Type C input path:** plotting from an NBC4 (line QUEBEC) is a different input than the NBC3 path — confirm UI handles both.
6. **Map-overlay renderer (deferred):** keep geometry engine renderer-independent so the separate georeferencing/plotting doc can add a map renderer later without touching the geometry.

---

*End of requirements for Modules 2 & 3. Provide the manual PDF in the build session so Table E-1/E-2/E-3/E-4, Table D-14, the CDM format, and the sample reports (Figures E-5…E-13) are transcribed from source — not from memory. Schematic (no-basemap) rendering is the v1 decision; map overlay is deferred to the separate georeferencing/plotting effort with its desktop-georeference / phone-plot workflow.*
