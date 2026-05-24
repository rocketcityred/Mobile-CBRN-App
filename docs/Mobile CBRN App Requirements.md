# Mobile CBRN App — Requirements (Module 1: Report Builder/Parser, v1: CHEM NBC1/2/3)

**Product name:** **Mobile CBRN App** — a single-page, offline-capable, iPhone-primary application that will grow to host multiple CBRN training capabilities ("modules"). This document specifies the **platform shell** plus the **first module**.
**Document status:** Requirements / specification for a build. NOT a build.
**Consumer of this document:** A future Claude Code session (and a developer new to the project).
**Source authority:** FM 3-11.3 / MCWP 3-37.2A / NTTP 3-11.25 / AFTTP(I) 3-2.56, 2 Feb 2006 ("the manual"). Public release; distribution unlimited.
**This document covers:** the Mobile CBRN App platform shell (§0.4) and **Module 1 — Report Builder/Parser** (CHEM NBC1/2/3, construct + parse). Other modules (plotting/mapping, downwind-message handling, etc.) are listed as planned in §0.5 and will be specified in their own documents.

---

## 0. Product, Use Restriction & Module Architecture

### 0.1 What the Mobile CBRN App is
A single, self-contained web app (one `index.html`, no server, runs offline, optimized for iPhone) that serves as a **training toolkit** for CBRN tasks. It is organized as a **platform shell** hosting independent **modules**. v1 ships the shell plus **Module 1 (Report Builder/Parser)**: constructing and deconstructing CBRN reports in the manual's ADatP-3-style set/field line format, scoped to the **CHEM** family, report types **NBC1, NBC2, NBC3**, both directions.

### 0.2 Use restriction (hard requirement — applies to the whole app and every module)
**TRAINING USE ONLY.** This is a proficiency aid; it is not an accredited messaging or hazard-prediction system and its output must not drive operational decisions.
- **R-0.2.1:** A persistent, non-dismissible banner `TRAINING USE ONLY` is visible on every screen of every module and included in any exported/copied output header where feasible. Not toggleable by config.

### 0.3 Accuracy doctrine (the point of the app)
"Accurate" means **faithful to the manual's documented definitions**, proven by reproducing the manual's own examples. For Module 1 that means faithful set/field definitions proven against the manual's sample reports. It does not mean full ADatP-3 standards compliance (the app implements the manual's CBRN subset only; see §6.2).

**Validation contract (non-negotiable, per module):**
- A capability is "done" only when automated tests show fidelity against the manual's reference example(s) for it (see §7).
- Any definition that cannot be pinned down from the manual is tagged `UNVERIFIED` in code and UI, not guessed.
- Prefer failing loudly (clear validation error) over silently emitting plausible-but-wrong output.

### 0.4 Platform shell requirements (shared by all modules)
- **R-0.4.1 — Single artifact:** the entire app (shell + all modules) is one self-contained `index.html` with inlined HTML/CSS/JS and vendored libraries; no build step, no server, no network required (§5, §6).
- **R-0.4.2 — Module navigation:** the shell presents a simple home/menu listing available modules; selecting one enters that module; a clear way back to the menu exists. Layout is touch-first and small-screen-first (§5).
- **R-0.4.3 — Module isolation:** each module is a self-contained unit (its own UI, logic, data, and tests) registered with the shell, so modules can be added or changed without destabilizing others. Keep a documented module interface/registration pattern.
- **R-0.4.4 — Shared services:** the shell provides common services modules reuse — the training-only banner, an in-app help/reference area, scenario file save/load (JSON), clipboard/export helpers, a toggleable debug log, and a shared validation/field-type utility layer where applicable.
- **R-0.4.5 — Naming/branding:** product name "Mobile CBRN App" appears in the shell header and the PWA/Home-Screen manifest (§5.1).

