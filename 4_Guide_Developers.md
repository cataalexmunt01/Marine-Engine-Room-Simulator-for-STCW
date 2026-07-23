# Developer Guide — STCW Engine Room Simulator

This guide describes the actual application architecture, verified by code and tests — not aspirational documentation.

## 1\. Architecture — single entry point

The project runs through a single server: server.py. package.json starts server.py directly.

* server.py — hand-rolled async HTTP \+ WebSocket server (Python asyncio, no websockets pip package required). Port: 3000\.

* engine\_core.py — physical core: EngineRoomModel, DieselGenerator, ElectricalBus, PowerPlant.

* scenario\_engine.py — JSON-driven scenario engine: initial\_state application, timed/conditional events, continuous\_trigger.

* exercise\_eval.py — STCW evaluation engine (score, penalties, category report).

* coach\_engine.py — e-Coach: global rules \+ per-scenario rules.

* excel\_handler.py — Excel template generation and parsing for engine profile import/export.

## 2\. File structure

stcw-engine-room-simulator/

├── engine\_core.py         \# Physical core

├── scenario\_engine.py     \# Scenario engine

├── exercise\_eval.py       \# STCW evaluation

├── coach\_engine.py        \# e-Coach rules

├── excel\_handler.py       \# Excel profile import/export

├── server.py              \# HTTP \+ WebSocket server (port 3000\)

├── engines\_config.json    \# Selectable engine/generator profiles

├── sala-masini.html       \# Student UI — main engine panel

├── tablou-electric.html   \# Student UI — electrical panel

├── configurare.html       \# Engine/generator profile selection \+ Excel import

├── pid-schema.html        \# Interactive P\&ID diagram (live gauges)

├── instructor.html        \# Instructor UI

├── common.js              \# Shared JS: WebSocket, i18n (EN/RO)

├── style.css              \# Shared styles

├── Template\_Profiluri\_Motoare\_ERS01.xlsx  \# Excel template for engine profiles

├── scenarios/

│   ├── catalog.json       \# ✅ read by instructor.html's "Quick load" practice picker

│   ├── exercise\_pack.json \# ✅ 35 exercises, read by instructor.html

│   └── scenario\_\*.json    \# Individual scenario files

├── logs/                  \# Session JSONL \+ CSV written by server.py

└── tests/

    ├── test\_faza1\_physics.py

    ├── test\_faza6\_physics.py

    ├── test\_faza7\_replay.py

    ├── test\_exercise\_eval.py

    ├── test\_scenario\_engine.py

    └── test\_scenario\_catalog.py

## 3\. Starting the server

python3 server.py

Then open: http://localhost:3000/sala-masini.html

## 4\. EngineRoomModel data model

All plant state lives on the EngineRoomModel object (plant.engine\_room). Key extension points:

* self.alarms: dict — any new alarms are added here with a default value of False.

* self.forced\_alarms: set — manual override mechanism (see §5).

* self.aux: dict\[str, AuxUnit\] — any new pump/compressor/separator is an AuxUnit(display\_name) entry.

* Simple scalar attributes (float / bool / int / None) for new physical quantities — added as direct attributes on EngineRoomModel, without an "engine\_room" prefix.

state() at the end of the class serialises everything sent to the UI — any new exposed field must be added explicitly there.

## 5\. The \_phys\_alarm / forced\_alarms mechanism

Any alarm calculated from real physics MUST be written via self.\_phys\_alarm(key, value), not directly self.alarms\[key\] \= value:

def \_phys\_alarm(self, key, value):

    if key not in self.forced\_alarms:

        self.alarms\[key\] \= bool(value)

Reason: scenarios can force an alarm (force\_alarm()) for procedural training. Writing directly overwrites any forced alarm on the next tick.

## 6\. How to add a new auxiliary unit

* In EngineRoomModel.\_\_init\_\_, add to self.aux: 'new\_key': AuxUnit('DisplayName').

* If it should be on by default, add the key to the \_duty\_key list.

* In step(), add its effect logic (exponential relaxation or linear integration).

* aux\_start / aux\_stop / aux\_fail / aux\_reset commands in server.py work generically on any key in self.aux — no new dispatch code needed.

* For a visible control in sala-masini.html, add a div.aux-unit with data-key="new\_key" — JS rendering is generic (reads state.aux).

* For RO/EN translation, add the key to the flat translations object in common.js (see §9).

## 7\. How to add a new scenario

Create a JSON file in scenarios/ with this schema:

{

  "id": "unique-scenario-id",

  "name": "Display name",

  "initial\_state": {

    "running": true,

    "throttle": 0.4,

    "mode\_index": 1,

    "clutch\_engaged": true,

    "aux": { "aux\_key": { "state": "ON" } },

    "any\_er\_attribute": 42.0

  },

  "events": \[

    { "id": "event-1", "trigger\_time": 5.0, "actions": \[

      { "type": "aux\_fail", "key": "sw1" },

      { "type": "set\_state", "path": "some\_attribute", "value": 1.5 },

      { "type": "alarm\_raised", "key": "alarm\_key" },

      { "type": "continuous\_trigger", "target": "path",

        "shape": "ramp", "start\_value": 1, "end\_value": 5, "duration": 10 }

    \]}

  \]

}