### 0.5 Planned modules (roadmap — NOT in this document's build scope)
Listed so the shell is designed to accommodate them; each will get its own requirements doc:
- **Module 1 — Report Builder/Parser** (THIS DOCUMENT): CHEM NBC1/2/3; later BIO/NUC/ROTA and NBC4/5/6.
- **Module 2 — Downwind Messages** (planned): CDM/EDM/CDR build/parse (feeds plotting).
- **Module 3 — Simplified Hazard Plotting** (planned): the chemical/bio/nuc/ROTA simplified plotting and map overlay work (specified in the separate plotting requirements doc; note the iPhone georeferencing constraints raised there).
- Additional reference/quiz/proficiency modules as desired.

> Build scope for THIS document = shell (§0.4) + Module 1 only. Do not build Modules 2/3 from this document.

---

## 1. Scope (Module 1)

### 1.1 In scope (Module 1, v1)
- Report types: **NBC1 CHEM**, **NBC2 CHEM**, **NBC3 CHEM**.
- **Construct:** structured field entry → correctly formatted report string.
- **Parse:** paste a report string → labeled, validated, human-readable fields.
- Round-trip integrity both ways (§3.4).
- Field-level validation against the manual's data-type and enumerated-value rules (§3.3).
- Runs within the platform shell as the first selectable module; single self-contained HTML file, offline, **iPhone-primary** (§5, §6).

### 1.2 Out of scope (Module 1, v1 — later tiers/modules)
- BIO, NUC, ROTA hazard families (later tiers of Module 1).
- Report types NBC4, NBC5, NBC6 (later tiers of Module 1).
- CDM/EDM/CDR downwind messages (Module 2).
- Any plotting, map, or coordinate-projection feature (Module 3 / separate doc).
- Transmission/networking, message routing, classification handling.
- Full ADatP-3 standard validation beyond the manual's stated rules.