Action types in \_execute\_action (scenario\_engine.py):

aux\_start, aux\_stop, aux\_fail, aux\_reset, set\_state, start, stop, set\_throttle, set\_mode, clutch\_engage, clutch\_disengage, continuous\_trigger, clear\_triggers, alarm\_raised, alarm\_cleared, set\_misfire, set\_thrust\_bearing\_fault, set\_turbo\_fault, purifier\_fire\_start, purifier\_qcv\_close, purifier\_co2\_release.

* initial\_state accepts any direct attribute on EngineRoomModel (without "engine\_room" prefix) via generic fallback.

* alarm\_raised on a physics-computed alarm remains forced until an explicit alarm\_cleared.

* Validate JSON before adding: python3 \-c "import json; json.load(open('file.json'))"

## 8\. How to add an assessed exercise (STCW)

Add an entry to scenarios/exercise\_pack.json (NOT catalog.json — the latter only feeds the instructor console's non-evaluated "Quick load" picker):

{

  "id": "exercise-id",

  "title": "Display title",

  "objective": "What the student must do",

  "description": "Full context",

  "stcw": "STCW reporting category",

  "path": "scenarios/scenario\_exercise-id.json",

  "criteria": \[

    { "type": "state", "path": "field", "operator": "==", "value": true,

      "points": 60, "label": "Criterion description", "stcw": "category" }

  \]

}

Criteria types supported by exercise\_eval.py: state (final status check, with path supporting aux., alarms., or direct attribute via fallback), sequence (order of actions in log), reaction\_time (response time from a trigger), penalty (deduction, negative points).

## 9\. Translation system (i18n)

common.js stores translations in a flat object with dot-in-the-key-name strings (NOT a nested object):

const translations \= {

  ro: { 'page.sala.alarmsTitle': 'Alarme', ... },

  en: { 'page.sala.alarmsTitle': 'Alarms', ... }

};

Correct lookup: obj\[path\] — direct access. Do NOT use path.split('.').reduce(...) treating the path as nested navigation.

* Default language: English.

* Fallback language: English.

* Current key parity: \~174 keys in both ro and en dictionaries.

* e-Coach messages (coach\_engine.py) and exercise titles/descriptions (exercise\_pack.json) are hardcoded in Romanian and do not go through this i18n system.

## 10\. Excel engine profile import

excel\_handler.py provides two functions:

* generate\_excel\_template() — creates a minimal XLSX file (stdlib only, no openpyxl) as a download template.

* parse\_excel\_engines(fileobj) — parses an uploaded XLSX and returns a list of engine profile dicts compatible with engines\_config.json.

The upload flow in server.py handles the upload\_excel\_b64 WebSocket action: decodes base64, parses, merges into engines\_config.json, and broadcasts excel\_upload\_result back to clients. configurare.html contains the upload UI.

## 11\. Session logging (Phase 7\)

server.py writes two session files to logs/ once per second (every 10 ticks at 10 Hz):

* logs/session\_\<timestamp\>.jsonl — one JSON record per second with full state snapshot.

* logs/session\_latest.jsonl \+ session\_latest.csv — always the most recent session; used by test\_faza7\_replay.py and the "Download CSV" button in instructor.html.

The new\_session action clears these files and resets the tick counter. The reset action also calls reset\_session\_log().

## 12\. Phase 6 — extended physics (implemented)

Phase 6 is fully implemented in engine\_core.py. The following subsystems are active:

* Dual HT/LT freshwater cooling circuits (fw\_ht, fw\_lt auxiliary units) — separate targets and separate relaxation from the base cooling model.

* Fuel viscosity calculation — depends on fuel temperature (preheated by fuel\_heater aux \+ boiler steam); alarm high\_fuel\_viscosity at \>20 cSt.

* Steam boiler — boiler\_water\_level, steam\_pressure, boiler\_feed\_pump aux; alarms for low water level and high pressure.

* Turbogenerator — runs when steam\_pressure ≥ 5 bar, outputs 120 kW (state fields: turbogenerator\_running, turbogenerator\_power\_kw).

* MARPOL Annex VI emissions — CO₂, SOx, NOx computed from fuel rate; scrubber and SCR system aux units reduce SOx/NOx; alarm emissions\_violation.

* Purifier drum RPM — modelled as a function of power supply and aux state.

## 13\. Testing

cd stcw-engine-room-simulator

python3 \-m pytest tests/ \-q

24 tests total; all 24 pass.

## 14\. Code conventions

* Code comments and user messages are in English/Romanian; identifiers in mixed English/Romanian following existing file conventions.

* Relaxation pattern: x \+= (target \- x) \* rate for any "slow" physical quantity — prefer over direct value sets.

* All WebSocket commands are handled in server.py handle\_command() — single dispatch point.

* Validate JS with: node \--check common.js

* Validate Python with: python3 \-m py\_compile engine\_core.py