### 1.3 Build order within Module 1 v1
1. Platform shell (§0.4) + Module 1 scaffold registered with the shell.
2. Set dictionary + field-type engine + validation (the shared core).
3. **NBC1 CHEM** construct + parse + round-trip test (the manual's Figure E-1 sample).
4. **NBC2 CHEM** (Figure E-2 sample).
5. **NBC3 CHEM** (Figure E-3 sample + the user's own provided sample).
Do not move to the next report type until the current one's round-trip test is green.

---

## 2. Report Format Model (from the manual)

### 2.1 Line/record syntax
A report is a sequence of **set lines**. Each set line has the form:

```
SETNAME/field1/field2/.../fieldN//
```

- Sets are delimited; each set line **ends with `//`**.
- Fields within a set are separated by `/`.
- A field with no value is represented by `-` (hyphen) as a placeholder (seen throughout the manual's samples, e.g. `PAPAA/01KM/-/10KM/-//`).
- Some sets allow **repeatable fields** (e.g., FOXTROT fields repeat up to 6 times to define a line/area; INDIA field 4 repeats up to 2 times). The model must support repeated occurrences within a set.
- The full report is the ordered concatenation of its set lines.

> Implementation note: do NOT hardcode parsing as "split on /". Build a small, explicit tokenizer that understands set names, the `//` set terminator, the `/` field separator, the `-` empty-field placeholder, and per-set repeatable fields. Keep this in one well-documented module.

### 2.2 The line-item dictionary (Table III-1, transcribed)
The set/line catalog used across CBRN reports (v1 uses the subset relevant to CHEM NBC1/2/3, but transcribe the whole table for forward use; mark which are used in v1):

| Set | Meaning |
|-----|---------|
| ALFA | Strike serial number |
| BRAVO | Location of the observer and direction of the attack or event |
| CHARLIE | DTG of the report or observation and end of the event |
| DELTA | DTG of attack or detonation and attack end |
| FOXTROT | Location of the attack or event |
| GOLF | Delivery and quantity information |
| HOTEL | Type of nuclear burst |
| INDIA | Release information on CB agent attacks or ROTA events |
| JULIET | Flash-to-bang time (seconds) |
| KILO | Crater description |
| LIMA | Nuclear burst angular cloud width at H+5 min |
| MIKE | Stabilized cloud measurement at H+10 min |
| MIKER | Description and status of ROTA event |
| NOVEMBER | Estimated nuclear yield (kt) |
| OSCAR | Reference DTG for estimated contour lines |
| PAPAA | Predicted attack or release and hazard area |
| PAPAB | Detailed fallout hazard prediction parameters |
| PAPAC | Radar-determined external contour of radioactive cloud |
| PAPAD | Radar-determined downwind direction of radioactive cloud |
| PAPAX | Hazard area location for the weather period |
| QUEBEC | Location/type of reading, sample, detection |
| ROMEO | Level of contamination, dose-rate trend, decay-rate trend |
| SIERRA | DTG of reading or initial detection |
| TANGO | Terrain/topography and vegetation description |
| WHISKEY | Sensor information |
| XRAYA | Actual contour information |
| XRAYB | Predicted contour information |
| YANKEE | Downwind direction and downwind speed |
| ZULU | Actual weather conditions |
| GENTEXT | General text |

> NOTE: The manual also uses **set occurrence by NBC-CDR weather period** (WHISKEY/XRAY/YANKEE as first/second/third 2-hour periods) in some contexts. For CHEM NBC1/2/3, YANKEE = downwind direction & speed as shown in the samples. Keep the dictionary general but implement only what the CHEM samples require in v1.

### 2.3 Field data-type notation (from the manual, §III "5g/5h")
The manual specifies field types with a count + class code. Implement validators for these:
- `N` = numeric; `A` = alphabetic; `X` = alphanumeric/any; `AN` = alphanumeric.
- A leading count or range gives length, e.g. `2A` (exactly 2 alpha), `1-3N` (1–3 digits), `1-10X`, `13AN`, `6-7AN`.
- Directional/angular values are stated in **degrees (3N)** or **mils (4N)**: 40° → `040`, 18 mils → `0018`.
- Mandatory/Optional/Conditional flags: **M** = mandatory, **O** = operationally determined (optional), **C** = conditional.

---

## 3. Set Definitions for v1 (CHEM NBC1/2/3) — transcribed from Table III-2

> These are the authoritative build targets. Field order, M/O/C flags, types, and enumerations are from the manual. The build session must verify each against the manual PDF provided alongside this doc; flag any discrepancy as `UNVERIFIED`.

### 3.1 Sets used by CHEM reports

**ALFA — Strike Serial Number** (`ALFA/n/o/s/t//`)
1. (M) Nationality `2A` **or** Area control center code `2-3AN`
2. (M) Code for originator `1-6X`
3. (M) Sequence number `1-10X`
4. (M) Type of incident `1-2A` — enumeration: `N`=Nuclear attack, `B`=Biological attack, `C`=Chemical attack, `RN`=Nuclear ROTA, `RB`=Bio ROTA, `RC`=Chem ROTA, `RU`=Unidentified ROTA. *(For CHEM reports this is normally `C`.)*
5. (O) Grading of message/report `1-3N`
- Example: `ALFA/US/A234/001/C//`

**BRAVO — Location of Observer & Direction of Attack/Event** (`BRAVO/loc/dir//`)
1. (M) Location of observer — one of: Geographic place name `1-30X`; LAT/LONG seconds `15AN`; UTM 10-m `13AN`; LAT/LONG minutes `11AN`; UTM 100-m `11AN`.
2. (M) Direction of attack/event from observer + unit of measurement `6-7AN` (degrees or mils per §2.3).
- Example: `BRAVO/32UNB062634/2500MLG//`

**CHARLIE — DTG of Report/Observation and End of Event** (`CHARLIE/dtg/dtg//`)
1. (M) DTG of report/observation, Zulu, month, year `14AN`.
2. (O) DTG event ended, Zulu, month, year `14AN`.

**DELTA — DTG of Attack/Detonation and Attack End** (`DELTA/dtg/dtg//`)
1. (M) DTG of attack/detonation, Zulu, month, year `14AN`.
2. (O) DTG attack ended `14AN`.
- Example: `DELTA/201405ZSEP2005//`

**FOXTROT — Location of Attack/Event** (`FOXTROT/loc/qual//`, fields 1–2 repeatable up to 6)
1. (M) Attack/event location — one of: place name `1-30X`; LAT/LONG sec `15AN`; UTM 10-m `13AN`; LAT/LONG min `11AN`; UTM 100-m `11AN`.
2. (M) Location qualifier `2A` — `AA`=Actual Area, `EA`=Estimated Area. *(Manual samples also show `EE`; treat the qualifier as a 2A enumeration and validate against the manual's allowed set — flag `EE` for verification.)*
- Repeatable: up to 6 (loc, qual) pairs to define a line/area attack.
- Example: `FOXTROT/32UNB058640/EE//`

**GOLF — Delivery & Quantity Information** (`GOLF/sus/del/num/cont/qty//`)
1. (M) Suspected/Observed `3A` — `SUS`=Suspected, `OBS`=Observed.
2. (M) Type of delivery `3A` — `AIR`, `BOM`, `CAN`, `MLR`, `MSL`, `MOR`, `PLT`, `RLD`, `SHP`, `TPT`, `UNK`.
3. (M) Number of delivery systems `1-3N`.
4. (M) Type of agent container `3A` — `BML`,`BOM`,`BTL`,`BUK`,`CON`,`DRM`,`GEN`,`MSL`,`RCT`,`RKT`,`SHL`,`SPR`,`STK`,`TNK`,`TOR`,`MNE`,`UNK`,`WST`.
5. (M) Number of agent containers `1-3N` **or** size of release `3A` — `SML` (<200 L/kg), `LRG`, `XLG` (>1,500 L/kg), `UNK`.
- Example: `GOLF/OBS/AIR/1/BML/-//`

**INDIA — Release Information CB Agent** (`INDIA/height/agent/persist/detect//`, field 4 repeatable up to 2)
1. (M) Type of agent-release-height `3-4A` — `AIR`, `SUBS`, `SURF`, `UNK`.
2. (O) Type of agent (Table III-3) `1-4A` **or** agent name (Table III-4) `1-4A` **or** UN/NA number (ERG) `4N`. *(e.g., `NERV`=nerve.)*
3. (O) Type of persistency `1-3A` — `P`=Persistent, `NP`=Nonpersistent, `T`=Thickened, `UNK`.
4. (O, repeatable ×2) Type of detection `3-5A` — `OTH` (use GENTEXT), `MPDS`, `UMPDS`, `MSDS`, `UMSDS`, `MSVY`, `UMSVY`.
- Examples: `INDIA/AIR/NERV/P/ACD//` (manual sample; `ACD` appears in field 4 — flag for verification against the detection enumeration) and `INDIA/SURF/NERV/NP//` (user's sample).

**PAPAA — Predicted Attack/Release and Hazard Area** (`PAPAA/.../.../.../...//`)
- Per the CHEM NBC3 sample, encodes attack-area and hazard-area extents and durations, e.g. `PAPAA/1KM/3-10DAY/10KM/2-6DAY//` and `PAPAA/01KM/-/10KM/-//`.
- Transcribe exact field definitions from Table III-2 (attack area radius / duration / hazard area radius / duration). **Flag any field whose meaning is not unambiguous in the manual as `UNVERIFIED`.**

**PAPAX — Hazard Area Location for the Weather Period** (`PAPAX/dtg/loc.../...//`)
- Per NBC3 sample, carries a DTG and may carry hazard-area location grids for the weather period (the manual's NBC3 NUC uses multi-grid PAPAX; for CHEM follow the CHEM sample). Examples: `PAPAX/271600ZAPR1999/-//` and `PAPAX/201600ZSEP2005/...//`.

**TANGO — Terrain/Topography & Vegetation** (`TANGO/terrain/veg//`)
- Example: `TANGO/FLAT/URBAN//`. Transcribe allowed values from the manual; flag unknowns.

**YANKEE — Downwind Direction & Speed** (`YANKEE/dir/speed//`)
1. (M) Downwind direction — degrees `3N` (e.g., `105DGT`) or mils.
2. (M) Downwind speed — e.g., `009KPH`.
- Examples: `YANKEE/270DGT/015KPH//`, `YANKEE/105DGT/009KPH//`.

**ZULU — Actual Weather Conditions** (`ZULU/.../.../.../.../...//`)
- Per samples, e.g. `ZULU/4/10C/7/5/1//` and `ZULU/4/18C/9/-/2//`. Transcribe exact field meanings (air stability category, temperature, etc.) from Table III-2; flag any unverified field.

**GENTEXT — General Text** (`GENTEXT/tag/freetext//`)
- e.g., `GENTEXT/CBRNINFO/TYPE A, CASE 1//`. Free text; tokenizer must not choke on spaces/commas inside the text field.

### 3.2 Which sets per CHEM report type (M/O from manual samples)
> Source: Figures E-1 (NBC1), E-2 (NBC2), E-3 (NBC3). The build must confirm the required-set list against Table III-2's per-report requirements, not only the samples.

- **NBC1 CHEM** (observer's report): BRAVO (M), DELTA (M), FOXTROT (O), GOLF (M), INDIA (M), TANGO (M), YANKEE (O), ZULU (O). *(Sample Figure E-1.)*
- **NBC2 CHEM** (evaluated data): ALFA (M), DELTA (M), FOXTROT (M), GOLF (M), INDIA (M), TANGO (M), YANKEE (O), ZULU (O). *(Sample Figure E-2.)*
- **NBC3 CHEM** (predicted hazard warning): ALFA (M), DELTA (M), FOXTROT (M), GOLF (O), INDIA (M), PAPAA (M), PAPAX (M), YANKEE (O), ZULU (O), GENTEXT (O). *(Sample Figure E-3.)*

> The exact required-set membership is the kind of thing that's easy to get subtly wrong. The build session MUST cross-check this table against Table III-2 in the manual and treat the M/O flags as authoritative from there; the samples are the round-trip fixtures.

### 3.3 Field validation rules
- Validate each field against its data-type (§2.3): length/range and class (N/A/X/AN).
- Validate enumerated fields against their allowed-value lists (transcribed above).
- Enforce M/O/C: missing mandatory set/field → validation error; omitted optional → allowed (rendered as absent or `-` per the set's convention).
- DTG fields: validate the `DDHHMMZmmmYYYY`-style 14AN pattern (e.g., `201405ZSEP2005`). Provide a date/time helper in the UI to build these.
- Directional fields: accept degrees (`3N`) or mils (`4N`) and enforce the zero-padding rule (40° → `040`).
- **Unknown/unverified values:** if an entered or parsed value isn't in the manual's enumeration, do not silently accept or "correct" it — surface it as a warning and tag `UNVERIFIED`.

### 3.4 Round-trip requirements
- **R-3.4.1 (construct→parse):** For every set the app can construct, parsing the constructed string must recover the identical field values.
- **R-3.4.2 (parse→construct):** Parsing a manual sample report and reconstructing it must reproduce the original string (modulo documented, lossless normalization of optional whitespace).
- **R-3.4.3:** Normalization rules (if any — e.g., trimming, empty-field representation) must be explicit, documented, and tested; nothing may be silently dropped.

---

## 4. Functional Requirements

### 4.1 Construct mode
- R-4.1.1: User selects report type (NBC1/2/3 CHEM). The form shows that type's sets in order, with M/O/C indicated and inline help (the set meaning + field types + enumerations).
- R-4.1.2: Enumerated fields use pickers (dropdowns) populated from the transcribed allowed-value lists — not free text — to prevent invalid codes.
- R-4.1.3: Repeatable fields (FOXTROT ×6, INDIA detect ×2) support add/remove of occurrences up to the manual's max.
- R-4.1.4: Live preview of the assembled report string updates as fields change; invalid/missing-mandatory state is clearly shown and blocks "finalize."
- R-4.1.5: "Copy report" (to clipboard) and "Download .txt" actions. (See §6 for iOS specifics.)

### 4.2 Parse mode
- R-4.2.1: User pastes a report string; the app tokenizes and displays each set with labeled, decoded fields (code → human-readable, e.g., `C` → "Chemical attack", `NERV` → "nerve agent", `NP` → "nonpersistent").
- R-4.2.2: Validation results shown per field: valid / invalid type / unknown enumeration (`UNVERIFIED`) / missing mandatory.
- R-4.2.3: Parser is tolerant of input quirks (trailing/leading whitespace, mixed case where the manual allows) but reports — does not silently fix — anything ambiguous.
- R-4.2.4: A parsed report can be loaded into Construct mode for editing, then re-emitted (supports the parse→edit→construct workflow).

### 4.3 Decode reference
- R-4.3.1: An accessible, in-app reference of the set dictionary and enumerations (so the trainee learns the codes). This doubles as documentation.

### 4.4 Error handling / logging / validation (per standing engineering prefs)
- R-4.4.1: Validate all input; never emit a report flagged invalid without an explicit user override that is itself recorded in the output as `UNVERIFIED`.
- R-4.4.2: Clear, specific error messages (which set, which field, what rule failed).
- R-4.4.3: A lightweight in-app log/console (toggleable) showing tokenization and validation steps, for debugging and learning.

---

## 5. UI / Platform — iPhone-primary (hard constraints)

- R-5.1: **Single self-contained `index.html`** — HTML + CSS + JS inlined, any libraries vendored inline, **no build step, no server, no network required.** Opens by double-tap/open in Safari; supports "Add to Home Screen" (include a minimal PWA manifest + icon for a clean full-screen shell). The manifest `name`/`short_name` and the shell header display the product name **"Mobile CBRN App"** (§0.4.5). The shell hosts Module 1 and is built to accept future modules (§0.4).
- R-5.2: **Touch-first, small-screen layout.** Sequential/stepwise flow (pick type → fill sets → preview → copy/export), not a desktop multi-pane dashboard. Minimum ~44px touch targets. No hover-only affordances.
- R-5.3: **Text in / text out only** — no images, maps, or files beyond plain-text report strings and optional `.txt`/`.json` scenario files. (This is why the slice is iPhone-clean.)
- R-5.4: **No reliance on browser storage** for correctness. iOS Safari evicts localStorage; do not depend on it. Persistence = explicit file save/load via the share sheet (§6). In-session state may live in memory (JS variables / component state) only.
- R-5.5: Works equally on desktop browsers (the app must not be iPhone-only); iPhone is the priority target for layout/interaction decisions, desktop is a superset that should also be comfortable.
- R-5.6: Training-only banner (§0.2) always visible, responsive to small screens.

---

## 6. Persistence & I/O (iOS-aware)

- R-6.1: **Save scenario** = serialize the current report (type + field values) to a downloadable **JSON file**; on iOS this routes through the share sheet ("Save to Files"). Document the exact JSON schema.
- R-6.2: **Load scenario** = file picker import of that JSON, repopulating Construct mode.
- R-6.3: **Copy to clipboard** for the report string (primary path on iPhone, since file download is fussier there). Provide a clear visual "copied" confirmation.
- R-6.4: **Export .txt** of the report string (secondary; via share sheet on iOS).
- R-6.5: All saved artifacts (JSON, txt) include a header line with the `TRAINING USE ONLY` marking and the source-manual citation.
- R-6.6: Honest caveat to document: iOS Safari file save/import works but is clunkier than desktop; clipboard copy is the most reliable mobile path. Set expectations in the in-app help.

---

## 7. Validation & Test Plan

### 7.1 Golden sample round-trip tests (mandatory)
Use the manual's sample reports as fixtures; assert parse→construct reproduces them and construct→parse recovers fields:
- **NBC1 CHEM** — Figure E-1 sample (`BRAVO/32UNB062634/2500MLG//`, `GOLF/OBS/AIR/1/BML/-//`, `INDIA/AIR/NERV/P/ACD//`, `TANGO/FLAT/URBAN//`, `YANKEE/270DGT/015KPH//`, `ZULU/4/10C/7/5/1//`, …).
- **NBC2 CHEM** — Figure E-2 sample (`ALFA/US/A234/001/B//`, …).
- **NBC3 CHEM** — Figure E-3 sample (`ALFA/US/A234/001/C//`, `PAPAA/1KM/3-10DAY/10KM/2-6DAY//`, `PAPAX/201600ZSEP2005/…//`, `GENTEXT/CBRNINFO/RECALCULATION…//`) **and** the user's provided sample (`FOXTROT/33UUB206300/AA//`, `INDIA/SURF/NERV/NP//`, `PAPAA/01KM/-/10KM/-//`, `YANKEE/105DGT/009KPH//`, `ZULU/4/18C/9/-/2//`, `GENTEXT/CBRNINFO/TYPE A, CASE 1//`).

### 7.2 Field-validation unit tests
- Data-type validators: each of `N/A/X/AN` with length/range, padding rules, degrees/mils.
- Enumeration validators: valid codes accepted; invalid codes flagged `UNVERIFIED` (not auto-corrected).
- M/O/C enforcement: missing mandatory → error; missing optional → OK.
- Repeatable fields: FOXTROT up to 6, INDIA detection up to 2; over-max → error.

### 7.3 Tokenizer tests
- Empty-field `-` handling, `//` set terminators, spaces/commas inside GENTEXT, trailing whitespace, mixed case.

### 7.4 Labeling tests
- `TRAINING USE ONLY` present in UI and in exported/copied headers.

### 7.5 `UNVERIFIED` handling tests
- Any set/field the build couldn't confirm against the manual is surfaced as `UNVERIFIED` in both construct and parse, and a test asserts that surfacing.

---

## 8. Open Items / Verify-During-Build
1. **Per-report required-set membership** (§3.2): confirm against Table III-2's per-report requirements, not just the samples.
2. **PAPAA / PAPAX / ZULU / TANGO exact sub-field definitions** (§3.1): transcribe precisely from Table III-2; several sub-fields are not fully unambiguous from the samples alone — flag `UNVERIFIED` where needed.
3. **FOXTROT qualifier enumeration:** manual text lists `AA`/`EA`; samples show `EE`. Reconcile against the manual and document the allowed set.
4. **INDIA field-4 vs. sample `ACD`:** the detection enumeration listed doesn't obviously include `ACD`; verify what `ACD` is in the sample and reconcile.
5. **Agent code/name tables (Table III-3, III-4):** transcribe the chemical agent codes/names for the INDIA agent field; until then, accept and flag unknown agent codes.
6. **Normalization rules for round-trip** (§3.4.3): define and document (whitespace, empty-field representation) so round-trip is lossless and tested.

## 9. Build-Order Checklist (hand to Claude Code in order)
1. Platform shell: single `index.html` scaffold, product name "Mobile CBRN App", module menu/navigation (§0.4), training banner, shared in-app help/reference area, shared scenario save/load + clipboard/export helpers, toggleable debug log, test harness (runnable in-browser or via a tiny test page), no build step. Register Module 1 with the shell.
2. Tokenizer + field-type validators + enumeration tables (transcribed from the manual PDF provided alongside this doc) + their unit tests (§7.2, §7.3).
3. Set-definition data (the CHEM sets, §3.1) as a documented data structure with manual-page citations; mark `UNVERIFIED` items.
4. **NBC1 CHEM** construct + parse + round-trip test (Figure E-1). Green before proceeding.
5. **NBC2 CHEM** (Figure E-2). Green before proceeding.
6. **NBC3 CHEM** (Figure E-3 + user sample). Green before proceeding.
7. Scenario save/load (JSON), copy/export, iOS share-sheet paths, PWA manifest.
8. In-app decode reference (§4.3) finalized as learning aid + documentation.

---

*End of requirements. This document specifies the **Mobile CBRN App** platform shell (§0.4) plus **Module 1 — Report Builder/Parser** (CHEM NBC1/2/3) only. Provide the manual PDF in the build session so the set dictionary, field types, enumerations, and sample reports are transcribed from source — not from memory. Future modules (downwind messages, simplified hazard plotting, etc.) are listed in §0.5 and will each be specified in their own document; plotting/map functionality is intentionally excluded here.*
